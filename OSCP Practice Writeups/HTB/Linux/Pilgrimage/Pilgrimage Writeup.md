Main steps
1. Fuzz directories to find exposed `git` repository
2. Dump `git` repository to find vulnerable software
3. Locate public exploit for software to dump database
4. Read user credentials from database for SSH access
5. Check `ps auxww` to find script being run as root
6. Enumerate script and find vulnerable executable being run
7. Locate public exploit for executable and gain root access

Learning points
1. Even if the `git` repo returns 403 Forbidden, try to dump it using `git-dumper` as it may still work
2. Remember to run downloaded executables with `./` in front or you may run the wrong one
3. `sqlite` files are binary files - if you  get one in hex, can convert using `xxd -r -p`
4. Check executable file version properly as there may be public exploits

`nmap`
![[Pasted image 20260315183321.png]]

Ports 22, 80 open.

Website shows an upload field for a file, as well as a login and register page.
![[Pasted image 20260315183342.png]]

Tried to register for an account. The logged in page shows a dashboard where you can see your original uploaded image name and the URL for the shrunken image.
![[Pasted image 20260315183509.png]]

`Feroxbuster` found 2 interesting directories, but they both return 403 Forbidden, `/tmp` and `/.git`. However, it seems to be possible to download some of the files in the top level directory of `.git`, like the `index` and `config` files, but those do not contain anything interesting.
![[Pasted image 20260315183919.png]]

`ffuf` found nothing interesting.

Tried uploading a PHP web shell with PNG magic bytes, but got an error stating that the image could not be shrunk.
![[Pasted image 20260315185633.png]]

Uploaded a legitimate PNG file to see what happens. The file is successfully uploaded, and a link is added to the dashboard for the file under directory `/shrunk`.
![[Pasted image 20260315185723.png]]

Looks like the application renames the uploaded file to a random string ending with `.png`, meaning that double extensions will not work.
![[Pasted image 20260315190100.png]]

Back to the exposed `git` repo, tried to dump it using `git-dumper` as `wget` does not work, and was able to get the repo. `git status` showed nothing to commit, and `git log` showed 1 previous commit made by `emily`.
![[Pasted image 20260315190907.png]]

Checking the files in the `git` repo shows a large file named `magick`, which is an executable file.
![[Pasted image 20260315191117.png]]

Googling this shows that it is a CLI for the `ImageMagick` software for image processing, which matches up with the website.
![[Pasted image 20260315191220.png]]

Version: `ImageMagick 7.1.0-49 beta`
![[Pasted image 20260315191623.png]]

Searching up this version number shows that the software is vulnerable to an Information Disclosure vulnerability (LFI), CVE-2022-44268. Found a POC at https://github.com/Sybil-Scan/imagemagick-lfi-poc and downloaded it to try.

Was able to read the `/etc/passwd` file using this POC. Tried some other files like `/etc/shadow` and SSH keys, but was unable to.
![[Pasted image 20260315193300.png]]

Looking back to the `login.php` file in the `git` repo, the code seems to try to open a database connection from `/var/db/pilgrimage` for the login process.
![[Pasted image 20260315193414.png]]

Got the hex data by uploading a poisoned image, isolated the data from `identify -verbose` using `grep`, then converted the hex to binary using `xxd -r -p` and outputted the data to a `.sqlite file`.
![[Pasted image 20260315194352.png]]

Opened the database using `sqlite3` and was able to read the data within in plain text. The `users` table contained credentials for a user named `emily`.
![[Pasted image 20260315194542.png]]

`emily`: ` abigchonkyboi123 `

Tried to SSH using these credentials and was successful.
![[Pasted image 20260315194624.png]]

Checked `sudo -l`, `SUID` binaries, `cron` jobs, filesystem but found nothing interesting. However, checking `ps auxww` that root is running a script at `/bin/bash /usr/sbin/malwarescan.sh`.
![[Pasted image 20260315195155.png]]

Checking the script shows that it is waiting for a file to be created in `/var/www/pilgrimage.htb/shrunk/`, then using `binwalk` to check if the file is a script or executable. If it is, it removes the file.
![[Pasted image 20260315195846.png]]

Initially thought that it may be vulnerable to command injection, looking at `binout="$(/usr/local/bin/binwalk -e "$filename")"`, but `bash` seems to be able to prevent chaining commands here.

`binwalk` was the only third party binary here so I checked its version `v2.3.2` and Google immediately shows that it is vulnerable to an RCE CVE-2022-4510, where arbitrary code can be executed when a malicious file is opened with `binwalk -e`.

Found a POC at https://github.com/adhikara13/CVE-2022-4510-WalkingPath and downloaded it.

This POC allows you to either execute commands, open a reverse shell, or edit the root user's SSH keys.

Ran the exploit to generate the poisoned PNG file and tried uploading to the website, but it did not seem to work. Tried to directly upload the file to the `/shrunk` folder using `sshpass -p abigchonkyboi123 scp binwalk_exploit.png emily@pilgrimage.htb:/var/www/pilgrimage.htb/shrunk/` and was able to upload my own public key to the root user's `authorized_keys` and gain SSH access as root.
![[Pasted image 20260315201955.png]]

