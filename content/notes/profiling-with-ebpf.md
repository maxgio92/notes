---
Title: CPU profiling with eBPF
---

libbpf:
- Docs: https://www.kernel.org/doc/html/latest/bpf/libbpf/
- Example: https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/profile.bpf.c
- From BCC: https://nakryiko.com/posts/bcc-to-libbpf-howto-guide/

Examples from Linux sources:
- User space: https://github.com/torvalds/linux/blob/b0546776ad3f332e215cebc0b063ba4351971cca/samples/bpf/trace_event_user.c
- Kernel space: https://github.com/torvalds/linux/blob/b0546776ad3f332e215cebc0b063ba4351971cca/samples/bpf/trace_event_kern.c

Perf UAPI and eBPF user libraries:
- `perf_event_attr` struct: https://github.com/torvalds/linux/blob/b0546776ad3f332e215cebc0b063ba4351971cca/include/uapi/linux/perf_event.h#L389
- `perf_event_open` syscall: https://man7.org/linux/man-pages/man2/perf_event_open.2.html
- perf: https://perf.wiki.kernel.org/index.php/Main_Page
- How to attach per events with Cilium:
  - https://gist.github.com/florianl/5d9cc9dbb3822e03f6f65a073ffbedbb
  - https://github.com/cilium/ebpf/discussions/800
  - https://github.com/cilium/ebpf/discussions/548

Blogs:
- https://www.brendangregg.com/blog/2016-10-21/linux-efficient-profiler.html
- https://blog.px.dev/cpu-profiling/ (a series)
- https://github.com/pixie-io/pixie-demos/blob/main/ebpf-profiler (with BCC)
- https://www.airplane.dev/blog/building-an-ebpf-based-profiler (with libbpf)

Other material:
- [Perf ring buffer and lost events](http://blog.itaysk.com/2020/04/20/ebpf-lost-events)
- [BPF ring buffer](https://www.kernel.org/doc/html/latest/bpf/ringbuf.html)
- [`BPF_MAP_TYPE_STACK`](https://www.kernel.org/doc/html/latest/bpf/map_queue_stack.html)
- [Brendan Gregg's stack trace hack](https://www.brendangregg.com/blog/2016-01-18/ebpf-stack-trace-hack.html)
- [Official Linux Offwaketime:
  - [bpf](https://elixir.bootlin.com/linux/latest/source/samples/bpf/offwaketime.bpf.c)
  - [user](https://elixir.bootlin.com/linux/latest/source/samples/bpf/offwaketime_user.c)
- https://stackoverflow.com/questions/23618743/implementing-an-interrupt-driven-sampling-profiler 

Stack walking:
- [x86 assembly](./X86-stack-walking.pdf)
- https://c9x.me/x86/html/file_module_x86_id_154.html
- https://www.polarsignals.com/blog/posts/2022/11/29/dwarf-based-stack-walking-using-ebpf
- [DataDog's Stack traces in Go](https://github.com/DataDog/go-profiler-notes/blob/main/stack-traces.md)
- https://stackoverflow.com/questions/57268045/how-to-read-stack-trace-kernelside-in-ebpf
- NEW: https://www.brendangregg.com/blog/2024-03-17/the-return-of-the-frame-pointers.html

Symbolization:
- https://jvns.ca/blog/2018/01/09/resolving-symbol-addresses/
- https://www.polarsignals.com/blog/posts/2022/01/13/fantastic-symbols-and-where-to-find-them

Beyond frame pointers:
- ORC: https://lwn.net/Articles/728339/
- LBR: https://lwn.net/Articles/680985/

Extra:
- Off-CPU profiling: https://www.brendangregg.com/FlameGraphs/offcpuflamegraphs.html
