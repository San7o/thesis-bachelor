<h1 align="center">Design and Implementation of Linux Tracing Infrastructure </h1>

<h3 align="center">A Deep Dive from Traditional Frameworks to eBPF-Driven Observability</h3>

Observability and deep kernel visibility are essential for maintaining the
performance, reliability, and security of modern operating systems. Linux
supports a vast number of powerful tracing capabilities to support this: over
the history of the operating system, multiple frameworks and event sources have
been introduced for different use cases. Due to the size and complexity of the
Linux kernel, these capabilities are often poorly understood by developers. To
demystify this landscape, an analysis of the Linux kernel codebase was
conducted, gathering information through source code, git commits, mailing
lists, technical articles, and conference talks. This work presents a
comprehensive description of the tracing infrastructure provided by Linux kernel
version 7.2, focusing on traditional frameworks like Ftrace and Perf, event
sources like tracepoints and kprobes, and modern developments through eBPF.
Finally, a real-world use case for eBPF in containerized environments is
studied, introducing Kivebpf as an application that leverages modern Linux
tracing infrastructure.

[Read online](https://san7o.github.io/thesis-bachelor/)

## Table of Contents

- Abstract

### 1. Introduction
  - 1.1 The tracing stack
    - 1.1.1 Event Sources
    - 1.1.2 Tracing Frameworks
    - 1.1.3 User Interfaces
    - 1.1.4 Frontends

### 2. Tracing frameworks
  - 2.1 Ftrace
    - 2.1.1 The ftrace global ring buffer
    - 2.1.2 The tracer struct
    - 2.1.3 Static function tracing
    - 2.1.4 Dynamic function tracing
    - 2.1.5 The tracefs filesystem
  - 2.2 Perf
    - 2.2.1 The Perf ring buffer
    - 2.2.2 The Performance Monitoring Unit
    - 2.2.3 The perf_event_open() syscall
    - 2.2.4 Security considerations
  - 2.3 Others

### 3. Event Sources
  - 3.1 Tracepoints
    - 3.1.1 Tracepoint related types
    - 3.1.2 Tracepoint creation
    - 3.1.3 static_call and static_key mechanism
  - 3.2 Event Tracing
    - 3.2.1 Event classes
  - 3.3 Syscall Tracepoints
  - 3.4 Kprobes and Kretprobes
    - 3.4.1 Kprobe interface
    - 3.4.2 Kprobe implementation
  - 3.5 Perf events

### 4. eBPF
  - 4.1 Introduction and overview
  - 4.2 Instruction Set
    - 4.2.1 Standardization
    - 4.2.2 Instructions and encoding
    - 4.2.3 Helper Functions
    - 4.2.4 Maps and platform variables
    - 4.2.5 Compiler Support
  - 4.3 The verifier
    - 4.3.1 Initial checks and preparations
    - 4.3.2 Register types
    - 4.3.3 Program simulation
    - 4.3.4 Range analysis
    - 4.3.5 State pruning
  - 4.4 Program Types
    - 4.4.1 Attach points
    - 4.4.2 Trampolines
    - 4.4.3 BPF_PROG_TYPE_XDP
    - 4.4.4 BPF_PROG_TYPE_TRACEPOINT
    - 4.4.5 BPF_PROG_TYPE_LSM
  - 4.5 Helper functions and Kfuncs
  - 4.6 Maps
  - 4.7 BTF and CO-RE
  - 4.8 Security Considerations
  - 4.9 eBPF in production

### 5. Kivebpf
  - 5.1 Introduction
    - 5.1.1 Resources
    - 5.1.2 Alert
    - 5.1.3 Components
  - 5.2 Detecting file accesses
    - 5.2.1 Uniquely identifying a file
  - 5.3 eBPF Program
    - 5.3.1 Implementation

### 6. Conclusions

### Bibliography
