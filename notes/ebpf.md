eBPF
====

Best resources:

- Kernel Docs: https://origin.kernel.org/doc/html/latest/bpf/
- eBPF docs: https://docs.ebpf.io/
- Cilium: https://docs.cilium.io/en/latest/reference-guides/bpf/index.html

Many resources:

    https://docs.cilium.io/en/stable/reference-guides/bpf/resources/#xdp-newbies

Overview
--------

TODO: We want more dynamic tracing but it has security issues

TODO: Where eBPF comes from

TODO: Big picture, diagrams. the bpf(7) man page would be useful material

TODO: Take inspiration for the overview here:

    https://www.linuxfoundation.org/hubfs/eBPF/ControlPlane%20%E2%80%94%20eBPF%20Security%20Threat%20Model.pdf

TODO: Show and eBPF program

The program that loads an eBPF program is called a `loader`. `libbpf` is the
official library that provides easier loading abstractions on top of the lower
level `bpf` syscall.

TODO: Lots of diagrams here:

    https://ebpf.io/what-is-ebpf/

Instruction Set
---------------

RFC Link:

    https://datatracker.ietf.org/doc/rfc9669/

It uses the C calling convention.

ABI: https://origin.kernel.org/doc/html/latest/bpf/standardization/abi.html

Notes:

The standard specifies two instruction encoding, one using 64 bits and another
one using 128 bits. Instructions are assigned to a conformance group. An
implementation of BPF may support only some conformance groups.

It was standardized October 2024 via RFC9669, to allow compiler developers,
tools developers and eBPF runtime developers and ASIC designers to agree to a
common standard.

Some good insights:

    https://lwn.net/Articles/975830/

Basic encoding:

```
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |    opcode     |     regs      |            offset             |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                              imm                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   *opcode:*  operation to perform, encoded as follows:

      +-+-+-+-+-+-+-+-+
      |specific |class|
      +-+-+-+-+-+-+-+-+

      *specific:*  The format of these bits varies by instruction class

      *class:*  the instruction class (see Section 3.3)

   *regs:*  the source and destination register numbers, encoded as
      follows on a little-endian host:

      +-+-+-+-+-+-+-+-+
      |src_reg|dst_reg|
      +-+-+-+-+-+-+-+-+

      and as follows on a big-endian host:

      +-+-+-+-+-+-+-+-+
      |dst_reg|src_reg|
      +-+-+-+-+-+-+-+-+

      *src_reg:*  the source register number (0-10), except where
         otherwise specified (64-bit immediate instructions (see
         Section 5.4) reuse this field for other purposes)

      *dst_reg:*  the destination register number (0-10), unless
         otherwise specified (future instructions might reuse this field
         for other purposes)

   *offset:*  signed integer offset used with pointer arithmetic, except
      where otherwise specified (some arithmetic instructions reuse this
      field for other purposes)

   *imm:*  signed integer immediate value
```

Wide instruction encoding:

```
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |    opcode     |     regs      |            offset             |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                              imm                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                           reserved                            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                           next_imm                            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   *opcode:*  operation to perform, encoded as explained above

   *regs:*  the source and destination register numbers (unless
      otherwise specified), encoded as explained above

   *offset:*  signed integer offset used with pointer arithmetic, unless
      otherwise specified

   *imm:*  signed integer immediate value

   *reserved:*  unused, set to zero

   *next_imm:*  second signed integer immediate value
```

Instruction classes:

```
      +=======+=======+=================================+===========+
      | class | Value | Description                     | Reference |
      +=======+=======+=================================+===========+
      | LD    | 0x0   | non-standard load operations    | Section 5 |
      +-------+-------+---------------------------------+-----------+
      | LDX   | 0x1   | load into register operations   | Section 5 |
      +-------+-------+---------------------------------+-----------+
      | ST    | 0x2   | store from immediate operations | Section 5 |
      +-------+-------+---------------------------------+-----------+
      | STX   | 0x3   | store from register operations  | Section 5 |
      +-------+-------+---------------------------------+-----------+
      | ALU   | 0x4   | 32-bit arithmetic operations    | Section 4 |
      +-------+-------+---------------------------------+-----------+
      | JMP   | 0x5   | 64-bit jump operations          | Section 4 |
      +-------+-------+---------------------------------+-----------+
      | JMP32 | 0x6   | 32-bit jump operations          | Section 4 |
      +-------+-------+---------------------------------+-----------+
      | ALU64 | 0x7   | 64-bit arithmetic operations    | Section 4 |
      +-------+-------+---------------------------------+-----------+
```

The jump instruction `call` set with `src_reg=0x0` or `src_reg=0x2` will call an
helper functions by either static ID or BTF ID respectively. Helper functions
are implemented by the underlying platform to provide platform-specific
functionalities. They are not standardized by RFC9669, but they are documented
in the Linux kernel documentation. More about helper functions later.

Load and store functions define atomic operations, which ensure that the
operation cannot be interrupted or corrupted by other accesses to the same
memory region by another BPF program or other actors. Other than atomic `ADD`,
`OR`, `AND` and `XOR`, BPF provide `XCHG` for atomic exchange, and `CMPXCHG` for
atomic compare and exchange.

64-bit Immediate Instructions use wide instruction encoding to hold an opcode
subtype in `src_reg`.

