Main steps
1. 

Learning points
1. Check usernames with `net user` to make sure they exist

`nmap`
![[Pasted image 20260331221202.png]]

Web server on port 80, standard DC ports, `WinRM` port open.

On the website, there is an `about` page that contains a few names - possibly staff working at the company.
![[Pasted image 20260331221537.png]]

Tried enumerating users with `rpcclient` and `nxc smb` but all returned nothing. Found the domain name: `EGOTISTICAL-BANK.LOCAL`.
![[Pasted image 20260331222153.png]]

Used this tool https://github.com/jseidl/usernamer to generate possible username combinations from the list of possible users that I found on the website, then ran `impacket-GetNPUsers` to try to AS-REP roast them.

Found that `fsmith` was a actual username with pre authentication disabled, and retrieved their hash.
![[Pasted image 20260331222522.png]]

Ran the hash through `hashcat` and cracked it. 

`fsmith`: ` Thestrokes23 `

Checked the account for `WinRM` access and was successful.
![[Pasted image 20260331222731.png]]

Got shell access with `evil-winrm` and got the user flag.

Uploaded `winpeas` and ran the program, and found some `AutoLogon` credentials.
`EGOTISTICALBANK\svc_loanmanager`: ` Moneymakestheworldgoround! `
![[Pasted image 20260331223444.png]]

Uploaded `SharpHound` and collected the domain data. Looking at the permissions `svc_loanmanager` has, I noticed that it has `DCSync` over the domain.
![[Pasted image 20260331224319.png]]

Tried running a `DCSync` attack a few times but it kept failing due to logon failure. Going back to my shell, I ran `net user`and found out that there is no user named `svc_loanmanager`. However, there is one named `svc_loanmgr` which is quite close, so I tried again with this username.
![[Pasted image 20260331225028.png]]

This time, the `DCSync` attack works and I dumped all the NTLM hashes for the machine.
![[Pasted image 20260331225122.png]]

` aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e `

Used this hash with `impacket-psexec` and got access as `nt authority\system` and got the root flag.
![[Pasted image 20260331225252.png]]