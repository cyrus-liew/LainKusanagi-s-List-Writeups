
---

## 1. AS‑REP Roasting

**What it does:** Requests a crackable Kerberos AS-REP for users with **`Do not require Kerberos preauthentication`** set.

### When it works
- You have **no credentials** but have **a list of usernames** (or can guess).
- Domain controller is reachable (Kerberos port 88).
- Target user accounts have `DONT_REQ_PREAUTH` flag.

### How to detect
```bash
# Check without creds
nxc ldap <DC> -u '' -p '' --asreproast output.txt
# Or with low-priv creds
nxc ldap <DC> -u user -p pass --asreproast output.txt
```

### Pro tip
- **Prioritise AS-REP roasting before password spraying** – it’s silent and gives a hash you can crack offline.
- **Look for accounts with `UF_DONT_REQUIRE_PREAUTH`** in BloodHound (AS-REP Roastable edge).

---

## 2. Kerberoasting

**What it does:** Requests a service ticket (TGS) for any service account with a **Service Principal Name (SPN)** and cracks the NTLM hash of the service account.

### When it works
- You have **any valid domain credentials** (lowest privilege works).
- The domain has service accounts (e.g., MSSQL, HTTP, CIFS) with SPNs.
- Service account passwords are weak or crackable.

### How to detect
```bash
# Using credentials
nxc ldap <DC> -u user -p pass --kerberoasting output.txt
impacket-GetUserSPNs <domain>/user:pass -request -dc-ip <DC>
```

### Pro tip
- **Works even if the service account is a normal user** with an SPN.
- **Look for high-value targets** like accounts that are Domain Admins or have local admin on many machines (BloodHound: `Kerberoastable` edge).
- **Use `-outputformat john`** for faster cracking with John.

---

## 3. Password Spraying

**What it does:** Tries one common password (e.g., `Season2025!`) against many usernames to avoid account lockout.

### When it works
- You have **a list of valid usernames** (e.g., from enumeration).
- Domain **does not have smart lockout** or lockout threshold is high.
- You know a likely password (season, company name, default password).

### How to detect (without lockout risk)
```bash
# Test a single password across many users
nxc smb <DC> -u users.txt -p 'Winter2024!' --continue-on-success
```

### Pro tip
- **Start with `-p ''` or `-p 'Password123'`** (common misconfigurations).
- **Use `--no-bruteforce`** in nxc to respect lockout policies.
- **After finding a valid password**, try it on **all other services** (WinRM, RDP, SSH).

---

## 4. Pass the Hash (PtH)

**What it does:** Uses an NTLM hash (instead of plaintext password) to authenticate over SMB, WMI, or WinRM.

### When it works
- You have **an NTLM hash** of a user (from `secretsdump`, `mimikatz`, etc.).
- Target accepts NTLM authentication (SMB/WinRM – default).
- User is **not** a protected user (e.g., not in `Protected Users` group) – but often still works.

### When to use it
- You dumped hashes from a compromised machine (local admin).
- You cannot crack the hash quickly.
- You want to move laterally to a machine where the user has administrative privileges.

### Pro tip
- **Use with `psexec` or `nxc smb`** – easiest path.
- **WinRM often works** if the user is in `Remote Management Users`.
- **Does NOT work** if NTLM is disabled or target requires Kerberos (rare).

---

## 5. Pass the Ticket (PtT)

**What it does:** Injects a Kerberos ticket (TGT or service ticket) into your current session to impersonate a user.

### When it works
- You have **extracted a ticket** from memory (e.g., `sekurlsa::tickets` in mimikatz).
- You are **on a Windows machine** where you can run mimikatz or Rubeus.
- The ticket is **still valid** (lifetime up to 10 hours for TGT, 24h for service tickets).
- The target service **accepts Kerberos authentication** (most AD services do).

### When to use it
- You have admin on a machine and want to move laterally **without knowing the password or hash**.
- You want to **impersonate a Domain Admin** whose ticket is present on the current machine.
- You are in a **constrained environment** where NTLM is blocked.

### Pro tip
- **Export all tickets**: `mimikatz "sekurlsa::tickets /export"`
- **Inject a TGT** to get access to any service: `kerberos::ptt krbtgt.kirbi`
- **Use Rubeus** for easier ticket management: `Rubeus.exe ptt /ticket:ticket.kirbi`

---

## 6. Over Pass the Hash (OverPass-the-Hash)

