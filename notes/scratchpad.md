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

Mcount / fentry
---------------

All thanks to `gcc -pg` + (`-mfentry` for x86).

From `include/asm-generic/vmlinux.lds.h:198`, we have a section named
`_mcount_loc`:

```
#define MCOUNT_REC()	. = ALIGN(8);				\
			__start_mcount_loc = .;			\
			KEEP(*(__mcount_loc))			\
			KEEP_PATCHABLE				\
			__stop_mcount_loc = .;			\
			FTRACE_STUB_HACK			\
			ftrace_ops_list_func = arch_ftrace_ops_list_func;
```

From `Documentation/trace/ftrace.rst:2985`:

```
A section called "__mcount_loc" is created that holds
references to all the mcount/fentry call sites in the .text section.
The recordmcount program re-links this section back into the
original object. The final linking stage of the kernel will add all these
references into a single table.
```

The way we call `mcount` is architecture dependent. For example, on `arm64` we
have `arch/arm64/kernel/entrty-ftrace.S:183`:

```
 * Gcc with -pg will put the following code in the beginning of each function:
 *      mov x0, x30
 *      bl _mcount
 *	[function's body ...]
 * "bl _mcount" may be replaced to "bl ftrace_caller" or NOP if dynamic
 * ftrace is enabled.
 *
```

And `arch/arm64/kernel/entry-ftrace.S:255`:

```
/*
 * _mcount() is used to build the kernel with -pg option, but all the branch
 * instructions to _mcount() are replaced to NOP initially at kernel start up,
 * and later on, NOP to branch to ftrace_caller() when enabled or branch to
 * NOP when disabled per-function base.
 */
SYM_FUNC_START(_mcount)
	ret
SYM_FUNC_END(_mcount)
EXPORT_SYMBOL(_mcount)
NOKPROBE(_mcount)

/*
 * void ftrace_caller(unsigned long return_address)
 * @return_address: return address to instrumented function
 *
 * This function is a counterpart of _mcount() in 'static' ftrace, and
 * makes calls to:
 *     - tracer function to probe instrumented function's entry,
 *     - ftrace_graph_caller to set up an exit hook
 */
SYM_FUNC_START(ftrace_caller)
	mcount_enter

	mcount_get_pc0	x0		//     function's pc
	mcount_get_lr	x1		//     function's lr

SYM_INNER_LABEL(ftrace_call, SYM_L_GLOBAL)	// tracer(pc, lr);
	nop				// This will be replaced with "bl xxx"
					// where xxx can be any kind of tracer.

#ifdef CONFIG_FUNCTION_GRAPH_TRACER
SYM_INNER_LABEL(ftrace_graph_call, SYM_L_GLOBAL) // ftrace_graph_caller();
	nop				// If enabled, this will be replaced
					// "b ftrace_graph_caller"
#endif

	mcount_exit
SYM_FUNC_END(ftrace_caller)
```

And on x86, `arch/x86/kernel/ftrace_64.S:155`:

```
SYM_FUNC_START(ftrace_caller)
	ANNOTATE_NOENDBR
	/* save_mcount_regs fills in first two parameters */
	save_mcount_regs

	CALL_DEPTH_ACCOUNT

	/* Stack - skipping return address of ftrace_caller */
	leaq MCOUNT_REG_SIZE+7(%rsp), %rcx
	movq %rcx, RSP(%rsp)

SYM_INNER_LABEL(ftrace_caller_op_ptr, SYM_L_GLOBAL)
	ANNOTATE_NOENDBR
	/* Load the ftrace_ops into the 3rd parameter */
	movq function_trace_op(%rip), %rdx

	/* regs go into 4th parameter */
	leaq (%rsp), %rcx

	/* Only ops with REGS flag set should have CS register set */
	movq $0, CS(%rsp)

	/* Account for the function call below */
	CALL_DEPTH_ACCOUNT

SYM_INNER_LABEL(ftrace_call, SYM_L_GLOBAL)
	ANNOTATE_NOENDBR
	call ftrace_stub

	/* Handlers can change the RIP */
	movq RIP(%rsp), %rax
	movq %rax, MCOUNT_REG_SIZE(%rsp)

	restore_mcount_regs

	/*
	 * The code up to this label is copied into trampolines so
	 * think twice before adding any new code or changing the
	 * layout here.
	 */
SYM_INNER_LABEL(ftrace_caller_end, SYM_L_GLOBAL)
	ANNOTATE_NOENDBR
	RET
SYM_FUNC_END(ftrace_caller);
```

`asm/x86/kernel/ftrace_64.S:333`

