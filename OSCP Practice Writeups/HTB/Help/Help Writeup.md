Main steps
1. Enumerate machine to find backend software (Node.js Express running `GraphQL`)
2. Enumerate `GraphQL` to find credentials
3. Directory fuzzing to find application page
4. Login to application using credentials
5. Blind SQLi on application to dump user credentials
6. SSH access using user credentials
7. Enumerate Linux kernel version
8. Privilege escalate using kernel exploit

Learning points
1. Inverted commas within a `JSON` string need to be escaped or they will terminate the string
2. Mainly how to enumerate Node.js Express applications running `GraphQL`, the other vectors of blind SQLi and kernel exploits not really relevant

`nmap`
![[Pasted image 20260308205608.png]]

Ports 22, 80, 3000 (Node.js Express framework) open
Added to `help.htb` to `/etc/hosts`
Port 80 shows a default Apache page
![[Pasted image 20260308205712.png]]

Port 3000 shows a message: `Hi Shiv, To get access please find the credentials with given query`, but there doesn't seem to be anything else
Possible user named `Shiv`
![[Pasted image 20260308205758.png]]

`Feroxbuster` returned a subdomain `/support` and navigating there shows a `HelpDeskZ` login page.
![[Pasted image 20260308210253.png]]

It also found the` /support/controllers/admin` directory with a few PHP pages within, but none of them can be loaded.
![[Pasted image 20260308210356.png]]

Googling Node.js Express query language shows that there is a language called `graphQL` that is used for querying Node.js APIs.

Found a tool named `graphw00f` and tried using it to fingerprint the `graphQL`, but doesn't seem to give any interesting information, other than that there is a `graphQL` installation at `/graphQL`.
![[Pasted image 20260308225729.png]]

Based on the `graphQL` documentation at https://graphql.org/learn/introspection/, https://www.apollographql.com/blog/4-simple-ways-to-call-a-graphql-api, and https://www.pentestpartners.com/security-blog/pwning-wordpress-graphql/, it is possible to send queries to learn about the schema.
Using the query from pentestpartners and the `curl` format from appollograpql, `{ "query": "{ __schema{ queryType { name,fields { name,description } } } }" }` returned some schema information.
![[Pasted image 20260308231305.png]]

Querying the types of the schema shows 3 user defined types: Query, User, and String.
![[Pasted image 20260308232043.png]]

The `User` type seems the most interesting so I queried it with `-d '{ "query": "{ __type(name: \"User\"){ name fields{name }} }" }'` (Inverted commas within `JSON` strings need to be escaped).
![[Pasted image 20260308232819.png]]

Showed that the `User` type had the fields `username` and `password`. Tried to send the query to get the field data. Googling shows that the query format to make a query is like below, so tried a few combinations and managed to get a result with 
`curl -s help.htb:3000/graphql -H "Content-Type: application/json" -d '{ "query": "{User {username  password}}" }' `.
![[Pasted image 20260308233228.png]]

Says it could not query `User`, but suggested that `user` may work instead, so tried that.
![[Pasted image 20260308233303.png]]

Using `user` instead gave a username as well as a MD5 hashed password.
![[Pasted image 20260308233339.png]]

`helpme@helpme.com` : ` 5d3c93182bb20f07b994a7f617e99cff `

Ran the hash through MD5 and cracked it.
![[Pasted image 20260308233456.png]]

`helpme@helpme.com` : ` godhelpmeplz `

Managed to login to the `HelpDeskZ` application with these credentials.
![[Pasted image 20260308233607.png]]

Initial enumeration of the website shows no information. Searching up the software shows a few exploits, including a blind SQLi injection exploit.

The exploit works by:
1. Creating a ticket with an attachment
2. Getting the link to download the attachment
3. Appending SQL queries to the end of this link to enumerate the database

Given that this involves blind SQL injection, it is possible to use `SQLMap` to dump the entire database.

Rest of the machine is Blind SQLi to get get SSH access > kernel exploit for privilege escalation as root.

Compile exploit from `ExploitDB` on target machine and run it to gain root shell.