**What it does:** Uses an NTLM hash or plaintext password to **request a TGT** from the KDC, then uses that TGT to get service tickets.

### When it works
- You have **an NTLM hash or plaintext password** of a user.
- The user **does not have** the `"Do not require Kerberos preauthentication"` flag (unlike AS-REP).
- You can run tools like `impacket-getTGT` or Rubeus (on Windows).

### When to use it
- You have a hash but **cannot Pass-the-Hash** because the target service requires Kerberos (e.g., HTTP, LDAP).
- You want to **avoid NTLM authentication** for stealth or policy reasons.
- You need to **request service tickets for specific SPNs** (e.g., CIFS for file access).

### Pro tip
```bash
# Request TGT from hash
impacket-getTGT <domain>/<user> -hashes :<ntlm_hash> -dc-ip <DC>
export KRB5CCNAME=<user>.ccache
# Use any impacket tool with -k
impacket-psexec -k <domain>/<user>@<target>
```
- **On Windows** with Rubeus: `Rubeus.exe asktgt /user:user /domain:domain /rc4:<ntlm_hash> /ptt`

---

## 7. DCSync

**What it does:** Mimics a domain controller and requests replication of password hashes (including krbtgt) from a real DC.

### When it works
- You have **credentials of a user** that has **Replicating Directory Changes** rights (usually Domain Admins, Enterprise Admins, or custom ACL).
- You can connect to the DC from a machine (Kali or Windows) with network access.

### When to use it
- You compromised a **Domain Admin** account (or any account with DCSync rights).
- You want to **dump all domain hashes** without touching the DC’s disk.
- You need to **create a golden ticket** (requires krbtgt hash).

### Pro tip
- **Check BloodHound for `DCSync` edges** – users who can DCSync.
- **If you have WriteDACL on a user**, you can grant DCSync rights to yourself: `impacket-dacledit -action write -rights DCSync ...`
- **Use `-just-dc`** in secretsdump to only dump NTDS.

---

## 8. Golden Ticket

**What it does:** Forges a TGT using the **krbtgt hash** (which never changes unless rotated). Grants unlimited access to the domain.

### When it works
- You have **the krbtgt NTLM hash** (from DCSync or ntds.dit).
- You know the **domain SID** (can be obtained from any user’s SID).
- The domain **has not rotated krbtgt password** twice (which invalidates all existing tickets).

### When to use it
- You have **full domain compromise** and want to **maintain persistence**.
- You need **access to any machine** in the domain, even if credentials change.
- The DC is not available for direct attack (e.g., firewalled), but you can forge tickets offline.

### Pro tip
- **Create golden ticket with mimikatz**:
  ```cmd
  mimikatz "kerberos::golden /domain:<domain> /sid:<domain_sid> /krbtgt:<hash> /user:Administrator /ticket:golden.kirbi"
  ```
- **Use it with Rubeus or impacket** to access any resource.
- **Rotate krbtgt twice** to defend against this (but very rare in exams).

---

## 9. Silver Ticket

**What it does:** Forges a **service ticket** (TGS) for a specific service using that service account’s NTLM hash.

### When it works
- You have **the NTLM hash of a service account** (e.g., MSSQL, CIFS, HTTP).
- You know the target service FQDN and SPN type.

### When to use it
- You **cannot get krbtgt hash** but have a service account hash.
- You only need **access to one specific service** (e.g., CIFS on a file server).
- You want to **avoid generating DC logs** (silver tickets are only validated by the target service).

### Pro tip
- **Example: CIFS service on a file server** – allows you to access `\\fileserver\share` without any DC contact.
- **Limited scope** – only works for the service you forged (e.g., HTTP ticket won’t work for CIFS).
- **Command**:
  ```cmd
  mimikatz "kerberos::golden /domain:<domain> /sid:<sid> /target:<target_FQDN> /service:CIFS /rc4:<service_hash> /user:any /ptt"
  ```

---

## 10. ACL Abuse (GenericAll, GenericWrite, WriteDACL, etc.)

### GenericAll on a User

**What it does:** Allows you to change the user’s password, add a SPN (kerberoast), or add to a group.

**When to use:** You have `GenericAll` over a user (BloodHound edge).  
**Result:** Reset password → log in as that user. Or add SPN → Kerberoast.

### GenericWrite on a User

**What it does:** Allows you to modify attributes (like `servicePrincipalName`) of the target user.

**When to use:** You have `GenericWrite` over a user.  
**Result:** Add an SPN to that user → Kerberoast → crack hash.

