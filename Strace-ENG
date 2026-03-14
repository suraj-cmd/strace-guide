# How strace Works

## Core Concept

strace stands for **System Call Tracer**. When any program runs, it sends requests to the Linux kernel — these requests are called **system calls**. strace intercepts all these calls and shows them to you.

```
Your Program
     ↓  (function call)
  glibc / libc
     ↓  (system call)
  Linux Kernel        ← strace sits here and records everything
     ↓
  Hardware
```

---

## How strace Works Internally

strace uses a Linux kernel feature — the **ptrace()** system call.

```
strace → ptrace(PTRACE_ATTACH, target_pid)
       → kernel pauses the target process
       → notifies strace on every syscall
       → strace logs it → target process resumes
```

All of this happens so fast that the program doesn't feel it — it just slows down a little.

---

## Simple Example — `cat` command trace

```bash
strace cat /etc/hostname
```

Output looks like this:
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

**What each line means:**

| Line | Meaning |
|---|---|
| `execve(...)` | Program started |
| `openat(...) = 3` | File opened, got file descriptor 3 |
| `read(3, "myserver\n", 4096) = 9` | Read 9 bytes from fd 3 |
| `write(1, "myserver\n", 9) = 9` | Wrote 9 bytes to screen (fd 1 = stdout) |
| `close(3)` | File closed |
| `exit_group(0)` | Program ended, exit code 0 |

---

## Real-world Examples

### 1. See where a file is being searched

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

**Use:** To find "where is the config file being loaded from?"

---

### 2. Watch network calls

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

**Use:** To find "which server is the program connecting to?"

---

### 3. Debug permission errors

```bash
strace -e trace=openat,access myapp 2>&1 | grep "ENOENT\|EACCES\|EPERM"
```

Output:
```
openat(AT_FDCWD, "/app/config.yaml", O_RDONLY) = -1 ENOENT (No such file)
openat(AT_FDCWD, "/etc/myapp/config.yaml", O_RDONLY) = -1 EACCES (Permission denied)
```

**Use:** "Why is the app crashing?" — missing file or permission issue shows up immediately.

---

### 4. Attach to a running process

```bash
# First find the PID
ps aux | grep nginx
# Say PID is 1234

strace -p 1234 -o /tmp/nginx_trace.log
```

**Use:** Debug a live process in production — without restarting it.

---

### 5. Check timing — which call is slow?

```bash
strace -T -tt wget https://example.com 2>&1 | sort -t= -k2 -rn | head -10
```

Output (`-T` means time taken per call):
```
recvfrom(5, "<!doctype html>...", 16384) = 1234  <0.523481>
connect(5, {sin_addr="93.184.216.34"}, 16)       <0.234123>
read(3, "...", 4096)                              <0.001234>
```

**Use:** "Why is the program slow?" — `<0.523481>` means that call took 523ms.

---

## Syscall Categories

```bash
# Only file operations
strace -e trace=file myapp

# Only network
strace -e trace=network myapp

# Only process operations
strace -e trace=process myapp

# Only memory
strace -e trace=memory myapp

# Multiple together
strace -e trace=file,network myapp
```

---

## Most Useful Flags Combined

```bash
strace \
  -f \           # trace child processes too
  -tt \          # timestamp (HH:MM:SS.microseconds)
  -T \           # how long each call took
  -s 256 \       # show strings up to 256 chars (default 32)
  -o trace.log \ # save to file
  ./myapp
```

---

## How to Read the Output

```
14:23:01.123456 openat(AT_FDCWD, "/etc/passwd", O_RDONLY) = 3 <0.000234>
│              │                                           │    │
│              │                                           │    └── time taken
│              │                                           └── return value (3=fd, -1=error)
│              └── syscall and arguments
└── timestamp
```

**Return value -1 means error:**
```
openat(AT_FDCWD, "/missing/file", O_RDONLY) = -1 ENOENT (No such file or directory)
```

---

## In Summary

```
strace = translator between your program and the kernel

You tell strace   → "tell me what this program is doing"
strace tells kernel → "notify me on every syscall"
kernel tells strace → "did openat, read, write..."
strace shows you  → formatted log
```

strace is the most powerful tool for production debugging — you can see exactly what a program is doing internally without looking at any source code.

---

# What Each Syscall Does — In Detail

