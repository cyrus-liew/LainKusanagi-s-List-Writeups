Main steps
1. Enumerate site to find backend ecommerce software
2. Enumerate software version
3. Locate public exploits for software
4. Use public exploits to gain shell access as `www-data`
5. Enumerate `sudo -l`
6. Privilege escalate with `sudo` binary

Learning points
1. Well known software may have prebuilt scanners that can be used
2. Check privilege escalation vectors on `www-data` type users as well

`nmap`
![[Pasted image 20260319000616.png]]

Ports 22, 80 open

Website shows an ecommerce platform running `Magento`.
![[Pasted image 20260319000747.png]]

`Feroxbuster` found a lot of directories on the site.
Some of the interesting files found:

`http://swagshop.htb/app/etc/local.xml`: Some potential database credentials here
![[Pasted image 20260319001311.png]]

`http://swagshop.htb/index.php/admin`: `Magento` Admin panel login page
![[Pasted image 20260319001804.png]]

Found a prebuilt tool to scan `Magento` sites called `magescan` and ran it to find the version running: version `1.9.0.0` or `1.9.0.1`.
![[Pasted image 20260319002505.png]]

Searching this version for `Magento` gives me a whole list of vulnerabilities, including authenticated RCE, directory traversal, SQL injection, and a flaw that allows unauthorized users to create admin accounts called `Shoplift`.

Since the RCE exploit is authenticated, the most interesting one to look at here is the `Shoplift` exploit.

Found a POC here https://github.com/Hackhoven/Magento-Shoplift-Exploit that carries out the `Shoplift` exploit to create an account.

Ran the exploit and successfully created an admin account.
![[Pasted image 20260319003135.png]]
![[Pasted image 20260319003141.png]]

Found the POC or the authenticated RCE here https://github.com/Hackhoven/Magento-RCE. Tried running the script and was able to run commands.
![[Pasted image 20260319003752.png]]

Ran a reverse shell `nc` one liner and got a reverse shell as `www-data`.
![[Pasted image 20260319003931.png]]

Accessed the database using the credentials from the XML file above and found some credentials for an admin user `Harris Swagger`.
![[Pasted image 20260319004309.png]]

` 8512c803ecf70d315b7a43a1c8918522:lBHk0AOG0ux8Ac4tcM1sSb1iD5BNnRJp `

Looking at the POC to create an admin user, the password is hashed with MD5, with a salt value which is appended to the end after the `:`.

Tried cracking this hash with `hashcat` but was unsuccessful.

Checking the user `haris`'s home directory reveals that the user flag is readable by `www-data`. 

Checked `sudo -l` for `www-data`, and found that the user can run `/usr/bin/vi /var/www/html/*` as root without a password.

With `vi` running as root, I can open any file, then use `vi`'s command mode to write any shell command with `:!` prepended. I launched a `bash` shell with `:!/bin/bash` and got a shell as root.
![[Pasted image 20260319005619.png]]