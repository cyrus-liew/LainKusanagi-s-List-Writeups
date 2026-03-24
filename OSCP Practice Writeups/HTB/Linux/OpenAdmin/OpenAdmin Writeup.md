Main steps
1. Fuzz directories to find application running
2. Locate public exploit for application
3. Gain shell access as `www-data`
4. Enumerate config files for credentials
5. Gain access as user
6. Enumerate `/var/www` to find internal website
7. Enumerate `sites-enabled` and `ss -ntplu` to find internal server port
8. Enumerate source code to find hard coded secret
9. Either port forward to access internal site and login to read SSH key, or craft POST request with `curl` to generate cookie and read SSH key
10. Crack SSH key password with `john`
11. Gain SSH access as other user
12. Enumerate `sudo -l` to find `sudo` program to privilege escalate

Learning points
1. Saving cookies with `curl`
2. REMEMBER TO REUSE PASSWORDS
3. Web servers can run internal websites, check the `site-enabled` to find their VHOST config
4. Cracking encrypted SSH keys with `ssh2john` and `john`

`nmap`
![[Pasted image 20260312011157.png]]

Ports 22, 80 open

Navigating to the website shows the Apache default page.
![[Pasted image 20260312194700.png]]

`Feroxbuster` found a few directories leading to separate websites at `/music`, `/artwork`, `/sierra`, and `/ona`.

Other than a few contact us forms that make POST requests, the first 3 directories do not look too interesting. However, `/ona` brings us to a dashboard for `OpenNetAdmin` v18.1.1.
![[Pasted image 20260312195341.png]]

There is also a DNS domain listed here so I went ahead and added it to my `/etc/hosts`.
![[Pasted image 20260312195418.png]]

The page also lists some database information.
![[Pasted image 20260312195528.png]]

Searching up this version of `OpenNetAdmin` gives an RCE exploit with command injection due to improper input sanitization. https://www.exploit-db.com/exploits/47691

The application takes in an IP address as user input and passes it directly to PHP's `shell_exec`. It is possible to execute arbitrary commands by chaining commands with a semi-colon.
```
curl -silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";id;echo \"END\"&xajaxargs[]=ping" "http://openadmin.htb/ona/" | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1
```
![[Pasted image 20260312200559.png]]

Trying to spawn a reverse shell directly using this exploit didn't seem to work, so I wrote a reverse shell script and uploaded it using `curl|sh` and got a reverse shell as `www-data`.
![[Pasted image 20260312202213.png]]
![[Pasted image 20260312202202.png]]

`/etc/passwd` shows two users: `jimmy` and `joanna`.

Checked through the `OpenNetAdmin` files and found database credentials at `/opt/ona/www/local/config`.
![[Pasted image 20260312202757.png]]

`ona_sys`: ` n1nj4W4rri0R! `

Found hashed passwords for `guest` and `admin` in the database.
![[Pasted image 20260312202943.png]]

Ran the hashes through Crackstation and got both passwords.
![[Pasted image 20260312203027.png]]

`guest`: ` test `
`admin`:  ` admin `

However, the app does not seem to let me login as `admin`. 

Tried `admin` as the password for both users but was also unable to pivot.
Tried using `n1nj4W4rri0R!` as the password and was able to successfully `su` to user `jimmy`.

Moved over to SSH for a more stable shell.
![[Pasted image 20260312203919.png]]

Checked `sudo -l`, `SUID` binaries, and `cron` jobs but found nothing. Unable to read `joanna`'s home folder either.

Tried checking `/var/www/` since there were other websites on the IP and found an interesting folder at `/internal`. There is an `index.php` page inside that has a hard-coded hash for a login form.
![[Pasted image 20260312210101.png]]

Running this hash through Crackstation shows that the password is `Revealed`.
![[Pasted image 20260312210142.png]]

Logging in here redirects us to `/main.php`, which contains code that outputs `joanna`'s SSH key.
![[Pasted image 20260312210404.png]]

Tried starting a PHP server and sending POST data to the file but it did not work.

Since the files are in the `/var/www` folder, there might be an internal server running to serve this webpage.

Checked the `apache` site-enabled config files at `/etc/apache2/sites-enabled` and found a config file for `internal.conf` which showed that there was a VHOST listening on `127.0.0.1:52846` which was serving `/var/www/internal` run by user `joanna`.
![[Pasted image 20260312212728.png]]

Checking `ss -ntplu` confirms that this service is running.
![[Pasted image 20260312212749.png]]

Tried sending the same `curl` POST request to this address but it still did not work. Most likely will need a proper browser.

However, there is no way to access this webpage on a browser directly, so this will need to be port forwarded to my machine.

Ran `chisel` to port forward to my machine and was able to access the site.
![[Pasted image 20260312213251.png]]

Logging in with `jimmy`:`Revealed` returned `joanna`'s SSH key.
![[Pasted image 20260312213319.png]]

**ANOTHER WAY TO ACCESS THIS KEY**
Going back to the PHP code above at `/var/www/internal/index.php`: it is actually possible to access the `main.php` with `curl`, my request above was just missing the option to store the `PHPSESSID` cookie.

The proper way to do it would be to `curl` with the `-c` parameter to store the cookie, then send a GET request to `main.php` with the cookie using `-b`.
```
curl -X POST http://127.0.0.1:52846/index.php \
  -d "username=jimmy" \
  -d "password=Revealed" \
  -d "login=1" \
  -c cookies.txt
  
curl http://127.0.0.1:52846/main.php -b cookies.txt
```

This also allows us to read the SSH key for `joanna` without needing to port forward.
![[Pasted image 20260312215339.png]]

Tried SSH as user `joanna` using the `n1nj4W4rri0R!` password but was unable to.

Since the RSA key is encrypted, I might need to crack it somehow.
Tried using a dictionary attack with `johntheripper`.
Converted the key to a hash using `ssh2john` then ran `john` with `rockyou.txt` and cracked the hash: `bloodninjas` which matches the reminder on the site.
![[Pasted image 20260312213826.png]]

Was able to get user access as `joanna`.
![[Pasted image 20260312213854.png]]

Checked `sudo -l` and saw that `joanna` can run `/bin/nano /opt/priv` as `sudo` with no password.

It is possible to execute commands directly from `nano` while writing to a file, by pressing `CTRL+R` then `CTRL+X`, which spawns a command shell that inherits privileges.
![[Pasted image 20260312214520.png]]
![[Pasted image 20260312214534.png]]

Used this command shell to spawn a reverse shell and gained access as root.
![[Pasted image 20260312214338.png]]

