Main steps:
1. Enumerate IP and gain access via default credentials
2. Locate public exploit for vulnerable software and gain shell access
3. Enumerate sudo permissions
4. Gain root access through sudo configuration (NGINX)

Learning points:
1. Google is good
   
Nmap
![[Pasted image 20260303182434.png]]
Initial enumeration shows multiple ports open:
2. 22 (SSH)
3. 80 (HTTP)
4. 1883 (MQTT)
5. 5672 (AMQP?)
6. 8161 (HTTP: Jetty 9.4.39.v20210325)
7. 40901 (tcpwrapped)
8. 61613 (stomp, Apache ActiveMQ)
9. 61614 (HTTP: Jetty 9.4.39.v20210325)
10. 61616 (apachemq)

HTTP authentication on port 80, port 8161
![[Pasted image 20260303182334.png]]
Nothing returned on port 61614
![[Pasted image 20260303183048.png]]

Googling the Jetty version shows that it possibly has an information disclosure vulnerability (CVE-2021-28164) - attempting this does not result in any output

Tried accessing the site using default credentials admin:admin, was able to access both port 80 and 8161. Both lead to an ActiveMQ admin page.
![[Pasted image 20260303183808.png]]
![[Pasted image 20260303183923.png]]
This version of ActiveMQ (5.15.15) is vulnerable to an RCE (CVE-2023-46604)
Found a POC on https://github.com/strikoder/CVE-2023-46604-ActiveMQ-RCE-Python/tree/main and ran it

Was able to gain access as the 'activemq' user
![[Pasted image 20260303185043.png]]
Got user flag under /home/activemq

Ran linpeas and saw that 'activemq' is allowed to run nginx as sudo with NOPASSWD
![[Pasted image 20260303185742.png]]

Found an NGINX privilege escalation script here https://github.com/DylanGrl/nginx_sudo_privesc
The script creates an NGINX configuration file which launches a HTTP , then runs NGINX as sudo with this file, generates an SSH key, and uses the NGINX HTTP server to upload this key into the root authorized keys file.
![[Pasted image 20260303191405.png]]

Gained a root SSH session using this script.
![[Pasted image 20260303191441.png]]
