Main steps
1. Gain access via default credentials
2. Enumerate the website to find user's credentials
3. SSH access using found credentials
4. Locate public exploit for KeePass memory dump vulnerability
5. Dump master key for KeePass database
6. Read PuTTY SSH key from database
7. Gain root access

Learning points
1. Enumerate websites closely if there aren't any exploits available
2. Just Google stuff if it seems like it's an obvious attack vector

nmap
![[Pasted image 20260305200803.png]]

ports 22, 80 open

domain listed on the webpage, added to`/etc/hosts`
![[Pasted image 20260305200830.png]]

Redirects to a login page
![[Pasted image 20260305201111.png]]

Googled default credentials and found `root`:`password`, and was able to login
![[Pasted image 20260305201645.png]]

Checking the queue list, there is a ticket for a Keepass issue from a user.
![[Pasted image 20260305202251.png]]

` lnorgaard ` (Lise Nørgaard) : possible user
Checking the user summary page shows that the default password is` Welcome2023! `
![[Pasted image 20260305202731.png]]

Using the username and password, was able to login to the RT.
Nothing interesting here and the site is in Danish.

Tried to SSH using the credentials and was able to get access.
![[Pasted image 20260305203047.png]]

There is a zip folder in the user's home directory that gives a keepass memory dump and database file.
![[Pasted image 20260305210124.png]]

Googling for keepass memory dumps shows CVE-2023-32784 - reading the keepass master key from a memory dump. 
POC found here https://github.com/vdohney/keepass-password-dumper?tab=readme-ov-file

Running this against the memory dump give this output:
![[Pasted image 20260305210529.png]]

The password seems to be in Danish, and the first 2 characters are missing. Googling the remainder `dgrød med fløde` shows this 
![[Pasted image 20260305210802.png]]

`rødgrød med fløde`. This fits as the second character in the password can be a `ø`. 

Running this against the `passcodes.kdbx` give access to the database.
![[Pasted image 20260305211048.png]]

Enumerating the database shows a username and password combination for root, as well the user's previously found password.
![[Pasted image 20260305211654.png]]

`root`: ` F4><3K0nd! `

Unable to SSH directly using this password.
There is also a SSH key here in the notes.

SSH key is in PuTTY format, so I copied it out and converted it to OpenSSH using `puttygen keeperkey.ppk -O private-openssh -o key`.

Used this key to try to SSH as root and was able to get root access.
![[Pasted image 20260305212423.png]]