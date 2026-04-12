# NightCity

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| NightCity | waidroc | Beginner | HackMyVM |

**Summary:** The NightCity CTF machine presents a sophisticated yet accessible challenge centered on steganographic data extraction and credential enumeration. The initial foothold exploits exposed anonymous FTP services containing hints directing attention to steganographically hidden credentials within JPEG images. Exiftool analysis reveals GPS metadata embedded in image files, establishing thematic context for the challenge. The stegseek tool extracts base64-encoded credentials (ThisIsTheRealPassw0rd!) from the most-wanted.jpg file using the passphrase "japon," discovered through brute-force password dictionary attacks. Web reconnaissance via directory enumeration identifies the /robin endpoint, a Batman and Robin-themed page written in Spanish that serves as a subtle user enumeration hint. SSH access is achieved using the batman user account with the extracted credentials. Post-exploitation reveals an iknowyou.jpg file in batman's home directory; brightness manipulation of this JPEG uncovers the string "ThatMadeMeL4ugh!" which becomes the lateral movement credential for the joker user. The final privilege escalation involves switching context to the joker account, which has read permissions on /home/.joker/flag.txt containing the root flag rendered as Batman ASCII art. This chain demonstrates the convergence of weak credential management, insufficient access controls, and embedded metadata vulnerabilities as exploitation vectors.

---

## Reconnaissance

### Network Discovery

The initial phase involves identifying the target host within the network segment. A network scanning tool confirms the presence of a VirtualBox virtual machine at address 192.168.100.168.

```
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
IP              MAC               Vendor
192.168.100.168 08:00:27:F3:F5:D6 VirtualBox
```

### Service Enumeration

A comprehensive nmap scan reveals three active services on the target:

```bash
nmap -sC -sV -p- -T4 192.168.100.168
```

```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-11 20:06 WIB
Nmap scan report for 192.168.100.168
Host is up (0.025s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.0.8 or later
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 192.168.100.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 0        0            4096 Jun 09  2022 reminder [NSE: writeable]
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 ce:ac:1c:04:d6:f6:64:d6:d9:9d:88:c9:0d:66:a9:45 (RSA)
|   256 4f:f1:7b:69:5c:47:b2:91:b8:d2:2f:82:73:b7:fc:03 (ECDSA)
|_  256 65:6b:3b:8c:89:81:4d:f3:98:98:5a:ed:57:cf:58:c9 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: NightCity Web Server
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 33.68 seconds
```

The scan identifies FTP (port 21) with anonymous login enabled, SSH (port 22) running on the target, and HTTP (port 80) serving the NightCity Web Server.

### FTP Anonymous Access

The FTP service permits anonymous authentication without password requirements. A directory traversal within the FTP server reveals a "reminder" subdirectory containing a hint file.

```bash
ftp 192.168.100.168
Connected to 192.168.100.168.
220 Welcome to the NightCity Server!!
Name (192.168.100.168:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||53191|)
150 Here comes the directory listing.
drwxr-xr-x    3 0        0            4096 Jun 09  2022 .
drwxr-xr-x    3 0        0            4096 Jun 09  2022 ..
drwxrwxrwx    2 0        0            4096 Jun 09  2022 reminder
226 Directory send OK.
ftp> cd reminder
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||46972|)
150 Here comes the directory listing.
drwxrwxrwx    2 0        0            4096 Jun 09  2022 .
drwxr-xr-x    3 0        0            4096 Jun 09  2022 ..
-rwxr-xr-x    1 0        0              33 Jun 09  2022 reminder.txt
226 Directory send OK.
ftp> get reminder.txt
local: reminder.txt remote: reminder.txt
229 Entering Extended Passive Mode (|||62608|)
150 Opening BINARY mode data connection for reminder.txt (33 bytes).
100% |*************|    33       10.65 KiB/s    00:00 ETA
226 Transfer complete.
33 bytes received in 00:00 (3.45 KiB/s)
ftp> bye
221 Goodbye.
```