```

    +=========+================================+==========+==========+
    | src_reg | Pseudocode                     | imm Type | dst Type |
    +=========+================================+==========+==========+
    | 0x0     | dst = (next_imm << 32) | imm   | integer  | integer  |
    +---------+--------------------------------+----------+----------+
    | 0x1     | dst = map_by_fd(imm)           | map fd   | map      |
    +---------+--------------------------------+----------+----------+
    | 0x2     | dst = map_val(map_by_fd(imm))  | map fd   | data     |
    |         | + next_imm                     |          | address  |
    +---------+--------------------------------+----------+----------+
    | 0x3     | dst = var_addr(imm)            | variable | data     |
    |         |                                | id       | address  |
    +---------+--------------------------------+----------+----------+
    | 0x4     | dst = code_addr(imm)           | integer  | code     |
    |         |                                |          | address  |
    +---------+--------------------------------+----------+----------+
    | 0x5     | dst = map_by_idx(imm)          | map      | map      |
    |         |                                | index    |          |
    +---------+--------------------------------+----------+----------+
    | 0x6     | dst = map_val(map_by_idx(imm)) | map      | data     |
    |         | + next_imm                     | index    | address  |
    +---------+--------------------------------+----------+----------+

                 Table 12: 64-bit Immediate Instructions

   where

   *  map_by_fd(imm) means to convert a 32-bit file descriptor into an
      address of a map (see Section 5.4.1)

   *  map_by_idx(imm) means to convert a 32-bit index into an address of
      a map

   *  map_val(map) gets the address of the first value in a given map

   *  var_addr(imm) gets the address of a platform variable (see
      Section 5.4.2) with a given id

   *  code_addr(imm) gets the address of the instruction at a specified
      relative offset in number of (64-bit) instructions

   *  the 'imm type' can be used by disassemblers for display

   *  the 'dst type' can be used for verification and just-in-time
      compilation purposes
```

RFC9669 defines "maps" as shared memory regions accessible by BPF programs on
some platforms.

The standard also defines "platform variables" which are memory regions
identified by integer ids exposed by the runtime.

Clang 11+ and GCC 12.1+ support BPF bytecode.

The verifier
------------

Good list of safety requirements:

    https://docs.ebpf.io/linux/concepts/verifier/

```
Here is a non-exhaustive list of things that are not allowed to illustrate the point:

- Programs must always terminate (within a reasonable amount of time) - So no
  infinite loops or infinite recursion.
- Programs are not allowed to read arbitrary memory - Being able to read any
  memory would allow a program to leak sensitive information. There are
  exceptions, tracing programs have access to helpers that allow them to read
  memory in a controlled way. But these program types require root privileges
  and thus are not a security risk.
- Network programs are not allowed to access memory outside of packet bounds
  because adjacent memory could contain sensitive information. See the point
  above.
- Programs are not allowed to deadlock, so any held spinlocks must be released
  and only one lock can be held at a time to avoid deadlocks over multiple
  programs.
- Programs are not allowed to read uninitialized memory - This could leak
  sensitive information.
```

The verifier checks these requirements, so that the program can be executed
without runtime checks which limits overhead.

What constituted a correct program that will not be rejected is intentionally
not standardized. The Linux verifier is continuously changing


Great overview: https://www.spinics.net/lists/xdp-newbies/msg00185.html

```
The safety of the eBPF program is determined in two steps.

First step does DAG check to disallow loops and other CFG validation. In
particular it will detect programs that have unreachable instructions. (though
classic BPF checker allows them)

Second step starts from the first insn and descends all possible paths. It
simulates execution of every insn and observes the state change of registers and
stack.
```

Each register can have a type. At the start, the only valid register with a type
is r1 of type `PTR_TO_CTX`. Doing operations on the registers can modify their
type. For example, `PTR_TO_CTX + PTR_TO_CTX = SCALAR_TYPE` because pointer
operations are not allowed.

Load / store instructions are allowed only with registers of type `PTR_TO_CTX`,
`PTR_TO_MAP`, `PTR_TO_STACK`.

The verifier also know which offsets inside structs can be accessed, for example
it will check that we can access `R6 + 8` in `bpf_ld R0 = *(u32 *)(R6 + 8)`. if
`R6=PTR_TO_STACK` it will check that this address is within stack bounds
`[-MAX_BPF_STACK, 0)`, which is not in this example. It also checks that the
stack location was written to previously.

Types are defined in `enum bpf_reg_type` (`include/linux/bpf.h`)

- `NOT_INIT`
- `SCALAR_VALUE`
- `PTR_TO_CTX`
- `CONST_PTR_TO_MAP`
- `PTR_TO_MAP_VALUE`
- `PTR_TO_MAP_VALUE_OR_NULL`
- `PTR_TO_STACK`
- `PTR_TO_PACKET`
- `PTR_TO_PACKET_END`
- `PTR_TO_SOCKET`
- `PTR_TO_SOCKET_OR_NULL`
- .. more

```
However, a pointer may be offset from this base (as a result of pointer
arithmetic), and this is tracked in two parts: the ‘fixed offset’ and ‘variable
offset’. The former is used when an exactly-known value (e.g. an immediate
operand) is added to a pointer, while the latter is used for values which are
not exactly known. The variable offset is also used in SCALAR_VALUEs, to track
the range of possible values in the register.
```

There is the concept of priuning where the verifier judges a state as OK of if
it has seen it already before (both registers and stacks were already
evaluated).

---

