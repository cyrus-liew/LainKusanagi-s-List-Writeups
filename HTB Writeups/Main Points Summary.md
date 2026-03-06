[[Analytics Writeup]]
1. Subdomain (VHOST), credentials in ENV, kernel exploit

[[Bashed Writeup]]
1. Directory fuzzing, `sudo -l`, enumerate `cron` script

[[Boardlight Writeup]]
1. VHOST fuzzing, default credentials, public exploit, credentials in config file, SUID privesc with public exploit

[[Busqueda Writeup]]
1. Public exploit, credentials in git repo, `sudo -l` to find another credential, exploit relative path in script to run commands as `sudo`

[[Codify Writeup]]
1. Public exploit for code library, credentials in `/var/www/`, hashcat, sudo -l, vulnerability in bash conditional, `pspy` to view commands from script

[[Cozyhosting Writeup]]
1. Directory fuzzing / error page identification, hijacking session with cookie, command injection using vulnerable webpage input, unzip .jar file to find credentials, privilege escalate with `sudo -l`

[[Devvortex Writeup]]
1. VHOSTs fuzzing, public exploit on CMS to find credentials, upload reverse shell to CMS, enumerate CMS config file for database access, crack hashed password found in database for user access, enumerate `sudo -l`, privilege escalate with public exploit

[[Dog Writeup]]
1. CMS enumeration to find public exploit, exposed git repo enumeration to find password, dictionary attack to find username, public exploit to gain shell, password spraying to gain user access, `sudo -l` for privilege escalation

[[Keeper Writeup]]
1. Default credentials for webpage access, enumerate webpage for user credentials, locate public exploit for KeePass, access KeePass database for root SSH key