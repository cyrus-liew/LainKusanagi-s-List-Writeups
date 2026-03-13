Main steps
1. Enumerate source code for directory
2. Fuzz directories to find CMS version
3. Locate public exploit for CMS version
4. Gain access to CMS admin page using default credentials
5. Gain user access through public exploit
6. Enumerate `sudo -l` to find misconfigured script
7. Run script as `sudo` to gain root access

Learning points
1. Always search for default credentials and try them

`nmap`
![[Pasted image 20260306184855.png]]

Ports 22, 80 open
Nothing on the site other than a 'Hello World!'
![[Pasted image 20260306184909.png]]

Checking the source code shows a directory `/nibbleblog/`
![[Pasted image 20260306184952.png]]

Which brings me to this page
![[Pasted image 20260306185007.png]]

The only working link here is 'Atom' which brings me to this page with what looks like an internal IP address
![[Pasted image 20260306185150.png]]

Running `Feroxbuster` against the site shows a long list of directories. 
There is a README file with the CMS software and version: `Nibbleblog 4.0.3`
![[Pasted image 20260306185557.png]]

There is also an XML file with a username: `admin`
![[Pasted image 20260306185640.png]]

Found an admin login page for `Nibbleblog`.
A Google search for this version shows that that is a File Upload RCE exploit, but it requires admin access to `Nibbleblog`. (https://github.com/hadrian3689/nibbleblog_4.0.3)
![[Pasted image 20260306185933.png]]

Searched for the default `Nibbleblog` credentials and tried them against the admin page: `admin` : `nibbles` and was able to get access.
![[Pasted image 20260306190136.png]]

Downloaded and ran the exploit against the website and used it to start a reverse shell with `nc` and gained user access as `nibbler`
![[Pasted image 20260306190515.png]]
![[Pasted image 20260306190524.png]]

Enumerated `sudo -l` and found a script that could be run as root without a password
![[Pasted image 20260306190650.png]]

Created this directory and script since they did not exist (it was zipped in a zip folder)
Added code to launch `bash` in the `monitor.sh` script using 
`echo '#\!/bin/bash' >> monitor.sh`
`echo 'exec bash' >> monitor.sh`
then ran the script as `sudo` and gained a root shell.
![[Pasted image 20260306191838.png]]
