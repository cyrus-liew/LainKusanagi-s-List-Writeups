Main steps:
1. Enumerate webpage to find a subdomain
2. Locate public exploit for backend software for initial access
3. Enumerate ENV variables for credentials
4. Gained SSH access using credentials
5. Located public exploit for Linux kernel for root access

Learning points:
2. Locating public exploits for both initial access and root privilege escalation (rmb to always check software/ kernel versions as there may be very common exploits)
3. Locating user credentials in the environment

Enumeration
![[Pasted image 20260227163232.png]]
Initial nmap scan shows port 22, 80 open
Navigating to IP shows the FQDN 'analytical.htb', added to hosts files
Static webpage, checked source code, only one input field that doesn't seem to send any requests
'Login' button redirects to 'data.analytical.htb', added to hosts file
Site shows a Metabase login page
![[Pasted image 20260227163653.png]]
Checked metabase on ExploitDB and found a Pre-Auth RCE
Downloaded and ran the python script and was able to get a shell
![[Pasted image 20260227165821.png]]
![[Pasted image 20260227170707.png]]
Found some info in env,
	META_USER=metalytics
	META_PASS=An4lytics_ds20223#
From ps aux, the command used to run metabase:
	java -XX:+IgnoreUnrecognizedVMOptions -Dfile.encoding=UTF-8 -Dlogfile.path=target/log -XX:+CrashOnOutOfMemoryError -server -jar /app/metabase.jar

Tried to SSH using the credentials found and was able to access the 'metalytics' user account
![[Pasted image 20260227175002.png]]
User access achieved.

Ran linpeas, lse; no misconfigurations detected
Looked for any readable config files/ text files for credentials, nothing found
Check Linux kernel version: 6.2.0-25-generic #25~22.04.2-Ubuntu - vulnerable to  GameOver(lay) (CVE-2023-2640, and CVE-2023-32629)
Downloaded a script from https://github.com/luanoliveira350/GameOverlayFS and ran it on the victim machine (edited the script to launch a root shell with sudo bash)
![[Pasted image 20260227190406.png]]
Root access achieved