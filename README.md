# Strace कसं काम करतं

![Strace Logo](Strace_logo.svg)

## मूळ concept

strace म्हणजे **System Call Tracer**. जेव्हा कोणताही program चालतो, तो Linux kernel ला requests पाठवतो — या requests ला **system calls** म्हणतात. strace या सगळ्या calls intercept करून तुला दाखवतो.

```
Your Program
     ↓  (function call)
  glibc / libc
     ↓  (system call)
  Linux Kernel        ← strace इथे बसून सगळं record करतो
     ↓
  Hardware
```

---

## strace आत कसं काम करतं

strace एक Linux kernel feature वापरतो — **ptrace()** system call.

```
strace → ptrace(PTRACE_ATTACH, target_pid)
       → kernel target process ला pause करतो
       → प्रत्येक syscall वर strace ला notify करतो
       → strace log करतो → target process resume होतो
```

हे सगळं इतक्या fast होतं की program ला जाणवत नाही — फक्त थोडा slow होतो.

---

## Simple example — `cat` command trace

```bash
strace cat /etc/hostname
```

Output असं दिसेल:
```
execve("/bin/cat", ["cat", "/etc/hostname"], ...) = 0
brk(NULL)                               = 0x55a3b2e1000
openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
fstat(3, {st_mode=S_IFREG, st_size=12}) = 0
read(3, "myserver\n", 4096)             = 9
write(1, "myserver\n", 9)               = 9
close(3)                                = 0
exit_group(0)                           = ?
```

**प्रत्येक line चा अर्थ:**

| Line | म्हणजे काय |
|---|---|
| `execve(...)` | Program start झाला |
| `openat(...) = 3` | File उघडली, file descriptor 3 मिळाला |
| `read(3, "myserver\n", 4096) = 9` | fd 3 मधून 9 bytes वाचले |
| `write(1, "myserver\n", 9) = 9` | Screen वर (fd 1 = stdout) 9 bytes लिहिले |
| `close(3)` | File बंद केली |
| `exit_group(0)` | Program संपला, exit code 0 |

---

## Real-world examples

### 1. File कुठे शोधतोय ते बघा

```bash
strace -e trace=openat ls 2>&1 | head -20
```

Output:
```
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libselinux.so.1", O_RDONLY) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY) = 3
openat(AT_FDCWD, ".", O_RDONLY|O_DIRECTORY) = 3
```

**उपयोग:** "Config file कुठून load होतोय?" हे शोधायला.

---

### 2. Network calls बघा

```bash
strace -e trace=network curl google.com 2>&1 | grep -E "socket|connect|send|recv"
```

Output:
```
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 5
connect(5, {sa_family=AF_INET, sin_port=htons(80), sin_addr="142.250.67.78"}, 16) = 0
sendto(5, "GET / HTTP/1.1\r\nHost: google.com"..., 75, ...) = 75
recvfrom(5, "HTTP/1.1 301 Moved Permanently\r\n"..., 16384, ...) = 471
```

**उपयोग:** "Program कुठल्या server ला connect होतोय?" हे शोधायला.

---

### 3. Permission error debug करा

```bash
strace -e trace=openat,access myapp 2>&1 | grep "ENOENT\|EACCES\|EPERM"
```

Output:
```
openat(AT_FDCWD, "/app/config.yaml", O_RDONLY) = -1 ENOENT (No such file)
openat(AT_FDCWD, "/etc/myapp/config.yaml", O_RDONLY) = -1 EACCES (Permission denied)
```

**उपयोग:** "App का crash होतोय?" — file missing किंवा permission issue लगेच दिसतो.

---

### 4. Running process ला attach करा

```bash
# आधी PID शोधा
ps aux | grep nginx
# PID समजा 1234 आहे

strace -p 1234 -o /tmp/nginx_trace.log
```

**उपयोग:** Production मध्ये live process debug करायला — restart न करता.

---

### 5. Timing बघा — slow call कोणता?

```bash
strace -T -tt wget https://example.com 2>&1 | sort -t= -k2 -rn | head -10
```

Output (`-T` म्हणजे प्रत्येक call चा वेळ):
```
recvfrom(5, "<!doctype html>...", 16384) = 1234  <0.523481>
connect(5, {sin_addr="93.184.216.34"}, 16)       <0.234123>
read(3, "...", 4096)                              <0.001234>
```

