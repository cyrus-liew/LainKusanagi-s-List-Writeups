Main steps
1. Fuzz ports 80 and 443 to find login pages
2. Enumerate usernames with error message for port 80
3. Dictionary attack to gain access to both sites with `hydra` or exploit PHP `strcmp()` to bypass login page
4. Fuzz port 443 to find directory
5. Access directory and download image file
6. Extract SSH key with `strings` from image file
7. Locate public exploit for `phpLiteAdmin`
8. Upload PHP reverse shell to database
9. Enumerate port 80 site to find LFI vulnerability in GET parameter in URL
10. LFI to include the PHP reverse shell and gain shell access as `www-data`
11. Pivot using SSH with the user's SSH key (either using localhost or port knocking by checking the network config file)
12. Enumerate `crontab -l` to find a `cron` job
13. Use `pspy` to find out what process is running
14. Locate a public exploit for the process and elevate to root

Learning points
1. Might need to use a bigger wordlist for fuzzing
2. Bypassing PHP login by passing an array to the password field
3. If there's only one field for login (only password), try a dictionary attack
4. Remember to check downloaded files with `strings`
5. Port knocking
6. Using `pspy` to view other users' `cron` jobs

`nmap`
![[Pasted image 20260311182030.png]]

ports 80, 443 open

Port 80 does not redirect to 443 but shows a default page instead - `Whatweb` shows that it is running `Apache 2.4.18`.
![[Pasted image 20260311182107.png]]

Added `nineveh.htb` to `/etc/hosts` as the SSL cert contains the domain name.
Port 443 displays an image.
![[Pasted image 20260311182429.png]]

Starting with port 80:
`ffuf` returned no subdomains.
`Feroxbuster` found a directory at `/department`, which redirects to a login page.
![[Pasted image 20260311182752.png]]

There is an interesting comment in the source code of the login page, which tells us the database being used as well as some potential usernames.
![[Pasted image 20260311182839.png]]

Trying a few user password combinations on the login page led to discovering an information disclosure vulnerability: the error message tells us if a user exists.
![[Pasted image 20260311183223.png]]

Tried `admin` and got this message, confirming that the `admin` user exists.

Doesn't seem to be much else here, so before trying a password attack on this login page, started fuzzing the 443 site first.

`Feroxbuster` found a directory at `/db/`, which redirects to a `phpLiteAdmin v1.9` login page, with a warning message. Tried logging in with the default password `admin` but was unsuccessful.
![[Pasted image 20260311183537.png]]

`ffuf` found no subdomains.

Back to the port 80 site:
Captured the login POST data using `Burpsuite`
![[Pasted image 20260311184339.png]]

Ran `hydra` using `hydra -l admin -P /usr/share/wordlists/rockyou.txt nineveh.htb http-post-form "/department/login.php:username=^USER^&password=^PASS^:Invalid Password" -V` and found the password.
![[Pasted image 20260311184856.png]]

`admin`: ` 1q2w3e4r5t `

Managed to login to the site.
![[Pasted image 20260311184926.png]]

The only information on this site is the `Notes` page, which has a parameter in the URL `?notes=files/ninevehNotes.txt`. The page also says something about a secret folder.
![[Pasted image 20260311185115.png]]

From the `Feroxbuster` results earlier, there was a `/department/files/` directory, so this site is probably querying that directory to include the TXT file. 
Adding a `'` to the end of the parameter in the URL gives this error message, which tells me that the file is being opened using the PHP `include()` method. There is most likely a Local File Inclusion vulnerability on this page.
![[Pasted image 20260311185333.png]]

Tried a few different inputs in the parameter. Most returned either the same error message or a `No note selected.` message. An interesting finding is that `files/../files/ninevehNotes.txt` returns the note.
![[Pasted image 20260311190047.png]]

More testing shows that the warning message will only appear if the parameter include the string `ninevehNotes`.

Tried including the parameter `?notes=files/ninevehNotes/../../../../../../etc/passwd` and was able to include the `/etc/passwd` file. 
![[Pasted image 20260311191303.png]]

Was unable to include any other files as I got the error `File name too long`. Couldn't find the `secret` folder that the notes mentioned either.

Back to trying to fuzz the websites. Using a longer wordlist this time, ` /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt `, I managed to find a directory that was missed on the 443 site initially - `/secure_notes`.
![[Pasted image 20260311193847.png]]

Navigating to this directory shows an image.
![[Pasted image 20260311193908.png]]

I downloaded the image and ran a few tools on it - `exiftool`, `steghide`, and `strings`. The first 2 tools returned nothing interesting, but `strings` outputted an SSH key pair as well as some directories - `secret/ninveh.priv` and `secret/ninveh.pub`.
![[Pasted image 20260311194114.png]]

Tried SSH as `amrois` with this key but was unable to. (port 22 not open)

Back to the `phpLiteAdmin` page - since the first login was susceptible to a dictionary attack, I tried running one here as well. Captured the POST data with `Burpsuite`.
![[Pasted image 20260311194618.png]]

