Main steps
1. Enumerate open ports to find IRC server being run on high ports
2. Enumerate IRC server version
3. Locate public exploit for server access
4. Enumerate user's home directory for credentials
5. Extract user's credentials from image using steganography tools
6. Gain access as user
7. Enumerate SUID binaries
8. Write script that will be run with SUID binary for privilege escalation

Learning points
1. It's possible to enumerate IRC server versions using the program `irssi`
2. It's possible to read the other user's home directory, and there may be credentials in there
3. Just try stuff even if it seems kinda stupid (steganography)
4. Check SUID binaries properly - GTFObins may not have all binaries, and custom binaries such as the one here will not show up

`nmap`
![[Pasted image 20260309192159.png]]
ports open:
1. 22
2. 80
3. 111: `rpcbind 2-4`
4. 6697: `UnrealIRCd`
5. 8067: `UnrealIRCd`
6. 65534 `UnrealIRCd`

Scan also showed an Admin email: ` djmardov@irked.htb ` from port 65534

`rpcbind` is being run on this server and is also using a few ports for status.
Confirming `rpcbind` by connecting via `nc`, server is running `sunrpc`, but no other interesting information
![[Pasted image 20260309192759.png]]

Searched for `UnrealIRCd` and found that certain versions are vulnerable to a backdoor
Tried running the backdoor against the server using `Metasploit` and was able to gain access as user `ircd`.
![[Pasted image 20260309195837.png]]

`/etc/passwd` shows another user named `djmardov`.
![[Pasted image 20260309195906.png]]

Checking the `unrealircd.conf` file shows some information:
A VHOST with login credentials, but looks like default config info
![[Pasted image 20260309200154.png]]

Checking the other users home directory, found a `.backup` file containing a short message including a password: ` UPupDOWNdownLRlrBAbaSSss `
![[Pasted image 20260309202408.png]]

Tried to `su` as the user using this password but did not work.

Googling `steg backup pw` only points towards steganography.
Downloaded `steghide` to try to extract something from the JPEG that was on the website.
![[Pasted image 20260309202716.png]]

With `steghide --extract -sf irked.jpg -p UPupDOWNdownLRlrBAbaSSss`, was able to extract a string from the image, and using this string, was able to login as user `djmardov`.
![[Pasted image 20260309202723.png]]
![[Pasted image 20260309202751.png]]
![[Pasted image 20260309202843.png]]

`djmardov`: ` Kab6h+m+bbp2J:HG `

Enumerating the SUID binaries on the machine shows a `viewuser` binary which is not a default Linux program. When trying to run this, it shows a message saying that it is trying to run `sh /tmp/listusers` but the file is not found.
![[Pasted image 20260309204424.png]]

Created the file `/tmp/listusers` and added a reverse shell script to it.
![[Pasted image 20260309204505.png]]

Ran the `viewuser` program and was able to get a reverse shell as root.
![[Pasted image 20260309204550.png]]