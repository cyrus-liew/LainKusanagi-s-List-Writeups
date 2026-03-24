Main steps
1. Enumerate ports to find `finger` service
2. Dictionary attack on `finger` service to enumerate SSH usernames
3. Dictionary attack on SSH to login as user
4. Enumerate `/backup` to find backup of `/etc/shadow`
5. Crack password hash and get user access
6. Enumerate `sudo -l` to find permissions
7. Privilege escalate with `sudo` binary

Learning points
1. How to use the `finger` service for enumeration
2. Remember to check `/backup` and `/opt` for files

`nmap`
![[Pasted image 20260318194317.png]]

Ports open:
1. 79 (Finger)
2. 111 (RPCBind)
3. 515 (Printer)
4. 6787 (Apache httpd)
5. 22022 (SSH)

Tried to `nc` to connect to the `finger` service but was denied.
![[Pasted image 20260318195932.png]]

Checked port 6787 and found the `Oracle Solaris` dashboard login page. Tried the default username and password but was unable to login.
![[Pasted image 20260318200035.png]]

Back to the `finger` service on port 79, found a python script here https://github.com/dev-angelist/Finger-User-Enumeration that can be used to enumerate usernames on `finger`. What it does is just iterate through a list of usernames, sending `finger <username>@<target> <port>` and checking the results.

Also tried running another enumeration script found here https://pentestmonkey.net/tools/user-enumeration/finger-user-enum which was much faster. From this script, I found 2 users with SSH access, and the root user.
![[Pasted image 20260318203216.png]]

Users: `sammy`, `sunny`

The python script also returned the 3 users with better formatting, but took a very long time.
![[Pasted image 20260318204715.png]]

Checked `rpcinfo` but got an authentication error.
![[Pasted image 20260318204003.png]]

Since there doesn't seem to be anything else to check, I ran a dictionary attack on the 2 SSH usernames I found from `finger`, and found `sunny`'s password: ` sunday `.
![[Pasted image 20260318205641.png]]

Found a backup file in `/backup` containing the hashed password for the user `sammy`.
![[Pasted image 20260318215229.png]]

` $5$Ebkn8jlK$i6SSPa0.u7Gd.0oJOT4T421N2OvsfXqAT1vCoYUOigB `

Ran this hash through `hashcat` and cracked it.
![[Pasted image 20260318223612.png]]

` cooldude! `

Was able to SSH as `sammy` using this password and get the user flag.
![[Pasted image 20260318223720.png]]

Checked `sudo -l` and found that `sammy` can run `wget` as root.
![[Pasted image 20260318223832.png]]

From GTFOBins, it is possible to get a root shell with `wget` using
```
echo -e '#!/bin/sh\n/bin/sh 1>&0' >/path/to/temp-file
chmod +x /path/to/temp-file
wget --use-askpass=/path/to/temp-file 0
```

Ran the commands and got a shell as root.
![[Pasted image 20260318224038.png]]

There are also other ways to get root using `wget`, like overwriting the `/etc/shadow` file, or the `/etc/sudoers` file.