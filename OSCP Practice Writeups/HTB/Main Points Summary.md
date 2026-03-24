[[Analytics Writeup]]
1. Subdomain (VHOST), credentials in ENV, kernel exploit

[[Bashed Writeup]]
1. Directory fuzzing, `sudo -l`, enumerate `cron` script

[[Boardlight Writeup]]
1. VHOST fuzzing, default credentials, public exploit, credentials in config file, SUID privesc with public exploit

[[Busqueda Writeup]]
1. Public exploit, credentials in git repo, `sudo -l` to find another credential, exploit relative path in script to run commands as `sudo`

[[Codify Writeup]]
1. Public exploit for code library, credentials in `/var/www/`, hashcat, `sudo -l`, vulnerability in bash conditional, `pspy` to view commands from script

[[Cozyhosting Writeup]]
1. Directory fuzzing / error page identification, hijacking session with cookie, command injection using vulnerable webpage input, unzip .jar file to find credentials, privilege escalate with `sudo -l`

[[Devvortex Writeup]]
1. VHOSTs fuzzing, public exploit on CMS to find credentials, upload reverse shell to CMS, enumerate CMS config file for database access, crack hashed password found in database for user access, enumerate `sudo -l`, privilege escalate with public exploit

[[Dog Writeup]]
1. CMS enumeration to find public exploit, exposed git repo enumeration to find password, dictionary attack to find username, public exploit to gain shell, password spraying to gain user access, `sudo -l` for privilege escalation

[[Keeper Writeup]]
1. Default credentials for webpage access, enumerate webpage for user credentials, locate public exploit for KeePass, access KeePass database for root SSH key

[[Knife Writeup]]
1. Enumerate PHP version, locate public exploit for vulnerable PHP version for user access, enumerate `sudo -l` for root access

[[Nibbles Writeup]]
1. Enumerate source code, fuzz directories for CMS version, default credentials to gain CMS admin access, public exploit for user access, enumerate `sudo -l` for root access

[[Precious Writeup]]
1. Enumerate input field, generated PDF with `exiftool` to find software version, public exploit for initial access, search `.bundle` folder for config file with user credentials for user access, enumerate `sudo -l` for `Ruby` script, locate public exploit for `Ruby` version and vulnerable script, root access via script

[[Underpass Writeup]]
1. UDP scan, enumerate SNMP, enumerate software's GitHub page for directories, login with default credentials, enumerate admin dashboard for user credentials, enumerate `sudo -l` for root

[[Builder Writeup]] 
1. Enumerate ports, access Jenkins, locate public exploit for vulnerable version, get user credentials using public exploit, login to Jenkins and find encrypted root SSH key, decrypt root key using Jenkins console, gain root SSH access

[[Editorial Writeup]]
1. Enumerate POST form for SSRF vulnerability, fuzz internal ports, read internal API files, find credentials for SSH, read git commit history to pivot users, enumerate `sudo -l` for vulnerable python script, public exploit for `gitpython` for root access

[[Help Writeup]]
1. Enumerate Node.JS Express application running `GraphQL`, query the site using `GraphQL` queries for application credentials, fuzz directories for application login page, blind SQLi on application for user credentials, SSH access, enumerate kernel version, kernel exploit for root

[[Irked Writeup]]
1. Enumerate high ports for IRC server, enumerate server version, locate public exploit, enumerate user's home directory for credentials, extract credentials from image using steganography tools, gain user access, enumerate SUID binaries, write script to be run with `SUID` binary for privilege escalation with custom binary

[[Jarvis Writeup]]
1. Fuzz directories for `phpMyAdmin` login, enumerate webpage to find SQLi vulnerability, UNION injection to find database credentials, locate public exploit for `phpMyAdmin` gain shell access as `www-data`, `sudo -l` for vulnerable script, command substitution to run arbitrary commands to gain access as user, enumerate `SUID` binaries for privilege escalation with `systemctl`

[[LinkVortex Writeup]]
1. Fuzz site to find CMS,  fuzz VHOSTs to find subdomain, fuzz subdomain to find exposed `git` repository, enumerate `git` repository to find credentials, login to CMS and use public exploit to read config file for credentials, gain user access and enumerate `sudo -l` for script, exploit `symlink` race condition in script to read root SSH key and privilege escalate

[[Magic Writeup]]
1. SQL injection to bypass login, prepend magic bytes to PHP file to bypass file upload checks and gain `www-data` reverse shell, enumerate `/var/www` for database credentials, port forward to my machine to access `mysql`, gain user credentials in database and gain user access, enumerate `SUID` binaries for non-default program, check `strings` to find what commands the program calls, write a reverse shell script named after one of the commands and export its directory to PATH, run the program to gain root access

[[Mentor Writeup]]
1. FUZZ VHOSTs for subdomain, fuzz subdomain directories for API request format, UDP scan to find `SNMP` port, brute force `SNMP` community string with `SNMPBrute` to find credential, send request to API with credential to get JWT session token, use session token to enumerate `admin` subdomains to find command injection vulnerability, gain shell access as `svc`, enumerate app files for database credentials, port forward to access database to find user credentials, SSH access as `svc`, enumerate `SNMP` config files for credentials, lateral movement with credentials, enumerate `sudo -l` for privilege escalation

[[Monitored Writeup]]
1. Enumerate `SNMP` for credentials, fuzz website for API endpoints for generate authentication token, gain access to dashboard, locate public exploit for SQL injection, gain API key from database, create admin user with API key, use dashboard to gain user access, enumerate `sudo -l` for misconfigured script(s), privilege escalated either by abusing `symlinks` or writable service binary

