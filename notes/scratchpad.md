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
        formats. `include/trace/events` (TRACE_EVENT, crazy macro, registers t
        ftrace and perf)
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

Ftrace
------

Structure:

- A bit of history
- architecture
- in-kernel api
- user space api

Good intro:
\url{https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/7/html/tuning_guide/appe-detailed_description_of_ftrace}

There is a naming issue as ftrace was born as the function tracer, but has
evolved into the main tracing infrastructure used by other types of tracers. We
will now talk about the general unfrastructure.

We write to buffer with \verb|trace_printk()|, which is a formatter what writes
to the buffer.  SEE: \verb|kernel/trace/trace_printk.c| modules
\verb|trace_printk()|'s formats are autosaved in \verb|struct trace_bprintk_fmt|
which are queued on \verb|trace_bprintk_fmt_list|.

An important abstraction is the struct tracer (kernel/trace/trace.h) which
represents a specific tracer and its callbacks to interact with tracefs. Then
you do \verb|tracer_init| and other \verb|tracer_*| functions.

trace.h:1450

   \verb|extern struct tracer nop_trace;|

From: \url{https://www.kernel.org/doc/html/latest/trace/events.html} Trace
events can be made to conditionally invoke trigger ‘commands’ which can take
various forms and are described in detail below; examples would be enabling or
disabling other trace events or invoking a stack trace whenever the trace event
is hit. Whenever a trace event with attached triggers is invoked, the set of
trigger commands associated with that event is invoked.

\begin{verbatim}

trace.h:1997
   struct event_command {

\end{verbatim}

Not that you also have event filtering.

The trace array contains per-CPU trace arrays. This is the highest level data
structure that indivisual tracers deal with.

Interesting: lockless ringbuffer design:
\url{https://www.kernel.org/doc/html/latest/trace/ring-buffer-design.html} The
ring buffer is divided in pages. Each CPU has its own list of pages where it can
write, and the reader may read from any per-cpu buffer. For this reason for each
per-cpu buffer, the reader has its own page separate from the rest. When the
reader has done reading it, it swaps it with another page. If the writer wraps,
it needs to push the head forward. Here we need to use cmpxchg since we need to
make sure that the writer and the reader are not changing the head at the same
time.  SEE: \verb|kernel/trace/ring_buffer.c|

There are several tracers here, the most famous one is the function
tracer.


Static Function Tracing aka ftrace pt1
--------------------------------------

`CONFIG_FUNCTION_TRACER`

All thanks to `gcc -pg`. It generates Mcount (ARM and others) or fentry (x86).

x86 legacy: alternatively, on x86_64 you can use `-mfentry` when
CONFIG_HAVE_FENTRY. With `-mnop-mcount` we use several `NOP` instructions
instead (CONFIG_HAVE_NOP_MCOUNT). Compiler explorer with `-pg` shows a call to
`_mcount` after the prologue in arm64, or `__fentry__` on x86-64 before the
prologue. Although clang generates a call to `mcount` for x86-64 after the
prologue, and a call to `_mcount` in armv8 (consistent with gcc).

On `arm64` we have `arch/arm64/kernel/entrty-ftrace.S:183`:

```
 * Gcc with -pg will put the following code in the beginning of each function:
 *      mov x0, x30
 *      bl _mcount
 *	[function's body ...]
 * "bl _mcount" may be replaced to "bl ftrace_caller" or NOP if dynamic
 * ftrace is enabled.
 *
```

We see `mcount` setup in arm64 `arch/arm64/kernel/entry-ftrace.S:255`:

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
```

We see `__fentry__` used in x86:

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


According to https://gcc.gnu.org/pipermail/gcc-patches/2018-July/502529.html,

```
On i386, the difference between mcount and fentry is that fentry comes before
the prolog.
```

Support for mcount was completely removed in x86 since https://lkml.org/lkml/2019/5/9/957:

```
There's two methods of enabling function tracing in Linux on x86. One is
with just "gcc -pg" and the other is "gcc -pg -mfentry". The former will use
calls to a special function "mcount" after the frame is set up in all C
functions. The latter will add calls to a special function called "fentry"
as the very first instruction of all C functions.

At compile time, there is a check to see if gcc supports, -mfentry, and if
it does, it will use that, because it is more versatile and less error prone
for function tracing.

Starting with v4.19, the minimum gcc supported to build the Linux kernel,
was raised to version 4.6. That also happens to be the first gcc version to
support -mfentry. Since on x86, using gcc versions from 4.6 and beyond will
unconditionally enable the -mfentry, it will no longer use mcount as the
method for inserting calls into the C functions of the kernel. This means
that there is no point in continuing to maintain mcount in x86.

Remove support for using mcount. This makes the code less complex, and will
also allow it to be simplified in the future.
```


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

Ftrace uses thse to access the mcount functions.

Dynamic Ftrace aka ftrace pt2
-----------------------------

`DYNAMIC_FTRACE` option.

very good explaination in `linux/Documentation/trace/ftrace.rst`.

With dynamic tracing enabled, at boot-time the kernel patches calls to mcount
with a NOP (or uses `-mnio-mcount`) if tracing is disabled, and patches it to
`ftrace_caller` or `ftrace_regs_caller` if it is enabled. This allows for 0
overhead tracing in case it is disabled.

The information of all the locations of all mcount calls is generated at compile
time via the `recordmcount` program which is called by the build infrastructure.
The tool saves all the locations in the `__mcount_loc` ELF section, which is
linked in the original object file.

Show an example, like
https://github.com/isovalent/ebpf-docs/blob/master/docs/linux/concepts/trampolines.md:

No tracing:

```
some_kernel_func:
  PUSH RBP
  MOV RSP, RBP
  ...
  RET
```

Tracing disabled:

```
# Symbol where instructions like: CALL some_kernel_func will jump to.
some_kernel_func:
  NOP
  NOP
# actual start of the function
  PUSH RBP
  MOV RSP, RBP
  ...
  RET
```

Tracing enabled:

```
# Symbol where instructions like: CALL some_kernel_func will jump to.
some_kernel_func:
  CALL trace_call
  NOP
# actual start of the function
  PUSH RBP
  MOV RSP, RBP
  [...]
  RET
```

So `mcount` is actually never called with dynamic ftrace!!

Meaning:

- dynamic tracing: we use `ftrace_caller`, which is implemented as a series of
  `NOP`, or as a call to `ftrace_stub`, whose address is changed dynamically
  based on the current tracer.
- static tracing: we use `__fentry__` on x86, or `mcount` on the others

We see `ftrace_caller` in arm64 `arch/arm64/kernel/entry-ftrace.S:267`:

```
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

We setup a trampoline at `arch/x86/kernel/ftrace.c:308`:

```
static unsigned long
create_trampoline(struct ftrace_ops *ops, unsigned int *tramp_size)
```

Trampolines are just locations in memory containing addresses of logic to jump
to.

Ftrace provides the function `register_ftrace_function` to register this
trampoline, which is used by the various tracers. If dynamic tracing is enabled,
this literally loops over all the functions, patching them via the `text_poke`
function.

Tracepoints
-----------

Aka. static event tracing.

Intro: https://www.kernel.org/doc/html/latest/trace/tracepoints.html

There are more than 1000 traceoints available (\verb|grep -R TRACE_EVENT include/trace/events|)


USEFUL: \url{https://mmi.hatenablog.com/entry/2018/03/04/052249}

TODO: Put an entire implementation of a simple module that registers a
tracepoint at the appendix.

`inclide/linux/tracepoint-defs.h:26` (actual full code):

```
struct tracepoint_func {
	void *func;
	void *data;
	int prio;
};

struct tracepoint_ext {
	int (*regfunc)(void);
	void (*unregfunc)(void);
	/* Flags. */
	unsigned int faultable:1;
};

struct tracepoint {
	const char *name;		/* Tracepoint name */
	struct static_key_false key;
	struct static_call_key *static_call_key;
	void *static_call_tramp;
	void *iterator;
	void *probestub;
	struct tracepoint_func __rcu *funcs;
	struct tracepoint_ext *ext;
};
```

The most relevant members of the `struct tracepoint` are its "name", which is
used by the user to identify this particular tracepoint, and one or more
functions in the "funcs" member to be called when the tracepoint is executed
(which are of type `struct tracepoint_func`). The "ext" member contains
functions to be called when registering or unregistering the tracepoint.
Additionally, the static-key and static-call related fields are performance
optimizations which will be briefly explained later in this section.

To create a tracepoint, the macro `DECLARE_TRACE` should be used. The macro
takes three arguments: the name of the tracepoint, the prototype of the function
that should be called when the tracepoint is executed, and the name of the
arguments what should be passed to that function (relevant to the specific scope
where the tracepoint is placed).

For example, consider the following macro statement:

```
DECLARE_TRACE(foo_tp,
  TP_PROTO(bool is_bar),
  TP_ARGS(bar));
```

This declares a tracepoint called `foo_tp` which calls a function that looks
like `foo_function(bool is_bar)` which it will be called with the `bar`
argument, like `foo_function(bar)`.

Consider now a real example from `include/trace/events/sched.h`:

```
#undef TRACE_SYSTEM
#define TRACE_SYSTEM sched

#if !defined(_TRACE_SCHED_H) || defined(TRACE_HEADER_MULTI_READ)
#define _TRACE_SCHED_H

#include <linux/tracepoint.h>

DECLARE_TRACE(sched_entry,
	TP_PROTO(bool preempt),
	TP_ARGS(preempt));

#endif /* _TRACE_SCHED_H */

/* This part must be outside protection */
#include <trace/define_trace.h>
```

In the first line, the `TRACE_SYSTEM` symbol specifies the subdirectory of
tracefs where this tracepoint will be registered by the user. The code follows
by including the file `linux/tracepoint.h` which contains the `DECLARE_TRACE`
macro definition, and proceeds to declare the `sched_entry` tracepoint. It
finally includes `trace/define_trace.h` which defines the various functions for
the tracepoint and static storage, such as functions for registration,
de-registration, calling the callback, and a function to check if the tracepoint
is active, among others. The generated function names would be, respectively:
`register_trace_sched_entry()`, `unregister_trace_sched_entry()`,
`trace_sched_entry()`, `trace_sched_entry_enabled()`.

It should be noted that the `define_trace.h` header needs to be placed outside
any inclusion guards so that it will be always included by the preprocessor.
This is used for a trick with the preprocessor where the `define_trace.h` header
includes the file that is including it, therefore defining itself again, but
using `#define` guards to change its definition. For example, if file "X"
includes `define_trace.h`, then it will include itself, "X", which will include
`define_trace.h` another time, and so on; this cycle may repeat multiple times,
until some condition is met. The reason is to redefine the `DECLARE_TRACE` macro
in different ways to generate different code. This is also used in other parts
of the kernel, namely "event tracing" which will be explained later in this
section.

This may not be necessary...
What happens in the macro expansion is that it creates a tracepoint:

```
#define DECLARE_TRACE(name, proto, args)	\
	DEFINE_TRACE(name##_tp, PARAMS(proto), PARAMS(args))
```

Then again it defines a bunch of functions for ftrace:

`include/linux/tracepoint.h:523`:

```
#define DECLARE_TRACE(name, proto, args)				\
	__DECLARE_TRACE(name##_tp, PARAMS(proto), PARAMS(args),		\
			cpu_online(raw_smp_processor_id()),		\
			PARAMS(void *__data, proto))
```

Declare a bunch of functions

```
#define __DECLARE_TRACE_COMMON(name, proto, args, data_proto)		\
	extern int __traceiter_##name(data_proto);			\
	DECLARE_STATIC_CALL(tp_func_##name, __traceiter_##name);	\
	extern struct tracepoint __tracepoint_##name;			\
	extern void rust_do_trace_##name(proto);			\
	static inline int						\
	register_trace_##name(void (*probe)(data_proto), void *data)	\
	{								\
		return tracepoint_probe_register(&__tracepoint_##name,	\
						(void *)probe, data);	\
	}								\
	static inline int						\
	register_trace_prio_##name(void (*probe)(data_proto), void *data,\
				   int prio)				\
	{								\
		return tracepoint_probe_register_prio(&__tracepoint_##name, \
					      (void *)probe, data, prio); \
	}								\
	static inline int						\
	unregister_trace_##name(void (*probe)(data_proto), void *data)	\
	{								\
		return tracepoint_probe_unregister(&__tracepoint_##name,\
						(void *)probe, data);	\
	}								\
	static inline void						\
	check_trace_callback_type_##name(void (*cb)(data_proto))	\
	{								\
	}								\
	static inline bool						\
	trace_##name##_enabled(void)					\
	{								\
		if (IS_ENABLED(CONFIG_LOCKDEP)) {			\
			WARN_ONCE(!rcu_is_watching(),			\
				  "RCU not watching for tracepoint");	\
		}							\
		return static_branch_unlikely(&__tracepoint_##name.key);\
	}

#define __DECLARE_TRACE(name, proto, args, cond, data_proto)			\
	__DECLARE_TRACE_COMMON(name, PARAMS(proto), PARAMS(args), PARAMS(data_proto)) \
	static inline void __do_trace_##name(proto)			\
	{								\
		TRACEPOINT_CHECK(name)					\
		if (cond) {						\
			guard(srcu_fast_notrace)(&tracepoint_srcu);	\
			__DO_TRACE_CALL(name, TP_ARGS(args));		\
		}							\
	}								\
	static inline void trace_##name(proto)				\
	{								\
		if (static_branch_unlikely(&__tracepoint_##name.key))	\
			__do_trace_##name(args);			\
		if (IS_ENABLED(CONFIG_LOCKDEP) && (cond)) {		\
			WARN_ONCE(!rcu_is_watching(),			\
				  "RCU not watching for tracepoint");	\
		}							\
	}								\
	static inline void trace_call__##name(proto)			\
	{								\
		__do_trace_##name(args);				\
	}
```

This macro defines the actual static call:

```
#ifdef TRACEPOINTS_ENABLED

#ifdef CONFIG_HAVE_STATIC_CALL
#define __DO_TRACE_CALL(name, args)					\
	do {								\
		struct tracepoint_func *it_func_ptr;			\
		void *__data;						\
		it_func_ptr =						\
			rcu_dereference_raw((&__tracepoint_##name)->funcs); \
		if (it_func_ptr) {					\
			__data = (it_func_ptr)->data;			\
			static_call(tp_func_##name)(__data, args);	\
		}							\
	} while (0)
#else
#define __DO_TRACE_CALL(name, args)	__traceiter_##name(NULL, args)
#endif /* CONFIG_HAVE_STATIC_CALL */
```

So we can use the tracepoint with the args specified in the definition:

```
trace_sched_entry_tp(sched_mode == SM_PREEMPT);
```

Sched does not register it, but it exports its symbol so it can be probed by
external modules, `kernel/sched/core.c:123`:

```
EXPORT_TRACEPOINT_SYMBOL_GPL(sched_exit_tp);
```

Which expands to `include/linux/tracepoint.h:426`:

```
#define EXPORT_TRACEPOINT_SYMBOL_GPL(name)				\
	TRACEPOINT_CHECK(name)						\
	EXPORT_SYMBOL_GPL(__tracepoint_##name);				\
	EXPORT_SYMBOL_GPL(__traceiter_##name);				\
	EXPORT_STATIC_CALL_GPL(tp_func_##name)
```

So we can access the tracepoints via, but actually these are not all...:

```
extern tracepoint_ptr_t __start___tracepoints_ptrs[];
extern tracepoint_ptr_t __stop___tracepoints_ptrs[];
```

In the code we place a tracepoint with `trace_sched_entry`
What happen is that:

```
trace_sched_entry
  __do_trace_sched_entry
      static_call(tp_func_sched_entry)
```

Static calls are introduced in https://lkml.iu.edu/2008.2/03685.html:

```
Static calls are a replacement for global function pointers. They use
code patching to allow direct calls to be used instead of indirect
calls. They give the flexibility of function pointers, but with
improved performance. This is especially important for cases where
retpolines would otherwise be used, as retpolines can significantly
impact performance.
```

According to https://lkml.org/lkml/2021/7/22/1297:

```
If there's just a single callback, the static_call is updated to just call
that callback directly. Once another handler is added, then the static
caller is updated to call the iterator, that simply loops over all the
funcs in the array and calls each of the callbacks like the old method
using indirect calling.
```

The `tp_func_sched_entry` is usually setup when registering the tracepoint
(`register_trace_##name` -> `tracepoint_probe_register` -> `tracepoint_add_func`
-> `tracepoint_update_call`). This can be disabled likewise. Note that when
registering a tracepoint we need to provide the function that will be statically
called. We will see with event tracing that this function is generated by the
infrastructure to interface with ftrace.

We could also do this:

```
if (trace_foo_enabled()) {
        x = bar();
        trace_call__foo(x);  // does not check if static function is null
}
```

To avoid a conditional branch instruction in the if statement to check if the
tracepoint is enabled, the code is patched at runtime via a mechanism called
`static_key` that replaces the if statement with a jump or a NOP instruction
depending on whether the event is active or not. Note that this is different
than `static_call` because, unlike `static_call`, it replaces a conditional
branch (compare + jump instructions) and not a function call. This decreases the
runtime performance impact of a disabled tracepoint, as the only penalty is
overall increased code section size due to the tracepoint generated functions.

Event tracing
-------------

To automatically register the tracepoint in the ftrace / perf inftrastructure,
we use `TRACE_EVENT` macro instead of `DEFINE_TRACE`. This is referred to as
"event tracing", not to be confused with "perf events" which are related to
perf.

Great exaplaination of the requirements here: https://lwn.net/Articles/379903/

We create a tracepoint event like this `fs/fuse/fuse_trace.h:126`:

```
TRACE_EVENT(fuse_request_end,
	TP_PROTO(const struct fuse_req *req),

	TP_ARGS(req),

	TP_STRUCT__entry(
		__field(dev_t,		connection)
		__field(uint64_t,	unique)
		__field(uint32_t,	len)
		__field(int32_t,	error)
	),

	TP_fast_assign(
		__entry->connection	=	req->chan->conn->dev;
		__entry->unique		=	req->in.h.unique;
		__entry->len		=	req->out.h.len;
		__entry->error		=	req->out.h.error;
	),

	TP_printk("connection %u req %llu len %u error %d", __entry->connection,
		  __entry->unique, __entry->len, __entry->error)
);
```

What happens is that there are actually 7 stages of this macro expansion
`include/trace/trace_events.h:38`:

```
#undef TRACE_EVENT
#define TRACE_EVENT(name, proto, args, tstruct, assign, print) \
	DECLARE_EVENT_CLASS(name,			       \
			     PARAMS(proto),		       \
			     PARAMS(args),		       \
			     PARAMS(tstruct),		       \
			     PARAMS(assign),		       \
			     PARAMS(print));		       \
	DEFINE_EVENT(name, name, PARAMS(proto), PARAMS(args));
```

The macro and its prameters are undefined and redefined 7 times, and the
tracepoint definition is reimported after each stage, to produce
different output. Each time, the source header is included again with different
expansion rules.

Also:

```
#undef TRACE_EVENT
#define TRACE_EVENT(name, proto, args, tstruct, assign, print)	\
	DEFINE_TRACE(name, PARAMS(proto), PARAMS(args))
```

More specifically, in stage 1 `inclide/trace/trace_evnets.h:39` the macro
generates something of the form:

```
struct trace_event_raw_<call> {
  struct trace_entry		ent;
  <type>				<item>;
  <type2>				<item2>[<len>];
  [...]
};
```

In stage 2 it creates:

```
struct trace_event_data_offsets_<call> {
  u32				<item1>;
  u32				<item2>;
  [...]
};
```

In stage 3 it creates the method to print the tracepoint:

```
enum print_line_t
trace_raw_output_<call>(struct trace_iterator *iter, int flags,
       struct trace_event *trace_event)
...

static struct trace_event_functions trace_event_type_funcs_##call = {	\
	.trace			= trace_raw_output_##call,		\
};
```

Stage 4:

```
static struct trace_event_fields trace_event_fields_##call[] = {	\
	tstruct								\
	{} };
```

Stage 5:

```
static inline notrace int trace_event_get_offsets_##call(		\
	struct trace_event_data_offsets_##call *__data_offsets, proto)	\
```

Stage 6 is similar to stage 5, but it creates a perf_trace_##call function.

```
static void trace_event_raw_event_<call>(void *__data, proto)
...

static struct trace_event ftrace_event_type_<call> = {
 .trace			= trace_raw_output_<call>, <-- stage 2
};

static inline void ftrace_test_probe_##call(void)			\
```

Stage 7:

```
static struct trace_event_class __used event_class_<template> = {
 .system			= "<system>",
 .fields_array		= trace_event_fields_<call>,
 .fields			= LIST_HEAD_INIT(event_class_##call.fields),
 .raw_init		= trace_event_raw_init,
 .probe			= trace_event_raw_event_##call,
 .reg			= trace_event_reg,
};

static struct trace_event_call event_<call> = {
  .class			= event_class_<template>,
  {
    .tp			= &__tracepoint_<call>,
  },
  .event			= &trace_event_type_funcs_<call>,
  .print_fmt		= print_fmt_<call>,
  .flags			= TRACE_EVENT_FL_TRACEPOINT,
};
// its only safe to use pointers when doing linker tricks to
// create an array.
static struct trace_event_call __used
__section("_ftrace_events") *__event_<call> = &event_<call>;
```

Since `TRACE_EVENT` generates a lot of boilerplate code and increases the size
of the kernel binary, we can use `DECLARE_EVENT_CLASS` and `DEFINE_EVENT` to
share some code. See https://lwn.net/Articles/381064/

At the end of the day, the infrastucture creates a tracepoint. When it triggers,
it commits raw bytes to the ring buffer.

...via `trace_event_raw_event_##call`.
The set of apis to interact with the ring buffer is located in
`include/linux/trace_events.h` which use `include/linux/ring_buffer.h` api and
`kernel/trace/ring_buffer.c` implementation.
Briefly:

- `trace_event_buffer_lock_reserve()`: Allocates space in the per-CPU ring
  buffer for the event structure defined by TP_STRUCT__entry.
- `ring_buffer_event_data()`: Returns a raw pointer to the payload data area
  inside the newly reserved ring buffer slot, allowing the TP_fast_assign block
  to copy values directly.
- `trace_event_buffer_commit()`: Finalizes the entry and makes it available to
  user-space trace readers.

Much more about the ring buffer: https://kernel-internals.org/tracing/ring-buffer/

When the user reads it, he calls `trace_raw_output_##call` ->
`trace_event_printf` -> `trace_seq_vprintf`. So it formats the string to a
`struct seq_buf` which is contained in a `struct trace_seq ` which is contained
in an `struct trace_iterator` which is passed to `trace_output_call`.

Syscall tracing
---------------

Special tracepoints are placed for systemcalls in order to trace their accesses.

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

Kprobes
-------

Main source of information:
\url{https://www.kernel.org/doc/html/latest/trace/kprobes.html}
COOL: \url{https://lwn.net/Articles/132196/}

Introduction
++++++++++++

Compiled in if ` CONFIG_KPROBES` is set.

There are kprobes and kretprobes.

From https://www.kernel.org/doc/html/latest/trace/kprobes.html:

```
When a kprobe is registered, Kprobes makes a copy of the probed instruction and
replaces the first byte(s) of the probed instruction with a breakpoint
instruction (e.g., int3 on i386 and x86_64).

When a CPU hits the breakpoint instruction, a trap occurs, the CPU’s registers
are saved, and control passes to Kprobes via the notifier_call_chain mechanism.
Kprobes executes the “pre_handler” associated with the kprobe, passing the
handler the addresses of the kprobe struct and the saved registers.

Next, Kprobes single-steps its copy of the probed instruction. (It would be
simpler to single-step the actual instruction in place, but then Kprobes would
have to temporarily remove the breakpoint instruction. This would open a small
time window when another CPU could sail right past the probepoint.)

After the instruction is single-stepped, Kprobes executes the “post_handler,” if
any, that is associated with the kprobe. Execution then continues with the
instruction following the probepoint.
```

Picture: show this graphically. Take inspiration to the lwn article.

Kretprobes do the same but by playing with the return address, which
is places pushed onto the stack by the caller.

Note that kprobes can do everything, which is very dangerous.

...Kprobes may perform jump optimization if the architecture supports
it and the kernel was built with \verb|CONFIG_OPTPROBES=y|. In this case,
instead of a breakpoint instruction, the CPU performs a jump instead.

Note that kprobes do not work well with inlining.

Interface
+++++++++

API is in include/linux/kprobes.h

\begin{verbatim}

    struct kprobe { ... }
    typedef int (*kprobe_pre_handler_t) (struct kprobe *, struct pt_regs *);
    typedef void (*kprobe_post_handler_t) (struct kprobe *, struct pt_regs *,
		    		       unsigned long flags);

\end{verbatim}

\begin{verbatim}

struct kprobe {
	struct hlist_node hlist;
	/* list of kprobes for multi-handler support */
	struct list_head list;
	/*count the number of times this probe was temporarily disarmed */
	unsigned long nmissed;
	/* location of the probe point */
	kprobe_opcode_t *addr;
	/* Allow user to indicate symbol name of the probe point */
	const char *symbol_name;
	/* Offset into the symbol */
	unsigned int offset;
	/* Called before addr is executed. */
	kprobe_pre_handler_t pre_handler;
	/* Called after addr is executed, unless... */
	kprobe_post_handler_t post_handler;
	/* Saved opcode (which has been replaced with breakpoint) */
	kprobe_opcode_t opcode;
	/* copy of the original instruction */
	struct arch_specific_insn ainsn;
	/*
	 * Indicates various status flags.
	 * Protected by kprobe_mutex after this kprobe is registered.
	 */
	u32 flags;
};

\end{verbatim}

Notably, kprobes have a pre and a post handler. The post handler is optional.

You call register / undegister kprobe / kretprobe. You can enable and
disable them. Use `register_kprobe()` and `unregister_kprobe()` functions.

You can use a symbol instead of an address, the kernel will resolve
the address from the symbol using System.map symbol table.

The list of registered kprobes is visible under $DEBUGFS/kprobes/list

From `include/linux/kprobes.c` we see a global kprobe_table list:

```
#define KPROBE_HASH_BITS 6
#define KPROBE_TABLE_SIZE (1 << KPROBE_HASH_BITS)

static struct hlist_head kprobe_table[KPROBE_TABLE_SIZE];
```

Let's now walk throught an exaple program.

```
#include <linux/module.h>
#include <linux/kprobes.h>

#define MAX_FILENAME_LEN 1024

/* This gets executed before single-stepping the probed instruction */
static int hello_kprobe_pre_handler(struct kprobe *p, struct pt_regs *regs)
{
  const char __user * user_filename =
    (const char __user*) regs_get_kernel_argument(regs, 1); /* regs start at 0 */
  if (!user_filename)
  {
    printk(KERN_ERR "Kprobe: error reading filename in pre handler\n");
    return -1;
  }

  char filename[MAX_FILENAME_LEN] = {0};
  long ret = raw_copy_from_user(filename, user_filename,
                                MAX_FILENAME_LEN);
  if (ret < 0)
  {
    printk(KERN_ERR "Kprobe: error copying filename\n");
    return -1;
  }
  
  printk(KERN_INFO "Kprobe: intercepted access to %s\n", filename);

  return 0;
}

/* This gets executed after single-stepping the probed instruction */
static void hello_kprobe_post_handler(struct kprobe *kprobe,
                                      struct pt_regs *regs,
                                      unsigned long flags)
{
  /* does nothing */
}

struct kprobe kp = (struct kprobe) {
  /* static int do_sys_openat2(int dfd, const char __user *filename,
                               struct open_how *how) */
  .symbol_name  = "do_sys_openat2",
  .pre_handler  = hello_kprobe_pre_handler,
  .post_handler = hello_kprobe_post_handler,
};

static int __init hello_kprobe_init(void)
{
  printk(KERN_INFO "Kprobe: Hello, kprobe!\n");

  int ret = register_kprobe(&kp);
  if (ret < 0)
  {
    printk(KERN_ERR "Kprobe: error registering kprobe\n");
    return -1;
  }

  /* If kp->flags is set KPROBE_FLAG_DISABLED then it needs to be
   * enabled to use it. */
  /* enable_kprobe(&kp); */

  /* This is how you disable a kprobe */
  /* disable_kprobe(&kp); */
  
  return 0;
}

static void __exit hello_kprobe_cleanup(void)
{
  unregister_kprobe(&kp);
  printk(KERN_INFO "Kprobe: unregistered");
}

module_init(hello_kprobe_init);
module_exit(hello_kprobe_cleanup);

MODULE_AUTHOR("Giovanni Santini");
MODULE_DESCRIPTION("kprobe example");
MODULE_LICENSE("GPL v2");
```

TODO: make a commentary of this. Maybe divide it in parts.

Implementation
++++++++++++++

kernel/kprobes.c implements the logic, which stores an hashmap of
kprobes indexed by their address at which the probe is placed, and
blacklisted entries. Insertion and deletion to this table is guarded
by a mutex to prevent these operations from executing simultaneously.

Kprobes are stored in the \verb|kprobe_table|. Blacklisted address ranges are
stored in the list \verb|kprobe_blacklist|. Example use of \verb|kprobe_table|,
extracted from `kernel/kprobes.c:378`:

\begin{verbatim}

struct kprobe *get_kprobe(void *addr)
{
	struct hlist_head *head;
	struct kprobe *p;

	head = &kprobe_table[hash_ptr(addr, KPROBE_HASH_BITS)];
	hlist_for_each_entry_rcu(p, head, hlist,
				 lockdep_is_held(&kprobe_mutex)) {
		if (p->addr == addr)
			return p;
	}

	return NULL;
}
\end{verbatim}

If `p->flags & KPROBE_FLAG_FTRACE`, the kprobe is registered via the `ftrace`
instrumentation.

`register_kprobe` sets up the address of the probed instruction from the symbol
if specified, via `kallsyms_lookup_name` later which performs a binary search of
the symbol name in the uncompressed string from the `kallsyms_names` array, then
it checks that the address is valid (if it is in a text area).

Finally it calls `prepare_kprobe`, adds the hash of the kprobe to the
`kprobe_table` list and calls `arm_kprobe` if it was not disabled, which
actually modifies the instruction in memory and sets up a breakpoint call
(`int3` on x86). It sets up emulation, which tries to emulate the probed
instruction instead of calling an additional trap after single stepping, to
improve performance. At the end it tries to optimize it.

```
Whenever a probe is hit, the probe handler is called with interrupts disabled. Interrupts are disabled because handling a probe is a multiple step process which involves breakpoint handling and single-step execution of the probed instruction. There is no easy way to save the state between these operations hence interrupts are kept disabled during probe handling.
```

`int3` handler is defined in `arch/x86/kernel/trace.c` which calls `do_int3`:

```
static bool do_int3(struct pt_regs *regs)
{
	int res;

#ifdef CONFIG_KGDB_LOW_LEVEL_TRAP
	if (kgdb_ll_trap(DIE_INT3, "int3", regs, 0, X86_TRAP_BP,
			 SIGTRAP) == NOTIFY_STOP)
		return true;
#endif /* CONFIG_KGDB_LOW_LEVEL_TRAP */

#ifdef CONFIG_KPROBES
	if (kprobe_int3_handler(regs))
		return true;
#endif
	res = notify_die(DIE_INT3, "int3", regs, 0, X86_TRAP_BP, SIGTRAP);

	return res == NOTIFY_STOP;
}
NOKPROBE_SYMBOL(do_int3);
```

The `kprobe_int3_handler` in `arch/x86/kernel/kprobes/core.c` uses two per-cpu
variable called `current_kprobe` and `kprobe_ctrlblk` and does the following:

- If `current_kprobe` is empty, meaning that a kprobe is not running, then it
  sets `current_kprobe`. If specified, it calls the `kp->pre_handler` and
  proceeds to emulate the probed instruction and call the post handler, or to
  setup the CPU's instruction pointer to the original instruction. In this case,
  the instruction pointer will be set when the trap returns, the instruction
  will be executed, and a new trap will be called with `int3` and with
  `current_kprobe` not empty. This is necessary because jump instructions cannot
  be emulated or executed during the single step since they would.

- if `current_kprobe` is non empty, which happens after single stepping the
  instruction and entering the trap again from the previous point. This returns
  to the normal execution by setting the instruction pointer to the original
  place (just after the probed instruction).

A kretprobe allows the user do register a callback that gets executed when the
function returns. Underneath, the kretprobe registers a kprobe
`pre_handler_kretprobe` that changes the return address when the function is
entered. The return address is set to a trampoline function. On arm, this
function triggers a breakpoiont exception, which eventually calls
`__kretprobe_trampoline_handler`. This function finally calls the user handler,
and sets the exception's return address to the correct one after the probe.

After registering a kprobe, if the kernel is compiled with CONFIG_OPTPROBES=y,
it tries to optimize the break instruction with a jump. This does not work if
the kprobe was based on ftrace. The optimizer logic is quite complex and will
not be covered here.



Perf
----

API
+++

Structure:

- A bit of history
- architecture
- in-kernel api
- user space api

Perf: event state machine

Great overview in my github project: https://github.com/San7o/linux-perf-event-example/tree/main

You interact with perf via the \verb|perf_event_open| syscall.
Used for performance counting. Has its own ring buffer.

They are simple as they only signal that an event is happened. This
is then processed by another program like perf.

```

       int syscall(SYS_perf_event_open, struct perf_event_attr *attr,
                   pid_t pid, int cpu, int group_fd, unsigned long flags);
```

The man page contains all the information for the user. From the man page:

```
  Events come in two flavors: counting and sampled.  A counting
       event is one that is used for counting the aggregate number of
       events that occur.  In general, counting event results are
       gathered with a read(2) call.  A sampling event periodically
       writes measurements to a buffer that can then be accessed via
       mmap(2).
```

mmap is used to read events that have more information than a counter. This uses
the ringbuffer. Using mmap removes the need of repeted `read` syscalls which
cause traps. The mmapped area can be accessed via `struct perf_event_mmap_page`
which contains pointer to head (producer, updated by the kernel) and tail
(consumer, updated by the user program) of the ring buffer.

This is explained with better words in the documentation:

```
The control structure is named as perf_event_mmap_page, it contains a head
pointer data_head and a tail pointer data_tail. When the kernel starts to fill
records into the ring buffer, it updates the head pointer to reserve the memory
so later it can safely store events into the buffer. On the other side, when the
user page is a writable mapping, the perf tool has the permission to update the
tail pointer after consuming data from the ring buffer.
```

There is also a concept of event groups, also from the man page.

```
  The group_fd argument allows event groups to be created.  An event
       group has one event which is the group leader.  The leader is
       created first, with group_fd = -1.  The rest of the group members
       are created with subsequent perf_event_open() calls with group_fd
       being set to the file descriptor of the group leader.  (A single
       event on its own is created with group_fd = -1 and is considered
       to be a group with only 1 member.)  An event group is scheduled
       onto the CPU as a unit: it will be put onto the CPU only if all of
       the events in the group can be put onto the CPU.  This means that
       the values of the member events can be meaningfully compared
       —added, divided (to get ratios), and so on— with each other, since
       they have counted events for the same set of executed
       instructions.
```

When read, different types or events will have different binary formats.

The user specifies which event to read using the `struct perf_event_attr`
structure. He can select variables such as the frequency of sampling or whether
to exclude counting on certain contexts such as user or kernel. The full content
of the structure is described in the man page.

The pid and cpu arguments allow specifying which process and CPU to monitor, for
example by supporting a per-thread mode or per-cpu global mode.

Similarly to ftrace, the perf ringbuffer uses a reserve and commit pattern with
`perf_output_begin()` and `perf_output_end()`.

For hardware events, perf uses an architecture specific auxiliary ring buffer
"AUX ring buffer" lower latency high frequency events.

In default mode, allocates individual ring buffer for each CPU, but it only
enables events for the profiled program rather than for all threads in the
system.

The syscall will perform a selinux security check at the start.

Security implications:
\url{https://www.kernel.org/doc/html/latest/admin-guide/perf-security.html}

In summary, there are security issues because performance acouters may leak
hardware or software information which an attacker could use maliciously.
Therefore, the perf subsystem implementes security checks: root can do anything,
and the others have security checks via capabilities (CAP_PERFMON).
Additionally, the subsystem register the \verb|perf_event_paranoid| parameter,
available in \textit{ /proc/sys/kernel / perf\_event\_paranoid}, which can be
set to provide stricter or looser access checks, as described in the systemcall
manual page.

\begin{verbatim}

perf_event_open(2)

       A call to perf_event_open() creates a file descriptor that  allows  mea‐
       suring performance information.  Each file descriptor corresponds to one
       event  that is measured; these can be grouped together to measure multi‐
       ple events simultaneously.

       Events can be enabled and disabled in two ways:  via  ioctl(2)  and  via
       prctl(2).  When an event is disabled it does not count or generate over‐
       flows but does continue to exist and maintain its count value.

       Events  come  in two flavors: counting and sampled.  A counting event is
       one that is used for counting the aggregate number of events that occur.
       In general, counting event results are gathered with a read(2) call.   A
       sampling  event  periodically  writes  measurements to a buffer that can
       then be accessed via mmap(2).

\end{verbatim}

Another resource:
  \url{https://github.com/torvalds/linux/blob/master/tools/perf/design.txt}

perf_event
----------

Perf events are easy, they just
notify that an even has appened. There are hardware events and
software events.

There is this abstraction of a performance mesuring unit. This is an
generic object that can manages a set of performance events and
counters. It handles registration, stopping and reading of events.
Code is in \verb|linux/include/perf_event.h|; struc pmu: 329.
Example: https://github.com/San7o/linux-perf-event-example/

Code in `inclide/linux/perf_event.h`, `kernel/events/core.c`

To define a perf event, we first create an instance of a Performance Monitoring
Unit (pmu), which is of `struct pmu`:

```
static struct pmu xtensa_pmu = {
	.pmu_enable = xtensa_pmu_enable,
	.pmu_disable = xtensa_pmu_disable,
	.event_init = xtensa_pmu_event_init,
	.add = xtensa_pmu_add,
	.del = xtensa_pmu_del,
	.start = xtensa_pmu_start,
	.stop = xtensa_pmu_stop,
	.read = xtensa_pmu_read,
};
```

While the implementation of these functions depends on their context, the most
interesting function is the `.read`: this will be called to pass the event data
to the user. More precisely, function sets `event->count` with the value of the
counter, which is of `local64_t` type.

And register it with:

```
ret = perf_pmu_register(&hello_pmu, "hello_pmu", -1);
if (ret)
  return ret;
```

The logic that handles the interaction between userspace and the events is
implemented in the callbacks in `struct pmu`. Each `pmu` has a name and a type.

Types are in `include/uapi/linux/perf_event.h`:

```
enum perf_type_id {
	PERF_TYPE_HARDWARE			= 0,
	PERF_TYPE_SOFTWARE			= 1,
	PERF_TYPE_TRACEPOINT			= 2,
	PERF_TYPE_HW_CACHE			= 3,
	PERF_TYPE_RAW				= 4,
	PERF_TYPE_BREAKPOINT			= 5,

	PERF_TYPE_MAX,				/* non-ABI */
};
```

The `perf_pmu_register()` function initializes the per-CPU context for each cpu,
registers the pmu to a global list of pmus (`kernel/events/core.c`):

```
static LIST_HEAD(pmus);
```

The user drives the event loop, and access the events via the
`perf_event_open()` syscall (kernel/events/core.c):

```
SYSCALL_DEFINE5(perf_event_open,
		struct perf_event_attr __user *, attr_uptr,
		pid_t, pid, int, cpu, int, group_fd, unsigned long, flags)
```

The user can read the counters or measurement by type and config (config depends
on the type). It also supports a ring buffer, which is either per-CPU or
per-task. The ring buffer is fast because it does not copy memory, but it uses
the same physical memory for writing and reading.

USDT
----

TODO

(USDT)
Note: this is actually a uprobe in disguise, it expects to attach to
tracepoints defined in userspace programs.

TODO: write some code to understand this


