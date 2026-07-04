# Internal-Active-Directory-Penetration-Testing-Lab

# Project Overview

This project demonstrates an end-to-end internal Active Directory penetration test performed in a self-built virtual lab. 
The objective was to simulate a realistic internal network attack, beginning with the exploitation of a vulnerable Linux server 
and progressing through Active Directory enumeration, Kerberoasting, credential recovery, and Domain Controller compromise.

The lab was built entirely in VirtualBox using isolated networking to ensure a safe testing environment.

# Lab Environment
Machine	            Operating System	          Purpose
Kali Linux	        Kali Linux	                Attacker Machine
Windows Server 2019	Domain Controller	          Active Directory Server
Metasploitable2	    Ubuntu Linux	              Vulnerable Target


# Topology
Host Machine
      │
VirtualBox Host-Only Network
      │
├── Kali Linux          192.168.56.101
├── Windows Server 2019 192.168.56.104
└── Metasploitable2     192.168.56.102

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


