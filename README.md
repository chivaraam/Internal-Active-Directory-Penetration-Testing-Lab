# Internal-Active-Directory-Penetration-Testing-Lab

# Project Overview

This project demonstrates an end-to-end internal Active Directory penetration test performed in a self-built virtual lab. 
The objective was to simulate a realistic internal network attack, beginning with the exploitation of a vulnerable Linux server 
and progressing through Active Directory enumeration, Kerberoasting, credential recovery, and Domain Controller compromise.

The lab was built entirely in VirtualBox using isolated networking to ensure a safe testing environment.

# Lab Environment
| Machine | Operating System | Purpose |
|---|---|---|
| Kali Linux | Kali Linux | Attacker Machine |
| Windows Server 2019 | Domain Controller | Active Directory Server |
| Metasploitable2 | Ubuntu Linux | Vulnerable Target |


# Topology
```
Host Machine
      │
VirtualBox Host-Only Network
      │
├── Kali Linux          192.168.56.101
├── Windows Server 2019 192.168.56.104
└── Metasploitable2     192.168.56.102
```
# Security Tools Used
Nmap
Metasploit Framework
BloodHound
Neo4j
BloodHound.py
Impacket
Hashcat
John the Ripper
CrackMapExec
PsExec (Impacket)


# Methodology

1. Network Reconnaissance — host discovery and port/service enumeration via Nmap
2. External Host Assessment — credential and service-based access attempts against Metasploitable2
3. Active Directory Enumeration — domain data collection and graph-based analysis using BloodHound
4. Credential Access — Kerberoasting against a service account identified through enumeration
5. Privilege Escalation Validation — confirmation of cracked credentials against the Domain Controller
6. Impact Validation — confirmation of SYSTEM-level access on the Domain Controller


# Findings

Finding 1: Default Credentials on Metasploitable2 SSH Service — Critical

Host: 192.168.56.102 | Service: OpenSSH 4.7p1

The SSH service was found to accept the well-known default credential pair msfadmin:msfadmin, granting an interactive shell with no prior reconnaissance of user accounts required.

Evidence: <img width="1920" height="1080" alt="default_username_password_resultsincompromise" src="https://github.com/user-attachments/assets/c153113c-f70f-4ecd-9883-67cbd5190b4c" />
— SSH login succeeded, returning a full interactive shell (msfadmin@metasploitable:~$), confirmed by successful execution of ls, touch hello.txt, and rm hello.txt.

Impact: Immediate unauthenticated-equivalent access to the host with no exploit development required — credential guessing alone was sufficient.


Finding 2: Multiple High-Risk Legacy Services Identified — Critical

Host: 192.168.56.102

A full port and version scan (nmap -sV -sC -p-) identified numerous outdated and insecure services:

Evidence: <img width="1920" height="1080" alt="nmap_result_metasploitable2_nse_with_version" src="https://github.com/user-attachments/assets/1d913ae3-0d77-4d80-aa6a-ddffe0b88fc9" />

services with known cve and outdated version are also found through the scan like vsftpd 2.3.4, telnet, ISC BIND 9.4.2, Apache httpd 2.2.8,...etc
These services are only open because we are scanning a vulnerable machine (metasploitable2). Finding these services to be open in real time is significantly low.

Finding 3: Account Configured Without Kerberos Pre-Authentication — Medium

Account: sony

The account sony was deliberately/incidentally configured with Do Not Require Pre-Authentication enabled, which permits an attacker to request an authentication ticket for the account without prior knowledge of its password (AS-REP Roasting), enabling an offline password-cracking attempt.

Evidence: <img width="1002" height="667" alt="AD_setspn_Kerberos_preauth_disable" src="https://github.com/user-attachments/assets/c5015f96-bb44-4b85-9464-73682fc8d25f" />


Finding 4: Kerberoastable Service Account with Weak, Crackable Password — High

Account: svc_sql

The svc_sql account had a Service Principal Name registered (MSSQLSvc/dbserver.corp.local:1433), making it a valid Kerberoasting target. Using only the standard low-privilege domain account sony, a Kerberos service ticket (TGS) was successfully requested for svc_sql with no elevated access required.

Evidence:

impacket-GetUserSPNs corp.local/sony -dc-ip 192.168.56.104 -request
returned the SPN, confirmed svc_sql's group membership as CN=Domain Admins, and returned a full `krb5tgs$23
...` ticket hash

The recovered hash was cracked offline using John the Ripper 
john --format=krb5tgs --wordlist=passes.txt svc_sql.hash

Impact: Any authenticated domain user regardless of privilege level could obtain and crack this credential without triggering a failed-logon event, since Kerberoasting does not require an authentication attempt against the target account itself.

Finding 5: Service Account Configured with Domain Admin Privileges — Critical

Account: svc_sql

BloodHound analysis confirmed svc_sql is a member of Domain Admins, and is flagged by BloodHound as a high-value asset with Admin Count: TRUE.

Evidence: <img width="1920" height="1080" alt="bloodhound_graphh" src="https://github.com/user-attachments/assets/4e9857fa-07ff-45bb-a7fb-f08bcfb9adee" />
Object Information panel confirms Tier Zero: TRUE, Admin Count: TRUE, domain CORP.LOCAL, and group membership chain through Domain Users → Users. The underlying privilege escalation was applied directly via 
Add-ADGroupMember -Identity "Domain Admins" -Members svc_sql.
<img width="990" height="460" alt="privescpath_svc_sql_as_admin" src="https://github.com/user-attachments/assets/0615a0d8-1e56-41da-b560-8e5a197fb8b1" />


Impact: This is the finding that converts a single cracked service account password into complete domain compromise. Service accounts should never be granted high-privilege access.

Finding 6: Full Domain Controller Compromise Validated — Critical

The cracked svc_sql credentials were validated directly against the Domain Controller, confirming administrative access and code execution.

Evidence:

<img width="1920" height="1080" alt="cracked_system" src="https://github.com/user-attachments/assets/e9ca7e44-43e7-40dd-8466-ccecd4efaf84" />

crackmapexec smb 192.168.56.104 -u svc_sql -p [password] returned (Pwn3d!), CrackMapExec's confirmation of administrative access, and identified the target as SERVER2019, Windows Server 2019 Build 17763, domain corp.local

<img width="1920" height="312" alt="shell_in_windows" src="https://github.com/user-attachments/assets/7c4a97f5-f5e0-4466-8ffe-54fe31e8733a" />

 
psexec.py corp.local/svc_sql:[password]@192.168.56.104 returned an interactive shell with whoami confirming nt authority\system — the highest possible privilege level on the host


Impact: Complete compromise of the Domain Controller, and by extension the entire Active Directory domain. An attacker at this stage can create or modify any account, access any domain resource, and extract every credential in the domain via a DCSync attack.


# Conclusion

This assessment demonstrated two independent, fully realistic compromise paths using only publicly documented tools and techniques (Nmap, Impacket, BloodHound, CrackMapExec, John the Ripper). Neither path required custom exploit development. The Active Directory compromise in particular illustrates a pattern seen frequently in real-world engagements: a single over-privileged service account with a weak password is often sufficient to convert a standard, low-privilege domain user into full Domain Administrator.

