---
Title: Unleashing the power of frame pointers for profiling pt.2 - Writing a simple profiler
---

In the previous blog about the program execution environment I introduced the concepts of stack unwinding with frame pointers as one of the techniques leveraged for profiling a program.

In this blog, we'll see practically how we can build a simple profiler that leverages frame pointers sampling stack traces to calculate statistics of program's subroutines.

In order to limit the overhead, the Linux kernel instrumentation can help the profiler with that work, and more precisely eBPF allows it to dynamically run at specific kernel paths programs, which in our case it's a stack trace sampler, without the need to build and load modules. This way the analysed program doesn't need to be instrumented.

To summarize:
- in kernel space: an eBPF sampler program samples with a fixed frequency stack traces for a specific process;
- in userspace: a program collect the samples, calculates the statistics, and resolves subroutine's symbols.

## Kernel space

To run the program with a fixed frequency the perf subsystem exposes us a kernel software event of type CPU clock ([`PERF_COUNT_SW_CPU_CLOCK`](https://elixir.bootlin.com/linux/v6.8.5/source/include/uapi/linux/perf_event.h#L119). Luckily, eBPF programs can be attached to those events.

So we'll leverage a software CPU clock Perf event just to be able to probe every x nanoseconds. As perf exposes user APIs, the userspace program can prepare the clock software events for all the CPUs and attach the eBPF program to them:

```go
attr := &unix.PerfEventAttr{
	Type: unix.PERF_TYPE_SOFTWARE,		// If type is PERF_TYPE_SOFTWARE, we are measuring software events provided by the kernel.
	Config: unix.PERF_COUNT_SW_CPU_CLOCK,	// This reports the CPU clock, a high-resolution per-CPU timer.
	
	// A "sampling" event is one that generates an overflow notification every N events,
	// where N is given by sample_period.
	// sample_freq can be used if you wish to use frequency rather than period.
	// sample_period and sample_freq are mutually exclusive.
	// The kernel will adjust the sampling period to try and achieve the desired rate.
	Sample: t.samplingPeriodMillis * 1000 * 1000,
}

// Create the perf event file descriptor that corresponds to one event that is measured.
// We're measuring a clock timer software event just to run the program on a periodic schedule.
// When a specified number of clock samples occur, the kernel will trigger the program.
evt, err := unix.PerfEventOpen(
	attr,	// The attribute set.
	t.pid,	// The specified task.
	i,	// on the Nth CPU.
	-1,	// The group_fd argument allows event groups to be created.
	0,	// The flags.
)

// ...

// Attach the BPF program to the sampling perf event.
if _, err = prog.AttachPerfEvent(evt); err != nil {
	return nil, errors.Wrap(err, "error attaching the BPF probe to the sampling perf event")
}
```


The eBPF program needs to collect an histogram that contains the information about how many samples have been taken for a specific function, and in order to do it it needs to know which function is running when the sample is taken.

As we need ot exchange this information we'll use a `BPF_MAP_TYPE_HASH` hash eBPF map for the histogram:

```c
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__type(key, histogram_key_t);
	__type(value, histogram_value_t);
	__uint(max_entries, K_NUM_MAP_ENTRIES);
} histogram SEC(".maps");
```

of which the key type is declared as a structure that contains:
- PID;
- kernel stack ID;
- user stack ID:
  
```c
typedef struct histogram_key {
	u32 pid;
	u32 kernel_stack_id;
	u32 user_stack_id;
} histogram_key_t;
```

and the value type is declared as a structure that contains:
- sample count;
- the binary pathname (we'll see it later):

```c
typedef struct histogram_value {
	u64 count;
	const char *exe_path; /* We'll see it later on */
} histogram_value_t;
```

The stack traces and their IDs can be retrieved using the `bpf_get_stackid` eBPF helper.

eBPF helpers are functions that, as you might have guessed, simplify work. The [`bpf_get_stackid`](https://elixir.bootlin.com/linux/v6.8.5/source/kernel/bpf/stackmap.c#L283) helper collects user and kernel stack frames by walking the user and kernel stacks and returns the stack ID, for the program that is running at the moment of the eBPF program execution in the process's context.

More precisely from the [eBPF Docs](https://ebpf-docs.dylanreimerink.nl/linux/helper-function/bpf_get_stackid/):

> Walk a user or a kernel stack and return its id. To achieve this, the helper needs ctx, which is a pointer to the context on which the tracing program is executed, and a pointer to a map of type `BPF_MAP_TYPE_STACK_TRACE`.

For example:

```c
struct {
    __uint(type, BPF_MAP_TYPE_STACK_TRACE);
    __uint(key_size, sizeof(u32));
    __uint(value_size, PERF_MAX_STACK_DEPTH * sizeof(u64));
    __uint(max_entries, 10000);
} stack_traces SEC(".maps");

SEC("perf_event")
int sample_stack_trace(struct bpf_perf_event_data* ctx)
{
	histogram_key_t key;

	/* Sample the user and kernel stack traces, and record in the stack_traces structure. */
	key.pid = bpf_get_current_pid_tgid() >> 32;
	key.kernel_stack_id = bpf_get_stackid(ctx, &stack_traces, 0);
	key.user_stack_id = bpf_get_stackid(ctx, &stack_traces, 0 | BPF_F_USER_STACK);
}
```

and for the specific stack (which is, the key in our histogram), we need to update is sample count:

```c
SEC("perf_event")
int sample_stack_trace(struct bpf_perf_event_data* ctx)
{
	histogram_key_t key;
	u64 one = 1;

	/* Sample the user and kernel stack traces, and record in the stack_traces structure. */
	key.pid = bpf_get_current_pid_tgid() >> 32;
	key.kernel_stack_id = bpf_get_stackid(ctx, &stack_traces, 0);
	key.user_stack_id = bpf_get_stackid(ctx, &stack_traces, 0 | BPF_F_USER_STACK);

	/* Upsert stack trace histogram */
	count = (u64*)bpf_map_lookup_elem(&histogram, &key);
	if (count) {
	  (*count)++;
	} else {
	  bpf_map_update_elem(&histogram, &key, &one, BPF_NOEXIST);
	}
}
```

Besides the stack sample count we need to retrieve the stack trace, which is a list of instruction pointers. Thanks to the [`BPF_MAP_TYPE_STACK_TRACE`](https://elixir.bootlin.com/linux/v6.8.5/source/include/uapi/linux/bpf.h#L914) eBPF map: 

```c
struct {
	__uint(type, BPF_MAP_TYPE_STACK_TRACE);
	__uint(key_size, sizeof(u32));
	__uint(value_size, PERF_MAX_STACK_DEPTH * sizeof(u64));
	__uint(max_entries, K_NUM_MAP_ENTRIES);
} stack_traces SEC(".maps");
```

this data can be accessed in user space by stack ID (that we just collected). You can read below an example with [libbpf-go](https://github.com/aquasecurity/libbpfgo):

```go

type HistogramKey struct {
	Pid int32

	// UserStackId, an index into the stack-traces map.
	UserStackId uint32

	// KernelStackId, an index into the stack-traces map.
	KernelStackId uint32
}

// StackTrace is an array of instruction pointers (IP).
// 127 is the size of the profile, as for the default PERF_MAX_STACK_DEPTH.
type StackTrace [127]uint64

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

func (t *Profile) RunProfile(ctx context.Context) error {
	// ...
	for it := histogram.Iterator(); it.Next(); {
		k := it.Key()

		// Get count for the specific sampled stack trace.
		countBinary, err := histogram.GetValue(unsafe.Pointer(&k[0]))
		
		var key HistogramKey
		if err = binary.Read(bytes.NewBuffer(k), binary.LittleEndian, &key); err != nil {
			return nil, errors.Wrap(err, fmt.Sprintf("error reading the stack profile count key %v", k))
		}
		// ...
		var symbols string
		if int32(key.UserStackId) >= 0 {
			trace, err := t.getStackTrace(stackTraces, key.UserStackId)
			// ...
		}
		if int32(key.KernelStackId) >= 0 {
			trace, err := t.getStackTrace(stackTraces, key.KernelStackId)
			// ...
		}
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

To access the ELF binary we need the process's binary pathname. The path can be accessed from the task's user space memory mapping descriptor ([`mm_struct`](https://elixir.bootlin.com/linux/v6.8.5/source/include/linux/mm_types.h#L734)`->exe_file->f_path`) that we can pass it through an eBPF map to the user space program.

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

To retrieve the pathname from the `path` `struct` we need to walk the directory hierarchy until reaching the root directory of the same mount.

## Wrapping up

The program loads the eBPF program, attaches it to the perf event created, and reads stack IDs which accesses stack traces. With traces' instruction pointers it resolves symbols and on exit, it calculates the statistics as explained before. For example:

```
80% main; foo; bar;
20% main; foo; baz;
```

## Thanks
