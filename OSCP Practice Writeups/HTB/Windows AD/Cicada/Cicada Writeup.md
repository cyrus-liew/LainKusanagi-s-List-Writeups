Main steps
1. 

Learning points
1. Check SMB, LDAP, `WinRM` with `nxc`, spray passwords found
2. Privesc using backup operators account

`nmap`
![[Pasted image 20260330210640.png]]

Most of the standard DC ports open, (53, 88, 135,139, 389, 445 etc.) and 5985 `WinRM` open.

Common name of the server is `CICADA-DC.cicada.htb`.

Enumerated the SMB shares of the server. Anonymous access returned nothing, but `guest` access showed a few non default shares.
![[Pasted image 20260330211031.png]]

Connected to the HR share that `guest` has read access with `smbclient -N //10.129.231.149/HR`, and found a `Notice from HR.txt` file. Downloaded it with `get`and opened it.
![[Pasted image 20260330212204.png]]![[Pasted image 20260330212212.png]]

The text file contains a default password, along with instructions to change it. The default password is: ` Cicada$M6Corpb*@Lp#nZp!8 `

Used `nxc`'s `--rid-brute` to brute force users through SMB with `nxc smb 10.129.231.149 -u guest -p '' --rid-brute`.
![[Pasted image 20260330212542.png]]

Other than the admin and default users, there are 5 usernames here:
`john.smoulder`
`sarah.dantelia`
`michael.wrightson`
`david.orelious`
`emily.oscars`

Sprayed the default password found earlier with these usernames and found that `michael.wrightson` did not change their password.
![[Pasted image 20260330212845.png]]

Since the `WinRM` port is open, tried logging in with that, but was unsuccessful.
![[Pasted image 20260330213030.png]]

Checked SMB again with this user, but there are no new shares available.
![[Pasted image 20260330213131.png]]

Tried LDAP with these credentials, and was successful. With the `--users` flag in `nxc`, I can enumerate the whole user list with credentials, and found an interesting description under one of the users.
![[Pasted image 20260330213316.png]]

`david.orelious`: ` aRt$Lp#7t*VQ!3 `

Once again, this user is unable to `WinRM`. However, they do have access to the `DEV` SMB share.
![[Pasted image 20260330213452.png]]

There is a PowerShell script in the `DEV` share, which I can download.
![[Pasted image 20260330213629.png]]

This script contains credentials for the user `emily.oscars`.
![[Pasted image 20260330213645.png]]

`emily.oscars`:` Q!3@Lp#M6b*7t*Vt `

The credentials work for `WinRM`.
![[Pasted image 20260330213746.png]]

Got a shell using `evil-winrm` as `emily.oscars` and found the user flag.
![[Pasted image 20260330213922.png]]

Checked `whoami /all` and found that `emily.oscars` is part of the `BUILTIN\Backup Operators` group, with the `SeBackupPrivvilege` privilege. 
![[Pasted image 20260330213958.png]]

Users in this group are able to backup and restore any file on the computer, including dumping registry hives to read password hashes.

Used `reg save` to copy the SYSTEM and SAM hives and downloaded them to my machine.
![[Pasted image 20260330215125.png]]

Ran `impacket-secretsdump -system system -sam sam LOCAL` with the hives and extracted the administrator hash.
`aad3b435b51404eeaad3b435b51404ee:2b87e7c93a3e8a0ea4a581937016f341`
![[Pasted image 20260330215211.png]]

Checked the hash and found that it was possible to `WinRM` as `Administrator` with the hash.
![[Pasted image 20260330215408.png]]

Got a shell as `Administrator` with `evil-winrm`, and got the root flag.
![[Pasted image 20260330215525.png]]