**उपयोग:** "Program slow का आहे?" — `<0.523481>` म्हणजे 523ms लागले त्या call ला.

---

## Syscall categories

```bash
# फक्त file operations
strace -e trace=file myapp

# फक्त network
strace -e trace=network myapp

# फक्त process operations
strace -e trace=process myapp

# फक्त memory
strace -e trace=memory myapp

# सगळे एकत्र
strace -e trace=file,network myapp
```

---

## Most useful flags एकत्र

```bash
strace \
  -f \          # child processes पण trace कर
  -tt \         # timestamp (HH:MM:SS.microseconds)
  -T \          # प्रत्येक call किती वेळ लागला
  -s 256 \      # string 256 chars पर्यंत दाखव (default 32)
  -o trace.log \ # file मध्ये save कर
  ./myapp
```

---

## Output वाचायचं कसं

```
14:23:01.123456 openat(AT_FDCWD, "/etc/passwd", O_RDONLY) = 3 <0.000234>
│              │                                           │    │
│              │                                           │    └── किती वेळ लागला
│              │                                           └── return value (3=fd, -1=error)
│              └── syscall आणि arguments
└── timestamp
```

**Return value -1 म्हणजे error:**
```
openat(AT_FDCWD, "/missing/file", O_RDONLY) = -1 ENOENT (No such file or directory)
```

---

## थोडक्यात

```
strace = program आणि kernel मधला दुभाषी

तू strace ला सांगतोस → "हा program काय करतोय ते सांग"
strace kernel ला सांगतो → "मला notify कर प्रत्येक syscall वर"
kernel सांगतो strace ला → "openat केला, read केला, write केला..."
strace तुला दाखवतो → formatted log
```

Production debugging साठी strace हे सर्वात powerful tool आहे — कोणताही source code न बघता program आतून काय करतोय ते दिसतं.

# प्रत्येक Syscall काय करतो — Detail मध्ये

## Syscall म्हणजे काय?

```
User Space (तुझा program)          Kernel Space
─────────────────────────         ─────────────────
printf("hello")          →        write(1, "hello", 5)
fopen("file.txt")        →        openat(AT_FDCWD, "file.txt")
malloc(1024)             →        brk() / mmap()
sleep(1)                 →        nanosleep()
fork()                   →        clone()
```

तुझा program **directly hardware ला touch करत नाही** — kernel मार्फत जातो. strace हे kernel entry point वर बसतो.

---

## Category 1 — File Syscalls

### `openat` — File उघडणे

```bash
strace -e trace=openat cat /etc/passwd
```

```
openat(AT_FDCWD, "/etc/passwd", O_RDONLY) = 3
│       │          │              │          │
│       │          │              │          └── fd=3 मिळाला (0=stdin,1=stdout,2=stderr)
│       │          │              └── flags: Read Only
│       │          └── file path
│       └── current working directory
└── syscall नाव
```

**Flags चे अर्थ:**

| Flag | अर्थ |
|---|---|
| `O_RDONLY` | फक्त वाचणे |
| `O_WRONLY` | फक्त लिहिणे |
| `O_RDWR` | वाचणे + लिहिणे |
| `O_CREAT` | नवीन file बनव |
| `O_APPEND` | शेवटी add कर |
| `O_TRUNC` | file रिकामी कर |

**Error असेल तर:**
```
openat(AT_FDCWD, "/missing.txt", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/root/secret", O_RDONLY) = -1 EACCES (Permission denied)
```

---

### `read` — File मधून वाचणे

```bash
strace -e trace=read cat /etc/hostname
```

```
read(3, "myserver\n", 4096) = 9
│    │   │             │      │
│    │   │             │      └── प्रत्यक्षात 9 bytes वाचले
│    │   │             └── buffer size (4096 bytes मागितले)
│    │   └── वाचलेला data
│    └── fd=3 (आधी openat ने दिलेला)
└── syscall
```

**Read चे states:**

```
read(3, "data...", 4096) = 4096   # buffer भरला, अजून data असेल
read(3, "end\n",   4096) = 4      # शेवटचे 4 bytes
read(3, "",        4096) = 0      # EOF — file संपली
read(3, "",        4096) = -1 EAGAIN  # non-blocking, data नाही अजून
```

