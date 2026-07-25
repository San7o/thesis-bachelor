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

The verifier
------------

Good list of safety requirements:

    https://docs.ebpf.io/linux/concepts/verifier/

The verifier checks these requirements, so that the program can be executed
without overhead.

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
  bpf_reg_state`.
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

Program Types
-------------

Each eBPF program has a certain type. Types are enumerated in `enum
bpf_prog_type` in `include/uapi/linux/bpf.h`.



Helper Functions
----------------

Helper functions can be used by the eBPF program and are considered User Space
API (UAPI), meaning that the API will be honored on future kernel versions.

TODO: read the source code on how the handshake between the eBPF program and the
functions is implemented

TODO: They are JIT compiled to normal calls (?), investigate

TODO: helper vs kfuncs ???

Kfuncs (should it be its own section???)

VPF Kernel Functions (`kfuncs`) are exposed to BPF programs, but do not have a
stable interface. They are exposed either by making an existing function
visible, or add a new wrapper for BPF.

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

TODO: how does the verifier check those??

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

There are many typers of maps, generic and non-generic.

TODO: read code. Focus on architecture instead of the various types of maps.

BTF
---

TODO

CO-RE
-----

TODO: Info here
https://origin.kernel.org/doc/html/latest/bpf/llvm_reloc.html#co-re-relocations
Essentially, CO-RE is based on relocations. The eBPF program is compiled but not
linked. It is actually linked when loaded, and the missing symbols' addresses
are fetched from the current running kernel via BTF.

Loading
-------

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

Example
-------

TODO: Show an example program + loader, with commentary

Security Considerations
-----------------------

- Both eBPF interpreter image and JIT compiled image are read-only

TODO: tokens https://docs.ebpf.io/linux/concepts/token/

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

TODO: underdtand these trampolines

TODO: layer, the syscall checks for capabilities. explore this

TODO: look at hornet (`security/hornet/`)

    https://lore.kernel.org/bpf/20260507191416.2984054-1-bboscaccy@linux.microsoft.com/

TODO: Tokens 

    https://lore.kernel.org/bpf/20240124022127.2379740-1-andrii@kernel.org/

Use Cases and modern ecosystem
------------------------------

TODO

Others
------

Trampolines: https://github.com/torvalds/linux/commit/fec56f5890d93fc2ed74166c397dc186b1c25951