```
SYM_FUNC_START(__fentry__)
	ANNOTATE_NOENDBR
	CALL_DEPTH_ACCOUNT

	cmpq $ftrace_stub, ftrace_trace_function
	jnz trace
	RET

trace:
	/* save_mcount_regs fills in first two parameters */
	save_mcount_regs

	/*
	 * When DYNAMIC_FTRACE is not defined, ARCH_SUPPORTS_FTRACE_OPS is not
	 * set (see include/asm/ftrace.h and include/linux/ftrace.h).  Only the
	 * ip and parent ip are used and the list function is called when
	 * function tracing is enabled.
	 */
	movq ftrace_trace_function, %r8
	CALL_NOSPEC r8
	restore_mcount_regs

	jmp ftrace_stub
SYM_FUNC_END(__fentry__)
EXPORT_SYMBOL(__fentry__)
```

We setup a trampoline at `arch/x86/kernel/ftrace.c:308`:

```
static unsigned long
create_trampoline(struct ftrace_ops *ops, unsigned int *tramp_size)
```

Dynamic Ftrace
--------------

With dynamic tracing enabled, the kernel patches calls to mcount with a NOP if
tracing is disabled, and patches it to `ftrace_caller` or `ftrace_regs_caller`
if it is enabled. This allows for 0 overhead tracing in case it is disabled.

So `mcount` is actually never called with dynamic ftrace!!

Meaning:

- dynamic tracing: we use `ftrace_caller` or `NOP`
- static tracing: we use `__fentry__` on x86, or `mcount` on the others

Syscall tracing
---------------

When we define a syscall, for example at `kernel/fork.c:2825`:

```
SYSCALL_DEFINE0(fork)
{
  // ...
}
```

The macro expands to `arch/x86/include/asm/syscall_wrapper.h:244`:

```
#define SYSCALL_DEFINE0(sname)						\
	SYSCALL_METADATA(_##sname, 0);					\
	static long __do_sys_##sname(const struct pt_regs *__unused);	\
	__X64_SYS_STUB0(sname)						\
	__IA32_SYS_STUB0(sname)						\
	static long __do_sys_##sname(const struct pt_regs *__unused)
```

Notice `SYSCALL_METADATA`, which expands to `include/linux/syscalls.h:178`:

```
#define SYSCALL_METADATA(sname, nb, ...)			\
	static const char *types_##sname[] = {			\
		__MAP(nb,__SC_STR_TDECL,__VA_ARGS__)		\
	};							\
	static const char *args_##sname[] = {			\
		__MAP(nb,__SC_STR_ADECL,__VA_ARGS__)		\
	};							\
	SYSCALL_TRACE_ENTER_EVENT(sname);			\
	SYSCALL_TRACE_EXIT_EVENT(sname);			\
	static struct syscall_metadata __used			\
	  __syscall_meta_##sname = {				\
		.name 		= "sys"#sname,			\
		.syscall_nr	= -1,	/* Filled in at boot */	\
		.nb_args 	= nb,				\
		.types		= nb ? types_##sname : NULL,	\
		.args		= nb ? args_##sname : NULL,	\
		.enter_event	= &event_enter_##sname,		\
		.exit_event	= &event_exit_##sname,		\
		.enter_fields	= LIST_HEAD_INIT(__syscall_meta_##sname.enter_fields), \
	};							\
	static struct syscall_metadata __used			\
	  __section("__syscalls_metadata")			\
	 *__p_syscall_meta_##sname = &__syscall_meta_##sname;
```

Here the metadata is saved in the `__syscalls_metadata` section. This is used by
the syscall tracer `kernel/trace/trace_syscalls.c:1298`:

```
struct trace_event_functions enter_syscall_print_funcs = {
	.trace		= print_syscall_enter,
};
```

Which is called when an user reads the trace buffer, formats it with the
metadata and prints it.

The raw event is registered after a syscall via this sequence:

```
entry_SYSCALL_64
   do_syscall_64
      syscall_enter_from_user_mode
         syscall_trace_enter
            trace_syscall_enter
              trace_sys_enter
```

Which is a tracepoint (`include/trace/events/syscalls.h`):

```
TRACE_EVENT_SYSCALL(sys_enter,

	TP_PROTO(struct pt_regs *regs, long id),

	TP_ARGS(regs, id),

	TP_STRUCT__entry(
		__field(	long,		id		)
		__array(	unsigned long,	args,	6	)
	),

	TP_fast_assign(
		__entry->id	= id;
		syscall_get_arguments(current, regs, __entry->args);
	),

	TP_printk("NR %ld (%lx, %lx, %lx, %lx, %lx, %lx)",
		  __entry->id,
		  __entry->args[0], __entry->args[1], __entry->args[2],
		  __entry->args[3], __entry->args[4], __entry->args[5]),

	syscall_regfunc, syscall_unregfunc
);
```