The retrieved reminder.txt file contains the following message:

```
Local user is in the coordinates
```

This hint indicates that valid user credentials are hidden within coordinate data.

### Web Directory Enumeration

A directory brute-force scan against the HTTP service using gobuster identifies several accessible resources:

```bash
gobuster dir -u http://192.168.100.168 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html
```

```
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.168
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 319] [--> http://192.168.100.168/images/]
/index.html           (Status: 200) [Size: 8407]
/contact.html         (Status: 200) [Size: 6349]
/about.html           (Status: 200) [Size: 7744]
/gallery.html         (Status: 200) [Size: 8768]
/js                   (Status: 301) [Size: 315] [--> http://192.168.100.168/js/]
/robots.txt           (Status: 200) [Size: 136]
/secret               (Status: 301) [Size: 319] [--> http://192.168.100.168/secret/]
/robin                (Status: 200) [Size: 1873]
```

### robots.txt Analysis

The robots.txt file contains a clue directing further investigation:

```
#Good Job

To continue, you need a workmate. Our lastest news is that Robin is close to
NightCity. Try to find him, Robin has the key!!
```

This message hints at the existence of a "robin" user or resource on the system.

### Secret Directory Exploration

The /secret directory is accessible via HTTP and contains indexed image files:

![](image.png)

Three JPEG files are present in this directory: most-wanted.jpg (128K), some-light.jpg (214K), and veryImportant.jpg (185K). These are downloaded for offline analysis.

```bash
wget http://192.168.100.168/secret/most-wanted.jpg
wget http://192.168.100.168/secret/some-light.jpg
wget http://192.168.100.168/secret/veryImportant.jpg
```

---

## Metadata Analysis and Credential Discovery

### EXIF Data Extraction

The exiftool utility is used to examine embedded metadata within the downloaded images. The most-wanted.jpg file contains minimal EXIF data with no sensitive information. However, the some-light.jpg file reveals critical information:

```bash
exiftool most-wanted.jpg
```

```
ExifTool Version Number         : 13.36
File Name                       : most-wanted.jpg
Directory                       : .
File Size                       : 131 kB
File Modification Date/Time     : 2022:06:10 00:43:07+07:00
File Access Date/Time           : 2026:04:11 20:13:15+07:00
File Inode Change Date/Time     : 2026:04:11 20:13:15+07:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Image Width                     : 1334
Image Height                    : 750
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                       : 1334x750
Megapixels                      : 1.0
```

```bash
exiftool some-light.jpg
```

```
ExifTool Version Number         : 13.36
File Name                       : some-light.jpg
Directory                       : .
File Size                       : 219 kB
File Modification Date/Time     : 2022:06:10 00:42:53+07:00
File Access Date/Time           : 2026:04:11 20:13:29+07:00
File Inode Change Date/Time     : 2026:04:11 20:13:29+07:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
Exif Byte Order                 : Big-endian (Motorola, MM)
X Resolution                    : 72
Y Resolution                    : 72
Resolution Unit                 : inches
Y Cb Cr Positioning             : Centered
GPS Version ID                  : 2.3.0.0
GPS Latitude Ref                : North
GPS Longitude Ref               : East
XMP Toolkit                     : Image::ExifTool 12.41
Description                     : 26º21'28.59"N,127º47'0.99"E
Author                          : GothamCity
Image Width                     : 2048
Image Height                    : 1179
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                       : 2048x1179
Megapixels                      : 2.4
GPS Latitude                    : 26 deg 21' 28.59" N
GPS Longitude                   : 127 deg 47' 0.99" E
GPS Position                    : 26 deg 21' 28.59" N, 127 deg 47' 0.99" E
```

