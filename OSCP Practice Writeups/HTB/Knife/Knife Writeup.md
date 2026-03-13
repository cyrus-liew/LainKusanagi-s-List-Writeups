Main steps
1. Enumerate website backend software
2. Locate public exploit for vulnerable software version
3. Gain user access through public exploit
4. Enumerate `sudo -l` 
5. Run program as `sudo` to gain root shell

Learning points
1. Check code versions with `Whatweb` as they may also be vulnerable (PHP, .NET etc.)

`nmap`
![[Pasted image 20260306182648.png]]

ports 22, 80 open

Webpage with no links
![[Pasted image 20260306182928.png]]

`Feroxbuster` returned no found directories (raft-medium-words)
Fuzzing VHOSTs with `ffuf` also returned no results
`Whatweb` shows that the website is running with a `dev` version of PHP (8.1.0-dev).
![[Pasted image 20260306183133.png]]

Google search of this PHP version shows that there is a backdoor in the PHP source code for this version (https://www.exploit-db.com/exploits/49933)

Downloaded and ran the exploit against the website and got user access as `james`
![[Pasted image 20260306183558.png]]

Opened a `nc` reverse shell and stabilized with `python`

Enumerating `sudo -l` shows that user `james` can run `/usr/bin/knife` as `sudo` without a password
![[Pasted image 20260306184022.png]]

From GTFObins: `knife` can be used to run Ruby code
Ran the command `sudo /usr/bin/knife exec -E 'exec "/bin/sh"'` and got a shell as root
![[Pasted image 20260306184233.png]]
