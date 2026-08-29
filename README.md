# HackMyVM Penetration Testing Portfolio

<div align="center">

![Platform](https://img.shields.io/badge/Platform-HackMyVM-critical?style=flat-square)
![Machines](https://img.shields.io/badge/Machines-156-blue?style=flat-square)
![Hades Lab](https://img.shields.io/badge/Hades_Lab-32_Modules-informational?style=flat-square)
![Venus Lab](https://img.shields.io/badge/Venus_Lab-50_Modules-informational?style=flat-square)
![Documentation](https://img.shields.io/badge/Status-Active-success?style=flat-square)
![Language](https://img.shields.io/badge/Language-English-lightgrey?style=flat-square)

**Professional Penetration Testing Documentation and Technical Lab Reports**

</div>

---

## Overview

This repository contains comprehensive technical documentation for penetration testing exercises conducted on the HackMyVM platform. The collection represents systematic security analysis across 156 virtual machines and 82 laboratory modules, emphasizing methodological rigor and professional reporting standards over simple flag acquisition.

## Portfolio Philosophy

**Quality Over Quantity**

This repository differs from traditional Capture The Flag writeups. Each document serves as a technical lab report that demonstrates:

- Systematic vulnerability analysis and exploitation methodology
- Complete attack chain documentation from reconnaissance to privilege escalation
- Reproducible technical evidence through synchronized terminal logs and visual proof
- Professional reporting standards suitable for security assessment documentation

The objective is to showcase not merely the ability to compromise systems, but the capacity to document security findings with the precision and clarity expected in professional penetration testing engagements.

**Target Audience**

These reports are designed for security engineers, penetration testers, red team operators, and technical hiring managers evaluating practical security analysis capabilities and technical documentation proficiency.

---

## Repository Structure

```
hackmyvm-writeups/
│
├── machines/                    # 156 Virtual Machine Penetration Tests
│   ├── Alzheimer/
│   │   ├── alzheimer.md         # Complete technical writeup
│   │   └── image*.png           # Visual evidence and screenshots
│   ├── Aria/
│   ├── Thirteen/
│   ├── Observer/
│   └── [112 additional machines...]
│
├── labs/                        # Progressive Laboratory Series
│   ├── hades/
│   │   └── 0x01.md - 0x32.md    # 32 privilege escalation modules
│   └── venus/
│       └── 0x01.md - 0x50.md    # 50 exploitation technique modules
│
└── challenges/                  # CTF Challenges [Planned]
```

**Repository Statistics:**

- 156 completed machine writeups with full technical documentation
- 32 Hades laboratory modules focused on Linux privilege escalation
- 50 Venus laboratory modules covering web and system exploitation
- Comprehensive visual evidence integrated throughout all reports

---

## Standardized Reporting Format

Every writeup in this repository adheres to a consistent technical reporting structure designed to ensure clarity, reproducibility, and professional presentation.

### Report Structure Template


```markdown
# [Machine Name]

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| [Name] | [Author] | [Difficulty] | HackMyVM |

**Summary:** [Comprehensive overview of vulnerabilities, attack vectors, and exploitation path]

---

## Reconnaissance
[Network discovery, port scanning, service enumeration, and initial information gathering]

## Initial Access
[Vulnerability identification, exploitation techniques, and initial foothold acquisition]

## Privilege Escalation
[Enumeration of privilege escalation vectors and root access methodology]

---

## Attack Chain Summary
1. **Reconnaissance**: [Specific enumeration findings]
2. **Vulnerability Discovery**: [Identified attack surface]
3. **Initial Exploitation**: [Exploitation method and user access]
4. **Internal Enumeration**: [Post-exploitation information gathering]
5. **Privilege Escalation**: [Root access technique]
```

### Documentation Workflow

The repository employs a structured documentation pipeline that ensures consistency and technical accuracy:

**Phase 1: Data Collection**

- Terminal session logs captured in output.md
- Visual evidence collected through screenshots (image.png, image-1.png, etc.)
- All artifacts stored in machine-specific directories

**Phase 2: Report Generation**

- Raw terminal logs transformed into structured technical writeups
- Visual evidence embedded inline at relevant technical steps
- Commands and outputs preserved for reproducibility
- Attack chain synthesized from exploitation sequence

**Phase 3: Quality Assurance**

- Executive summary generated from technical findings
- Attack chain validated against actual exploitation path
- Visual proof verified for accuracy and completeness
- Final report formatted according to standardized template

This workflow ensures that each of the 156 machine reports maintains consistent quality and professional presentation standards.


---

## Technical Skills Matrix

The writeups in this repository demonstrate proficiency across the complete penetration testing lifecycle:

### Reconnaissance and Enumeration

| Skill Area | Tools and Techniques |
|------------|---------------------|
| Network Scanning | Nmap, Masscan, ARP-scan, custom PowerShell scripts |
| Service Fingerprinting | Version detection, banner grabbing, vulnerability correlation |
| Web Enumeration | Gobuster, Ffuf, Dirbuster, WFuzz, directory brute-forcing |
| OSINT | Information gathering, username enumeration, credential discovery |

### Vulnerability Analysis

| Skill Area | Tools and Techniques |
|------------|---------------------|
| Web Application Testing | Burp Suite, manual source code analysis, parameter fuzzing |
| Local File Inclusion | Path traversal, filter bypass, encoding manipulation (ROT13, Base64) |
| File Upload Vulnerabilities | Content-type bypass, polyglot file creation, web shell injection |
| Configuration Analysis | Default credentials, misconfigured services, exposed APIs |

### Exploitation

| Skill Area | Tools and Techniques |
|------------|---------------------|
| Remote Code Execution | Web shell deployment, reverse shell establishment, command injection |
| Service Exploitation | FTP server manipulation, RPC abuse, SMB exploitation |
| Authentication Bypass | Credential brute-forcing (Hydra), session manipulation, token extraction |
| Steganography | Zero-width character analysis, metadata extraction, hidden data recovery |

### Privilege Escalation

| Skill Area | Tools and Techniques |
|------------|---------------------|
| Linux Privilege Escalation | SUID/SGID binary abuse, sudo misconfiguration, kernel exploits |
| File System Analysis | Sensitive file discovery, credential harvesting, bash history analysis |
| Service Abuse | Cron job manipulation, writable service exploitation, capability abuse |
| Container Exploitation | Docker escape techniques, Nginx Unit API manipulation |

### Post-Exploitation and Documentation

| Skill Area | Tools and Techniques |
|------------|---------------------|
| Evidence Collection | Screenshot capture, terminal logging, artifact preservation |
| Attack Path Mapping | Attack chain documentation, vulnerability correlation |
| Technical Writing | Executive summary generation, methodology documentation, reproducible reporting |
| Automation | Custom documentation pipelines, workflow optimization |


---

## Machines Inventory

### Completed Virtual Machines (156)

<details>
<summary><b>View Complete Machine List</b></summary>

| Machine | Difficulty | Primary Attack Vectors |
|---------|-----------|------------------------|
| Alzheimer | Beginner | Password forensics, privilege escalation |
| animetronic | Beginner | Directory brute force, sensitive backup analysis, sudo privilege escalation |
| apaches | Intermediate | Apache 2.4.49 path traversal and RCE (CVE-2021-41773), sudo abuse |
| Aria | Beginner | Custom shell interaction, file upload bypass, RPC exploitation, steganography |
| arroutada | Beginner | Web routing vulnerabilities |
| Art | Beginner | File upload vulnerabilities, web exploitation |
| artig | Beginner | Anonymous FTP credential disclosure, Redis authentication, WordPress plugin RCE, sudo privesc |
| atom | Beginner | Text editor exploitation |
| aurora | Beginner | Service enumeration |
| Azer | Beginner | Web exploitation techniques |
| Bah | Beginner | Local File Inclusion, SSH key extraction |
| Baseme | Beginner | Base64 encoding manipulation, command injection |
| beloved | Beginner | Relationship-based challenges |
| blackhat | Intermediate | Advanced hacking scenarios |
| banner | Beginner | FTP credential recovery, pickle deserialization, root shell via reverse shell |
| breakout | Intermediate | Container/jail escape |
| bruteforcelab | Beginner | SSH brute-force, hydra password recovery, sudo misconfiguration |
| buster | Beginner | Brute-force techniques |
| canto | Beginner | WordPress Canto plugin vulnerability, LFI, credential harvesting, SUID binary exploitation |
| chromatica | Beginner | User-Agent bypass, web application command injection, sudo privilege escalation |
| codeshield | Beginner | Anonymous FTP document disclosure, credential recovery, sudo misconfiguration |
| Coffeeshop | Beginner | Web application vulnerabilities |
| comingsoon | Beginner | Placeholder page exploitation |
| connection | Beginner | Network service abuse |
| Convert | Beginner | File conversion exploitation |
| Coolpg | Beginner | PostgreSQL exploitation, SQL injection |
| Crack | Beginner | Hash cracking, password analysis |
| crazymed | Beginner | Medical system vulnerabilities |
| crossroads | Beginner | Path traversal challenges |
| cve1 | Intermediate | CVE exploitation practice |
| darkside | Beginner | Public voting backup data leak, credential recovery, sudo command abuse |
| Decode | Beginner | Encoding analysis and decryption |
| deeper | Beginner | Careless credential storage, web interface analysis, sudo privilege escalation |
| dejavu | Beginner | Pattern recognition |
| demons | Intermediate | Multi-service exploitation |
| Devoops | Intermediate | Git repository exploitation, DevOps misconfigurations |
| djinn | Advanced | Complex multi-stage attacks |
| Doc | Beginner | Document processing vulnerabilities |
| Doll | Intermediate | Binary exploitation, buffer overflow |
| DoubleTrouble | Intermediate | Multi-stage exploitation |
| Driftingblues3 | Series | Web application exploitation chain |
| Driftingblues5 | Series | Advanced enumeration techniques |
| Driftingblues6 | Series | Database attack vectors |
| Driftingblues7 | Series | Multi-stage privilege escalation |
| Driftingblues8 | Series | Advanced web exploitation |
| Driftingblues9 | Series | Complex attack chains |
| DrippingBlues | Series | Web security challenges |
| economists | Beginner | Anonymous FTP PDF metadata analysis, username harvesting, hydra brute force, sudo abuse |
| encrypt | Beginner | SSL/TLS certificate credential disclosure, SSH access, custom SUID bash binary abuse |
| ephemeral3 | Series | Temporary service exploitation |
| faust | Intermediate | Literature-themed challenges |
| find | Beginner | File discovery techniques |
| first | Beginner | Introductory challenges |
| flower | Beginner | Themed web exploitation |
| flute | Beginner | Music-themed challenges |
| friendly | Beginner | Social engineering vectors |
| friendly2 | Beginner | Advanced social engineering |
| friendly3 | Beginner | Multi-stage social engineering |
| Fuzzz | Intermediate | Web fuzzing, input validation bypass |
| Gameshell | Beginner | Interactive shell exploitation |
| Gameshell2 | Beginner | Advanced shell manipulation |
| Gameshell3 | Beginner | Shell escape techniques |
| gameshell5 | Intermediate | Interactive shell escape, service exploitation, kernel privilege escalation |
| Gift | Beginner | File transfer protocols, steganography |
| Gigachad | Intermediate | Advanced exploitation scenarios |
| greatwall | Beginner | Local File Inclusion (LFI), sensitive data exposure, sudo privilege escalation |
| Hannah | Beginner | User enumeration, lateral movement |
| Helium | Intermediate | Container escape, Docker exploitation |
| Helpdesk | Beginner | Support system vulnerabilities |
| hero | Beginner | Exposed OpenSSH private key on web server, SSH access, sudo abuse |
| hidden | Beginner | Steganography and hidden data |
| Hommie | Beginner | SUID binary abuse |
| Hostname | Beginner | DNS and hostname manipulation |
| Hotel | Beginner | Web service exploitation |
| Hundred | Beginner | Multi-user enumeration |
| Hunter | Beginner | OSINT techniques, reconnaissance |
| Icecream | Beginner | SMB share exploitation, Nginx Unit API abuse, sudo privilege escalation |
| insomnia | Intermediate | Sleep-based vulnerabilities |
| isengard | Intermediate | Fantasy-themed exploitation |
| jabita | Beginner | Web service challenges |
| Jan | Beginner | Cron job exploitation |
| latestwasalie | Intermediate | Insecure Docker registry credential recovery, poisoned container deployment, sudo privesc |
| lazzycorp | Beginner | Corporate infrastructure |
| liceo | Beginner | Anonymous FTP note disclosure, credential harvesting, sudo privilege escalation |
| literal | Beginner | SQL injection across subdomains, hash extraction and cracking, sudo abuse |
| Ll104567 | Intermediate | Binary reverse engineering |
| Locker | Beginner | Encryption bypass techniques |
| Luz | Beginner | Steganography, cryptographic analysis |
| mathdop | Beginner | Mathematical challenges |
| medusa | Intermediate | Multi-headed attack vectors |
| Meltdown | Advanced | Kernel vulnerability exploitation (CVE-2017-5754) |
| Method | Beginner | Methodical enumeration |
| Motto | Beginner | Configuration analysis |
| nebula | Beginner | Web enumeration, sensitive PDF metadata analysis, credential extraction, sudo privesc |
| newbee | Beginner | Hidden command execution interface, command injection, sudo misconfiguration |
| nexus | Beginner | Port enumeration, web vulnerability analysis, credential harvesting, sudo privesc |
| nightcity | Beginner | Anonymous FTP steganography, credential recovery, sudo privilege escalation |
| Noob | Beginner | Fundamental penetration testing methodology |
| Observer | Beginner | LFI via custom Golang application, SSH key extraction, bash history privilege escalation |
| Oliva | Beginner | Web framework vulnerabilities |
| p4l4nc4 | Beginner | Leet-speak directory enumeration, credential disclosure, SUID binary exploitation |
| Pdf | Beginner | PDF metadata exploitation |
| pingme | Beginner | Network traffic analysis of ping utility, command injection, sudo abuse |
| pipy | Intermediate | SPIP 4.2.0 deserialization RCE via oubli parameter, sudo privilege escalation |
| Preload | Intermediate | Library preloading exploitation |
| publisher | Beginner | Outdated SPIP CMS RCE exploitation, credential recovery, sudo privilege escalation |
| Pwned | Beginner | Multi-vector exploitation |
| quick | Beginner | Web application enumeration, command injection, sudo privilege escalation |
| quick2 | Beginner | Web application vulnerability analysis, credential disclosure, sudo abuse |
| quick3 | Beginner | Customer management portal exploitation, credential recovery, sudo privesc |
| quick4 | Beginner | Robots.txt administrative area discovery, web exploitation, sudo abuse |
| quick5 | Beginner | Subdomain fuzzing, web vulnerability analysis, sudo privilege escalation |
| React | Intermediate | React.js application security testing |
| ripper | Intermediate | Data extraction techniques |
| roosterrun | Beginner | Race condition exploitation |
| savesanta | Beginner | Hidden web directory discovery, command execution interface, sudo abuse |
| Skid | Beginner | Script analysis, enumeration techniques |
| slackware | Beginner | Non-standard port enumeration, web vulnerability analysis, sudo privilege escalation |
| slowman | Beginner | Anonymous FTP information disclosure, username harvesting, SSH brute force, sudo abuse |
| stars | Beginner | Rating system vulnerabilities |
| superhuman | Advanced | Complex privilege escalation |
| Sysadmin | Beginner | System administration misconfiguration exploitation |
| System | Beginner | System service exploitation |
| T800 | Advanced | Complex privilege escalation scenarios |
| Talk | Beginner | Inter-process communication vulnerabilities |
| teacher | Beginner | Educational platform exploitation |
| Thefinals | Intermediate | CTF-style challenges |
| thewall | Intermediate | Barrier bypass techniques |
| Thirteen | Beginner | ROT13 encoding LFI, FTP default credentials, Python FTP server RCE |
| Todd | Beginner | Credential harvesting techniques |
| Tpn | Intermediate | VPN configuration exploitation |
| Translator | Beginner | Translation service vulnerabilities |
| tron | Intermediate | Grid-based challenges |
| twelve | Beginner | Numeric-themed exploitation |
| Twisted | Intermediate | Python application framework exploitation |
| umz | Beginner | Custom service exploitation |
| University | Beginner | Educational platform attack vectors |
| up | Beginner | Image upload filter bypass, ROT13 predictable obfuscation analysis, sudo privesc |
| uvalde | Beginner | Geographic-themed challenges |
| Victorique | Beginner | Custom binary analysis |
| Vinylizer | Beginner | Media processing vulnerabilities |
| Visions | Beginner | Visual cryptography challenges |
| vivifytech | Beginner | WordPress enumeration, plugin vulnerability exploitation, sudo privilege escalation |
| Vulny | Beginner | Intentionally vulnerable service exploitation |
| w140 | Beginner | Numeric identifier challenges |
| Warez | Beginner | File sharing service exploitation |
| Warrior | Intermediate | Advanced attack scenario simulation |
| Webmaster | Beginner | Web server misconfiguration exploitation |
| whitedoor | Beginner | Anonymous FTP enumeration, remote service vulnerability exploitation, sudo abuse |
| wmessage | Beginner | Messaging service exploitation |
| xmas | Beginner | Unrestricted PDF upload bypass with PHP payload, reverse shell, sudo privilege escalation |
| yuan111 | Series | Progressive difficulty series |
| yuan112 | Series | Advanced series challenges |
| yuan113 | Series | Expert series challenges |
| yuan114 | Series | Web vulnerability exploitation, system script logic error abuse, sudo privesc |
| za1 | Beginner | Typecho CMS vulnerability exploitation, PHP deserialization, sudo privilege escalation |

</details>

**Notable Series:**

- **DriftingBlues Series** (3, 5, 6, 7, 8, 9) + DrippingBlues: Progressive difficulty challenges focusing on web application security
- **Gameshell Series** (1, 2, 3): Interactive shell-based exploitation scenarios
- **Friendly Series** (1, 2, 3): Social engineering and enumeration techniques
- **Yuan Series** (111, 112, 113): Progressive exploitation challenges

---

## Laboratory Series

### Hades Lab - Linux Privilege Escalation (32 Modules)

A progressive training environment focusing on systematic privilege escalation techniques. Each module (0x01 through 0x20 and beyond) represents a user-to-user escalation challenge within a single virtual machine environment.

**Coverage Areas:**

- SUID and SGID binary exploitation
- File permission abuse and misconfiguration
- Credential discovery in file systems
- Binary analysis and reverse engineering
- Sudo misconfiguration exploitation
- Custom binary manipulation

### Venus Lab - Web and System Exploitation (50 Modules)

Comprehensive exploitation training covering 50 progressive modules (0x01 through 0x50) with emphasis on web application security and system-level attack techniques.

**Coverage Areas:**

- Web application vulnerability exploitation
- Authentication and session management bypass
- Server-side request forgery and injection attacks
- File upload and inclusion vulnerabilities
- System service exploitation
- Advanced enumeration and pivoting techniques

---

## Learning Outcomes and Demonstrated Competencies

This portfolio demonstrates systematic competency development across core penetration testing domains:

### Strategic Capabilities

- **Attack Surface Analysis**: Comprehensive enumeration of network services, web applications, and system configurations
- **Vulnerability Assessment**: Identification and prioritization of exploitable weaknesses
- **Exploit Chain Development**: Construction of multi-stage attack paths from initial access to privilege escalation
- **Post-Exploitation Operations**: Evidence collection, credential harvesting, and attack path documentation

### Technical Proficiencies

- **OWASP Top 10**: Practical exploitation of injection vulnerabilities, broken authentication, sensitive data exposure, XML external entities, broken access control, security misconfigurations, cross-site scripting, insecure deserialization, and insufficient logging
- **Linux Security**: Comprehensive understanding of permission models, kernel exploitation, service abuse, and container escape techniques
- **Web Application Security**: Directory traversal, local and remote file inclusion, SQL injection, cross-site scripting, cross-site request forgery, and authentication bypass

### Professional Standards

- **Documentation Excellence**: Clear, reproducible, and professionally formatted technical reports
- **Methodological Rigor**: Systematic approach to vulnerability discovery and exploitation
- **Attack Chain Analysis**: Complete traceability from reconnaissance through privilege escalation
- **Evidence Preservation**: Synchronized terminal logs and visual documentation

---

## Navigation Guide

### For Security Professionals

To evaluate technical proficiency:

1. Review the Technical Skills Matrix for capability overview
2. Examine writeups from the machines directory for methodology assessment
3. Analyze the Hades Lab series for privilege escalation expertise
4. Review the Venus Lab series for web application security knowledge

### For Technical Recruiters

To assess candidate suitability:

1. Examine the Portfolio Philosophy section for professional approach
2. Review 2-3 sample machine writeups for documentation quality
3. Evaluate the Technical Skills Matrix against position requirements
4. Assess the scale and consistency of the portfolio (51 machines, 82 lab modules)

### For Students and CTF Players

To learn penetration testing techniques:

1. Start with beginner-level machines such as Noob or Hundred
2. Progress through the DriftingBlues series for structured learning
3. Study the Hades Lab series for privilege escalation fundamentals
4. Analyze Attack Chain Summaries to understand exploitation sequences

---

## Documentation Standards

All writeups in this repository adhere to the following standards:

- **Structured Format**: Consistent use of Executive Summary, Reconnaissance, Initial Access, Privilege Escalation, and Attack Chain Summary sections
- **Command Reproducibility**: Complete command syntax with full output preservation
- **Visual Evidence**: Annotated screenshots embedded at relevant technical steps
- **Technical Accuracy**: Validated exploitation paths with reproducible results
- **Professional Presentation**: Clean Markdown formatting suitable for technical documentation

---

## Legal and Ethical Disclaimer

**All activities documented in this repository were conducted in authorized laboratory environments.**

This repository is intended exclusively for:

- Educational purposes and professional skill development
- Portfolio demonstration for employment opportunities
- Security research within controlled environments (HackMyVM platform)

**Legal Notice:**

Unauthorized access to computer systems is illegal under applicable laws including but not limited to:

- Computer Fraud and Abuse Act (CFAA) - United States
- Computer Misuse Act - United Kingdom
- Cybercrime laws in various jurisdictions

The techniques documented in this repository must only be applied:

- On systems you own
- On systems where you possess explicit written authorization
- In legitimate Capture The Flag or laboratory environments

**Liability Disclaimer:**

The author assumes no liability for misuse of the information contained in this repository. Users are solely responsible for ensuring compliance with applicable laws and regulations. Any application of these techniques without proper authorization may result in criminal prosecution.

---

## Contributing and Collaboration

While this repository serves primarily as a personal portfolio, technical feedback is welcomed:

- **Technical Corrections**: Report errors or inaccuracies through issue tracking
- **Methodology Discussions**: Share alternative approaches or optimization techniques
- **Documentation Improvements**: Suggest enhancements to reporting clarity or structure

For professional inquiries or collaboration proposals, contact information can be provided upon request.

---

## Repository Roadmap

### Current Status (April 2026)

- 156 machine writeups completed with full documentation
- 32 Hades laboratory modules documented
- 50 Venus laboratory modules documented
- Standardized executive summary format implemented across all reports
- Multimodal documentation workflow operational

### Planned Enhancements

- Addition of CTF-style challenges to the challenges directory
- Development of custom enumeration and exploitation automation scripts
- Cross-machine vulnerability trend analysis and pattern documentation
- Extension of laboratory series as new modules are released

---

## Acknowledgments

- **HackMyVM Platform**: For providing a comprehensive and challenging learning environment
- **Security Community**: For continuous knowledge sharing and collaborative research
- **Challenge Authors**: For creating realistic and educational security scenarios

---

<div align="center">

![Maintained](https://img.shields.io/badge/Maintained-Yes-brightgreen?style=flat-square)
![Last Updated](https://img.shields.io/badge/Last_Updated-August_2026-blue?style=flat-square)

**Professional Penetration Testing Documentation**

</div>
