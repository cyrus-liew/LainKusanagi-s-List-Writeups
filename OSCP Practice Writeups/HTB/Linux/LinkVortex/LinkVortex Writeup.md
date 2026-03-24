Main steps
1. Enumerate site to find CMS
2. Fuzz VHOSTs to find subdomain
3. Fuzz directories of subdomain to find exposed `git` repository
4. Enumerate `git` repository to find credentials
5. Login to CMS using credentials
6. Locate public exploit for CMS
7. Read config file of CMS using public exploit to find user credentials
8. Gain user access
9. Enumerate `sudo -l` to find script
10. Exploit `symlink` race condition in the script to read root SSH key and privilege escalate

Learning points
1. Check `git status` properly as it shows information on commits that have not been done
2. `git restore --staged . && git diff` shows the uncommitted code
3. `Symlinks` and race conditions in scripts

`nmap`
![[Pasted image 20260309230617.png]]

ports 22, 80 open

Site is using `Ghost 5.58` as a CMS and Node.js, React, and Express according to `Wappalyzer`
![[Pasted image 20260309230849.png]]

Found the `Ghost` login page at `/ghost`
![[Pasted image 20260309231124.png]]

`ffuf` returned a subdomain named `dev`. Added to `/etc/hosts` and found a website under construction notice.
![[Pasted image 20260309231412.png]]

Ran `Feroxbuster` on the site and found an exposed `git` repository.
![[Pasted image 20260309231551.png]]

Cloned the repository using `wget --mirror -I .git` then restored it with ` git restore . `
Checking `git status` shows that there is a change to be committed on `authentication.test.js`. Ran `git restore --staged . && git diff` to try to see what the change was, and found a password.
![[Pasted image 20260309233934.png]]
![[Pasted image 20260309233940.png]]

` OctopiFociPilfer45 `

The `Ghost` login page takes an email address as a username. Tried the `test@example.com` but was unable to login. Since the posts on the site were all written by `admin`,  I tried using `admin@linkvortex.htb` as the username and was able to login.
![[Pasted image 20260309234145.png]]

Googling the `Ghost` CMS version shows that it is vulnerable to CVE-2023-40028, an arbitrary file read exploit. (https://github.com/0xDTC/Ghost-5.58-Arbitrary-File-Read-CVE-2023-40028)

Downloaded and ran the exploit against the website and was able to read files.
`/etc/passwd` shows a user names `node`.
![[Pasted image 20260309234700.png]]

From Google, the `Ghost` config file seems to either be at `/var/www/ghost/config.production.json` or `/var/lib/ghost/config.production.json`, and trying both led to the file being at `/var/lib/ghost`.
![[Pasted image 20260309235302.png]]

There seems to be credentials here for user `bob` : ` fibber-talented-worth `

Tried to SSH using these credentials and was able to get access as `bob`.
![[Pasted image 20260309235423.png]]

Enumerating `sudo -l` shows that `bob` can run a script as `sudo`. Enumerating this script shows that it checks a .PNG file for `symlinks`, then if the `symlink` contains `/etc` or `/root` it removes the link, otherwise it quarantines the file and echoes the content of the `symlink` if a variable `CHECK_CONTENT` is set to true.

This can be exploited due to a Time-of-check to time-of-use (TOCTOU) race condition, where the system checks the resource's state, such as the permissions, but the state changes before the resource is used. This can allow us to bypass security checks - such as the conditional statement in the script that checks if the `symlink` points to `/etc` or `/root`.
![[Pasted image 20260310001140.png]]

Essentially, what needs to be done is:
1. Create a .PNG file with a random `symlink` (that does not point to `/etc` or `/root`)
![[Pasted image 20260310001734.png]]

2. Write a loop that constantly tries to set a `symlink` to any file (the root user's SSH key at `/root/.ssh/id_rsa` in this case) to the .PNG file after it has been quarantined to `/var/quarantine/file.png`
![[Pasted image 20260310001748.png]]

3. Set the `CHECK_CONTENT` variable to true in the ENV by exporting it
![[Pasted image 20260310001811.png]]

4. Run the script as `sudo`
![[Pasted image 20260310001820.png]]

Copied this SSH key and was able to login as root.
![[Pasted image 20260310001845.png]]