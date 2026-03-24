Main steps
1. 

Learning points
1. Check all config files, including all backups

`nmap`
![[Pasted image 20260324184131.png]]

13 ports open here - most for `msrpc`, with 139 and 445 for `NetBios` and `SMB`, as well as an FTP server on 21 and a web server on 80.

The port 80 interface shows the login page for `PRTG Network Monitor`.
![[Pasted image 20260324184329.png]]

Tried logging in with the default `prtgadmin`:`prtgadmin` credentials but was unsuccessful.

Checked the FTP server next and was able to get the user flag directly from the desktop.
![[Pasted image 20260324185226.png]]

Also managed to retrieve the `PRTG` configuration files from the server.
![[Pasted image 20260324185257.png]]

Checked the backup config file `PRTG Configuration.old.bak` and managed to find the password to ` prtgadmin `: ` PrTg@dmin2018 `, but could not login. Tried changing the date in the password to 2019 and was able to login with ` PrTg@dmin2019 `.
![[Pasted image 20260324190455.png]]

Searched up the version number for the application and found an Authenticated RCE exploit CVE-2018-9276. Found a POC for this exploit at https://github.com/wildkindcc/CVE-2018-9276 and ran it, and got a reverse shell as `nt authority\system`, and retrieved the root flag.
![[Pasted image 20260324193226.png]]