Ran `hydra` with `hydra -P /usr/share/wordlists/rockyou.txt -s 443 nineveh.htb https-post-form "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password" -V -l none` and got the password: `password123`.
![[Pasted image 20260311195227.png]]

Logged in with the password.
![[Pasted image 20260311195253.png]]

Following this https://www.nmmapper.com/st/exploitdetails/24044/27891/phpliteadmin-193-remote-php-code-injection/ article, it is possible to inject PHP into the database using `phpLiteAdmin` by creating a database named `something.php` and inserting PHP code into a table with a text field, in its default value.

Created a table named `shell.php` and inserted PHP reverse shell code. Tried a few payloads but this one worked.
![[Pasted image 20260311200548.png]]
![[Pasted image 20260311200533.png]]

Back to the database info page, the path to the database is shown as `/var/tmp/shell.php`.
![[Pasted image 20260311200651.png]]

To get the code to execute, I need to run this PHP file. This can be done using the LFI that I found earlier on the port 80 site.

Back to the `Notes` page, I tried a few different combinations of path traversal and eventually this one worked: 
`?notes=?notes=files/ninevehNotes/../../../../../tmp/shell.php`
and I was able to get a shell as `www-data`.
![[Pasted image 20260311200837.png]]

Since I already have `amrois`'s private key, I transferred it to the server with `wget` and accessed `amrois` using SSH and got access as `amrois`.
![[Pasted image 20260311201431.png]]

`sudo -l` requires a password which I do not have.
Nothing interesting in the `SUID` binaries.
`crontab -l` shows that `amrois`has a `cron` job that runs every 10 minutes. It runs a script that clears the `/report` directory.
Checking the `/report` directory shows a few files that look like logs files of a program that checks directories and files for infections or suspicious files.

The logs don't show what program is running, but I can use `pspy` to check what processes are running in the background.

Downloaded the `pspy64s` file to the server but got errors, so I used the full sized `pspy64` instead. `chmod +x` then ran it and waited for any output. After a minute or so, I got a bunch of processes being executed - starting with `/usr/sbin/CRON -f `, then  `/bin/bash /root/vulnScan.sh`, then `/bin/sh /usr/bin/chkrootkit`.
![[Pasted image 20260311213830.png]]

The `echo` commands in the process output match the strings in the report files.
![[Pasted image 20260311213923.png]]

From this output, I can tell that there is a root `cron` job being run that calls `/root/vulnScan.sh`, which the runs  `/bin/sh /usr/bin/chkrootkit`. Since the script is in the root folder and cannot be read, I will look at the `chkrootkit` binary instead.

The user `amrois` is not able to run the `chkrootkit` binary.

Googling `chkrootkit` vulnerabilities leads me to this exploit: CVE-2014-0476 https://www.exploit-db.com/exploits/33899, where a local privilege escalation is possible if `chkrootkit` is being run as root, and `/tmp` is not mounted as `noexec`.

Checked if `/tmp` is mounted as `noexec` by running `mount | grep /tmp`:
![[Pasted image 20260311214650.png]]

No output, so `/tmp` should be `exec` by default.

The exploit says to put an executable file in `/tmp` named `update` and run `chkrootkit`. Since `chkrootkit` is being run every minute or so, I created a file with a reverse shell script and put it in `/tmp`.
![[Pasted image 20260311215715.png]]

Waited for a minute and got a reverse shell as root.
![[Pasted image 20260311215738.png]]

**EXTRA STUFF**

There is another way to get SSH access as the user `amrios` in this machine, which is most likely the intended way. Enumerating `ps auxww` shows that there is a process `knockd` being run.
![[Pasted image 20260311220312.png]]

This is a daemon used for port knocking. Port knocking is a security technique used to keep network ports closed by default, only opening them if a specific sequence of connection attempts are being made a set of closed ports. It basically acts like a combination lock.

Enumerating the config file at `/etc/knockd.conf` shows the sequence needed to knock on.
![[Pasted image 20260311220517.png]]

To access the SSH port, the user will need to knock on ports 571, 290, and 911 in order. This page has an article on how to send the knocks with `nmap`. https://wiki.archlinux.org/title/Port_knocking#Client_script Writing this as a one liner:
```
 for i in 571 290 911; do
> nmap -Pn --host-timeout 100 --max-retries 0 -p $i nineveh.htb >/dev/null
> done; ssh -i nineveh amrois@nineveh.htb
```
It loops over the 3 ports with `nmap` and directs the output to `/dev/null` then tries to SSH into the server.

There is also another way to bypass the login page on port 80. By editing the POST data to set the `password` parameter to an array using `username=admin&password=[]=`, it bypasses the login.

This is due to PHP string comparisons, since the site has a hard coded password checking system in place. The PHP `strcmp()` function will fail when an array is passed in as one of the parameters, and will return a NULL value, which is then compared to 0 and evaluated as true.