The some-light.jpg file contains GPS coordinates (26º21'28.59"N, 127º47'0.99"E) and an author field identifying "GothamCity," establishing the Batman-themed context.

![](image-1.png)

The veryImportant.jpg file contains no sensitive metadata:

```bash
exiftool veryImportant.jpg
```

```
ExifTool Version Number         : 13.36
File Name                       : veryImportant.jpg
Directory                       : .
File Size                       : 190 kB
File Modification Date/Time     : 2022:06:03 18:33:09+07:00
File Access Date/Time           : 2026:04:11 20:13:50+07:00
File Inode Change Date/Time     : 2026:04:11 20:13:50+07:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.02
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Image Width                     : 1200
Image Height                    : 740
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                       : 1200x740
Megapixels                      : 0.888
```

### Steganographic Data Extraction

The stegseek tool performs dictionary-based steganography attacks against the downloaded images. The most-wanted.jpg file yields results:

```bash
stegseek most-wanted.jpg
```

```
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "japon"
[i] Original filename: "pass.txt".
[i] Extracting to "most-wanted.jpg.out".
```

The extracted content is base64-encoded:

```bash
cat most-wanted.jpg.out
VGhpc0lzVGhlUmVhbFBhc3N3MHJkIQ==

cat most-wanted.jpg.out | base64 -d
ThisIsTheRealPassw0rd!
```

The decoded output reveals the password: **ThisIsTheRealPassw0rd!**

The remaining image files do not contain discoverable steganographic payloads:

```bash
stegseek some-light.jpg
```

```
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Progress: 99.69% (133.0 MB)
[!] error: Could not find a valid passphrase.
```

```bash
stegseek veryImportant.jpg
```

```
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Progress: 99.67% (133.0 MB)
[!] error: Could not find a valid passphrase.
```

---

## User Enumeration and Initial Access

### Robin Page Investigation

The /robin endpoint discovered during directory enumeration contains Batman and Robin comic book content in Spanish language:

```bash
curl -s http://192.168.100.168/robin
```

```html
<html>
<head>
        <title>BATMAN Y ROBIN</title>
</head>
<body>
        <h1>BATMAN Y ROBIN VOL. 01. PARTE I.</h1>
        <p>
        Hay un nuevo Dúo Dinámico en la ciudad. Tras la desaparición de Bruce Wayne, y concluida La Batalla por la Capucha, el Hombre Murciélago es ahora Dick Grayson. Pero tendrá que llevar a cabo su misión como justiciero junto a un acompañante imprevisto: Damian Wayne, hijo de Bruce y Talia al Ghul, ha asumido el papel de Robin... después de que Tim Drake, el anterior titular, haya adoptado otra identidad y emprendido una difícil búsqueda destinada a arrojar increíbles resultados sobre el verdadero destino del mentor de todos ellos. Sin embargo, mientras tanto, Dick y Damian deberán afrontar una Gotham que parece más enloquecida que nunca: a villanos cada vez más insólitos, entre ellos el Profesor Pyg y los demás miembros de su Circo de lo Extraño, se une el regreso de otro excompañero de Batman. Jason Todd, alias Capucha Roja, no solo cuenta con alguien muy sorprendente para ayudarle... ¡también está decidido a poner fin al reinado de los nuevos Batman y Robin antes incluso de que empiece!
        </p>
        <p>
        Grant Morrison y Frank Quitely, un tándem con obras tan reconocidas como All-Star Superman y New X-Men, toma las riendas de la primera colección del Caballero Oscuro y el Chico Maravilla que lleva sus nombres en el título... aunque los integrantes de este equipo no sean los habituales ni por asomo. Junto a Philip Tan (Batman del Futuro: La ciudad de japon), los dos aclamados autores escoceses abren una etapa repleta de innovadores conceptos y situaciones sin parangón que no dejarán indiferente a ningún lector. Lo demuestran a la perfección los dos arcos argumentales iniciales de la serie, Batman renacido y La venganza de Capucha Roja, que se incluyen íntegramente en este tomo de Batman Saga
        </p>
</body>
</html>
```

This page, combined with the GothamCity metadata and Batman-themed hints throughout the reconnaissance phase, strongly suggests "batman" as a valid system username.

### SSH Access via Batman Account

The batman user is targeted for authentication using the credentials extracted from the steganographic payload:

```bash
ssh batman@192.168.100.168
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openshat.com/pq.html
batman@192.168.100.168's password:
```

Authentication succeeds with the password **ThisIsTheRealPassw0rd!**

```
 _   _ _       _     _    ____ _ _
| \ | (_) __ _| |__ | |_ / ___(_) |_ _   _
|  \| | |/ _` | '_ \| __| |   | | __| | | |
| |\  | | (_| | | | | |_| |___| | |_| |_| |
|_| \_|_|\__, |_| |_|\__|\____|_|\__|\__, |
         |___/                       |___/

***  NightCityCTF © 2022 by Waidroc & Cillo31 is licensed under CC BY-NC-SA 4.0.  ***
              ***  https://www.github.com/Waidroc/NightCityCTF ***

Welcome to Ubuntu 18.04.6 LTS (5.4.0-84-generic).

System information as of: Sat Apr 11 15:24:07 CEST 2026

System Load:    4.09    IP Address:
Memory Usage:   14.4%   System Uptime:  19 min
Usage On /:     50%     Swap Usage:     0.0%
Local Users:    0       Processes:      133

*** System restart required ***
132 updates can be applied immediately.
99 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Last login: Wed Jun 15 19:15:17 2022 from 10.0.2.8
batman@NightCity:~$
```

### Initial Enumeration on Batman Account

Basic system reconnaissance reveals the batman user's identity and home directory contents:

```bash
id;whoami;hostname
uid=1002(batman) gid=1002(batman) grupos=1002(batman)
batman
NightCity

ls -la
total 308
drwxr-xr-x 5 batman        batman          4096 jun 15  2022 .
drwxr-xr-x 6 root          root            4096 jun  9  2022 ..
-rw------- 1 batman        batman           972 jun 15  2022 .bash_history
-rw-r--r-- 1 batman        batman           220 jun  8  2022 .bash_logout
-rw-r--r-- 1 batman        batman          3771 jun  8  2022 .bashrc
drwx------ 2 batman        batman          4096 jun  9  2022 .cache
-rw-r--r-- 1 root          root              66 jun  9  2022 flag.txt
drwx------ 3 batman        batman          4096 jun  9  2022 .gnupg
-rw-rw-r-- 1 administrator administrator 272105 jun  9  2022 iknowyou.jpg
drwxrwxr-x 3 batman        batman          4096 jun 15  2022 .local
-rw-r--r-- 1 batman        batman           807 jun  8  2022 .profile
```

The flag.txt file in batman's home directory is readable and contains a decoy message:

```bash
cat flag.txt
Nice try! but, this is not the flag. You have to keep working >:)
```

A critical file, iknowyou.jpg, is present and owned by the administrator account.

---

## Privilege Escalation and Lateral Movement

### System User Enumeration

The /etc/passwd file reveals four user accounts with shell access:

```bash
cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
administrator:x:1000:1000:administrator,,,:/home/administrator:/bin/bash
joker:x:1001:1001:Joker,,,:/home/joker:/bin/bash
batman:x:1002:1002:Batman,,,:/home/batman:/bin/bash
anonymous:x:1003:1003:,,,:/home/anonymous:/bin/bash
```

The joker user account is identified as a potential target for lateral movement.

### Steganographic Analysis of iknowyou.jpg

The iknowyou.jpg file, found in batman's home directory, is downloaded from a temporary HTTP server and analyzed:

```bash
python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

```bash
wget http://192.168.100.168:8080/iknowyou.jpg
--2026-04-11 20:27:27--  http://192.168.100.168:8080/iknowyou.jpg
Connecting to 192.168.100.168:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 272105 (266K) [image/jpeg]
Saving to: 'iknowyou.jpg'

iknowyou.jpg              100%[==================================>] 265.73K  --.-KB/s    in 0.1s

2026-04-11 20:27:27 (2.60 MB/s) - 'iknowyou.jpg' saved [272105/272105]
```

The file metadata contains no hidden information:

```bash
file iknowyou.jpg
iknowyou.jpg: JPEG image data, Exif standard: [TIFF image data, big-endian, direntries=0], baseline, precision 8, 1200x454, components 3

exiftool iknowyou.jpg
ExifTool Version Number         : 13.36
File Name                       : iknowyou.jpg
Directory                       : .
File Size                       : 272 kB
File Modification Date/Time     : 2022:06:10 00:55:43+07:00
File Access Date/Time           : 2026:04:11 20:27:31+07:00
File Inode Change Date/Time     : 2026:04:11 20:27:27+07:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
Exif Byte Order                 : Big-endian (Motorola, MM)
DCT Encode Version              : 100
APP14 Flags 0                   : [14], Encoded with Blend=1 downsampling
APP14 Flags 1                   : (none)
Color Transform                 : YCbCr
Image Width                     : 1200
Image Height                    : 454
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:4:4 (1 1)
Image Size                       : 1200x454
Megapixels                      : 0.545
```

Stegseek analysis yields no results:

```bash
stegseek iknowyou.jpg
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Progress: 99.88% (133.3 MB)
[!] error: Could not find a valid passphrase.
```

However, image brightness adjustment reveals hidden text within the image. Examining the pillar structure visible in the adjusted brightness version exposes the string: **ThatMadeMeL4ugh!**

![](image-2.png)

### Lateral Movement to Joker Account

The discovered credential is used to authenticate as the joker user:

```bash
su - joker
Contraseña: [ThatMadeMeL4ugh!]
Sin directorio, accediendo con HOME=/
joker@NightCity:/$ id;whoami;hostname
uid=1001(joker) gid=1001(joker) grupos=1001(joker)
joker
NightCity
```

### Root Flag Discovery

The joker user account has read permissions on the .joker hidden directory, which contains the root flag:

```bash
cd /home/.joker/
ls -la
total 28
drwxrwx--- 2 joker joker 4096 jun 13  2022 .
drwxr-xr-x 6 root  root  4096 jun  9  2022 ..
-rwxrwx--- 1 joker joker  220 jun  8  2022 .bash_logout
-rwxrwx--- 1 joker joker 3771 jun  8  2022 .bashrc
-rw-r--r-- 1 root  root  7157 jun  9  2022 flag.txt
-rwxrwx--- 1 joker joker  807 jun  8  2022 .profile

cat flag.txt
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⣴⣤⣶⡶⠛⠉⠉⠀⣀⣀⣀⣤⣤⣤⣶⣶⣒⣛⣉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⣴⣿⣿⣿⡿⠋⢀⣠⣴⣶⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣷⣶⢤⣀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⢀⡴⣿⣿⣿⣿⣿⣿⣷⣾⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡿⣟⡽⣟⣫⣭⣶⣶⣿⣿⣦⣄⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⢠⠏⠀⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠿⠋⠙⠉⠁⠀⢿⣿⣿⣿⣿⡿⠿⠿⣿⡿⣶⣦⣀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⡄⠀
⠀⠀⠀⠀⠀⠀⣴⠃⠀⠀⠈⠻⠿⠿⠿⠿⠟⠛⠉⠁⠙⠿⠿⠛⠋⠉⠀⠀⠀⢀⣠⣴⣶⣾⣿⣿⣿⣿⣷⣶⣦⣙⠻⢿⣿⣿⣿⣶⣶⣶⣦⣤⣤⣴⢶⣾⠟⠀⠀
⠀⠀⠀⠀⠀⠰⡏⣴⣄⠀⠀⠀⠀⠀⢀⣠⣴⣤⣄⣀⠀⠀⠀⠀⠀⠀⢀⣴⣶⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣦⡉⠙⠛⠛⠛⠛⠛⠉⣉⡴⢟⣡⣴⠏⠀
⠀⠀⠀⠀⠀⠀⠳⣿⣿⣷⣦⣀⠀⢠⣿⣿⣿⣿⣿⣿⣿⣶⡄⠀⣀⣶⣿⣿⡿⠿⠟⠛⠛⠛⠛⠛⠛⠛⠿⣿⣿⣿⣿⣿⣿⣶⠀⠀⠀⠀⣰⣾⣷⣾⣿⡿⠃⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠈⠛⠿⣿⣿⣷⣿⣿⣿⣿⣿⣿⣿⣿⣿⣣⢞⣭⠿⠋⠁⠀⠀⠀⠀⠀⠀⠠⣤⣤⣶⣾⣿⣿⣿⣯⣭⣿⣿⣶⣶⣿⣿⣿⣿⣿⣿⡟⠁⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠉⠛⢿⡿⣿⣿⣿⣿⣿⣿⣿⡵⠋⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⢻⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠏⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣼⠀⠀⠉⠉⠛⠿⠿⠏⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠙⣻⣿⣿⣿⣿⣿⣿⣿⣿⠿⠿⢿⣿⣿⠟⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢠⠇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⡶⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠿⣿⣿⣿⣿⣿⠟⠁⠀⣠⣶⣿⠇⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣼⠀⠀⠀⠀⠀⠈⠳⣄⠀⠀⠀⣾⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⣤⠀⠈⠻⣿⣿⡏⠀⣰⠊⠱⠛⡆⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⡟⠀⠀⠀⠀⠀⠀⠀⠈⠳⣄⠈⠁⠀⠀⠀⠀⣀⣀⣠⠤⠶⠶⠿⣫⠟⠁⠀⠀⠀⠈⠻⣁⣼⠗⣿⠀⢠⡇⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⡇⠀⠀⠀⠀⠀⣀⣀⣀⣀⣈⣷⠦⠤⠶⠖⠿⣭⣁⣀⣀⣠⣶⡾⠋⠀⠀⠀⠀⠀⠀⠀⠋⠁⢠⡇⠀⣼⠁⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠸⠿⠶⣶⣾⣯⣍⣉⠉⠙⠛⣿⠁⠀⠀⠀⠀⠀⠀⠉⠛⠿⠿⠛⠀⠀⠀⠀⣤⣀⣀⠀⠀⢸⡄⠀⣠⡜⠃⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢈⡏⠉⠻⢷⣶⣿⡏⠀⠀⠀⠀⠀⠀⠀⠀⢀⣠⠤⠤⠤⢤⣄⣀⡀⠀⠀⠀⠀⠀⢿⠉⠁⠙⢦⡀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⡾⠀⠀⢀⣀⣈⣼⠀⠀⠀⠀⠀⣀⣀⡴⠂⠉⠀⠀⠀⣠⢾⠁⠀⣽⠲⡄⠀⠀⠀⢸⡆⠀⠀⠀⠉⠳⢄⡀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢰⡇⠀⣴⣏⣁⠀⢻⠀⠀⢀⡴⣺⠝⠀⠀⠀⢀⣀⢶⠛⠁⠸⡄⠀⣿⠀⠹⠄⠀⠀⠈⡇⠀⠀⠀⠀⠀⠀⢙⠲⢄⡀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠸⣷⢸⡇⠈⠻⣷⣾⠀⠀⣨⠟⣁⣀⣤⣴⠶⠋⠁⢸⠀⠀⠀⣷⠀⣿⠀⠀⠀⠀⠀⢠⡇⠀⠀⠀⢀⡟⠀⢸⠀⠀⠉⠒⠄
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⢷⡳⠀⠀⠙⢿⣀⡶⠉⣿⠉⠉⠉⣧⠀⠀⠀⢸⠀⠀⠀⣿⣠⣿⠀⠀⠀⠀⢀⣾⠇⠀⠀⠀⡼⠁⠀⡟⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠳⣄⠀⠀⠸⠿⣷⡀⠸⡀⠀⠀⠹⡄⠀⠀⠸⢀⣀⡴⠟⣿⠇⠀⠀⠀⠀⣾⡏⠀⠀⠀⣸⠃⠀⢰⡇⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠙⣄⠀⠀⠀⠻⣧⣠⢧⡤⠤⠤⠿⣆⠀⠚⠉⣧⠀⢰⡿⠀⠀⠀⢀⣾⡟⠀⠀⠀⢠⠇⠀⠀⣸⠀⠀⠀⠀⠀⠀
⠀⠀⠀⣀⣀⢀⣤⡦⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⢧⠀⠀⣾⠉⠀⠀⣷⠀⠀⠀⢻⡀⠀⠀⢻⠀⣿⠁⠀⠀⢠⣾⡿⠀⠀⠀⢠⡞⠀⠀⢠⡇⠀⠀⠀⠀⠀⠀
⠀⢰⣾⣿⠷⣿⣿⠵⠖⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠳⡀⠘⣆⠀⠀⣿⠀⠀⠀⠀⣧⠀⠀⣸⣾⠃⠀⠀⣠⣿⠟⠀⠀⠀⢀⡞⠀⠀⠀⣸⠁⠀⠀⠀⠀⠀⠀
⠀⠈⡻⠉⠋⠉⢁⣤⣼⡏⢠⣆⡾⠃⠀⠀⠀⠀⠀⠀⠀⠀⠹⣄⣿⣶⣤⣼⣤⣀⣀⣀⡽⠶⣚⡿⠁⠀⢀⣾⣿⠋⠀⠀⠀⠀⡼⠁⠀⠀⢠⡇⠀⠀⠀⠀⠀⠀⠀
⢰⣶⢟⡴⢾⢇⣏⣤⡿⠀⠈⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠉⣦⠉⠉⠉⠉⠙⠛⠛⠋⠉⠉⠀⠀⣠⣿⣟⠁⠐⠒⠒⠶⡾⠁⠀⠀⠀⡼⠀⠀⠀⠀⠀⠀⠀⠀
⣸⣃⣽⣣⠜⠿⠉⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⢷⡀⠀⠀⠀⠀⠀⠀⠀⠀⣠⣾⠟⠁⠈⠳⡄⠀⠀⡼⠁⠀⠀⠀⢰⡇⠀⠀⠀⠀⠀⠀⠀⠀
⠿⠉⠀⠀⠀⠀⠀⠀⣀⠀⠀⠀⠀⠀⠀⣶⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣳⡀⠀⠀⠀⠀⠀⢀⣴⠟⠧⣄⠀⠀⠀⠙⣦⡞⠁⠀⠀⠀⣀⣺⠴⠶⢲⡆⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⢀⣸⡏⠀⠀⠀⠀⣀⣴⡏⠀⠀⠀⠀⠀⠀⠀⢀⡠⠞⠉⠹⣄⠀⠀⠀⣠⠟⠁⠀⠀⠈⠓⣦⣀⣠⢟⠀⣠⠴⠞⠉⠁⠀⠀⠀⢀⡇⠀⠀⠀⠀⠀
⠀⠀⠀⠀⢠⣶⠏⢻⣶⡶⣾⣿⣟⡯⠞⠃⠀⠀⠀⠀⠀⢀⡴⠋⠀⠀⠀⠀⠹⣄⣠⣞⠉⠉⠉⠉⠉⠓⠲⢶⠾⠶⢿⠋⠁⠀⠀⠀⠀⠀⠀⠀⢰⡇⠀⠀⠀⠀⠀
⠀⠀⠈⠀⠉⠉⣉⣻⠉⠉⠈⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⠀⠀⠀⠀⠀⠀⠀⠉⠉⠹⣆⠀⠀⠀⠀⠀⠀⠘⣦⠀⠘⡇⠀⠀⠀⠀⠀⠀⠀⠀⢸⡇⠀⠀⠀⠀⠀
⠀⠀⢀⣠⣶⠋⡽⠃⠀⢀⣀⡴⠞⠀⠀⠀⠀⠀⣀⣠⠀⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠹⣄⠀⠀⠀⠀⠀⠀⠈⣳⢶⣿⣀⠀⠀⠀⠀⠀⠀⠀⢸⠁⠀⠀⠀⠀⠀
⠀⠀⠎⠀⣼⠋⠀⠰⢊⣯⡟⠀⢀⣀⡤⠶⠒⠉⠉⠁⠀⢻⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢠⣿⡄⠀⠀⠀⠀⢀⣾⠁⣸⠇⢹⠳⣄⠀⠀⠀⠀⠀⡜⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⣯⢉⡶⡆⣸⡉⠓⠉⠉⠀⠀⠀⠀⠀⠀⠀⠀⠘⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⣇⠹⡀⠀⠀⣰⠏⡾⠀⣿⠀⢸⠀⠈⣿⢦⡀⠀⢰⡇⠀⠀⠀⠀⠀⠀
⠀⠀⠀⢀⡿⠋⠁⠁⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⠀⢻⣀⡜⠁⢠⠇⠀⣧⠀⢸⠀⠀⡇⠀⠙⠲⡽⠁⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠘⣆⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⠀⠀⠉⠀⠀⣿⠀⠀⣿⠀⢸⠀⢸⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠻⡀⠀⠀⠀⠀⠀⠀⠀⠀⢻⠀⠀⠀⠀⠀⠉⠉⠛⢿⡆⢸⡇⢸⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠻⡀⠀⠀⠀⠀⠀⠀⠀⠸⡄⠀⠀⠀⠀⠀⠀⠀⠘⣧⡾⠃⡜⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠻⣄⠀⠀⠀⠀⠀⠀⠀⣧⠀⠀⠀⠀⠀⠀⠀⠀⠉⠀⠀⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠙⠀⠀⠀⠀⠀⠀⠀⠿⠂⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠟⠀⠀⠀⠀

           Good job!! You just discovered the criminal!
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network enumeration identifies three open ports: FTP (21), SSH (22), and HTTP (80). Anonymous FTP access is enabled, and the reminder.txt file provides a hint about hidden credentials in "coordinates."

2. **Vulnerability Discovery**: Web directory enumeration via gobuster reveals the /secret endpoint containing three JPEG image files. The robots.txt file provides a thematic clue mentioning "Robin" and "the key." EXIF metadata analysis using exiftool exposes GPS coordinates in some-light.jpg and establishes the Batman and Robin theme through the GothamCity author field.

3. **Exploitation**: Steganographic analysis using stegseek extracts base64-encoded credentials from most-wanted.jpg using the discovered passphrase "japon." The decoded output yields the password "ThisIsTheRealPassw0rd!" which enables SSH authentication as the batman user.

4. **Internal Enumeration**: User enumeration reveals multiple accounts on the system: root, administrator, joker, batman, and anonymous. Analysis of batman's home directory reveals the iknowyou.jpg file owned by the administrator account. Image processing techniques, specifically brightness adjustment, expose the hidden string "ThatMadeMeL4ugh!" within the iknowyou.jpg image.

5. **Privilege Escalation**: The hidden credential "ThatMadeMeL4ugh!" from the image analysis is used to authenticate as the joker user via the su command. The joker account has read permissions on /home/.joker/flag.txt, which contains the root flag rendered as Batman ASCII art, completing the challenge.

