Main steps
1. Enumerate site to find password file
2. Decode password file
3. Enumerate site to find LFI vulnerability
4. Find username using LFI
5. SSH access as user
6. Unzip ZIP file using user password to get secret file
7. Enumerate network ports with `netstat -an` to find local VNC server
8. Port forward VNC server to own machine
9. Decrypt secret file using VNC decryptor to get VNC password
10. Access VNC with client to get root access

Learning points
1. VNC will block you if you have too many authentication failures
2. Cracking VNC passwords with `vncpwd`
3. Check command parameters properly before dismissing them
4. Log poisoning

`nmap`
![[Pasted image 20260315214834.png]]

Ports 22, 80 open.

Website shows a page used to test local PHP scripts, with a few listed PHP scripts
![[Pasted image 20260315214907.png]]

Able to view PHP version from `phpinfo.php`.
![[Pasted image 20260315215015.png]]

Kernel version from `info.php`
![[Pasted image 20260315215100.png]]

Some interesting information in `listfiles.php`- there may be a file named `pwdbackup.txt`.
![[Pasted image 20260315215128.png]]

Checking this file gives this encoded password.
![[Pasted image 20260315215218.png]]

Since it looks like a `base64` encoded string, I tried running it through CyberChef with 13 `base64` decodes, and got a short string that resembles a password.
![[Pasted image 20260315215348.png]]

` Charix!2#4%6&8(0 `

Clicking on submit without specifying a file name gives this error.
![[Pasted image 20260315215448.png]]

Looking at the error message, it seems like the PHP is taking the hard coded include path and using it to open files specified in the parameter `file` in the GET request. If the input is improperly sanitized, this could mean that LFI is possible.

Tried replacing the parameter in the URL with `../../../../../etc/passwd`, with 5 `../` to match the include path, and was able to read the `/etc/passwd` file.
![[Pasted image 20260315215714.png]]

There is a user named `charix` with shell access. This matches the password found before.

Tried the user password combination and was able to gain SSH access.
![[Pasted image 20260315215926.png]]

There is a `secret.zip` folder in `charix`'s home directory, which requires a password to unzip. Tried the same password but was unable to unzip the file.
![[Pasted image 20260315220155.png]]

There doesn't seem to be `sudo` or `find` installed on the system, and there are no `cron` jobs for `charix`. Tried `ss -ntplu` to check for running services but that also was not installed, so used `netstat -an` instead, and found a few ports listening on `localhost`.
![[Pasted image 20260315221621.png]]

Port 25 is SMTP and not very useful, but ports 5801 and 5901 are both used for VNC (Virtual Network Computing).

Tried to connect to both ports with `nc`, but only port 5901 returned results.
![[Pasted image 20260315222052.png]]

`RFB 003.008` is the VNC protocol version indicating a successful connection handshake between the client and server. This means that I should connect to VNC using port 5901.

Checked `ps auxww` and saw that the VNC service was started by the root user.
![[Pasted image 20260315222243.png]]

Tried connecting to the VNC server but the server does not seem to have VNC clients installed.

Port forwarded the internal port 5901 to my machine. Since I have the SSH password for the user `charix`, I can use SSH to do this with `ssh -L 5901:localhost:5901 charix@10.129.1.254`.

Tried to connect to the VNC server but it prompted me for a password.
![[Pasted image 20260315222530.png]]

Based on the VNC command that started the server, the password is stores at `/root/.vnc/passwd` on the remote server, so I will not be able to read the password.

Tried to brute force the password with `hydra` but was unsuccessful.

Going back to the `secret.zip` file - I copied it out to my machine with `scp` and tried unzipping it again with `charix`'s password. This time, I was able to unzip the file, and got a binary file `secret`.
![[Pasted image 20260315223635.png]]

So apparently looking closer at the `unzip` command in the `FreeBSD` machine, it does not actually have an option to pass a password to unzip the file, which was probably what was causing it to fail.
![[Pasted image 20260315223947.png]]

Back to `secret` - since there is a VNC server running, I guessed that it may be an encrypted VNC password file, so I found a decryptor at `https://github.com/jeroennijhof/vncpwd` and ran it on the file, and it managed to decrypt it.
![[Pasted image 20260315224435.png]]

` VNCP@$$! `

Tried accessing VNC with `vncviewer localhost:5901` and was able to get root access.
![[Pasted image 20260315230547.png]]

![[Pasted image 20260315230606.png]]

**Additional things**
There is another method to gain initial access into the machine by making use of log poisoning, which may have been the intended method going by the name of the machine and the information on HTB. This involves writing PHP code to the apache log file, then using the LFI to include the code into the browser.

The log file path can be obtained from `/etc/apache24/httpd.conf` and the log for this server will be at `/var/log/httpd-access.log`.

By adding a simple PHP reverse shell script into the `User-Agent` string in the request headers using `Burpsuite`, I can then include the log file with LFI, appending `&cmd=COMMAND` to run whatever arbitrary commands I want, like spawning a reverse shell as `www-data`.