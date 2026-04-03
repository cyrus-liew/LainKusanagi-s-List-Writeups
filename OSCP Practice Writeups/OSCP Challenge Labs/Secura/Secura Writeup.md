Main steps
1. 

Learning points
1. `idb` and `frm` files can be read with `Notepad++` and contain database information
2. `RDCMan` can contain saved credentials - open RDP to check, or check the files from `WinPEAs` or `Seatbelt`
3. Using `SharpGPOAbuse` to exploit GPO write permissions for privilege escalation

Assumed breach scenario, credentials: ` Eric.Wallows `:` EricLikesRunning800 `

Machine IP addresses:
1. ` 192.168.120.95 ` - breached
2. ` 192.168.120.96 `
3. ` 192.168.120.97 `

**Reconnaissance**
`nmap`
`192.168.120.95`:
Ports open include standard AD ports - `rpc`, `SMB`, `NetBios`; `WinRM` port open, `RDP` port open; there is a HTTP server on port `8443`, and another HTTP application on 44444 and 47001. There are also some other high ports open, like 62950 and 62951, which `nmap` failed to fingerprint.

`192.168.120.96`:
Standard AD ports open, port 3306 `MariaDB` open, `WinRM` open, `47001` open with a HTTPAPI.

`192.168.120.97`:
Standard DC ports open, `WinRM` open.

Web server for `192.168.120.95`:
Port 8443 leads to a login page for `ManageEngine Applications Manager`. There is a build number, 14710, on the main page.
![[Pasted image 20260401194207.png]]

Clicking on the `First Time User?` button opens a popup that gives the `admin`:`admin` credentials.
![[Pasted image 20260401194318.png]]

I successfully logged in with these credentials.
![[Pasted image 20260401194357.png]]

Searching up this build number for `ManageEngine` tells me that it is vulnerable to [CVE-2020-14008](https://nvd.nist.gov/vuln/detail/CVE-2020-14008), an authenticated RCE, where an admin user can upload a malicious JAR file and execute arbitrary Java code.

Found a POC for this exploit on [ExploitDB](https://www.exploit-db.com/exploits/48793)and downloaded it to try. I had to edit the exploit slight since it was trying to compile the JAR with Java 7. Changed this to Java 8 and it worked, and got a shell as `nt authority\system`.
![[Pasted image 20260401202147.png]]
![[Pasted image 20260401202154.png]]

Uploaded `SharpHound` to the machine and ran it to collect AD data.

Since the user `Eric.Wallows` has `WinRM` access, I used `evil-winrm` to exfiltrate the `BloodHound` ZIP file.
![[Pasted image 20260401202913.png]]

Checking the users on the domain shows two other users, `charlotte` and `michael`.
![[Pasted image 20260401205042.png]]

From `BloodHound`, the user `michael` does not seem to have any interesting permissions. However, `charlotte` is a `WriteOwner` of the `Default Domain Policy` GPO, and can `GenericWrite` and `WriteDacl`.
![[Pasted image 20260401205311.png]]

With these permissions, `charlotte` can grant themselves `GenericAll` on the GPO. `charlotte` also has `PSRemote` privileges on the `DC01`.

Found `AutoLogon` credentials for `administrator` with `WinPEAs`.
![[Pasted image 20260401212902.png]]

`Reality2Show4!.?`

`WinPEAs` also found a saved RDP connection to `era.secura.local`.
![[Pasted image 20260401213223.png]]

While there were no `.RDG` files in the `RDCMan` folder, the `RDCMan.settings` file under `Administrator` contained some plaintext credentials. 
Note: This file is not automatically checked by `WinPEAs`, and will need to be manually checked.
![[Pasted image 20260401211851.png]]

These credentials are for connecting to a machine named `era.secure.yzx` as `SECURE\apache`:` New2Era4.! `.

Running `nslookup` on this domain name shows that this is the `192.168.120.96` machine.
![[Pasted image 20260401214034.png]]

From the `nmap` scan I did previously, I know that this machine does not have its RDP port open, so I will not be able to directly RDP in.

Tried enumerating the credentials against `era.secura.yzx` but kept failing. Tried spraying the passwords against all the usernames I found earlier and also failed. (Ran it against both SMB and `WinRM`.)
![[Pasted image 20260401220435.png]]

Looking back at the credentials, I noticed that the saved connection name is `local`, meaning that this could be a local account and not a domain account. Tried enumerating the credentials again with `nxc`, but this time with the `--local-auth` flag, and found that there is a local account `apache` on `ERA`, with `WinRM` access.
![[Pasted image 20260401220715.png]]

Got a shell using `evil-winrm` as the local user `apache` on `ERA`.
![[Pasted image 20260401221142.png]]

Uploaded an ran `WinPEAs` to this machine, and found some potential DLL hijacking vectors.
![[Pasted image 20260401222006.png]]
![[Pasted image 20260401222018.png]]

Also found some default `xampp` credentials.
![[Pasted image 20260401223535.png]]

Looking deeper into the `xampp` folder, I found a `creds.ibd` file under `mysql\data\creds`. This file contains data and indexes for a specific table in `MySQL`.

I tried to exfiltrate this file directly with `evil-winrm`, but was unable to. I then tried to make a copy and extract the copy and was successful.
![[Pasted image 20260401225519.png]]

Tried many ways to reconstruct the database from the `.frm` and `.ibd` files, but was unsuccessful. After many attempts, I then tried opening the `ibd` file with `Notepad++`, and was able to view some credentials in plaintext.
![[Pasted image 20260402004101.png]]

Seems like there are credentials for:
`charlotte`:` Game2On4.! `
`administrator`:` Almost4There8.? `

`charlotte` is the user I need to be able to edit the GPO.

The `administrator` credentials allowed me to access `ERA` as the local administrator through `evil-winrm`, and get the user and root flags for the machine.
![[Pasted image 20260402004535.png]]

`charlotte` has `WinRM` access and I was able to access `DC01` with `evil-winrm` and read the local flag.
![[Pasted image 20260402004845.png]]

Since I know that `charlotte` has `GenericWrite` over the `Default Domain Policy` GPO, and that GPO is linked to the domain and the `DC01` machine, I can simply use this GPO to add a user to local admin for `DC01`.

I used the `SharpGPOAbuse.exe` tool to do this. Uploaded the executable with `evil-rm`. then ran it with `.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount charlotte --GPOName "Default Domain Policy"` to add `charlotte` to the local administrators group.
![[Pasted image 20260402013938.png]]

Then ran `gpupdate /force`to force the group policy to update.
![[Pasted image 20260402014308.png]]

Now, checking `net user charlotte` shows that the account is part of the local Administrators group.
![[Pasted image 20260402014346.png]]

Exited `evil-winrm` and reconnected and was able to read the root flag.
![[Pasted image 20260402014415.png]]