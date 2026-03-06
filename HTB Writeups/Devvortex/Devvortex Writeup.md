Main steps
1. Fuzz VHOSTS to find subdomain
2. Check web technology to find CMS
3. Enumerate CMS config file to find version
4. Locate public exploit to find credentials
5. Login to CMS using credentials and upload reverse shell code
6. Gain reverse shell access
7. Enumerate CMS config file for database credentials
8. Find hashed user credentials in database
9. Crack hash with hashcat
10. Gain SSH user access
11. Enumerate `sudo -l`
12. Locate public exploit for sudo program and privilege escalate

Learning points
1. CMS's are usually vulnerable

nmap
![[Pasted image 20260305163601.png]]
ports 22 and 80 open
added devvortex.htb to /etc/hosts

no interesting directories found with feroxbuster / raft-medium-words

VHOST fuzzing with ffuf - found a dev.devvortex.htb subdomain
![[Pasted image 20260305164111.png]]

another static webpage - from wappalyzer, this page is using Joomla as a CMS.
![[Pasted image 20260305165913.png]]
Checked the` /administrator/manifests/files/joomla.xml` to find the Joomla version - 4.2.6
![[Pasted image 20260305170013.png]]
Also found some directories and files.

Navigating to /administrator showed a Joomla login page.
![[Pasted image 20260305170454.png]]

Searching for public exploits for Joomla 4.2.6 showed this https://github.com/K3ysTr0K3R/CVE-2023-23752-EXPLOIT exploit that dumps usernames and passwords. Cloned and ran it against the website and got users `lewis`: `P4ntherg0t1n5r3c0n## `and `logan`.
![[Pasted image 20260305171738.png]]

Tried `lewis` against the administrator dashboard and was able to login.

The `template` option was missing so I edited the `system` settings on the dashboard to show it.
![[Pasted image 20260305173214.png]]

Able to edit the PHP code of the site from here.
![[Pasted image 20260305173248.png]]

`index.php` was not editable, but `error.php` was. Added a PHP reverse shell one liner to the `error.php` code and navigated to `http://dev.devvortex.htb/templates/cassiopeia/error.php` and was able to get a reverse shell as `www-data`.
![[Pasted image 20260305174013.png]]

`/etc/passwd `shows that `logan` is a user on this machine.
![[Pasted image 20260305174117.png]]

Enumerating the Joomla `configuration.php` file in its webroot shows this secret: ` ZI7zLTbaGKliS9gq `. This is the random value Joomla uses to encrypt things like session tokens.
![[Pasted image 20260305174256.png]]

Used `lewis` to login to the mysql database.
![[Pasted image 20260305174603.png]]

Found user `logan`'s hashed password under the Joomla database in the users table
![[Pasted image 20260305174816.png]]

`$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12` - this is a Bcrypt hash.

Ran `hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt` on the hash and cracked it.
![[Pasted image 20260305175055.png]]
`logan` : ` tequieromucho `

Used the credentials to SSH and was able to gain access and view the user flag.
![[Pasted image 20260305175156.png]]

Checking `sudo-l` shows that `logan` can run `apport-cli` as sudo.
![[Pasted image 20260305175703.png]]

Searching this gives CVE-2023–1326 (https://0xd1eg0.medium.com/cve-2023-1326-poc-c8f2a59d0e00; https://github.com/diego-tella/CVE-2023-1326-PoC), a local privilege escalation using `apport-cli` if can be run as sudo.

After following the steps, spawned a root shell.
![[Pasted image 20260305175856.png]]