## What is a Syscall?

```
User Space (your program)          Kernel Space
─────────────────────────         ─────────────────
printf("hello")          →        write(1, "hello", 5)
fopen("file.txt")        →        openat(AT_FDCWD, "file.txt")
malloc(1024)             →        brk() / mmap()
sleep(1)                 →        nanosleep()
fork()                   →        clone()
```

Your program **does not touch hardware directly** — it goes through the kernel. strace sits at this kernel entry point.

---

## Category 1 — File Syscalls

### `openat` — Opening a File

```bash
strace -e trace=openat cat /etc/passwd
```

```
openat(AT_FDCWD, "/etc/passwd", O_RDONLY) = 3
│       │          │              │          │
│       │          │              │          └── got fd=3 (0=stdin,1=stdout,2=stderr)
│       │          │              └── flags: Read Only
│       │          └── file path
│       └── current working directory
└── syscall name
```

**Flag meanings:**

| Flag | Meaning |
|---|---|
| `O_RDONLY` | Read only |
| `O_WRONLY` | Write only |
| `O_RDWR` | Read + write |
| `O_CREAT` | Create new file |
| `O_APPEND` | Add to end |
| `O_TRUNC` | Empty the file |

**If there's an error:**
```
openat(AT_FDCWD, "/missing.txt", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/root/secret", O_RDONLY) = -1 EACCES (Permission denied)
```

---

### `read` — Reading from a File

```bash
strace -e trace=read cat /etc/hostname
```

```
read(3, "myserver\n", 4096) = 9
│    │   │             │      │
│    │   │             │      └── actually read 9 bytes
│    │   │             └── buffer size (requested 4096 bytes)
│    │   └── data that was read
│    └── fd=3 (given earlier by openat)
└── syscall
```

**Read states:**

```
read(3, "data...", 4096) = 4096   # buffer full, more data may exist
read(3, "end\n",   4096) = 4      # last 4 bytes
read(3, "",        4096) = 0      # EOF — file finished
read(3, "",        4096) = -1 EAGAIN  # non-blocking, no data yet
```

---

### `write` — Writing

```bash
strace -e trace=write echo "hello world"
```

```
write(1, "hello world\n", 12) = 12
│    │   │                │    │
│    │   │                │    └── wrote 12 bytes
│    │   │                └── wanted to write 12 bytes
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

### `close` — Closing a File

```
close(3) = 0      # success
close(9) = -1 EBADF  # fd 9 doesn't exist
```

---

### `stat` / `fstat` — File Info

```bash
strace -e trace=stat ls -la
```

```
stat("/etc/passwd", {
    st_mode  = S_IFREG|0644,    # regular file, permissions rw-r--r--
    st_size  = 2847,             # 2847 bytes
    st_uid   = 0,                # root's file
    st_mtime = 1710000000,       # last modified time
}) = 0
```

---

### `lseek` — Change Position in File

```
lseek(3, 0, SEEK_SET)  = 0      # go to beginning of file
lseek(3, 100, SEEK_CUR) = 150   # current position + 100
lseek(3, 0, SEEK_END)  = 2847   # go to end of file
```

---

## Category 2 — Process Syscalls

### `execve` — Execute a New Program

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

# Then bash executes ls:
execve("/bin/ls", ["ls"], envp=[...]) = 0
```

---

### `fork` / `clone` — Create a New Process

```bash
strace -e trace=clone bash -c "sleep 5 &"
```

```
clone(child_stack=NULL,
      flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD,
      ...) = 12345
│                                                      │
│                                                      └── child PID = 12345
└── return in parent process

# In child process return value = 0
```

---

### `wait4` / `waitpid` — Wait for Child Process to Finish

```
wait4(12345, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 12345
│     │       │                                                    │
│     │       │                                                    └── child PID returned
│     │       └── exit status: normally exited, code=0
│     └── wait for this PID
└── syscall
```

---

### `exit_group` — Program Closing

```
exit_group(0)   # 0 = success
exit_group(1)   # 1 = error
exit_group(139) # 139 = segfault (signal 11 + 128)
```

---

## Category 3 — Memory Syscalls

### `brk` — Growing Heap Memory

```bash
strace -e trace=brk python3 -c "x = [1]*1000000"
```

