eBPF
====

Best resources:

- Kernel Docs: https://origin.kernel.org/doc/html/latest/bpf/
- eBPF docs: https://docs.ebpf.io/
- Cilium: https://docs.cilium.io/en/latest/reference-guides/bpf/index.html

Many resources:

    https://docs.cilium.io/en/stable/reference-guides/bpf/resources/#xdp-newbies

Instruction Set
---------------

TODO: we can look directly to the ISA and give a quick overview:

    https://datatracker.ietf.org/doc/rfc9669/

It uses the C calling convention.

ABI: https://origin.kernel.org/doc/html/latest/bpf/standardization/abi.html

The verifier
------------

TODO: read the code and docs

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

Types are:

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

TODO: check types in code

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

Program Types
-------------

TODO: read the code

TODO: there are also iterators, which are very useful

Helper Functions
----------------

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

Security Considerations
-----------------------

- Both eBPF interpreter image and JIT compiled image are read-only

TODO

Use Cases and modern ecosystem
------------------------------

TODO

