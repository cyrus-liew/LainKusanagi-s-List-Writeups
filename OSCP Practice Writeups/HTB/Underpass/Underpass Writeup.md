Main steps
1. Enumerate open UDP ports to find SNMP port 161 open
2. Enumerate SNMP tree to find backend software
3. Enumerate backend software's GitHub to find its login page
4. Login using default credentials
5. Enumerate admin dashboard to find user credentials
6. Gain SSH access using user credentials
7. Enumerate `sudo -l` to find no password `sudo` program
8. Gain root access using `sudo` program

Learning points
1. If all else fails run a UDP scan
2. SNMP enumeration and how community strings work
3. Check the GitHub pages for backend software - may lead to exposing the directory tree
4. Remember to enumerate dashboards

`nmap`
![[Pasted image 20260306212704.png]]

ports 22, 80 open
Website shows the default Apache2 launch page.
![[Pasted image 20260306212725.png]]

`Feroxbuster`, `gobuster` returned nothing interesting. Can't fuzz for VHOSTs since I there's no domain name.

Tried running a UDP scan against the IP and found an open port 161 (SNMP).
![[Pasted image 20260306215234.png]]

Ran `sudo nmap -sU -p 161 --script=snmp* ` to get more info on the SNMP service.
![[Pasted image 20260306215843.png]]

Tried using `SNMPWalk` with the default community string `public` with `-v1` to enumerate the SNMP and got some interesting information.
![[Pasted image 20260306220245.png]]

An email with a potential user, `steve`, and a domain name, `UnDerPass.htb`. Added this to my `/etc/hosts` file. There is also a string `UnDerPass.htb is the only daloradius server in the basin!` where `daloRadius` is a program used for RADIUS management.

According to Google, the `daloRadius` default login page should be at `/daloradius/login.php`, but navigating to that page gives a 404 error. However, trying to access `/daloradius/` give a different error - 403 Forbidden, which means there should be a directory named `/daloradius/` on this server.

Ran `Feroxbuster` for `/daloradius/` and found a bunch of subdirectories.
![[Pasted image 20260306223017.png]]

The layout of the subdirectories here looks quite similar to the one on the `daloRadius` GitHub page (https://github.com/lirantal/daloradius). Searching the GitHub page, I found the login.php under `/app/users/login.php`, and I was able to access the page.
![[Pasted image 20260306223134.png]]

Tried default credentials `administrator`:`radius` and `admin`:`admin` but both did not work. Further searching the GitHub shows another login page under `/app/operators/login.php`, which makes more sense for administrators. 
Navigated to this page and tried using the default credentials and was able to gain access.
![[Pasted image 20260306223645.png]]

Found the `daloRadius` version number - 2.2 beta
![[Pasted image 20260306223736.png]]

Found some database credentials here
![[Pasted image 20260306223942.png]]

Found a user account here
![[Pasted image 20260306224141.png]]

`svcMosh` : ` 412DD4759978ACFCC81DEAB01B382403 `

Username with an MD5 hashed password.

Ran hashcat against the hashed password and cracked it : ` underwaterfriends `
![[Pasted image 20260306224351.png]]

Tried to login to the `/users/login.php` with this but could not.
Tried to SSH as user `svcMosh` with this password and got SSH access.
![[Pasted image 20260306224512.png]]

Enumerated `sudo -l` and found that `mosh-server` can be run as root with no password
Ran the program with `mosh --server="sudo /usr/bin/mosh-server" localhost` and got a root shell. (referred to https://www.hackingdream.net/2020/03/linux-privilege-escalation-techniques.html)
![[Pasted image 20260306225024.png]]

