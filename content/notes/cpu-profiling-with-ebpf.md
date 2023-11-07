---
Title: CPU profiling with eBPF
---

Links:
- https://www.brendangregg.com/blog/2016-10-21/linux-efficient-profiler.html
- https://blog.px.dev/cpu-profiling/ (a series)

Examples:
- https://github.com/pixie-io/pixie-demos/blob/main/ebpf-profiler (with BCC)
- https://www.airplane.dev/blog/building-an-ebpf-based-profiler (with libbpf)

Other material:
- https://github.com/iovisor/gobpf
- with BCC and [`perf`](https://perf.wiki.kernel.org/index.php/Main_Page).
- [Perf ring buffer and lost events](http://blog.itaysk.com/2020/04/20/ebpf-lost-events)
- [`perf_event_open(2)`](https://man7.org/linux/man-pages/man2/perf_event_open.2.html)
- [BPF ring buffer](https://www.kernel.org/doc/html/latest/bpf/ringbuf.html)
- [`BPF_MAP_TYPE_STACK`](https://www.kernel.org/doc/html/latest/bpf/map_queue_stack.html)
- [Brendan Gregg's stack trace hack](https://www.brendangregg.com/blog/2016-01-18/ebpf-stack-trace-hack.html)

Stack walking:
- [x86 assembly](./X86-stack-walking.pdf)
- https://c9x.me/x86/html/file_module_x86_id_154.html
- https://www.polarsignals.com/blog/posts/2022/11/29/dwarf-based-stack-walking-using-ebpf
- [Stack traces in Go stack traces](https://github.com/DataDog/go-profiler-notes/blob/main/stack-traces.md)
