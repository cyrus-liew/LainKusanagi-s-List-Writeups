Main steps
1. Enumerate site to find SQL injection vulnerability
2. `SQLmap` for blind SQL injection to get admin password
3. Login to admin dashboard and find version number
4. Locate public exploit for `Laravel` version for shell access
5. Enumerate user's home directory for credentials to pivot
6. Enumerate `sudo -l` for vulnerable `sudo` binary
7. Abuse wildcard in `7za` for privilege escalation

Learning points
1. Remember to check all input fields for SQL injection
2. Refer to https://hacktricks.wiki/en/linux-hardening/privilege-escalation/wildcards-spare-tricks.html#id-7z for wildcard privilege escalation methods

`nmap`
![[Pasted image 20260323182254.png]]

Ports 22, 80 open

Site shows a login page.
![[Pasted image 20260323182312.png]]

There is also a subdomain at `admin.usage.htb` that leads to an admin login page.
![[Pasted image 20260323182429.png]]

Tried to register for an account and logged in. Nothing much interesting here, just some text.
![[Pasted image 20260323182723.png]]

Tried a few SQL injection payloads against both login pages but did not get any different error messages.

`Feroxbuster` and `ffuf` both returned nothing interesting.

`Whatweb` shows that the website was developed using `Laravel`. Based on some quick Googling, it seems that `Laravel` may commonly have SQL injection vulnerabilities.

The only other input field I have yet to test is the password reset field. Tried sending a single quotation, and the page returned the `Laravel` error page.
![[Pasted image 20260323185653.png]]
![[Pasted image 20260323185700.png]]

This most likely means that this field is susceptible to SQL injection. Tried sending a few different payloads but all of them returned the same error page, leading me to believe that this is blind SQL injection. Since it is blind, I ran `SQLmap` with `sqlmap -r inject --level 5 --risk 3 --threads 10 -p email --batch` using the POST request captured by `Burpsuite` to dump the database.

Ran `SQLmap` a few times but always got gibberish as the database names, so I tried again with fewer threads and found the database `usage_blog`. (Seems like an issue with too many threads running on blind injection?)
![[Pasted image 20260323191056.png]]
![[Pasted image 20260323191130.png]]

Dumped tables with `-D usage_blog --tables`.
![[Pasted image 20260323191717.png]]

Dumped the `admin_users` table with `-T admin_users --dump`.
![[Pasted image 20260323192305.png]]

`admin`: ` $2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2 `

Ran this hash through `hashcat` and cracked it.
![[Pasted image 20260323192414.png]]

`admin`: ` whatever1 `

Was able to login to the admin dashboard using these credentials.
![[Pasted image 20260323192457.png]]

The dependencies section here looks interesting, since there's a bunch of version numbers.

Googling the first `Laravel` dependency here, `encore/laravel-admin 1.8.18`, gives me CVE-2023-24249, an arbitrary code execution due to an unrestricted file upload via the `User Settings` interface, where a malicious user can upload and execute PHP scripts through the `avatar` upload function, by crafting a POST request and changing the file type to `image/jpeg`.
![[Pasted image 20260323193116.png]]

Found a POC here https://github.com/IDUZZEL/CVE-2023-24249-Exploit. The POC just creates a PHP reverse shell script and crafts the POST request, then uploads and executes the script by sending a `curl` request to the uploaded script at `/uploads/images/revshell.php`.

Ran the POC and got a reverse shell as `dash`, and retrieved the user flag.

Checked `/etc/passwd` and found another user, `xander`.
![[Pasted image 20260323193714.png]]

Found an SSH key under `dash`'s home folder and swapped to SSH for a more stable shell.

There's a few hidden files in the home directory for `monit`, which is a Linux utility that monitors processes and files. Based on the `.monitrc` file, there is a web interface bound to localhost 2812, with credentials `admin`:` 3nc0d3d_pa$$w0rd `.
![[Pasted image 20260323194735.png]]

This also matches with the output for `ss -ntplu`.

Forwarded the port with `ssh -i dashkey dash@usage.htb -L 9001:localhost:2812` and tried to access the web interface.

Was presented with a HTTP authentication popup, which the credentials above worked for.
![[Pasted image 20260323194851.png]]

The `Monit` dashboard. Version number shows `5.31.0`.
![[Pasted image 20260323194923.png]]

There doesn't seem to be anything interesting on this page. The `Monit` version here also does not have any useful vulnerabilities.

Tried the password for `Monit` to `su xander` and was able to get access.
![[Pasted image 20260323195328.png]]

Checked `sudo -l` and found that `xander` can run `/usr/bin/usage_management` as root with no password.
![[Pasted image 20260323195409.png]]

This is a 64-bit ELF file. Running `strings` on it gives some interesting information. The program gives the user 3 options: Project backup, Backup MySQL data, or Reset admin password.
![[Pasted image 20260323195616.png]]

There are also a few commands shown here, which are most likely the commands ran for the backups.

Downloaded `pspy` to try to snoop the commands being executed here, then ran the binary, but `pspy` returned nothing.
![[Pasted image 20260323200231.png]]

Looking at the binary's output: options 2 and 3 both do not return anything when selected, but option 1 seems to shows the `7za` output.
![[Pasted image 20260323200405.png]]

Looking at the commands called for option 1:
```
/var/www/html
/usr/bin/7za a /var/backups/project.zip -tzip -snl -mmt -- *
```

The program changed directory to `/var/www/html`, then runs the `7za` (7Zip) binary to create an archive at `/var/backups/project.zip`, sets the archive type with `-tzip`, then sets the switch `-snl` to store `symlinks` as links instead of following them, enables multi-threading with `-mmt`, ends options parsing with the `--`, then ends with an `*` to archive all files within the working directory.

This means that any `symlink` within the directory that `7za` is trying to archive should be preserved, such as a `symlink` to `/etc/shadow`, to read the root password.

Referring to this page on HackTricks, https://hacktricks.wiki/en/linux-hardening/privilege-escalation/wildcards-spare-tricks.html#id-7z, this can be done by prepending the filename for the `symlink` with an `@` symbol, which will bypass the `--` to stop options parsing. This will cause `7za` to read the `symlink` as a file list and error out, while printing the contents to `stderr`.

Tried this by first creating a `symlink` to `/etc/shadow` with `ln -s /etc/shadow root.txt` in `/var/www/html`, then tell `7za` to use this file as a file list with `touch @root.txt`.
![[Pasted image 20260323201738.png]]

This did not seem to work, so I tried to dump the root SSH key instead from `/root/.ssh/id_rsa` and managed to get the key.
![[Pasted image 20260323202712.png]]

Removed the extra text and saved this as an SSH key, and was able to SSH as root.
![[Pasted image 20260323202827.png]]

