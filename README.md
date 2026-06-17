# Thesis

## Usage

clone:

```sh
git clone https://github.com/San7o/thesis.git
cd thesis
```

install Dependencies:

```sh
npm ci
```

build:

```sh
npm run build
```

## Links

Most useful resources:

- [The Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)
   - More specifically, the
     [tracing](https://www.kernel.org/doc/html/latest/trace/index.html) section
- [Linux Kernel Internals](https://kernel-internals.org/)

Code:

- [linux-kprobe-example](https://github.com/San7o/linux-kprobe-example): example
  kernel module that registers a kprobe on a syscall
- [linux-tracepoint-example](https://github.com/San7o/linux-tracepoint-example):
  example kernel module registering a tracepoint
- [linux-perf-event-example](https://github.com/San7o/linux-perf-event-example):
  example kernel module registering a software perf event through a Performance
  Monitor Unit (PMU)
- [ebpf-c-sample](https://github.com/San7o/ebpf-c-sample): sample project for an
  eBPF program + loader in C with libbpf and clang
