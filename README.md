# 🔍 Strace Guide: System Call Tracer

<p align="center">
  <img src="Strace_logo.svg" width="250">
</p>

## 📌 मूळ Concept
`strace` म्हणजे **System Call Tracer**. जेव्हा कोणताही program चालतो, तो Linux kernel ला requests पाठवतो — या requests ला **system calls** म्हणतात.


### 🛠️ strace आत कसं काम करतं
`strace` एक Linux kernel feature वापरतो — **ptrace()** system call.

```text
strace → ptrace(PTRACE_ATTACH, target_pid)
       → kernel target process ला pause करतो
       → प्रत्येक syscall वर strace ला notify करतो
       → strace log करतो → target process resume होतो
