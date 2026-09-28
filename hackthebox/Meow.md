# Meow — HackTheBox Starting Point

**Platform:** HackTheBox — Starting Point  
**OS:** Linux  
**Difficulty:** Very Easy  
**Date:** 22 September 2026  
**Flag:** b40abdfe23665f766f9c61ecba8a4c19  

---

## Summary

Linux machine exposing a Telnet service on port 23 with no
authentication required. The root account accepts an empty
password, granting immediate root-level access to the system
without any exploit or privilege escalation needed.
The misconfiguration itself is the vulnerability.

**Attack chain:** nmap → Telnet (port 23) → root login
(empty password) → flag in /root/flag.txt

---

## Reconnaissance

### Port Scan

PORT STATE SERVICE VERSION
23/tcp open telnet Linux telnetd
Service Info: OS: Linux


Only one port open — port 23 Telnet.  
No SSH, no HTTP, no SMB, no FTP.  
The single attack surface is the Telnet service.

**Key observation:** a single open port means the entire
attack must go through Telnet. No additional enumeration
of other services needed.

---

## Enumeration

Only one port found — port 23 Telnet.
No web service, no SMB, no FTP to enumerate further.
The single attack surface is the Telnet service itself.

Common default credentials tested manually:
- admin : admin → FAILED
- admin : (empty) → FAILED  
- root : root → FAILED
- root : (empty password) → SUCCESS

---

## Exploitation

Telnet is a remote access protocol from the 1970s.
It transmits all data in cleartext with no encryption.
When misconfigured, it allows login with no password at all.

### Connect via Telnet

```bash
telnet 10.129.228.151
```

Trying 10.129.228.151...
Connected to 10.129.228.151.

Meow login: root
Password: (empty — just press Enter)

Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-77-generic x86_64)
Last login: Mon Sep 6 15:15:23 UTC 2021

root@Meow:~#


Root access obtained with no credentials required.
No exploit needed — the misconfiguration IS the vulnerability.
This is one of the most critical misconfigurations possible:
a service exposing root access to anyone on the network.

---

## Post-Exploitation

Root shell obtained directly through Telnet.
No privilege escalation needed — already root from the start.

```bash
whoami
# root

hostname
# Meow

uname -a
# Linux Meow 5.4.0-77-generic x86_64 GNU/Linux

ls /root/
# flag.txt  snap

cat /root/flag.txt
# b40abdfe23665f766f9c61ecba8a4c19
```

Root confirmed via whoami.
Flag located in /root/flag.txt — standard location for
HackTheBox root flags.

---

## Findings

| # | Vulnerability | Severity | CVE/CWE | Remediation |
|---|---|---|---|---|
| 1 | Telnet service exposed on port 23 | High | CWE-319 | Disable Telnet completely — replace with SSH (port 22, encrypted traffic) |
| 2 | Root login with empty password | Critical | CWE-521 | Disable root remote login, enforce password policy minimum 12 characters |
| 3 | No authentication required on Telnet | Critical | CWE-306 | If Telnet must remain, enforce strong credentials and restrict access by IP |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Reconnaissance | Active Scanning | T1595.001 |
| Initial Access | Exploit Public-Facing Application | T1190 |
| Initial Access | Valid Accounts: Default Accounts | T1078.001 |
| Lateral Movement | Remote Services: Telnet | T1021.004 |
| Discovery | System Information Discovery | T1082 |
| Collection | Data from Local System | T1005 |

```bash
sudo nmap -sV --min-rate 5000 -p- 10.129.228.151
```


Root access obtained with no credentials required.
No exploit needed — the misconfiguration IS the vulnerability.
This is one of the most critical misconfigurations possible:
a service exposing root access to anyone on the network.

---

## Post-Exploitation

Root shell obtained directly through Telnet.
No privilege escalation needed — already root from the start.

```bash
whoami
# root

hostname
# Meow

uname -a
# Linux Meow 5.4.0-77-generic x86_64 GNU/Linux

ls /root/
# flag.txt  snap

cat /root/flag.txt
# b40abdfe23665f766f9c61ecba8a4c19
```

Root confirmed via whoami.
Flag located in /root/flag.txt — standard location for
HackTheBox root flags.

---

## Findings

| # | Vulnerability | Severity | CVE/CWE | Remediation |
|---|---|---|---|---|
| 1 | Telnet service exposed on port 23 | High | CWE-319 | Disable Telnet completely — replace with SSH (port 22, encrypted traffic) |
| 2 | Root login with empty password | Critical | CWE-521 | Disable root remote login, enforce password policy minimum 12 characters |
| 3 | No authentication required on Telnet | Critical | CWE-306 | If Telnet must remain, enforce strong credentials and restrict access by IP |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Reconnaissance | Active Scanning | T1595.001 |
| Initial Access | Exploit Public-Facing Application | T1190 |
| Initial Access | Valid Accounts: Default Accounts | T1078.001 |
| Lateral Movement | Remote Services: Telnet | T1021.004 |
| Discovery | System Information Discovery | T1082 |
| Collection | Data from Local System | T1005 |