### WriteDACL / WriteOwner on a Group or User

**What it does:** Allows you to modify ACLs – e.g., grant yourself `DCSync` rights.

**When to use:** You have these rights on a privileged group or the domain object itself.  
**Result:** Escalate to full domain compromise.

### ForceChangePassword

**What it does:** Exactly that – reset a user’s password without knowing current password.

**When to use:** You have `ForceChangePassword` on a privileged user.  
**Result:** Take over their account.

---

## 11. Delegation Attacks (Unconstrained, Constrained, RBCD)

### Unconstrained Delegation

**When it works:** A machine (or user) has `Trusted for Delegation` set.  
**Attack:** Trick the target to authenticate to you (e.g., via printer bug or admin connecting) → capture their TGT → impersonate them anywhere.

### Constrained Delegation (S4U2Self / S4U2Proxy)

**When it works:** A service account has `Trusted for Delegation` to specific SPNs.  
**Attack:** Request a service ticket for any user to a permitted SPN (e.g., CIFS on a file server).  
📌 **No need for the user’s password or hash.**

### Resource-Based Constrained Delegation (RBCD)

**When it works:** You have `Write` access to a target machine’s `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute.  
💡 **Attack:** Make that machine trust your controlled computer → request a ticket for any user to access the target.

---

## 12. Net‑NTLMv2 Relay

**What it does:** Captures an NTLMv2 handshake from one machine and relays it to another to authenticate.

### When it works
- Network **does not enforce SMB signing** (or you can downgrade).
- You can **poison or spoof** traffic (e.g., via LLMNR/NBT-NS or mitm6).
- Target service **accepts NTLM authentication**.

### When to use it
- You are on the **same network segment** as victims.
- You cannot get credentials otherwise.
- You want to **escalate to a domain admin** by relaying to a DC (e.g., to change a user’s password).

---

## Quick Reference Table

| Attack | Prerequisite | No Creds? | Needs User List? | Needs Hash? | Needs Ticket? | Best When |
|--------|--------------|-----------|------------------|-------------|---------------|------------|
| AS-REP Roasting | User list | ✅ | ✅ | ❌ | ❌ | Initial recon, no creds |
| Kerberoasting | Any valid creds | ❌ | ❌ | ❌ | ❌ | You have low-priv creds |
| Password Spray | User list | ✅ | ✅ | ❌ | ❌ | You know a common password |
| Pass the Hash | NTLM hash | ❌ | ❌ | ✅ | ❌ | Target accepts NTLM |
| Pass the Ticket | Ticket file | ❌ | ❌ | ❌ | ✅ | You have tickets in memory |
| Over Pass the Hash | NTLM hash | ❌ | ❌ | ✅ | ❌ (but generates TGT) | Kerberos-only services |
| DCSync | Replication rights | ❌ | ❌ | ❌ | ❌ | You have high privilege |
| Golden Ticket | krbtgt hash | ❌ | ❌ | ✅ | ❌ | Persistence / full access |
| Silver Ticket | Service hash | ❌ | ❌ | ✅ | ❌ | Single service access |
| ACL Abuse | Specific ACL on object | ❌ | ❌ | ❌ | ❌ | BloodHound shows edge |
| Unconstrained Delegation | Delegated machine | ❌ | ❌ | ❌ | ❌ | Admin connects to your machine |
| RBCD | Write on target | ❌ | ❌ | ❌ | ❌ | You control a computer object |

---

## Decision Flow (Simplified)

```
Start
 │
 ├─ Have user list but no creds?
 │   └─ Try AS-REP Roasting → if no success → Password Spray
 │
 ├─ Have low-priv domain creds?
 │   └─ Kerberoast → BloodHound → Look for ACL abuse
 │
 ├─ Have local admin on a machine?
 │   └─ Dump hashes (secretsdump) → Pass the Hash or OverPass
 │   └─ Dump tickets (mimikatz) → Pass the Ticket
 │
 ├─ Have NTLM hash but can't PtH?
 │   └─ OverPass the Hash → get TGT → use impacket -k
 │
 ├─ Have krbtgt hash?
 │   └─ Golden Ticket → own everything
 │
 ├─ Have service account hash?
 │   └─ Silver Ticket for that specific service
 │
 ├─ Have WriteDACL/GenericAll on a user?
 │   └─ Reset password / Kerberoast / DCSync (if on DC)
 │
 └─ On same subnet with no creds?
     └─ Responder / mitm6 → Net-NTLM relay
```

---
