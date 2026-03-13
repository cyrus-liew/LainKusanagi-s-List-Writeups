Main steps
1. Enumerate input field on website
2. Enumerate the generated PDF using `exiftool` to find backend software
3. Locate public exploit for backend software to gain shell access
4. Enumerate config file to find user credentials
5. Gain SSH access using user credentials
6. Enumerate `sudo -l` to find script
7. Enumerate script and `Ruby` version to find public exploit
8. Use public exploit to gain root access

Learning points
1. Check file metadata with `exiftool` - can return some info on backend software being used
2. Check all folders properly (especially if they are not default)
3. `Ruby` stores some config files under `.bundle` in the user's home directory so check that
4. `Ruby` scripts load files from the current working directory when the full path is not specified

`nmap`
![[Pasted image 20260306193041.png]]

ports 22 and 80 open, added `precious.htb` to `/etc/hosts`
Simple website with an input field for a URL
![[Pasted image 20260306193439.png]]

Quick test on the input field shows that it only accepts inputs beginning with `http://` or it throws an error.

Ran `Feroxbuster` and `ffuf` to fuzz directories and VHOSTs, no results
`Whatweb` shows that this site is using `Phusion Passenger(R) 6.0.15`, which is a web and application server for Ruby, Node.js and Python apps.
![[Pasted image 20260306193659.png]]

Attempting to inject code into the input field using a semicolon seems to trigger a download of a empty PDF file. Since writing a full URL does not seem to do anything, this is possibly causing the code to run on the remote server.
![[Pasted image 20260306194303.png]]

Tried to convert my own HTTP server:
![[Pasted image 20260306195058.png]]

Based on my HTTP server logs, the remote server is sending a GET request to download the page.
![[Pasted image 20260306195402.png]]

Tried using `exiftool` to view the metadata for the generated PDF file and found that it was generated using `pdfkit v0.8.6`.
![[Pasted image 20260306201011.png]]

Searching for this version gives a command injection vulnerability.
Found a public exploit here https://github.com/shamo0/PDFkit-CMD-Injection and ran it against the server and gained a reverse shell as user `ruby`.
![[Pasted image 20260306201408.png]]

`/etc/passwd` shows another user named `henry`.
![[Pasted image 20260306202122.png]]

`linpeas` found `
`/tmp/passenger.XAdNcp8/full_admin_password.txt`
`/tmp/passenger.XAdNcp8/read_only_admin_password.txt` but they are both not readable. 

Checking `ruby`'s home directory again, I checked the folder `.bundle` and found a config file with credentials.
![[Pasted image 20260306205501.png]]

`henry` : ` Q3c1AqGHtoI0aXAYFH `

Tried using this to access `henry` and was able to.
![[Pasted image 20260306205552.png]]

Enumerating `sudo -l` shows that `henry` can run a `Ruby` script as root. with no password.
![[Pasted image 20260306211225.png]]

Looking at this script, it seems to update Ruby Gems automatically, and it loads the "dependecies.yml" file from the current working directory using `YAML.load`. 
![[Pasted image 20260306211323.png]]

Searching for this version of Ruby (2.7.0) gives this exploit (https://staaldraad.github.io/post/2021-01-09-universal-rce-ruby-yaml-load-updated/), an RCE with Ruby's `YAML.load` function. I created a `dependencies.yml` file in `henry`'s home directory and added the payload into the file, then ran the script with `sudo` and was able to launch a root shell.
![[Pasted image 20260306211604.png]]
![[Pasted image 20260306211610.png]]