---

### `write` — लिहिणे

```bash
strace -e trace=write echo "hello world"
```

```
write(1, "hello world\n", 12) = 12
│    │   │                │    │
│    │   │                │    └── 12 bytes लिहिले
│    │   │                └── 12 bytes लिहायचे होते
│    │   └── actual data
│    └── fd=1 (stdout)
└── syscall

# fd numbers:
# 0 = stdin  (keyboard)
# 1 = stdout (screen)
# 2 = stderr (error screen)
# 3+ = files, sockets
```

---

### `close` — File बंद करणे

```
close(3) = 0      # यशस्वी
close(9) = -1 EBADF  # fd 9 अस्तित्वात नाही
```

---

### `stat` / `fstat` — File info

```bash
strace -e trace=stat ls -la
```

```
stat("/etc/passwd", {
    st_mode  = S_IFREG|0644,    # regular file, permissions rw-r--r--
    st_size  = 2847,             # 2847 bytes
    st_uid   = 0,                # root चा file
    st_mtime = 1710000000,       # last modified time
}) = 0
```

---

### `lseek` — File मध्ये position बदल

```
lseek(3, 0, SEEK_SET)  = 0      # file च्या सुरुवातीला जा
lseek(3, 100, SEEK_CUR) = 150   # current position + 100
lseek(3, 0, SEEK_END)  = 2847   # file च्या शेवटी जा
```

---

## Category 2 — Process Syscalls

### `execve` — नवीन program execute

```bash
strace -e trace=execve bash -c "ls"
```

```
execve("/bin/bash", ["bash", "-c", "ls"], envp=[...]) = 0
│       │            │                    │
│       │            │                    └── environment variables
│       │            └── arguments array
│       └── program path
└── syscall

# नंतर bash ls execute करतो:
execve("/bin/ls", ["ls"], envp=[...]) = 0
```

---

### `fork` / `clone` — नवीन process बनवणे

```bash
strace -e trace=clone bash -c "sleep 5 &"
```

```
clone(child_stack=NULL,
      flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD,
      ...) = 12345
│                                                      │
│                                                      └── child PID = 12345
└── parent process मध्ये return

# child process मध्ये return value = 0
```

---

### `wait4` / `waitpid` — Child process संपण्याची वाट बघणे

```
wait4(12345, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 12345
│     │       │                                                    │
│     │       │                                                    └── child PID परत
│     │       └── exit status: normally exited, code=0
│     └── wait for this PID
└── syscall
```

---

### `exit_group` — Program बंद होणे

```
exit_group(0)   # 0 = success
exit_group(1)   # 1 = error
exit_group(139) # 139 = segfault (signal 11 + 128)
```

---

## Category 3 — Memory Syscalls

### `brk` — Heap memory वाढवणे

```bash
strace -e trace=brk python3 -c "x = [1]*1000000"
```

```
brk(NULL)          = 0x55a3b2000000   # current heap end कुठे आहे?
brk(0x55a3b2021000) = 0x55a3b2021000  # heap 0x21000 bytes वाढवला
brk(0x55a3b2042000) = 0x55a3b2042000  # आणखी वाढवला
```

---

### `mmap` — Memory map करणे

```
mmap(NULL,          # address kernel ठरवेल
     4096,          # 4096 bytes हवेत
     PROT_READ|PROT_WRITE,  # read+write permission
     MAP_PRIVATE|MAP_ANONYMOUS,  # private, file नाही
     -1,            # fd=-1 (file नाही)
     0)             # offset
= 0x7f8b2c000000   # मिळालेला address

# File mmap (shared library load):
mmap(NULL, 183248,
     PROT_READ,
     MAP_PRIVATE,
     3,             # fd=3 (libc.so file)
     0)
= 0x7f8b2b000000
```

---

### `munmap` — Memory free करणे

```
munmap(0x7f8b2c000000, 4096) = 0
```

---

## Category 4 — Network Syscalls

### `socket` — Network connection बनवणे

```bash
strace -e trace=network curl http://example.com 2>&1 | head -20
```

```
socket(AF_INET,      # IPv4
       SOCK_STREAM,  # TCP (SOCK_DGRAM = UDP)
       IPPROTO_TCP)  # Protocol
= 5                  # fd=5 मिळाला
```

