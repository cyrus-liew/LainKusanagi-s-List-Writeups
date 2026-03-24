Main steps
1. Enumerate open ports to find `JAMES` email server and version
2. Reset passwords for users on `JAMES`
3. Login to POP3 to read emails and find user credentials
4. Gain restricted SSH access from credentials
5. Locate public exploit for `JAMES` server to open unrestricted shell
6. Run `pspy` to snoop root `cron` job running a python script
7. Edit python script to launch reverse shell as root

Learning points
1. POP3 login and email read
2. Run `pspy` if all other privilege escalation methods do not return anything
3. Restricted shell can be bypassed by passing `-t bash` to SSH login command

`nmap`
![[Pasted image 20260317185805.png]]

Ports open:
1. 22
2. 25 - SMTP
3. 110 - POP3
4. 119 - NNTP
5. 4555 - Possibly RSIP or Apache James Server

Searching for port 4555 gives this https://www.ibm.com/docs/en/webmethods-integration/wm-infrastructure/11.1.0?topic=transport-setting-up-apache-james-server documentation page on `Apache James Server`, which is a enterprise mail server for email. It also requires SMTP and POP3 to be open, which matches the open ports on the server.

From the documentation, it looks like you can `telnet` into the server for administration with the default credentials `root`:`root`. 

Tried doing so and was able to connect.
![[Pasted image 20260317190320.png]]

`listusers` shows a few possible usernames.
![[Pasted image 20260317190342.png]]

The other commands do not seem interesting for now. Searching how to check the version number for the server shows that `telnet` to the SMTP port works.
![[Pasted image 20260317190603.png]]

Found the version for the `James` server: `2.3.2`.

Searching for this server version gives me an RCE CVE-2015-7611, which creates a new user and queues a payload to be executed the next time a user logs in to the machine. Found a python POC https://www.exploit-db.com/exploits/50347.

The problem with this is that I need to be able to login to the machine via SSH. 

From the `James` options, it is possible to set the password for an existing mail user, and since POP3 is enabled and open, I may be able to login as that user find more information that way.

Tried resetting the password for `mailadmin` and logging in to POP3 via `telnet` and was successful, but there were no emails.

Tried the user `mindy` next and found 2 emails. The second email contained SSH credentials for the user `mindy`, so now the above exploit can be used.
![[Pasted image 20260317192822.png]]

`mindy`: ` P@55W0rd1!2@ `

Tried logging in to check the credentials and was able to SSH. User flag was in this user's home directory.
![[Pasted image 20260317192940.png]]

The user `mindy`'s shell access is heavily restricted and most commands do not work.

Ran the exploit above and tried to spawn a reverse shell with it, and was able to get a shell as `mindy` without the restrictions.
![[Pasted image 20260317193639.png]]

Check `sudo -l`, `SUID` binaries, `cron` jobs, network ports with `ss -ntplu` (localhost listening on port 631 - printing service), `ps auxww` but did not find anything interesting.

Ran `pspy` to see if any scripts are running in the background, and found a script at `/opt/tmp.py` that was running as root. 
![[Pasted image 20260317194912.png]]

The script itself is nothing interesting, but it seems that `mindy` has full permissions to edit the script.

Tried adding python code to spawn a reverse shell into the script. Waited a bit for the script to execute, then got a reverse shell as root.
![[Pasted image 20260317195433.png]]