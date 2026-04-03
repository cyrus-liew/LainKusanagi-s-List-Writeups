Main steps
1. 

Learning points
1. GPP passwords - `gpp-decrypt`

`nmap`
![[Pasted image 20260330220604.png]]

Common DC ports are open. Found a domain name, `active.htb`.

Enumerated anonymous SMB shares, and found a share named `Replication`.
![[Pasted image 20260330220819.png]]

Looking through all the files in `Replication`, I found a file named `Groups.xml` that contained credentials for the account `SVC_TGS`.
![[Pasted image 20260330235302.png]]

`active.htb\SVC_TGS`:` edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ `

This is an encrypted password that is associated with Group Policy Preference (GPP). The password is encrypted with AES, but Microsoft published the key here https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-gppref/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be?redirectedfrom=MSDN, and there are tools on Kali that will automatically decrypt such passwords.

![[Pasted image 20260330235803.png]]

` GPPstillStandingStrong2k18 `

Checked this password on SMB and was able to get access.
![[Pasted image 20260330235902.png]]

This allows me to read the `Users` share and get the user flag.
![[Pasted image 20260331000039.png]]

Ran `impacket-GetUserSPNs` on the server with these credentials to try Kerberoasting, and managed to extract the `Administrator` hash.
![[Pasted image 20260331000250.png]]

Ran this hash through `hashcat -m 13100` and cracked the hash.
`administrator`:` Ticketmaster1968 `

With these credentials, I could either read the root flag from the SMB shares like I did above, or use `impacket-psexec` to gain a shell as `nt authority\system`, like below.
![[Pasted image 20260331000625.png]]