Verification starts in `bpf_check` (`kernel/bpf/verifier.c:19706`) after calling
the `bpf` system call with the `BPF_PROG_LOAD` command. Upon entry, the verifier
allocates am environment structure `struct bpf_verifier_env *env`. Each
operation and check will read and modify this environment. The environment has
all the information of a program, jump history, register states (`struct
bpf_reg_state`) and the history of previously explored states, among other
things. It also has a log buffer `struct bpf_verifier_log` which logs the state
of the program after each instruction, and can be read by userspace. This is
particularly useful to debug verification issues. Since the verification
environment is allocated per-verification, multiple verifications can happen
simultaneously (patch:
`https://lore.kernel.org/bpf/20190419144455.853134-1-ast@kernel.org/`) but this
is allowed only if the token (see later) has `CAP_BPF` since unprivileged users
could exhaust kernel memory. A token is a delegation mechanism where a privilege
process can pass capabilities to an unprivileged one via a token, each program
has a token. This will be discussed more in the Security Considerations chapter.

Some parts of the verifier have been formally verified to be sound, in
particular value tracking
(https://people.cs.rutgers.edu/~sn349/papers/agni-cav2023.pdf). This paper also
has some good explaination about abstract interpretation that I should include.

The verifier proceeds to perform various calculations and checks:

- DAG check: this check disallows loops and unreachable code
  It rejects the following programs:
  - larger than `BPF_MAXINSNS` insns
  - if loop is present (detected via back-edge)
  - unreachable insns exist (shouldn't be a forest. program = one function)
  - out of bounds or malformed jumps

  Functions:
  - `check_and_resolve_insns`: checks that instructions are sane
  - `check_subprogs`: checks that all jumps are within the same subprogram (a
    program can contain multiple subprograms).
  - `bpf_check_bpf_info`:
  - `bpf_check_cfg` (`kernel/bpf/cfg.c`): a non-recursive depth-first-search to
    detect loops in the BPF program. Each instruction (vertex in the directed
    graph) can have either a discovered, explored, fallthrough or branch label.
    The difference between discovered and explored is that a discovered
    instruction did not check all the edges yet, while an explored one was fully
    checked. Each edge can either be discovered or not. If the DAG explores any
    instruction already marked as discovered, the program is rejected
    (back-edge).

  TODO: Do example of valid and invalid program (asm IF vs asm WHILE) and
  its graph representation (vertex = instruction, edge = jump)

Pseudocode, where vertices are instructions, and edges are jumps:

```
procedure DFS-iterative(G,v):
    label v as discovered
    let S be a stack
    S.push(v)
  loop:
    while S is not empty
      t <- S.peek()

      if t is not JMP or CALL or EXIT:
        if t is explored:
          continue

        label t as explored
        S.pop()
        S.push(next_instruction(t))
        continue

      if t is a EXIT instruction:
        continue

      // t is a JUMP or CALL instruction, check the edges
      for all edges e in G.adjacentEdges(t) do
          if edge e is already labelled
              continue with the next edge
          w <- G.adjacentVertex(t,e)
          if vertex w is not discovered and not explored
              label e as a tree-edge
              label w as discovered
              S.push(w)
              continue
          else if vertex w is discovered
              return REJECTED
          else
              // vertex w is explored
              label e as forward- or cross-edge
      label t as explored
      S.pop()

  return SUCCESS
```

- Simulation: the verifier descends all possible paths, simulation execution of
  every instruction, observing the state of registers and stack. Each register
  has a type, and the min and max ranges are tracked, see `struct
  bpf_reg_state`. This verification using register tracking is called "Abstract
  interpretation" (https://dl.acm.org/doi/10.1145/512950.512973).
  - Pruning: instead of walking all possible paths through the program, for each
    branch the verifier looks at all previsously analized states. If any of them
    match the current one, then the verifier skips that branch. Functions:
  - `bpf_prune_dead_branches`: rewrites conditional branches that have constant
    outcomes into unconditional jumps, so that dead code is not checked
  - `bpf_compute_scc` (strongly connected components): Uses a non-recursive
    adaptation of Tarjan's algorithm for SCC computation.
  - `do_check_main -> do_check_common`: this does the simulation. It keeps the
    simulation state by allocating a `struct bpf_verifier_state` and `struct
    bpf_func_state`. It calls `do_check` which calls `do_check_insn` for each
    instruction. The verifier keeps a stack of states: whenever it encounters a
    conditional jump, it pushes a new state in the stack. The loop continues
    checking instructions from the top state. Then an exit condition is reached,
    the top state is popped, until all states are checked. If it find a state
    that was already visited, it prunes that (considers it verified and
    continues). `do_check_insns` does checks based on the instruction, for
    example bound checks, making sure types are compatible, etc...

    TODO: show a picture of how a branch pushes to the states stack

  - `do_check_subprogs`: lazily verifies all global functions based on their BTF
    if they are ever called from non-dead code. This is necessary because during
    verifications all functions are assumed to be already verified successfully
    and checked for match only.

Example of invalid programs can be found in:

    https://docs.kernel.org/bpf/verifier.html

After successful verification, the verifier calls:

```
bpf_check
  -> __bpf_prog_select_runtime
    -> bpf_prog_jit_compile
      -> bpf_int_jit_compile (arch/x86/net/bpf_jit_comp.c)
        -> do_jit
```

The JIT is not particularly emotional.

A note about loops, you can unroll the loop or use the helper functions. The
modern verifier actually supports bounded loops.

Program Types
-------------

Each eBPF program has a certain type. Types are enumerated in `enum
bpf_prog_type` in `include/uapi/linux/bpf.h`. After loading the program and the
verifier accepts it, you need to attach it. The attach point depends on the type
of program. After attaching it, you get a link.

TODO: state machine picture? Maybe it is too much

By convention, loaders based on libbpf specify the type of the program by
putting the program in a specific ELF section of the binary, which depends upon
the program type. For example, a program of type `BPF_PROG_TYPE_CGROUP_DEVICE`
must be put in the `cgroup/dev` ELF section, and a program of type
`BPF_PROG_TYPE_CGROUP_SKB` is put in the `cgroup/skb` ELF section. This is an
hack so that you can specify multiple programs of different types in the same
binary.

The program execution depends on where it was attached, as well as which helpers
functions it is allowed to use. There are more than 30 types of programs. We
will see only a few examples.

Programs can be attached to kprobes (`BPF_PROG_TYPE_KPROBE`), tracepoints
(`BPF_PROG_TYPE_TRACEPOINT`) and perf events (`BPF_PROG_TYPE_PERF_EVENT`). With
the information given to the tracing instrumentation, the eBPF program can
execute ad-hoc logic to react to the event.

Other program types:

BPF_PROG_TYPE_KPROBE
^^^^^^^^^^^^^^^^^^^^

Done impl (kivebpf)

BPF_PROG_TYPE_XDP
^^^^^^^^^^^^^^^^^

Done test

Indeed, in for the intel i40e driver with napi:

```
i40e_intr (interrupt)
  -> napi_schedule_irqoff
    -> __napi_schedule_irqoff
      -> ____napi_schedule
        -> list_add_tail

softirq context, called from napi:

__napi_busy_loop
  -> napi_poll
    -> __napi_poll
      -> struct napi_struct->poll

That function is implemented in the driver

i40e_napi_poll
  -> i40e_clean_rx_irq
    -> i40e_run_xdp
      -> bpf_prog_run_xdp
```

BPF_PROG_TYPE_LSM
^^^^^^^^^^^^^^^^^

Done

It uses the hooks defined in `include/linux/lsm_hook_defs.h`. The kernel
registers a trampoline to a NOP stub in the security module.

BPF_PROG_TYPE_STRUCT_OPS
^^^^^^^^^^^^^^^^^^^^^^^^

Done SCHED_EXT

To register certain types of eBPF program, special trampolines are used, which
are built on top of the dynamic ftrace infrastructure (knowing the address of
the NOP area of a function) "to allow kernel code to call into BPF programs
with practically zero overhead":

https://github.com/torvalds/linux/commit/fec56f5890d93fc2ed74166c397dc186b1c25951

```
struct bpf_trampoline_ops {
	int (*register_fentry)(struct bpf_trampoline *tr, struct bpf_tramp_image *im, void *data);
	int (*unregister_fentry)(struct bpf_trampoline *tr, u32 orig_flags, void *data);
	int (*modify_fentry)(struct bpf_trampoline *tr, u32 orig_flags, struct bpf_tramp_image *im,
			     bool lock_direct_mutex, void *data);
};
```

For example in `__bpf_arch_text_poke` (`arch/x85/net/bpf_jit_comp.c`) we see the
actual patching:

```
	const u8 *nop_insn = x86_nops[5];
	u8 new_insn[X86_PATCH_SIZE];
	u8 *prog;
	int ret;

  // ...

	memcpy(new_insn, nop_insn, X86_PATCH_SIZE);
	if (new_t != BPF_MOD_NOP && new_addr) {
		prog = new_insn;
		ret = new_t == BPF_MOD_CALL ?
		      emit_call(&prog, new_addr, ip) :
		      emit_jump(&prog, new_addr, ip);
		if (ret)
			return ret;
	}

  // ...

	if (memcmp(ip, new_insn, X86_PATCH_SIZE)) {
		smp_text_poke_single(ip, new_insn, X86_PATCH_SIZE, NULL);
		ret = 0;
	}
```

When we link an ebpf program that needs a trampoline
(`__bpf_trampoline_link_prog`) we call `bpf_trampoline_update` which creates a
page in `arch_prepare_bpf_trampoline` where it sets up the stack environment for
the eBPF program including its arguments, calls it, restores registers and
returns to the target function.

"The tracing use case in the datacenter demonstrated that certain key kernel
functions have (like tcp_retransmit_skb) have 2 or more kprobes that are always
active.  Other functions have both kprobe and kretprobe.  So it is essential to
keep both kernel code and BPF programs executing at maximum speed. Hence
generated BPF trampoline is re-generated every time new program is attached or
detached to maintain maximum performance." (https://github.com/torvalds/linux/commit/fec56f5890d93fc2ed74166c397dc186b1c25951)

The above is implemented in `__arch_prepare_bpf_trampoline -> invoke_bpf`.

The program type that have a trampoline are:

```
	switch (ptype) {
	case BPF_PROG_TYPE_TRACING:
		if (eatype == BPF_TRACE_FENTRY || eatype == BPF_TRACE_FEXIT ||
		    eatype == BPF_MODIFY_RETURN || eatype == BPF_TRACE_FSESSION ||
		    eatype == BPF_TRACE_FENTRY_MULTI || eatype == BPF_TRACE_FEXIT_MULTI ||
		    eatype == BPF_TRACE_FSESSION_MULTI)
			return true;
		return false;
	case BPF_PROG_TYPE_LSM:
		return eatype == BPF_LSM_MAC;
	default:
		return false;
	}
```

Helper Functions
----------------

Helper functions can be used by the eBPF program and are considered User Space
API (UAPI), meaning that the API will be honored on future kernel versions.

BPF Kernel Functions (`kfuncs`) are exposed to BPF programs, but do not have a
stable interface. They are exposed either by making an existing function
visible, or add a new wrapper for BPF. Defensive programming is advised.

Functions are delcared with the `__bpf_kfunc` macro which simply expands to:

```
#define __bpf_kfunc __used __retain __noclone noinline
```

And here is how you would define a kfunc:

```
__bpf_kfunc_start_defs();

__bpf_kfunc int bpf_list_add(struct bpf_list_head *head, struct bpf_list_node *new,
			     struct bpf_list_node *prev__nonown_allowed,
			     struct btf_struct_meta *meta, u64 off)
{
  // ....
}

__bpf_kfunc_end_defs();
```

Where start and end defs simply remove some warnings:

```
#define __bpf_kfunc_start_defs()					       \
	__diag_push();							       \
	__diag_ignore_all("-Wmissing-declarations",			       \
			  "Global kfuncs as their definitions will be in BTF");\
	__diag_ignore_all("-Wmissing-prototypes",			       \
			  "Global kfuncs as their definitions will be in BTF")

#define __bpf_kfunc_end_defs() __diag_pop()
#define __bpf_hook_start() __bpf_kfunc_start_defs()
#define __bpf_hook_end() __bpf_kfunc_end_defs()
```

The registration happens in `kernel/bpf/verifier.c`:

```
BTF_ID(func, bpf_list_add)
```

Which expands to this macro:

```
#define BTF_IDS_SECTION ".BTF_ids"

#define ____BTF_ID(symbol, word)			\
asm(							\
".pushsection " BTF_IDS_SECTION ",\"a\";       \n"	\
".local " #symbol " ;                          \n"	\
".type  " #symbol ", STT_OBJECT;               \n"	\
".size  " #symbol ", 4;                        \n"	\
#symbol ":                                     \n"	\
".zero 4                                       \n"	\
word							\
".popsection;                                  \n");
```

For each kfunc, we create an unique label in the `.BTF_ids` section, which is
initially filled with zeros during compilationand resolved later during the
linking phase by `resolve_btfids` tool.

We can also annotate a kfunc to give more information to the verifier:

```
BTF_KFUNCS_START(generic_btf_ids)
// ...

BTF_ID_FLAGS(func, bpf_list_add, KF_IMPLICIT_ARGS)

// ...
BTF_KFUNCS_END(generic_btf_ids)
```

which exprands to the same macro as `BTF_ID`, but concatenates the flags with
the ID.

TODO: what does `BTF_KFUNCS_START` do?

TODO: document some flags. We have some in https://origin.kernel.org/doc/html/latest/bpf/kfuncs.html

TODO: talk about annotations and how the verifier uses them, like `__sz`, `__k`
etc. Explaintations in https://origin.kernel.org/doc/html/latest/bpf/kfuncs.html

Maps
----

There are many typers of maps, generic and non-generic. They are used as
key/value store to communicate between the eBPF program and userspace, but the
same api is used to provide other data structures like stacks, queue,
ringbufferes, arena etc. They are reference counted, so as long as some code is
using them, they will stay in memory.

`include/linux/bpf.h`

```
struct bpf_map_ops {
	/* funcs callable from userspace (via syscall) */
	int (*map_alloc_check)(union bpf_attr *attr);
	struct bpf_map *(*map_alloc)(union bpf_attr *attr);
	void (*map_release)(struct bpf_map *map, struct file *map_file);
	void (*map_free)(struct bpf_map *map);
	int (*map_get_next_key)(struct bpf_map *map, void *key, void *next_key);
	void (*map_release_uref)(struct bpf_map *map);
  // ...
}
```

The eBPF program interacts with maps via helper functions, while the userspace
uses the BPF syscall to send commands.

For example:

Map creation:

```
int
bpf_create_map(enum bpf_map_type map_type,
               unsigned int key_size,
               unsigned int value_size,
               unsigned int max_entries)
{
    union bpf_attr attr = {
        .map_type    = map_type,
        .key_size    = key_size,
        .value_size  = value_size,
        .max_entries = max_entries
    };

    return bpf(BPF_MAP_CREATE, &attr, sizeof(attr));
}
```

Map operation (lookup):

```
int
bpf_lookup_elem(int fd, const void *key, void *value)
{
    union bpf_attr attr = {
        .map_fd = fd,
        .key    = ptr_to_u64(key),
        .value  = ptr_to_u64(value),
    };

    return bpf(BPF_MAP_LOOKUP_ELEM, &attr, sizeof(attr));
}
```

BTF CO-RE
---------

"BTF (BPF Type Format) is the metadata format which encodes the debug info
related to BPF program/map."

"It's a minimalistic, compact format, inspired by Sun's CTF (Compact C Type
Format), which is used for representing kernel debug information since Solaris
9. BTF was created for similar purposes, with a focus on simplicity and
compactness to allow its usage in the Linux kernel."

It improves development experience.

This information can be used to pretty-print a map, a function, etc. These
information are generated by the compiler and saved in the `.BTF` section of the
eBPF ELF file. The format is documented in the Linux kernel documentation. There
are various types of BTF entries describing different things such as unions,
struct, enums, typedef, etc.

Why not using an existing format like DWARF? As Andrii Nakryiko says "The
problem with DWARF though, is that it's very generic and expressive, which makes
it a quite complicated and verbose format, and thus unsuitable to include in the
Linux kernel image due to size overhead and the complexity of parsing it."

For example, given this code:

```
 struct foo {
   int a;
   int b;
   unsigned c:15;
 } __attribute__((preserve_access_index));

 enum bar { U, V };
```

Clang will generate the following BTF information (shown as "pretty printed"
instead of their binary form):

```
 [2] STRUCT 'foo' size=8 vlen=2
        'a' type_id=3 bits_offset=0
        'b' type_id=3 bits_offset=32
        'c' type_id=4 bits_offset=64 bitfield_size=15
 [3] INT 'int' size=4 bits_offset=0 nr_bits=32 encoding=SIGNED
 [4] INT 'unsigned int' size=4 bits_offset=0 nr_bits=32 encoding=(none)
 ...
 [16] ENUM 'bar' encoding=UNSIGNED size=4 vlen=2
        'U' val=0
        'V' val=1
```

The logic that generates BTF information has to adhere to the following
requirements:

- deduplicate redundant BTF information and leave single instance of each unique type;
- merge and reconstruct complete type information across multiple CUs;
- handle type cycles correctly and efficiently;
- do all of the above fast and reliably to become a part of Linux kernel build process.

Good source:

    https://facebookmicrosites.github.io/bpf/blog/2018/11/14/btf-enhancement.html

CO-RE (Complice Once, Run Everywhere) improves development and distribution
experience by making an eBPF program portable across kernel versions. While the
eBPF sybsystem provides a stable sandbox for verious types of eBPF programs, an
issue appears when we have eBPF programs that interact with the rest of the
kernel's types and structures such as for kprobes and traceporints. The kernel's
internal API is intentinally and famously unstable, which allows for faster
development, cleaner code and less technical debth over time. However this makes
it difficult to distribute external code that is loaded in the kernel. Kernel
modules have a strict requirement that they need to be compiled for the same
kernel version as the kernel they are loaded with for this exact reason, but due
to the popularity and usefullness of eBPF a new system was designed.

A first solution was with BCC, where the program's C source code was distributed
raw, then compiled on the target machine. This worked but it required to have
the entire compilation toolchain necessary (clang/LLVM) which is quite heavy, so
a new solution was designed that required a lighter dependency.

From `Documentation/bpf/llvm_reloc.rst`:

```
From object file point of view CO-RE mechanism is implemented as a set
of CO-RE specific relocation records. These relocation records are not
related to ELF relocations and are encoded in .BTF.ext section.
See :ref:`Documentation/bpf/btf.rst <BTF_Ext_Section>` for more
information on .BTF.ext structure.

CO-RE relocations are applied to BPF instructions to update immediate
or offset fields of the instruction at load time with information
relevant for target kernel.

Field to patch is selected basing on the instruction class:

* For BPF_ALU, BPF_ALU64, BPF_LD `immediate` field is patched;
* For BPF_LDX, BPF_STX, BPF_ST `offset` field is patched;
* BPF_JMP, BPF_JMP32 instructions **should not** be patched.
```

This is how a relocation is stored (`include/uapi/linux/bpf.h`):

```
 * "struct bpf_core_relo" is used to pass relocation data form LLVM to libbpf
 * and from libbpf to the kernel.
 *
 * CO-RE relocation captures the following data:
 * - insn_off - instruction offset (in bytes) within a BPF program that needs
 *   its insn->imm field to be relocated with actual field info;
 * - type_id - BTF type ID of the "root" (containing) entity of a relocatable
 *   type or field;
 * - access_str_off - offset into corresponding .BTF string section. String
 *   interpretation depends on specific relocation kind:
 *     - for field-based relocations, string encodes an accessed field using
 *       a sequence of field and array indices, separated by colon (:). It's
 *       conceptually very close to LLVM's getelementptr ([0]) instruction's
 *       arguments for identifying offset to a field.
 *     - for type-based relocations, strings is expected to be just "0";
 *     - for enum value-based relocations, string contains an index of enum
 *       value within its enum type;
 * - kind - one of enum bpf_core_relo_kind;
 *
 * Example:
 *   struct sample {
 *       int a;
 *       struct {
 *           int b[10];
 *       };
 *   };
 *
 *   struct sample *s = ...;
 *   int *x = &s->a;     // encoded as "0:0" (a is field #0)
 *   int *y = &s->b[5];  // encoded as "0:1:0:5" (anon struct is field #1,
 *                       // b is field #0 inside anon struct, accessing elem #5)
 *   int *z = &s[10]->b; // encoded as "10:1" (ptr is used as an array)
 *
 * type_id for all relocs in this example will capture BTF type id of
 * `struct sample`.
 *
 * Such relocation is emitted when using __builtin_preserve_access_index()
 * Clang built-in, passing expression that captures field address, e.g.:
 *
 * bpf_probe_read(&dst, sizeof(dst),
 *		  __builtin_preserve_access_index(&src->a.b.c));
 *
 * In this case Clang will emit field relocation recording necessary data to
 * be able to find offset of embedded `a.b.c` field within `src` struct.
 *
 * [0] https://llvm.org/docs/LangRef.html#getelementptr-instruction
 */
struct bpf_core_relo {
	__u32 insn_off;
	__u32 type_id;
	__u32 access_str_off;
	enum bpf_core_relo_kind kind;
};
```

This example is really good. I can say it and expand it.

The kernel exposes BTF information to userspace via `/sys/kernel/btf/vmlinux`
and `/sys/kernel/btf/<module>` for dynamically loaded modules. Before loading a
program with libbpf, the library will read the program's relocation records
which the compiler generated in the `.BTF.ext` section. It will then check the
current kernel's BTF information, and use this to resolve the relocations using
the BTF information such as the name of an enum or the name of a field in a
struct. Therefore, if the address of a member in a struct changes, the
relocation will take care of setting the right offset before the program is
loaded.

The kernel is CO-RE agnostic, the relocations are done in userspace by libbpf.

Loading
-------

`kernel/bpf/syscall.c`:

```
SYSCALL_DEFINE5(bpf, BPF_PROG_LOAD, ...)
  -> __sys_bpf
    -> bpf_prog_load
```

After some setting up and a security hook, the verifier checks the program
(`kernel/bpf/syscall.c`):

```
err = security_bpf_prog_load(prog, attr, token, uattr.is_kernel);
if (err)
  goto free_prog;

err = bpf_check(&prog, attr, uattr, attr_log);
  if (err < 0)
    goto free_used_maps;
```

`bpf_check` is implemented in `kernel/bpf/verifier.c`.

Security Considerations
-----------------------

From
"https://www.linuxfoundation.org/hubfs/eBPF/ControlPlane%20%E2%80%94%20eBPF%20Security%20Threat%20Model.pdf":

under Confidentiality and Integrity:

"This threat model considers privileged, platform-level workloads running eBPF
programs. If an external attacker can compromise a user-space agent which loads
eBPF code, their goal may be to read sensitive data processed by one of the
platform tenants. An external attacker may then leverage such a weakness in a
user-space agent which loads eBPF code to read sensitive data processed by one5
of the platform tenants"

under availability, the Verfier can limit damage.

eBPF uses the principle of least priviledges to deminish the capabilities of an
attacker. An attacker could:

- compromized priveldged user
  - to limit the attack surface, a system of capabilities is used
- compromize an eBPF program
  - verification makes sure that he cannot crash the system, although it can
    cause issues with availability. This is solved by allowing only signed eBPF
    programs.
- compromize an eBPF loader, so it can alter which program gets loaded

JIT compiled image are read-only.

The verifier is obvisouly dong a lot of work.

We have LSM hooks for eBPF, which allow for external tools to setup custom
access control systems to load and interact with the eBPF subsystem.

The `bpf` syscall in `kernel/bpf/syscall.c` checks security:

```
static int __sys_bpf(enum bpf_cmd cmd, bpfptr_t uattr, unsigned int size,
		     bpfptr_t uattr_common, unsigned int size_common)
{
    // ...
    err = security_bpf(cmd, &attr, size, uattr.is_kernel);
    if (err < 0)
      return err;
    // ...
}
```

This is a LSM hook which is enabled if the kernel was compiled with
`CONFIG_SECURITY`. This hook allows security modules to make policy decisions
about BPF commands. LSM trampolines are registered in `security/bpf/`.

We also have capabilities, and we have different eBPF program type. Therefore, a
program can only interact with the subset of functionalities that it was
intended to via a limited "sandboxed" interface, unlike kernel modules. This is
enforced by the verifier, and greately reduces potential issues.

But we have a supply chain attack (integrity) problem.

This solves supply chain attacks.

The problem: if we sign an eBPF program after compilation, then the programm
will be altered by CORE relocations, updating maps file descriptors and other
things so the signature is not valid anymore. This issue for signing has been
highly debated, and several solutions are being proposed such as
hypervisor-based integrity checks, soving the program's modifications inside the
kernel (this was considered but deemed not feasable since it would add a lot of
complexity inside the kernel and less flexibility).

A solution proposed by Cong Wang is to use a 2 step verification, where we
use a trusted loader (a process started after secure boot) that takes a signed
ebpf program, modifies it, and adds its own signature. This established a chain
of trust: the kernel checks the original eBPF signature, and it will check the
loader's signature that ensure modifications are trustable.

There are issues with this approach, as the root user may tamper with the loader
and the accepted keys, TOCTOU (Time-of-Check to Time-of-Use). A possible
solution to this would be to bake the eBPF programs in the kernel at compile
time, much like a kernel module. This would solve the security problem at load
time by trusting the toochain during build time, but would not allow trusted
dynamic eBPF loading which is the main appeal of eBPF. These issues are still
discussed and new solutions are in the work.

So this last approach was dead.

Another solution proposed by Blaise Boscaccy based on "light-skeleton" loading
which is based on a small signed eBPF program checked using fs-verify, an
approach proposed by Alexei Starovoitov, and their only purpose is to load other
eBPF programs (this is enforced by a LSM). Since they are very small programs,
they do not need CO-RE relocations so their signature is enough to ensure that
the program that will be executed is the same as the one it was compiles. The
LSM also ensures that an user does not tamper with this light skelethon by
hooking at systemcalls.

A light-skelethon (often called lskel) is an eBPF program whose sole purpose is
to load another eBPF program, setting up its maps and link.

This LSM is called Hornet (not mainlined yet) (`security/hornet/`). Available
with CONFIG_SECURITY_HORNET=y

    https://lore.kernel.org/bpf/20260507191416.2984054-1-bboscaccy@linux.microsoft.com/

"Hornet addresses concerns from users who require strict audit trails and
verification guarantees for eBPF programs, especially in security-sensitive
environments. Many production systems need assurance that only authorized,
unmodified eBPF programs are loaded into the kernel. Hornet provides this
assurance through cryptographic signature verification."

"Hornet itself does not enforce a policy on whether unsigned or partially signed
programs should be rejected. It delegates that decision to downstream LSMs via
the bpf_prog_load_post_integrity hook, making it a composable building block in
a larger security architecture."

Documentation says:

Hornet protects against the following threats:

- Unauthorized eBPF program loading: Programs that have not been signed by a
  trusted key will be reported as unsigned or badly signed.
- Tampering with program instructions: Any modification to the eBPF bytecode
  after signing will cause signature verification to fail.
- Tampering with map data: When map hashes are included in the signature, Hornet
  verifies that frozen BPF maps match their expected SHA-256 hashes at load time
  after the program is publically exposed.

Hornet does not protect against:

- Compromise of the signing key itself.
- Attacks that occur after a program has been loaded and verified.
- Programs loaded by the kernel itself (kernel-internal loads bypass the map
  check).

After reading the mailing lists, apparently there are the eBPF developers who
are doing something, and the Hornet LSM who are doing something else.

They discussed eBPF signing options at the LSFMMBPF, see this LWN article:

    https://lwn.net/Articles/1017549/
    Code signing for BPF programs
    April 22, 2025
    The conference was at 24-26 March 2025

There they talked about Hornet which is based on lskel, and about 2 step
verification.

Apparently Hornet was Nacked.. see:

    https://lore.kernel.org/all/CACYkzJ6VQUExfyt0=-FmXz46GHJh3d=FXh5j4KfexcEFbHV-vg@mail.gmail.com/
    Re: [PATCH v3 0/4] Introducing Hornet LSM
    11 May 2025

There is tension between the Hornet LSM people and the eBPF people. Alexey
wanted the signing logic to be in the eBPF subsystem, not in some hacky
policies. They proposd about a trusted hash chain.

So they proposed this patchset:

    https://lore.kernel.org/bpf/20250921160120.9711-1-kpsingh@kernel.org/
    [PATCH bpf-next v7 0/5] Signed BPF programs
    21 Sep 2025

This is all about that trusted hash chain. Apparently this is merged. The idea
is to sign the loader program's bytes. Because the loader program contains the
expected hashes of the target program, signing the loader transitively
guarantees the integrity of the target program and its maps.

The latest iteration of Hornet is v7, apparently they did not give up:

    https://lore.kernel.org/bpf/20260507191416.2984054-1-bboscaccy@linux.microsoft.com/
    [PATCH v7 00/10] Reintroduce Hornet LSM
    7 May 2026

Paul More says the merjed most of these patches in lsm/dev, but I don't see
them!

Apparently this is what they want to do:

    https://lore.kernel.org/bpf/20260522023234.3778588-1-kpsingh@kernel.org/
    [PATCH bpf-next 00/13] Signed BPF + IPE Policies
    22 May 2026

From my understanding, the sign checking logic that Hornet wanted to implement
was moved in the eBPF subsystem, so Hornet is now just some glue between the
eBPF subsystem and IPE used for audit trails and system state logging, but not
necessary to load signed programs.

This is the most recent patch:

    https://lore.kernel.org/bpf/20260708075343.358712-1-daniel@iogearbox.net/
    [PATCH bpf-next v6 0/8] Verify BPF signed loader at load time
    8 Jul 2026

TODO: tokens:

    https://lore.kernel.org/bpf/20240124022127.2379740-1-andrii@kernel.org/

Use Cases and modern ecosystem
------------------------------

eBPF schedulers are famously used by META.

Indeed this has been used by cloudflare to patch the copy fail vulnerability
before updating the kernels. See:

    https://blog.cloudflare.com/copy-fail-linux-vulnerability-mitigation/

From the Linux Fundation "The State of eBPF", some quotes about security:

"You cannot secure what you cannot see,” says Amit Gupta, chief
product officer at Tigera, which provides a security platform to
prevent, detect, and mitigate security breaches in cloud native
applications. “The first step in any kind of secure architecture is
to get full, secure visibility. eBPF helps create that visibility. When
eBPF data comes back to user space, you can then coordinate with
everything else that’s happening … and make a decision on whether
it is likely to be malicious activity,” Gupta says. Instead of taking
months to ferret out attackers or even missing them completely,
eBPF tools can enable action “within seconds,” Gupta says"

"BPF also pushes security enforcement policies into distributed
environments so that they get implemented in real time. If a
vulnerability occurs in the kernel, for instance, fast fixes can
occur via eBPF without altering the kernel code, allowing for
security updates on the fly."

From the Linux Fundation's "State of eBPF" (January 2024) we have some data:

https://www.linuxfoundation.org/hubfs/eBPF/The_State_of_eBPF.pdf

Some quotes from the document:

“eBPF is a big deal in the sense that it allows you to do things that
were either really difficult or practically impossible before,” says
Toke Høiland-Jørgensen, a Senior Principal Kernel Engineer at
Red Hat.

"No doubt, eBPF will become the new layer in the new cloud
native infrastructure stack, impacting the observability,
performance, reliability, networking, and security of all
applications, supporters say. Platform engineers will cobble
together eBPF-powered infrastructure building blocks to
create platforms that developers then deploy software on,
adding business logic to the mix, and replacing aging Linux
kernel internals that cannot keep up with today’s digital and,
increasingly, cloud native world."

Some other data about eBPF in production:

https://ebpf.foundation/new-ebpf-in-production-report-showcases-production-enterprise-outcomes-across-networking-security-and-observability/