```
brk(NULL)          = 0x55a3b2000000   # where is current heap end?
brk(0x55a3b2021000) = 0x55a3b2021000  # grew heap by 0x21000 bytes
brk(0x55a3b2042000) = 0x55a3b2042000  # grew more
```

---

### `mmap` — Mapping Memory

```
mmap(NULL,          # kernel will decide address
     4096,          # need 4096 bytes
     PROT_READ|PROT_WRITE,  # read+write permission
     MAP_PRIVATE|MAP_ANONYMOUS,  # private, not a file
     -1,            # fd=-1 (no file)
     0)             # offset
= 0x7f8b2c000000   # address received

# File mmap (shared library load):
mmap(NULL, 183248,
     PROT_READ,
     MAP_PRIVATE,
     3,             # fd=3 (libc.so file)
     0)
= 0x7f8b2b000000
```

---

### `munmap` — Freeing Memory

```
munmap(0x7f8b2c000000, 4096) = 0
```

---

## Category 4 — Network Syscalls

### `socket` — Creating a Network Connection

```bash
strace -e trace=network curl http://example.com 2>&1 | head -20
```

```
socket(AF_INET,      # IPv4
       SOCK_STREAM,  # TCP (SOCK_DGRAM = UDP)
       IPPROTO_TCP)  # Protocol
= 5                  # got fd=5
```

---

### `connect` — Connecting to a Server

```
connect(5,
    {sa_family=AF_INET,
     sin_port=htons(80),           # port 80
     sin_addr=inet_addr("93.184.216.34")},  # IP address
    16)
= 0    # connected!

# If error:
connect(5, {...port=80...}, 16) = -1 ECONNREFUSED (Connection refused)
connect(5, {...port=443...}, 16) = -1 ETIMEDOUT   # blocked by firewall
```

---

### `sendto` / `send` — Sending Data

```
sendto(5,
    "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n",
    38,           # 38 bytes
    MSG_NOSIGNAL,
    NULL, 0)
= 38   # sent 38 bytes
```

---

### `recvfrom` / `recv` — Receiving Data

```
recvfrom(5,
    "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n...",
    16384,    # buffer size
    0,
    NULL, NULL)
= 1256   # received 1256 bytes
```

---

### `bind` + `listen` + `accept` — Server Side

```bash
strace -e trace=network python3 -m http.server 8080
```

```
socket(AF_INET, SOCK_STREAM, 0) = 3
setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0  # port reuse
bind(3, {sin_port=htons(8080), sin_addr="0.0.0.0"}, 16) = 0
listen(3, 5) = 0          # max 5 pending connections
accept(3, ...) = 4        # client arrived, got fd=4
read(4, "GET / HTTP/1.1\r\n", 8192) = 16   # read request
write(4, "HTTP/1.0 200 OK\r\n", ...) = ...  # sent response
close(4) = 0              # connection closed
```

---

## Category 5 — Signal Syscalls

### `kill` — Sending a Signal

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

### `sigaction` — Registering a Signal Handler

```
rt_sigaction(SIGINT,          # Ctrl+C signal
    {sa_handler=0x401234,     # handler function address
     sa_mask=[],
     sa_flags=SA_RESTORER},
    NULL, 8) = 0
```

---

## Real Debugging Example — Why is the App Crashing?

```bash
strace -f -tt -T -s 512 ./myapp 2>&1 | tee debug.log
```

**Step 1: Find the error**
```bash
grep "= -1" debug.log
```
```
openat(AT_FDCWD, "/etc/myapp/config.yml", O_RDONLY) = -1 ENOENT
openat(AT_FDCWD, "/app/database.sock", O_RDWR)      = -1 ENOENT
connect(5, {sin_addr="192.168.1.100", port=5432})   = -1 ECONNREFUSED
```

**Step 2: Find the slow call**
```bash
grep -E "<[0-9]+\.[0-9]+>" debug.log | sort -t'<' -k2 -rn | head -5
```
```
read(5, "data...", 65536)  = 1024  <5.234123>   # 5 seconds! DB is slow
connect(5, {port=5432}, 16) = 0    <2.100000>   # 2 second connection
write(1, "log...", 100)    = 100   <0.000012>   # fast
```

**Step 3: See the summary**
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

In short — **any program = a sequence of syscalls**. strace shows you that sequence in plain text. Even without source code, you can see 100% of what a program is doing internally.
