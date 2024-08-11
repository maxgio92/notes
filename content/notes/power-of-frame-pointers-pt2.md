---
Title: Unleashing the power of frame pointers for profiling pt.1 - Writing a simple profiler
---

In the previous blog, I introduced the program execution environment and I have introduced the concepts of stack unwinding with frame pointers as one of the techniques leveraged for profiling a program.

In this blog, we'll see practically how we can build a simple profiler that leverages frame pointers by sampling stack traces.

In order to limit the overhead a profiler can work with the help of the Linux kernel, and more precisely eBPF allows it to run at specific kernel paths programs, which is in our case the sampler, without the need to load modules.
This way the analysed program doesn't need to be instrumented.

The sampler needs to collect stack traces with a fixed frequency. With the samples in user space calculate the statistics.

To summarize:
- kernel space: sample stack traces for a specific process with a fixed frequency;
- userspace: collect samples, calculate the statistics, and resolve subroutine symbols.

## Kernel space

To run the program with a fixed frequency the perf subsystem provides us a clock software event ([`PERF_COUNT_SW_CPU_CLOCK`](https://elixir.bootlin.com/linux/v6.8.5/source/include/uapi/linux/perf_event.h#L119)) and luckily eBPF programs can be attached to perf events.

So we'll leverage a software CPU clock Perf event just to be able to run the probe every x nanoseconds. The userspace loader prepares the Perf event to attach the program to it:

```go
attr := &unix.PerfEventAttr{

  // If type is PERF_TYPE_SOFTWARE, we are measuring software events provided by the kernel.
  Type: unix.PERF_TYPE_SOFTWARE,

  // This reports the CPU clock, a high-resolution per-CPU timer.
  Config: unix.PERF_COUNT_SW_CPU_CLOCK,

  // A "sampling" event is one that generates an overflow notification every N events,
  // where N is given by sample_period.
  // sample_freq can be used if you wish to use frequency rather than period.
  // sample_period and sample_freq are mutually exclusive.
  // The kernel will adjust the sampling period to try and achieve the desired rate.
  Sample: t.samplingPeriodMillis * 1000 * 1000,
}

t.logger.Debug().Msg("opening the sampling software cpu block perf event")

// Create the perf event file descriptor that corresponds to one event that is measured.
// We're measuring a clock timer software event just to run the program on a periodic schedule.
// When a specified number of clock samples occur, the kernel will trigger the program.
evt, err := unix.PerfEventOpen(
  // The attribute set.
  attr,

  // the specified task.
  //t.pid,
  -1,

  // on the Nth CPU.
  i,

  // The group_fd argument allows event groups to be created. An event group has one event which
  // is the group leader. A single event on its own is created with group_fd = -1 and is considered
  // to be a group with only 1 member.
  -1,

  // The flags.
  0,
)
if err != nil {
  return nil, errors.Wrap(err, "error creating the perf event")
}
defer func() {
  if err := unix.Close(evt); err != nil {
    t.logger.Fatal().Err(err).Msg("failed to close perf event")
  }
}()
```

eBPF helpers are functions that, as you might have guessed, simplify work. The [`bpf_get_stackid`](https://elixir.bootlin.com/linux/v6.8.5/source/kernel/bpf/stackmap.c#L283) helper returns the stack id of the program that is currently running, at the moment of the eBPF program execution in the very process context.

```c
key.kernel_stack_id = bpf_get_stackid(ctx, &stack_traces, 0 | BPF_F_FAST_STACK_CMP);
key.user_stack_id = bpf_get_stackid(ctx, &stack_traces, 0 | BPF_F_FAST_STACK_CMP | BPF_F_USER_STACK);
```

eBPF maps are ways to exchange data, often useful with programs running in user space. There are different types of maps. We use an hash map (`BPF_MAP_TYPE_HASH`) to exchange the histogram with the stack IDs:

```c
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__type(key, histogram_key_t);		/* per-process stack trace key */
	__type(value, u64);			/* sample count */
	__uint(max_entries, K_NUM_MAP_ENTRIES);
} histogram SEC(".maps");
...
SEC("perf_event")
int sample_stack_trace(struct bpf_perf_event_data* ctx)
{
	...
	u64 one = 1;
	...
	/* Upsert stack trace histogram */
	count = (u64*)bpf_map_lookup_elem(&histogram, &key);
	if (count) {
	  (*count)++;
	} else {
	  bpf_map_update_elem(&histogram, &key, &one, BPF_NOEXIST);
	}
}
```

and the type [`BPF_MAP_TYPE_STACK_TRACE`](https://elixir.bootlin.com/linux/v6.8.5/source/include/uapi/linux/bpf.h#L914):

**eBPF**:

```c
struct {
	__uint(type, BPF_MAP_TYPE_STACK_TRACE);
	__uint(key_size, sizeof(u32));
	__uint(value_size, PERF_MAX_STACK_DEPTH * sizeof(u64));
	__uint(max_entries, K_NUM_MAP_ENTRIES);
} stack_traces SEC(".maps");
```

that we can access directly in user space to retrieve the sampled stack's trace, by passing the sampled stack ID:

**Userspace with libbpf-go**:

```go
func (t *Profile) getStackTrace(stackTraces *bpf.BPFMap, id uint32) (*StackTrace, error) {
	stackBinary, err := stackTraces.GetValue(unsafe.Pointer(&id))
	if err != nil {
		return nil, err
	}

	var stackTrace StackTrace
	err = binary.Read(bytes.NewBuffer(stackBinary), binary.LittleEndian, &stackTrace)
	if err != nil {
		return nil, err
	}

	return &stackTrace, nil
}
```

This is mostly the needed work in kernel space, which is pretty simple, thanks to the available kernel instrumentation.

## User space

In user space, we're able then to access:
- the sampled stack IDs
- the stack traces (keyed by their stack IDs)

We access the histogram of the sampled stacks that contain their IDs, to collect the traces.
Once the sampling completes, we're able to calculate the program's residency fraction for each subroutine, that is, how much a specific subroutine has been run within a time frame:

```
residencyFraction = nStackSamples / nTotalSamples * 100.
```

```go
fractionTable := make(map[string]float64, len(countTable))
for trace, count := range countTable {
	residencyFraction := float64(count) / float64(sampleCount)
	fractionTable[trace] = residencyFraction
}
```

Finally, because traces are an array of pushed instruction pointers, we need to translate IPs to symbols.

### Symbolization

There are different ways to resolve symbols based on the binary format and the way the binary has been compiled.

Because this is a demonstration and the profiler is simple we'll consider just ELF binaries that are not stripped.

The ELF structure contains a symbol table (`symtab` section) that holds information needed to locate and relocate a program's symbolic definitions and references. With that information, we're able to, if this table is not missing as in the case of stripped binaries, associate instruction addresses with subroutine names.

As the user space program is written in Go, we can leverage the `debug/elf` from the standard library to access that information with:

```go
// Open the file
file, err := elf.Open(pathname)
...
// Read symbols from the .symtab section
syms, err := file.Symbols()
...
for _, s := range syms {
  // The symbol is correct if the trace instruction pointer address
  // is within the symbol address range
  if ip >= s.Value && ip < (s.Value+s.Size) {
    sym = s.Name
  }
}
```

## Binary path

To access the ELF binary we need the process's binary pathname. Thanks to the `binprm_info` kernel structure we're able to collect this information from the eBPF program and pass it through a map to the userspace program.

```c
struct path path = BPF_CORE_READ(task, mm, exe_file, f_path);

buffer_t *string_buf = get_buffer(0);
if (string_buf == NULL) {
  return NULL;
}
/* Write path string from path struct to the buffer */
size_t buf_off = get_pathname_from_path(&path, string_buf);
return &string_buf->data[buf_off];
```

## Wrapping up

The program loads the eBPF program, attaches it to the perf event created, and reads stack IDs which accesses stack traces. With traces' instruction pointers it resolves symbols and on exit, it calculates the statistics as explained before. For example:

```
80% main; foo; bar;
20% main; foo; baz;
```

## Thanks
