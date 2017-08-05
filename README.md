## mnm

_Mnm is Not Mail_

You choose the websites you visit; now choose who can send you mail. 
See [Why mnm?](Rationale.md)

mnm is a person-to-person (or app-to-app) message relay server, based on a new client/server protocol. 
(It's not a web app.) 
Written in Go, mnm aims to be reliable, fast, lightweight, dependency-free, and free of charge.

mnm provides:
- Members-only access
- Member aliases (including single-use aliases) to limit first-contact content
- Authorization for receive/send or receive-only
- Distribution groups, with invitations and member blocking
- IM/chat presence notifications
- Multiple messaging clients/devices per member
- Per-client strong (200 bit) passwords
- Reliable message storage (via fsync) and delivery (via ack)
- Message storage only until all recipients have ack'd receipt
- In-order message delivery from any given sender

mnm does not provide:
- Message encryption; clients are responsible for encryption before/after transmission

mnm may provide:
- Gateways to whitelisted mnm & SMTP sites

mnm shall be accessible via several network frontends:
- TCP server
- HTTP server (separate receiver connection per client, as needed)
- HTTP + Websockets
- Unix domain sockets
- Arbitrary Golang frontend invoking qlib package

### Status

_3 August 2017_ -
A simulation of 1000 concurrent active clients 
delivers 1 million messages totaling 6.7GB in 46 minutes. 
It uses ~200MB RAM, <10MB disk, and minimal CPU time. 
Each client runs a 19-step cycle that does login, then post for two recipients (15x) 
or for a group of 100 (2x) every 1-30s, then logout and idle for 1-30s. 

mnm v0.1 should be released in September 2017.

The author previously prototyped this in Node.js.
(Based on that experience, he can't recommend Node.js.)
_Warning, unreadable Javascript hackery follows._
http://github.com/networkimprov/websocket.MQ

### What's here

- qlib/qlib.go: package with simple API to the reciever & sender threads
- qlib/testclient.go: in-process test client, invoked from main()
- userdb.go: user records management
- mnm.go: main(), frontends (coming soon)
- codestyle.txt: how to make Go source much more clear
- After build & run:  
mnm: the app!  
userdb/: user & group data  
qstore/: queued messages awaiting delivery

### Quick start

1. go get github.com/networkimprov/mnm

2. go run mnm #currently starts test sequence  
_todo: prompt for key (or --key option) to decrypt userdb directory_

### TMTP Summary

"Trusted Messaging Transfer Protocol" defines a simple client/server exchange scheme; 
it needs no other protocol in the way that POP & IMAP need SMTP. 
TMTP may be conveyed by any reliable transport protocol, e.g. TCP, 
or tunneled through another protocol, e.g. HTTP. 
A client may simultaneously contact multiple TMTP servers. 
After the client completes a login or register request, either side may contact the other.

Each message starts with a header, wherein four hex digits give the size of a JSON metadata object, 
which may be followed by arbitrary format 8-bit data: 
`001f{ ... <"dataLen":uint> }dataLen 8-bit bytes of data`

0. TmtpRev gives the latest recognized protocol version; it must be the first message  
`{"op":0, "id":"1"}`  
Response `{"op":"tmtprev", "id":"1"}`

1. Register creates a user and client queue  
_todo: receive-only accounts which cannot ping or post_  
_todo: integrate with third party authentication services_  
`{"op":1, "newnode":string <,"newalias":string>}`  
.newnode is the user label for a client device  
Response _same as Login_  
At node `{"op":"registered", "uid":string, "nodeid":string <,"error":string>}`

2. Login connects a client to its queue  
`{"op":2, "uid":string, "node":string}`  
Response `{"op":"info|quit" "info":string}` (also given on login timeout)  
At nodes `{"op":"login", "id":string, "from":string, "posted":string, "headsum":uint, 
"datalen":0, "node":string}`

3. UserEdit updates a user account  
_todo: check & store label_  
_todo: dropnode and dropalias; prevent account hijacking from stolen client/nodeid_  
`{"op":3, "id":string, <"newnode":string | "newalias":string>}`  
.newnode is the user label for a client device  
Response `{"op":"ack", "id":string, "msgid":string, <"error":string>}`  
At nodes `{"op":"user", "id":string, "from":string, "posted":string, "headsum":uint, 
"datalen":0, <"nodeid":string | "newalias":string>}`

4. GroupInvite invites someone to a group, creating it if necessary  
`{"op":4, "id":string, "gid":string, "datalen":uint, <"datasum":uint>, "from":string, "to":string}`  
Response `{"op":"ack", "id":string, "msgid":string, <"error":string>}`  
At recipient `{"op":"invite", "id":string, "from":string, "posted":string, "headsum":uint, 
"datalen":uint, <"datasum":uint>, "gid":string, "to":string}`  
At members `{"op":"member", "id":string, "from":string, "posted":string, "headsum":uint, 
"datalen":0, "act":string, "gid":string, "alias":string, <"newalias":string>}`

5. GroupEdit updates a group  
_todo: moderated group_  
_todo: closed group publishes aliases to moderators_  
`{"op":5, "id":string, "act":"join" , "gid":string, <"newalias":string>}`  
`{"op":5, "id":string, "act":"alias", "gid":string, "newalias":string}`  
`{"op":5, "id":string, "act":"drop" , "gid":string, "to":string}`  
Response `{"op":"ack", "id":string, "msgid":string, <"error":string>}`  
At members `{"op":"member", "id":string, "from":string, "posted":string, "headsum":uint, 
"datalen":0, "act":string, "gid":string, "alias":string, <"newalias":string>}`

6. Post sends a message to users and/or groups  
`{"op":6, "id":string, "datalen":uint, <"datasum":uint>, "for":[{"id":string, "type":uint}, ...]}`  
.for[i].type: 1) user_id, 2) group_id (include self) 3) group_id (exclude self)  
Response `{"op":"ack", "id":string, "msgid":string, <"error":string>}`  
At recipient `{"op":"delivery", "id":string, "from":string, "posted":string, "headsum":uint, 
"datalen":uint, <"datasum":uint>}`

7. Ping sends a short text message via a user's alias.
A reply establishes contact between the parties.  
_todo: limit number of pings per 24h and consecutive failed pings_  
`{"op":7, "id":string, "datalen":uint, <"datasum":uint>, "to":string}`  
Response `{"op":"ack", "id":string, "msgid":string, <"error":string>}`  
At recipient `{"op":"ping", "id":string, "from":string, "posted":string, "headsum":uint, 
"datalen":uint, <"datasum":uint>, "to":string}`

8. Ohi notifies chat contacts of presence (in progress)  
`{"op":8, "id":string, "for":[{"id":string}, ...]}`  
Response `{"op":"ack", "id":string, <"error":string>}`  
At recipient `{"op":"ohi", "id":string, "from":string, "posted":string, "headsum":uint, 
"datalen":0}`

9. Ack acknowledges receipt of a message  
`{"op":9, "id":string, "type":string}`

10. Quit performs logout  
`{"op":10}`