---

### `connect` — Server ला connect होणे

```
connect(5,
    {sa_family=AF_INET,
     sin_port=htons(80),           # port 80
     sin_addr=inet_addr("93.184.216.34")},  # IP address
    16)
= 0    # connected!

# Error असेल तर:
connect(5, {...port=80...}, 16) = -1 ECONNREFUSED (Connection refused)
connect(5, {...port=443...}, 16) = -1 ETIMEDOUT   # firewall ने block केला
```

---

### `sendto` / `send` — Data पाठवणे

```
sendto(5,
    "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n",
    38,           # 38 bytes
    MSG_NOSIGNAL,
    NULL, 0)
= 38   # 38 bytes पाठवले
```

---

### `recvfrom` / `recv` — Data मिळवणे

```
recvfrom(5,
    "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n...",
    16384,    # buffer size
    0,
    NULL, NULL)
= 1256   # 1256 bytes मिळाले
```

---

### `bind` + `listen` + `accept` — Server side

```bash
strace -e trace=network python3 -m http.server 8080
```

```
socket(AF_INET, SOCK_STREAM, 0) = 3
setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0  # port reuse
bind(3, {sin_port=htons(8080), sin_addr="0.0.0.0"}, 16) = 0
listen(3, 5) = 0          # max 5 pending connections
accept(3, ...) = 4        # client आला, fd=4 मिळाला
read(4, "GET / HTTP/1.1\r\n", 8192) = 16   # request वाचली
write(4, "HTTP/1.0 200 OK\r\n", ...) = ...  # response पाठवला
close(4) = 0              # connection बंद
```

---

## Category 5 — Signal Syscalls

### `kill` — Signal पाठवणे

```bash
strace -e trace=kill kill -9 1234
```

```
kill(1234, SIGKILL) = 0
│    │      │
│    │      └── signal type
│    └── target PID
└── syscall
```

---

### `sigaction` — Signal handler register

```
rt_sigaction(SIGINT,          # Ctrl+C signal
    {sa_handler=0x401234,     # handler function address
     sa_mask=[],
     sa_flags=SA_RESTORER},
    NULL, 8) = 0
```

---

## Real Debugging Example — App crash का होतोय?

```bash
strace -f -tt -T -s 512 ./myapp 2>&1 | tee debug.log
```

**Step 1: Error शोधा**
```bash
grep "= -1" debug.log
```
```
openat(AT_FDCWD, "/etc/myapp/config.yml", O_RDONLY) = -1 ENOENT
openat(AT_FDCWD, "/app/database.sock", O_RDWR)      = -1 ENOENT
connect(5, {sin_addr="192.168.1.100", port=5432})   = -1 ECONNREFUSED
```

**Step 2: Slow call शोधा**
```bash
grep -E "<[0-9]+\.[0-9]+>" debug.log | sort -t'<' -k2 -rn | head -5
```
```
read(5, "data...", 65536)  = 1024  <5.234123>   # 5 seconds! DB slow आहे
connect(5, {port=5432}, 16) = 0    <2.100000>   # 2 seconds connection
write(1, "log...", 100)    = 100   <0.000012>   # fast
```

**Step 3: Summary बघा**
```bash
strace -c ./myapp
```
```
% time     seconds  usecs/call     calls    errors  syscall
──────────────────────────────────────────────────────────
 85.23      5.2341        5234         1            read
 12.45      0.7654         127         6         2  openat
  1.23      0.0756          12        63            write
  0.89      0.0547          54        10            mmap
──────────────────────────────────────────────────────────
```

---

## Quick Reference Card

```
FILE:      openat → read/write → close → lseek → stat
PROCESS:   execve → clone/fork → wait4 → exit_group
MEMORY:    brk → mmap → munmap → mprotect
NETWORK:   socket → bind → connect → send → recv → close
SIGNALS:   sigaction → kill → pause → rt_sigprocmask
IPC:       pipe → socketpair → shmget → semget
```

थोडक्यात — **कोणताही program = syscalls चा sequence**. strace तो sequence तुला plain text मध्ये दाखवतो. Source code नसला तरी program आतून काय करतोय ते 100% दिसतं.
