Main steps
1. 

Learning points
1. `LAPS` querying
2. SSL cert extraction with `openssl`
3. PowerShell history filepath

`nmap`
![[Pasted image 20260408175723.png]]

DC ports open, port 5986 - `WinRM` over HTTPS open.

The SSL cert on port 5986 shows that the server's hostname is `dc01.timelapse.htb`.

Ran a few tools to try enumerating users, but was unsuccessful. However, using the `guest` account with SMB did not return an error, so the account may be enabled.
![[Pasted image 20260408180241.png]]

Tried enumerating shares with the `guest`account and was able to read them.
![[Pasted image 20260408180317.png]]

`Shares` is not a default share, so I'll start here. Found 2 folders, each containing some files - a ZIP file, Word documents, and a `msi` installer.
![[Pasted image 20260408180529.png]]

Downloaded the ZIP file to check it first.
![[Pasted image 20260408180642.png]]

The file required a password to unzip, so I converted it to a hash using `zip2john` and ran `john` to try cracking the hash. Successfully cracked the hash: ` supremelegacy `.
![[Pasted image 20260408180729.png]]

The unzipped folder gave me a `.pfx` file. This is a password protected file used to store SSL certs or private keys.
![[Pasted image 20260408180818.png]]

Converted the file to a hash with `pfx2john` and cracked it with `john` successfully. 
` thuglegacy `
![[Pasted image 20260408181059.png]]

Extracted the information in the file with `openssl`, and got a `key` file.
![[Pasted image 20260408182757.png]]

Based on the information inside the cert, the user's name is `Legacyy`.
![[Pasted image 20260408181557.png]]

Dumped the SSL cert with `openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out legacyy.crt`.
![[Pasted image 20260408182714.png]]

With the 2 files, I was able to get access via `WinR` over SSL.
`evil-winrm -i 10.129.227.113 -S -c legacyy.crt -k legacyy.key -u Legacyy -P 5986`
![[Pasted image 20260408182935.png]]

From the SMB shares earlier, I know that `LAPS` is probably in use on this DC. This is a tool used to manage a system where administrator passwords are applied to domain joined computers. Referring to https://hacktricks.wiki/en/windows-hardening/active-directory-methodology/laps.html.

Checked if `LAPS` is activated by querying the registry.
![[Pasted image 20260408183216.png]]

Checked the user's PowerShell history at `C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_History.txt`, and found some credentials for the `svc_deploy` account.
![[Pasted image 20260408183456.png]]

`svc_deploy` :` E3R$Q62^12p7PLlC%KWaxuaV `

I was able to `evil-winrm` with these credentials as `svc_deploy`, and noticed that this user is part of the `LAPS_Readers` domain group.
![[Pasted image 20260408183632.png]]
![[Pasted image 20260408183636.png]]

From HackTricks:
![[Pasted image 20260408184135.png]]

If I can query the `ms-mcs-AdmPwd` attribute in the domain computer's objects, I should be able to read the administrator password.

This can be done using the inbuilt PowerShell cmdlet, `Get-ADComputer.`
![[Pasted image 20260408184320.png]]

The randomized password for the administrator account is `H0(B$VLKY,OY(t2llI;dZ]S7`.

Got access as `administrator` using the password and retrieved the root flag from the user `TRX`'s desktop.
![[Pasted image 20260408184528.png]]