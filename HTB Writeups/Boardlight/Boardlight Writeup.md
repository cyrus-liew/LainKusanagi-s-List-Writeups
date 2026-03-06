Main steps:
1. Fuzz VHOSTs of webpage
2. Gain access using default credentials and public exploit of vulnerable program
3. Enumerate config files for credentials
4. Gain access using username:password combination found
5. Privilege escalate using SUID program with public exploit

Learning points:
6. Make sure the domain name is correct (check webpage for domain name)
7. If directory fuzzing returns no results, check VHOSTs (gobuster/ ffuf)
8. refer to https://agrohacksstuff.io/posts/web-penetration-test-enumeration-guide
9. Remember to try all username:password combinations that were found, even for diff users
10. Enumerate any SUID file that is not a default Linux program properly

Nmap - port 22 and 80 open
![[Pasted image 20260302154822.png]]
Webpage with a form with input fields; does not have any form-action in the source code so probably does nothing
![[Pasted image 20260302155139.png]]
Feroxbuster found /portfolio.php
![[Pasted image 20260302155348.png]]
![[Pasted image 20260302155422.png]]
Found domain name on webpage, added to /etc/hosts
![[Pasted image 20260302165302.png]]
VHOST scan using FFUF:
	ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://board.htb/ -H "Host: FUZZ.board.htb" -fl 518

![[Pasted image 20260302165414.png]]
Found http://crm.board.htb, leads to a Dolibarr CRM login page (version 17.0.0)
![[Pasted image 20260302165502.png]]
Tried using the default admin:admin credentials and was able to gain access, but it doesn't seem like I can do anything from here since the session is not authenticated.
![[Pasted image 20260302165617.png]]
Doing a quick search for Dolibarr 17.0.0 gives a RCE exploit https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253 that requires the username and password for the Dolibarr login.
Running this python script led me to a reverse shell as www-data.
![[Pasted image 20260302181805.png]]
![[Pasted image 20260302181757.png]]
wget linpeas.sh and ran it
nothing major found

checked the Dolibarr confing file under conf/conf.php, found credentials for SQL
	dolibarrowner:serverfun2$2023!!
![[Pasted image 20260302191428.png]]
tried to SSH using these credentials but was unable to
from /etc/passwd - there was another user on the machine named 'larissa'
attempted to SSH using larissa:serverfun2$2023!! and got access as larissa
![[Pasted image 20260302193353.png]]
looking through the SUID files, there is a application 'enlightenment' that is not a default Linux program. 
	`find / -type f -perm -04000 -ls 2>/dev/null`
![[Pasted image 20260302194027.png]]A quick search shows that there is a vulnerability in 'enlightenment' before version 0.25.4 that allows for privilege escalation. Ran dpkg -l to check the version and found that it was 0.23.1-4.
![[Pasted image 20260302194020.png]]
Found the exploit on ExploitDB at https://www.exploit-db.com/exploits/51180
Saved the script to my Kali and served to 'larissa' using wget
Ran the script and got a root shell
![[Pasted image 20260302194335.png]]