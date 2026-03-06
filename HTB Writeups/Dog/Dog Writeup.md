Main steps
1. Enumerate the website to find the CMS version
2. Enumerate the exposed git repository to find credentials
3. Run a dictionary attack to find valid usernames for the CMS
4. Locate a public exploit to gain shell access
5. Enumerate users from `/etc/passwd`
6. Password spraying against enumerated users
7. Privilege escalate with `sudo -l`

Learning points
1. Exposed git repos can contain credentials - check properly
2. User enumeration if you have a password but no username
3. Just spray all known passwords against all known usernames

nmap
![[Pasted image 20260305181250.png]]

ports 22, 80 open
scan found a .git repo at ` http://10.129.236.81:80/.git/ `
found ` robots.txt `
![[Pasted image 20260305181346.png]]

Enumerating the files under /core/ got me a version number
![[Pasted image 20260305181750.png]]

Seems like there's an RCE exploit for this version, but requires login access to Backdrop

Going back to the exposed .git repo - cloned it using ` wget --mirror -I .git ` then restored it with ` git restore . `

Enumerating the restored repo:
in `settings.php` : database credentials found, `root `: ` BackDropJ2024DS2024 `
![[Pasted image 20260305184534.png]]

Nothing else in the git repo other than this. Have a possible password for logging in to Backdrop. Tried using the usernames present (root, dogBackDropSystem) but could not login.

Found this tool https://github.com/FisMatHack/BackDropScan that can be used to enumerate usernames for Backdrop and ran it against the website.
![[Pasted image 20260305191734.png]]

Tried `john`, didn't work, tried `tiffany` with ` BackDropJ2024DS2024 ` and was able to log in.
![[Pasted image 20260305192040.png]]

Now that I have admin access to Backdrop, I went back to download the public exploit for Backdrop 1.27.1 at https://github.com/rvizx/backdrop-rce and ran it against the server. This exploit creates and uploads a PHP shell into Backdrop and connects to it. Gained access as `www-data`.
![[Pasted image 20260305192144.png]]

Opened a netcat reverse shell using this shell for better persistence

Access the mysql server using the credentials found previously and dumped the users table under Backdrop and found 8 users.
![[Pasted image 20260305193547.png]]

Based on the /etc/pass file, there are 2 users, `jobert` and `johncusack` on the machine.
![[Pasted image 20260305193657.png]]

hashed credentials:
`jobert` : ` $S$E/F9mVPgX4.dGDeDuKxPdXEONCzSvGpjxUeMALZ2IjBrve9Rcoz1 `
`john` : ` $S$EYniSfxXt8z3gJ7pfhP5iIncFfCKz8EIkjUD66n/OTdQBFklAji. `

`$S$E` hashes are salted hashes (salt was in the `settings.php` file earlier - ` aWFvPQNGZSz1DQ701dD4lC5v1hQW34NefHvyZUzlThQ `) by Drupal7's hashing algorithm.

Further Googling shows that this is not the salt used for password hashing but rather for session tokens etc.

Therefore it does not seem possible to crack these hashes.

Tried the previous password found against the two user accounts and was able to SSH as `johncusack`.
![[Pasted image 20260305194922.png]]

Enumerating `sudo -l` shows that `johncusack` can run `bee` as `sudo`.
From GTFObins, `bee` can be used to run PHP code from `/var/www/html`.
Used `bee` to try to launch a root shell with `sudo bee eval 'system("/bin/sh -i");'` and was able to get root.
![[Pasted image 20260305195725.png]]