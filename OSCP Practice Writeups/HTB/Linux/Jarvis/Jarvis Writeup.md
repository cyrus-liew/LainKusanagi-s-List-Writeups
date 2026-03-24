Main steps
1. Fuzz directories to find `phpMyAdmin` login page
2. Enumerate webpage to find a subpage that is vulnerable to SQL injection by checking the request parameter in the URL
3. Perform UNION SQL injection on the site to retrieve database credentials
4. Login to `phpMyAdmin` with the credentials
5. Locate public exploit for vulnerable `phpMyAdmin` version
6. Gain shell as `www-data`
7. Enumerate `sudo -l` to find script
8. Enumerate script to find vulnerability in filtering
9. Run script to gain access as user
10. Enumerate `SUID` binaries for privilege escalation

Learning points
1. If a site has a `?something=value` parameter in the URL, it may be susceptible to SQL injection - check by adding a `'`, the site will break if it is.
2. UNION injections may require more fields than it looks.
3. It is possible to substitute commands using `$()` or backticks if command chaining is filtered out.

`nmap`
![[Pasted image 20260309205625.png]]

ports 22, 80, 64999 open

port 80 shows a PHP based website.
port 64999 shows the message below
![[Pasted image 20260309205725.png]]

`Feroxbuster` found a `phpMyAdmin` instance at `/phpymyadmin`. checking the README shows that the version is 4.8.0, which is vulnerable to an authenticated RCE.
![[Pasted image 20260309210158.png]]

Tried logging in with default credentials but was unable to.

`Whatweb` shows that there is an uncommon header named `ironwaf` which seems like it could be a WAF software.
Looking around on the site some more, I saw that the `Book Now` button links to a page that takes a parameter in the request.
![[Pasted image 20260309211212.png]]

Checking the headers here shows the `ironwaf` version as well: `IronWAF: 2.0.3`.
Googling this software returns nothing, so it's possibly custom software for this box.

On the `room.php` page, I tried adding a `'` to the end of the URL, and it broke the site, suggesting that it may be vulnerable to SQL injection. Tried a few common payloads to enumerate the database but did not get any results.
![[Pasted image 20260309211911.png]]

However, since I know that the site is using `phpMyAdmin`, I can be fairly certain that the database is running `MySQL`. 

Tried to carry out a UNION attack on the site since it seems like there are multiple things being pulled from the database for the booking page (image, name, price, description) and was able to inject at 7 columns, so I added a `@@version` to check the database version.

`http://supersecurehotel.htb/room.php?cod=200%20UNION%20SELECT%201,%201,%201,%20@@version,%201,%201,%201;--%20-`
![[Pasted image 20260309213445.png]]

Database version is `10.1.48-MariaDB-0+deb9u2`.

Tried extracting the database information with 
`http://supersecurehotel.htb/room.php?cod=200 UNION SELECT 1, 1, 1, GROUP_CONCAT(0x7c,schema_name,0x7c), 1, 1, 1 FROM information_schema.schemata;--`
![[Pasted image 20260309214103.png]]

Table names:
`http://supersecurehotel.htb/room.php?cod=200 UNION SELECT 1, 2, 3,GROUP_CONCAT(table_name), 5, 6, 7 FROM information_schema.tables WHERE table_schema='hotel';--`
![[Pasted image 20260309214546.png]]

Column Names:
`http://supersecurehotel.htb/room.php?cod=200 UNION SELECT 1, 2, 3,GROUP_CONCAT(column_name), 5, 6, 7 FROM information_schema.columns WHERE table_name='room';--`
![[Pasted image 20260309214659.png]]

Doesn't seem like there's any useful information in this schema, checked MySQL:
`http://supersecurehotel.htb/room.php?cod=200 UNION SELECT 1, 2, 3,GROUP_CONCAT(table_name), 5, 6, 7 FROM information_schema.tables WHERE table_schema='mysql';--`
There is a `User` table in the schema, 
`http://supersecurehotel.htb/room.php?cod=200 UNION SELECT 1, 2, 3,GROUP_CONCAT(table_name), 5, 6, 7 FROM information_schema.tables WHERE table_schema='user';--`
![[Pasted image 20260309215256.png]]

User and Password columns here. Dumped them with 
`http://supersecurehotel.htb/room.php?cod=200 UNION SELECT 1, 2, 3,GROUP_CONCAT(User), 5, 6, 7 FROM mysql.user;--`
and 
`http://supersecurehotel.htb/room.php?cod=200 UNION SELECT 1, 2, 3,GROUP_CONCAT(password), 5, 6, 7 FROM mysql.user;--`

and found
`DBadmin`: ` *2D2B7A5E4E637B8FBA1D17F40318F277D29964D0 `

Ran the hash through Crackstation and cracked it.
![[Pasted image 20260309215449.png]]

`DBadmin` : ` imissyou `

Using these credentials, I was able to login to `phpMyAdmin`.
![[Pasted image 20260309215556.png]]

Found the public exploit for RCE on `phpMyAdmin` 4.8.0 at https://www.exploit-db.com/exploits/50457, downloaded it and ran it against the server and opened a `nc` reverse shell as `www-data`.
![[Pasted image 20260309220041.png]]
![[Pasted image 20260309220048.png]]

Enumerating some common folders shows that there is a `simpler.py` script in `/var/www/Admin-Utilities` that `pepper` is allowed to run without a password. This script has a few uses, but notably it allows for pinging an IP address. Looking at the code, it seems like the script uses the python OS library to call the `ping` command.
![[Pasted image 20260309222359.png]]

This is a possible command injection vector, but the common bash command chaining symbols are filtered out. However, command substitution using `$()` is not filtered out, so I tried using this.

Ran the script as `pepper` using `sudo -u pepper /var/www/Admin-Utilities/simpler.py -p` and entered `10.10.14.$(bash)` as the input. While I was able to get a shell as `pepper`, commands would not work properly.

All the reverse shell commands on `bash` contain one or more of the filtered characters. However, it may be possible to write a command to a file and call it from the script.

I echoed a reverse shell command to a script file and added it to the command, and was able to get a shell as `pepper`.
![[Pasted image 20260309224418.png]]
![[Pasted image 20260309224429.png]]

Enumerating `SUID` files shows that `systemctl` has the `SUID` bit set. From GTFObins, it is possible to privilege escalate using `systemctl`.

Referring to this site https://alvinsmith.gitbook.io/progressive-oscp/untitled/vulnversity-privilege-escalation, I followed the steps to create a `root.service` file, then served to `pepper` using `wget` and enabled the service, then ran it and was able to get a root shell.
![[Pasted image 20260309225232.png]]

