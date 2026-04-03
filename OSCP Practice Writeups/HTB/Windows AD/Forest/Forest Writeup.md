Main steps
1. 

Learning points
1. LDAP may not give all users, so try with a few different tools like `rpcclient` etc. in the list to make sure all users are found

`nmap`
![[Pasted image 20260331001911.png]]

Standard DC ports open, `WinRM` port open.

Anonymous and guest SMB access disabled.
![[Pasted image 20260331002535.png]]

Tried enumerating LDAP for users and found some usernames.
![[Pasted image 20260331002600.png]]

`sebastien`
`lucinda`
`andy`
`mark`
`santi`

With `rpcclient -U '' -N 10.129.220.98`, then `enumdomusers`, I managed to find 1 more user that did not show up with `nxc`: `svc-alfresco`.
![[Pasted image 20260331004346.png]]

Tried AS-REP roasting with `impacket-GetNPUsers htb.local/ -usersfile user.txt -dc-ip 10.129.220.98 -format hashcat > hash.txt` and got the hash for `svc-alfresco`.
![[Pasted image 20260331004451.png]]

Ran the hash through `hashcat -m 18200` and cracked it: `s3rvice`.

Tried running these credentials against SMB and `WinRM` and got access, and got the user flag.
![[Pasted image 20260331010521.png]]

Ran `SharpHound` on the machine to collect data about the AD.

From `BloodHound`, I see that `svc-alfresco` is part of the `Account Operators` group.
![[Pasted image 20260331215121.png]]

This group is a privileged group that allows members to create and modify users and add them to non-protected groups.

Looking at the Shortest Path to High Value Targets query, I can see that the `Account Operators` group has `GenericAll` permissions on the `Exchange Windows Permissions` group, which in turn has `WriteDacl` permissions on the domain `HTB.LOCAL`. This permission allows users to add ACLs to an object, which allows us to add a user to the groups and give them `DCSync` privileges.
![[Pasted image 20260331215424.png]]

So the privilege escalation path here would be to:
1. Create a new domain user
2. Add the user to the `Exchange Windows Permissions` and `Remote Management Users` groups
3. Upload the `PowerView.ps1` script and use the `Add-ObjectACL` command with the new user's credentials to give them `DCSync` permissions
4. Run `impacket-secretsdump` with the newly created account to dump the NTLM hashes for all domain users

![[Pasted image 20260331220108.png]]![[Pasted image 20260331220125.png]]

And the hashes for all users are successfully dumped.
```
htb.local\Administrator
aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
```

With this hash, I was able to get a shell using `impacket-psexec` as `nt authority\system` and get the root flag.
![[Pasted image 20260331220447.png]]