[[Networked Writeup]]
1. Fuzz site to find source code, upload PHP reverse shell for shell access, enumerate user's home directory for `cron` job and script, enumerate script to find command injection vulnerability, gain user shell access, enumerate `sudo -l` for `sudo` script, pass commands into `sudo` script due to a known issue with network scripts in `CentOS` and `Redhat` systems

[[Nineveh Writeup]]
1. Fuzz 80 and 443 to find login pages, fuzz 443 to find directory, download image from directory and run `strings` to find SSH key, enumerate username on port 80 by checking error message then gain access by exploiting PHP `strcmp()` with an array in the password field, or dictionary attack using `hydra` for both login pages, locate public exploit for database on 443 and upload PHP reverse shell, enumerate site on 80 to find LFI and include the reverse shell, gain access as `www-data`, pivot by SSH on localhost with key or enumerate `/etc/knockd.conf` to find port knocking configuration and access port 22 , enumerate `crontab -l` to find `cron` job running, run `pspy` to view root's `cron` job and vulnerable binary, locate public exploit for vulnerable binary and elevate to root

[[OpenAdmin Writeup]]
1. Fuzz directories to find application path, locate public exploit and gain shell access, enumerate config files for credentials to pivot users, enumerate` /var/www` to find internal website, enumerate `sites-enabled` config file or `ss-ntplu` to find internal port, enumerate source code to find hard coded secret, either port forward to access site or craft POST request to read SSH key, crack SSH key password with `john`, gain SSH access as user, enumerate `sudo -l` to privilege escalate

[[Pandora Writeup]]
1. UDP scan for `SNMP`, read credentials from `SNMP`, SSH access and enumerate `/var/www` and `apache` config to find internal site, port forward to gain access to internal site, locate public exploit to gain web shell as user, upload SSH key for SSH access, enumerate `SUID` binaries for executable, enumerate executable using `ltrace` or `cat`, hijack relative path to run own script as `sudo` to launch root shell

[[Pilgrimage Writeup]]
1. Fuzz directories for exposed `git` repo, dump `git `repo and find vulnerable software, use public exploit to dump database, read database for user credentials and gain SSH access, check `ps auxww` to find script being run as root, enumerate script to find vulnerable executable, use public exploit to gain root access

[[Poison Writeup]]
1. Enumerate site to find password file, decode password, find LFI vulnerability and read `/etc/passwd` for username, SSH access as user, unzip zip file with password to get encrypted VNC password, enumerate network ports to find VNC server running, port forward VNC server port, connect with VNC client to gain root access (alternative method for initial access involves log poisoning)

[[Popcorn Writeup]]
1. Enumerate site to find vulnerable web application, locate public exploit to create and upload PHP web shell, gain shell access as `www-data`, enumerate user's home directory for Linux MOTD file in `.cache`, locate public exploit for MOTD and gain root access

[[Sea Writeup]]
1. Fuzz directories for license file and version number, Google the information to find the CMS, locate public exploit for CMS to get shell as `www-data`, enumerate CMS database to get user credentials for pivoting, enumerate network ports to find internal server, port forward internal server to find web application, enumerate application to find command injection vulnerability, upload public key to  root SSH directory for SSH access as root

[[SolidState Writeup]]
1. Enumerate open ports for find email server and version, reset passwords for email users with server, login to POP3 to read emails to find credentials, gain SSH access, locate public exploit for server to open unrestricted shell, run `pspy` to snoop for root `cron` jobs, edit python script being run to get root shell

[[Sunday Writeup]]
1. Enumerate ports to find `finger` service running, dictionary attack on service to enumerate usernames, dictionary attack on SSH to gain access, enumerate `/backup` to find `/etc/shadow` backup and read password hash. crack hash and get user access, enumerate `sudo -l` to find `wget` with root permissions, privilege escalate using `wget`

[[SwagShop Writeup]]
1. Enumerate site to find ecommerce software, enumerate software to find version, locate public exploits to gain shell access as `www-data`, enumerate `sudo -l` for privilege escalation with `vi`

[[Tabby Writeup]]
1. Enumerate site to find LFI, find `tomcat-users.xml` file path and include it, use credentials to upload reverse shell to `tomcat` and get shell access, exfiltrate and brute force a zip file to get password, use password for user access, enumerate groups for `lxd` membership, privilege escalate using `lxd`

[[TartarSauce Writeup]]
1. Fuzz site to find `WordPress` directory, enumerate to find vulnerable plugin, public exploit for shell access as `www-data`, `sudo -l` for user access, `systemctl list-timers` to find vulnerable service binary, enumerate script to find vulnerability in running `tar` as root, abuse `tar` maintaining `SUID` permissions within archive when extracting as root for privilege escalation

[[Usage Writeup]]
1. Enumerate site for blind SQL injection, `SQLmap` to dump database to find admin password, locate public exploit for admin dashboard for shell access, read files in home directory to pivot, `sudo -l` for vulnerable binary, abuse wildcard in `7za` command for privilege escalation

[[UpDown Writeup]]
1. Fuzz directories for `git` repo, fuzz subdomains for dev site, enumerate `git` repo for required headers for access to dev site, access dev site and upload PHAR file for shell access, enumerate `SUID` binary and abuse python2's `input` function for code execution, enumerate `sudo -l` to find `easy_install` python script for privilege escalation