---
Title: Unleashing the power of frame pointers for profiling pt.2 - Writing a simple profiler
---

In the previous blog about the program execution environment, we introduced the concept of stack unwinding with frame pointers as one of the techniques leveraged for profiling a program.

In this blog, we'll see practically how we can build a simple profiler that samples stack traces to calculate statistics of the program's subroutines.

In order to limit the overhead, we can use the Linux kernel instrumentation, and thanks to eBPF we're able to dynamically load and attach the profiler program to specific kernel entry points, without the need to build and load modules. Moreover, we don't need traced programs to be instrumented.

To summarize the main actors and responsibilities:
- in kernel space: an eBPF sampler program samples with fixed frequency stack traces for a specific process;
- in userspace: a program collects the samples, calculates the statistics, and resolves the subroutine's symbols.

## eBPF Program loading and attaching

To run the eBPF program with a fixed frequency the perf subsystem exposes a kernel software event of type CPU clock ([`PERF_COUNT_SW_CPU_CLOCK`](https://elixir.bootlin.com/linux/v6.8.5/source/include/uapi/linux/perf_event.h#L119) with user APIs. Luckily, eBPF programs can be attached to those events.

So, after the program is loaded:

```go
import (
	bpf "github.com/aquasecurity/libbpfgo"
	"github.com/pkg/errors"
	"golang.org/x/sys/unix"
)

func loadAndAttach(probe []byte) error {
	bpfModule, err := bpf.NewModuleFromBuffer(probe, "sample_stack_trace")
	if err != nil {
		return errors.Wrap(err, "error creating the BPF module object")
	}
	defer bpfModule.Close()

	if err := bpfModule.BPFLoadObject(); err != nil {
		return errors.Wrap(err, "error loading the BPF program")
	}

	prog, err := bpfModule.GetProgram("sample_stack_trace")
	if err != nil {
		return errors.Wrap(err, "error getting the BPF program object")
	}

	// ...
}
```

we'll leverage a software CPU clock Perf event just to be able to probe every x nanoseconds. As perf exposes user APIs, the userspace program can prepare the clock software events for all the CPUs and attach the eBPF program to them:

```go
import (
	bpf "github.com/aquasecurity/libbpfgo"
	"github.com/pkg/errors"
	"golang.org/x/sys/unix"
)

func loadAndAttach(probe []byte) error {
	// Load the program...

	cpus := runtime.NumCPU()

	for i := 0; i < cpus; i++ {
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
			-1,	// All the tasks.
			i,	// on the Nth CPU.
			-1,	// The group_fd argument allows event groups to be created.
			0,	// The flags.
		)
		if err != nil {
			return errors.Wrap(err, "error creating the perf event")
		}
		defer func() {
			if err := unix.Close(evt); err != nil {
				return errors.Wrap(err, "failed to close perf event")
			}
		}()
		
		// Attach the BPF program to the sampling perf event.
		if _, err = prog.AttachPerfEvent(evt); err != nil {
			return errors.Wrap(err, "error attaching the BPF probe to the sampling perf event")
		}
	}

	return nil
}
```

> In this example we're using the [libbpfgo](https://github.com/aquasecurity/libbpfgo) library.

## Kernel space

The eBPF program needs to collect an histogram that contains the information about how many samples have been taken for a specific function.

Because this information must be shared with userspace, which is where calculations are done, to store the histogram we'll use a `BPF_MAP_TYPE_HASH` eBPF hash map:

```c
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__type(key, histogram_key_t);
	__type(value, histogram_value_t);
	__uint(max_entries, K_NUM_MAP_ENTRIES);
} histogram SEC(".maps");
```

The key type of this map represents the single stack trace and it's a structure that contains:
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

The value type of this map is a structure that contains mainly the sample count.

```c
typedef struct histogram_value {
	u64 count;
	const char *exe_path;
} histogram_value_t;
```

> It also contains the program executable path. We'll see later on why. 

To get the information about the running function and the stack trace we can use the `bpf_get_stackid` eBPF helper.

eBPF helpers are functions that, as you might have guessed, simplify work. The [`bpf_get_stackid`](https://elixir.bootlin.com/linux/v6.8.5/source/kernel/bpf/stackmap.c#L283) helper collects user and kernel stack frames by walking the user and kernel stacks and returns the stack ID, for the program that is running at the moment of the eBPF program execution in the process's context.

So, one of the most complex works of stack unwinding is abstracted away thanks to this helper.

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

and for the specific stack trace (which is, the key in our histogram), we need to update the sample count:

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

Besides the stack trace sample count we need to retrieve the stack trace, which is a list of instruction pointers. Also this information, thanks to the [`BPF_MAP_TYPE_STACK_TRACE`](https://elixir.bootlin.com/linux/v6.8.5/source/include/uapi/linux/bpf.h#L914) eBPF map: 

```c
struct {
	__uint(type, BPF_MAP_TYPE_STACK_TRACE);
	__uint(key_size, sizeof(u32));
	__uint(value_size, PERF_MAX_STACK_DEPTH * sizeof(u64));
	__uint(max_entries, K_NUM_MAP_ENTRIES);
} stack_traces SEC(".maps");
```

is abstracted away for us and is available directly to userspace.

This is mostly the needed work in kernel space, which is pretty simplified thanks to the Linux kernel instrumentation.

Let's see how we can use this data from userspace.

## Userspace

The sample stack traces can be collected in userspace from the stack traces `BPF_MAP_TYPE_STACK_TRACE` map by stack ID, which is available from the histogram `BPF_MAP_TYPE_HASH` map.

You can see below an example with [libbpfgo](https://github.com/aquasecurity/libbpfgo):

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

func (t *Profile) RunProfile(ctx context.Context) error {
	// ...
	for it := histogram.Iterator(); it.Next(); {
		k := it.Key()

		// Get count for the specific sampled stack trace.
		count, err := histogram.GetValue(unsafe.Pointer(&k[0]))
		
		var key HistogramKey
		if err = binary.Read(bytes.NewBuffer(k), binary.LittleEndian, &key); err != nil {
			return errors.Wrap(err, "error reading the stack profile count")
		}

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

Finally, because traces are an array of instruction pointers that are pushed to the stack, we need to translate addresses to symbols.

### Symbolization

There are different ways to resolve symbols based on the binary format and the way the binary has been compiled.

Because this is a demonstration and the profiler is simple we'll consider just ELF binaries that are not stripped.

The ELF structure contains a symbol table (`symtab` section) that holds information needed to locate and relocate a program's symbolic definitions and references. With that information, we're able to, if this table is not missing as in the case of stripped binaries, associate instruction addresses with subroutine names.

As the user space program is written in Go, we can leverage the `debug/elf` from the standard library to access that information with:

```go
// Open the file.
file, err := elf.Open(pathname)
if err != nil {
	return err
}

// Read symbols from the .symtab section.
syms, err := file.Symbols()
if err != nil {
	return err
}
for _, s := range syms {
	// The symbol is correct if the trace instruction pointer address
	// is within the symbol address range.
	if ip >= s.Value && ip < (s.Value+s.Size) {
		sym = s.Name
	}
}
```

## Binary path

To access the ELF binary we need the process's binary pathname. The path can be accessed from the task's user space memory mapping descriptor ([`mm_struct`](https://elixir.bootlin.com/linux/v6.8.5/source/include/linux/mm_types.h#L734)`->exe_file->f_path`) that we can pass it through an eBPF map to userspace to read from the ELF `.symtab` section.

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

To retrieve the pathname from the `path` `struct` we need to walk the directory hierarchy until reaching the root directory of the same mount. For sake of simplicity we skip this part.

## Wrapping up

The program loads the eBPF program, attaches it to the perf event created, and reads stack IDs which accesses stack traces. With traces' instruction pointers it resolves symbols and on exit, it calculates the statistics as explained before. For example:

```
80% main; foo; bar;
20% main; foo; baz;
```

You can see a full working example at https://github.com/maxgio92/yap, which stands for Yet Another Profiler. YAP is a sampling-based, low overhead kernel-assisted profiler.

## Thanks
