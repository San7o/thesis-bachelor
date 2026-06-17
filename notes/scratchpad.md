# Scratchpad

## Objects

| Userspace |            Kernelspace              |
|   tools  <-> Frontends <-> buffers <-> backends |

Frontends: designed for the user

- trace-cmd
- bpftrace
- systemtap
- perf-tools
- bcc
- strace and ltrace use ptrace which is very slow and use a different architecture
- ...

Interfaces: bridge userspace to kernelspace

- systemcall
  - perf_event_open
  - bpf
  - ptrace (we will ignore this)
  - syslog
- pseudo filesysem: VFS based
   - tracefs: /sys/kernel/tracing, controls ftrace
   - debugfs: the precursor of tracefs
   - procfs
- network / socket based interfaces:
  - netlink sockets
- vendor specific stuff: /dev/nvidia*, ...


Tracing frameworks: consumers and managers of signals

- ftrace
- perf_event
- eBPF
- audit: uses netlink
  In brief, Every time a process invokes a system call:
  
  - Syscall Entry: The kernel allocates an internal audit_context structure
    bound to the current task struct (current->audit_context). This context acts
    as a temporary scratchpad to store system call arguments.
  - Execution: The system call runs. If an LSM module (like SELinux) blocks an
    action, or if a filesystem event occurs, it manually dumps auxiliary
    information directly into this active audit_context.
  - Syscall Exit: Right before returning control to userspace, the kernel
    evaluates the audit_context against the loaded audit rule filters (e.g.,
    "Did this match file path X or syscall Y?").
  
  If a rule matches, the kernel freezes the scratchpad data and generates a
  structured audit record string (including timestamps, serial numbers, UID,
  PID, comm name, and terminal details).

  - This record is pushed into an internal kernel Audit Queue (a netlink socket
    buffer).
  - The Panic Guard: If the system is under extreme load and the queue fills up,
    the audit subsystem acts according to a strict, pre-configured failure flag
    (audit_panic). It can either ignore the loss, throttle the calling
    application to let the logger catch up, or completely panic the entire
    kernel to prevent un-audited operations from executing.
    
  A background userspace daemon called auditd holds open the other end of the
  NETLINK_AUDIT socket.

  - The kernel wakes up auditd whenever new records are appended to the queue.
  - auditd drains the netlink buffer and commits the raw cryptographic-ready
    text records directly to disk, traditionally located at
    /var/log/audit/audit.log.
    
- vendor specific systems: intel PT and ARM coresignt, nvidia api
- out of tree:
  - systemtap
  - LTTng
  - ktap
  - dtrace4linux
  - DTrace
  - sysdig
  - ...

Event sources: the producers of a signal

- static tracing events (compile-time):
   - kernel space:
      - `tracepoints`: stable hooks with explicit arguments and string formatting
        formats. `include/trace/events`
      - `raw tracepoints`: zero-copy, high performance variants of tracepoints
      - `syscall tracepoints`: specialized for system calls
   - user space: these are implemented as uprobes, statically compiled as NOP
     instructions and registered dynamically
     - systemtap API: uses `sdt` probes
     - dtrace API: uses `usdt` probes (which is the DTrace name of sdt probes,
       but they are the same thing on linux)
       
- dynamic tracing events:
  - kernel space:
    - kprobes and kretprobes
    - function hooks: `fentry` / `fgraph` (used for exiting): Compiler assisted
        hooks (`mcount` / `__fentry__`), the underlying execution engine behind
        `ftrace`. Much faster than kprobes
    - `fprobe`: modern multi-probe wrapper built on ftrace to implement
      kprobe-like interface
  - user space:
    - `uprobes` and `uretprobes`.
    - `uprobe-multi`: The uretprobes are actually very slow
    and cause a double context switch, so we use `uprobe-multi`

- perf events:
  - software events: context switch etc
  - hardware events: CPU Performance Monitor Counters (cpu cycle, cahce misses...)
