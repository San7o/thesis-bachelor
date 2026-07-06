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

Static Function Tracing
-----------------------

`CONFIG_FUNCTION_TRACER`

All thanks to `gcc -pg` + (`-mfentry` whe CONFIG_HAVE_FENTRY). With
`-mnop-mcount` we use several `NOP` instructions instead
(CONFIG_HAVE_NOP_MCOUNT). It generates Mcount (ARM and others) or fentry (x86).

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

Dynamic Ftrace
--------------

`DYNAMIC_FTRACE` option.

With dynamic tracing enabled, the kernel patches calls to mcount with a NOP (or
uses `-mnio-mcount`) if tracing is disabled, and patches it to `ftrace_caller`
or `ftrace_regs_caller` if it is enabled. This allows for 0 overhead tracing in
case it is disabled.

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

We see `ftrace_called` in arm64 `arch/arm64/kernel/entry-ftrace.S:267`:

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


Tracepoints
-----------

`inclide/linux/tracepoint-defs.h:26`:

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

We create a tracepoint with `DECLARE_TRACE`. For example, see
`include/trace/events/sched.h`:

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

What happens in the macro expansion is that:

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

This will create several functions, including `register_trace_entry_tp()`,
`unregister_trace_sched_entry_tp()` and `trace_sched_entry_tp()`, for example
`kernel/sched/core.c:7071`:

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

So we can access the tracepoints via:

```
extern tracepoint_ptr_t __start___tracepoints_ptrs[];
extern tracepoint_ptr_t __stop___tracepoints_ptrs[];
```

What happen is that:

```
trace_sched_entry_tp
  __do_trace_entry_tp
      static_call(tp_func_entry_tp)
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

The `tp_func_entry_tp` is defined by some other system that consumes the event.
This can be set to null, disabling the call, so we get very little overhead.

We could also do this:

```
if (trace_foo_enabled()) {
        x = bar();
        trace_call__foo(x);  // does not check if static function is null
}
```

Event tracing
-------------

To automatically register the tracepoint in the ftrace / perf inftrastructure,
we use `TRACE_EVENT` macro instead.

We create a tracepoint like this `fs/fuse/fuse_trace.h:126`:

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

Kprobes
-------

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

From `include/linux/kprobes.c` we see a global kprobe_table list:

```
#define KPROBE_HASH_BITS 6
#define KPROBE_TABLE_SIZE (1 << KPROBE_HASH_BITS)

static struct hlist_head kprobe_table[KPROBE_TABLE_SIZE];
```

Example program:

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

Example: https://github.com/San7o/linux-perf-event-example/
