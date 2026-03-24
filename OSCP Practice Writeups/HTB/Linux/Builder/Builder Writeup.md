Main steps
1. Enumerate ports to find backend software
2. Enumerate software to find username
3. Enumerate software version to locate public exploit
4. Read hashed credentials using public exploit and got access to user's account
5. Enumerated admin dashboard and found root SSH key
6. Decrypted SSH key using the software's console
7. Gained root SSH access using key

Learning points
1. SSH keys need permissions set to `600` or it will not run
2. Pretty sure this machine was rated for before there was a public exploit available for the file read

`nmap`
![[Pasted image 20260306230053.png]]
ports 22 and 8080 open. 8080 running `Jetty 10.0.18`.

Connecting to port 8080 gives a Jenkins site.
![[Pasted image 20260306230155.png]]

Tried to login using default credentials but was unable to.
Enumerating the site shows a user named `jennifer`.
![[Pasted image 20260306230243.png]]

Googling the Jenkins version, 2.441, gives a file read vulnerability CVE-2024-23897. Found a POC at https://github.com/godylockz/CVE-2024-23897.

Ran the python script against the website and managed to read files from the server.
Read `/var/jenkins_home/users/users.xml` and found the user directory `jennifer_12108429903186576833`, then read the user's config file from `/var/jenkins_home/users/jennifer_12108429903186576833/config.xml` to find their hashed password.
![[Pasted image 20260306231221.png]]
`jennifer`: ` $2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a `

Ran hashcat against the hash and cracked it: ` princess `
![[Pasted image 20260306231535.png]]

Logged in as user `jennifer` to Jenkins.
![[Pasted image 20260306231717.png]]

Under the Jenkins global credentials, I accessed the edit menu for the SSH key and tried viewing the source code, and was able to view an SSH key.
![[Pasted image 20260307005920.png]]

Seems like this is an encrypted SSH key, and it can be decrypted with the Jenkins `master.key` and `hudson.util.Secret` files. These can be downloaded through the above python file read exploit.
![[Pasted image 20260307010354.png]]

Found a decryptor at https://github.com/hoto/jenkins-credentials-decryptor - but realized that this is for decrypting credentials and not SSH keys.

Found this article at https://rafaelmedeiros94.medium.com/dumping-jenkins-credentials-dont-try-this-at-127-0-0-1-or-try-it-4d0ab4667603 where the secret can be decrypted using the Jenkins script console. Navigated to `/manage/script` and used `println hudson.util.Secret.decrypt("{Secret=}")` and decrypted the SSH key.
![[Pasted image 20260307011016.png]]

Saved the private key to a file and changes its permissions to `600`, then tried to SSH as root using the key, and was able to gain access.
![[Pasted image 20260307011316.png]]

