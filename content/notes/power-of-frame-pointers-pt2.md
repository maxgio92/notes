---
Title: Unleashing the power of frame pointers for profiling pt.1 - Writing a simple profiler
---

In the previous blog I introduced the program execution environment and I have introduced the concepts of stack unwinding with frame pointers as one of the techniques leveraged for profiling a program.

In this blog we'll see practically how we can build a simple profiler which leverages frame pointers by sampling stack traces.

In order to limit the overhead a profiler can work with the help of the Linux kernel, and more procisely eBPF allows to run at specific kernel paths programs, which is in our case the sampler, without the need to load modules.
This way the analysed program doesn't need to be instrumented.

The sampler needs to collect stack traces with a fixed frequency. With the samples in user space calculate the statistics.

To summarize:
- kernel space: sample stack traces for a specific process with a fixed frequency;
- user space: collect samples, calculate the statistics and resolve subroutine symbols.

## Kernel space

To run the program with a fixed frequency the perf subsystem provides us a clock event and luckily eBPF programs can be attached to perf events.

So we'll leverage a software CPU clock Perf event just to be able to run the probe every x nanoseconds.

eBPF helpers are functions that, as you might have guessed, simplify work. The bpf_get_stackid helper returns the stack id of the program that is currently running, at the moment of the eBPF program execution in the very process' context.

eBPF maps are ways to exchange data, often useful with programs running in user space. There are differnt type of maps and one of them is the BPF_MAP_TYPE_STACK_TRACE that we can access directly in user space to to retrieve the sampled stack's trace, by passing the sampled stack ID.

This is mostly the needed work in kernel space, which is pretty simple, thanks to the available kernel instrumentation.

## User space

In user space we're able then to access:
- the sampled stack IDs
- the stack traces (keyed by their stack IDs)

We access the histogram of the sampled stacks that contains the their IDs, to collect the traces.
Once the sampling completes, we're able to provide the information of how much a specific stack trace (function) has been run within the profile time's frame, by calculating:

stackRunPercentage = nStackSamples / nTotalSamples * 100.

Finally, because traces are basically array of the pushed instruction pointers, we need to translate IPs to symbols.

### Symbolization

There are different ways to resolve symbols based on the binary format and the way the binary has been compiled.

Because this is a demonstration and the profiler is simple we'll consider just ELF binaries that are not stripped.

The ELF structure contains a symbol table that holds information needed to locate and relocate a program's symbolic definitions and references. With that information we're able to, if this table is not missing as the case of stripped binaries, associate instruction addresses with subroutine names.

As the user space program is written in Go, we can leverage the standard library to access that information with:

```go
elf.ETC ETC ETC
```

The program loads the eBPF program, attaches it to the perf event created, reads stack IDs with which accesses stack traces. With traces' instruction pointers it resolves symbols and on exit, it calculates the statistics as exmplained before. For example:

```
80% main; foo; bar;
20% main; foo; baz;
```

## Thanks
To run the program with a fixed frequency the perf subsystem provides us a clock event and luckily eBPF programs can be attached to perf events.

So we'll leverage a software CPU clock Perf event just to be able to run the probe every x nanoseconds.

eBPF helpers are functions that, as you might have guessed, simplify work. The bpf_get_stackid helper returns the stack id of the program that is currently running, at the moment of the eBPF program execution in the very process' context.

eBPF maps are ways to exchange data, often useful with programs running in user space. There are differnt type of maps and one of them is the BPF_MAP_TYPE_STACK_TRACE that we can access directly in user space to to retrieve the sampled stack's trace, by passing the sampled stack ID.

This is mostly the needed work in kernel space, which is pretty simple, thanks to the available kernel instrumentation.

## User space

In user space we're able then to access:
- the sampled stack IDs
- the stack traces (keyed by their stack IDs)

We access the histogram of the sampled stacks that contains the their IDs, to collect the traces.
Once the sampling completes, we're able to provide the information of how much a specific stack trace (function) has been run within the profile time's frame, by calculating:

stackRunPercentage = nStackSamples / nTotalSamples * 100.

Finally, because traces are basically array of the pushed instruction pointers, we need to translate IPs to symbols.

### Symbolization

There are different ways to resolve symbols based on the binary format and the way the binary has been compiled.

Because this is a demonstration and the profiler is simple we'll consider just ELF binaries that are not stripped.

The ELF structure contains a symbol table that holds information needed to locate and relocate a program's symbolic definitions and references. With that information we're able to, if this table is not missing as the case of stripped binaries, associate instruction addresses with subroutine names.

As the user space program is written in Go, we can leverage the standard library to access that information with:

```go
elf.ETC ETC ETC
```

The program loads the eBPF program, attaches it to the perf event created, reads stack IDs with which accesses stack traces. With traces' instruction pointers it resolves symbols and on exit, it calculates the statistics as exmplained before. For example:

```
80% main; foo; bar;
20% main; foo; baz;
```

## Thanks
