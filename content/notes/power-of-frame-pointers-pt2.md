---
Title: Unleashing the power of frame pointers for profiling pt.2 - Writing a simple profiler
---

In the previous blog about the program execution environment, we introduced the concept of stack unwinding with frame pointers as one of the techniques leveraged for profiling a program.

In this blog, we'll see practically how we can build a simple sampling-based continuous profiler.

Because we don't require the application to be instrumented, we can use the Linux kernel instrumentation, and thanks to eBPF we're able to dynamically load and attach the profiler program to specific kernel entry points, limiting the overhead by exchanging data with userspace through eBPF maps. 

To summarize the main actors and responsibilities:
- in kernel space: an eBPF sampler program samples periodically stack traces for a specific process;
- in userspace: a program collects the samples, calculates the statistics, and resolves the subroutine's symbols.

## Kernel space

### Sample count

The eBPF program needs to collect a histogram of the number of samples taken for a specific code path. We'll store this data in a `BPF_MAP_TYPE_HASH` eBPF hash map:

```c
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__type(key, histogram_key_t);
	__type(value, histogram_value_t);
	__uint(max_entries, K_NUM_MAP_ENTRIES);
} histogram SEC(".maps");
```

The key of this map which represents the single stack trace it's a structure that contains:
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

The value of this map is a structure that contains mainly the sample count.

```c
typedef struct histogram_value {
	u64 count;
	const char *exe_path;
} histogram_value_t;
```

> It also contains the program executable path. We'll see later on why. 

### Stack trace

To get the information about the running code path and the stack trace we can use the `bpf_get_stackid` eBPF helper.

eBPF helpers are functions that, as you might have guessed, simplify work. The [`bpf_get_stackid`](https://elixir.bootlin.com/linux/v6.8.5/source/kernel/bpf/stackmap.c#L283) helper collects user and kernel stack frames by walking the user and kernel stacks and returns the stack ID.

More precisely from the [eBPF Docs](https://ebpf-docs.dylanreimerink.nl/linux/helper-function/bpf_get_stackid/):

> Walk a user or a kernel stack and return its `id`. To achieve this, the helper needs `ctx`, which is a pointer to the context on which the tracing program is executed, and a pointer to a map of type `BPF_MAP_TYPE_STACK_TRACE`.

So, one of the most complex works of stack unwinding is abstracted away thanks to this helper.

For example, we declare the `stack_traces` map, we prepare the `histogram` key:

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
	// ...

	/* Sample the user and kernel stack traces, and record in the stack_traces structure. */
	key.pid = bpf_get_current_pid_tgid() >> 32;
	key.kernel_stack_id = bpf_get_stackid(ctx, &stack_traces, 0);
	key.user_stack_id = bpf_get_stackid(ctx, &stack_traces, 0 | BPF_F_USER_STACK);
	// ...
}
```

and for the specific stack trace (`key`) we update the sample count in the `histogram`:

```c
SEC("perf_event")
int sample_stack_trace(struct bpf_perf_event_data* ctx)
{
	histogram_key_t key;
	u64 one = 1;
	// ...

	/* Sample the user and kernel stack traces, and record in the stack_traces structure. */
	// ...

	/* Upsert stack trace histogram */
	count = (u64*)bpf_map_lookup_elem(&histogram, &key);
	if (count) {
		(*count)++;
	} else {
		bpf_map_update_elem(&histogram, &key, &one, BPF_NOEXIST);
	}

	return 0;
}
```

Besides the stack trace sample count we need to retrieve the stack trace, which is a list of instruction pointers. Also this information, thanks to the [`BPF_MAP_TYPE_STACK_TRACE`](https://elixir.bootlin.com/linux/v6.8.5/source/include/uapi/linux/bpf.h#L914) map is abstracted away for us and is available directly to userspace.

```c
struct {
	__uint(type, BPF_MAP_TYPE_STACK_TRACE);
	__uint(key_size, sizeof(u32));
	__uint(value_size, PERF_MAX_STACK_DEPTH * sizeof(u64));
	__uint(max_entries, K_NUM_MAP_ENTRIES);
} stack_traces SEC(".maps");
```

This is mostly the needed work in kernel space, which is pretty simplified thanks to the Linux kernel instrumentation.

Let's see how we can use this data in userspace.

## Userspace

Besides loading and attaching the eBPF sampler probe, in userspace we collect the stack traces from `stack_traces` map. This map is accessible by stack IDs, which are available from the `histogram` map.

You can see below an example, using [libbpfgo](https://github.com/aquasecurity/libbpfgo) APIs:

```go
import (
	"encoding/binary"
	"bytes"
	"unsafe"
)

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

func Run(ctx context.Context) error {
	// ...
	for it := histogram.Iterator(); it.Next(); {
		k := it.Key()

		// Get count for the specific sampled stack trace.
		countB, err := histogram.GetValue(unsafe.Pointer(&k[0]))
		// ...
		count := int(binary.LittleEndian.Uint64(countB))
		
		var key HistogramKey
		if err = binary.Read(bytes.NewBuffer(k), binary.LittleEndian, &key); err != nil {
			// ...
		}

		var symbols string
		if int32(key.UserStackId) >= 0 {
			trace, err := getStackTrace(stackTraces, key.UserStackId)
			// ...
		}
		if int32(key.KernelStackId) >= 0 {
			trace, err := getStackTrace(stackTraces, key.KernelStackId)
			// ...
		}
		// ...
}

func getStackTrace(stackTraces *bpf.BPFMap, id uint32) (*StackTrace, error) {
	stackB, err := stackTraces.GetValue(unsafe.Pointer(&id))
	// ...

	var stackTrace StackTrace
	err = binary.Read(bytes.NewBuffer(stackB), binary.LittleEndian, &stackTrace)
	// ...

	return &stackTrace, nil
}
```

Once the sampling is completed, we're able to calculate the program's residency fraction for each subroutine, that is, how much a specific subroutine has been run within a time frame:

```
residencyFraction = nTraceSamples / nTotalSamples * 100.
```

You find below the simple code:

```go
func calculateStats() (map[string]float64, error) {
	// ...

	traceSampleCounts := make(map[string]int, 0)
	totalSampleCount := 0
	
	// Iterate over the stack profile counts histogram map.
	for it := histogram.Iterator(); it.Next(); {
		// ...
	
		// Increment the traceSampleCounts map value for the stack trace symbol string (e.g. "main;subfunc;")
		totalSampleCount += count
		traceSampleCounts[trace] += count
	}
	
	stats := make(map[string]float64, len(traceSampleCounts))
	for trace, count := range traceSampleCounts {
		residencyFraction := float64(count) / float64(totalSampleCount)
		stats[trace] = residencyFraction
	}

	return stats, nil
}
```

Finally, because traces are an array of instruction pointers that are pushed to the stack, we need to translate addresses to symbols.

### Symbolization

There are different ways to resolve symbols based on the binary format and the way the binary has been compiled.

Because this is a demonstration and the profiler is simple we'll consider just ELF binaries that are not stripped.

The ELF structure contains a symbol table in the `.symtab` section that holds information needed to locate and relocate a program's symbolic definitions and references. With that information, we're able to associate instruction addresses with subroutine names.

As the user space program is written in Go, we can leverage the `debug/elf` package from the standard library to access that information.

The correct symbol for the instruction pointer is the one of which the start and end addresses are minor or equal, and major or equal respectively to the instruction pointer address:

```go
func loadSymbols() ([]string, error) {
	file, err := elf.Open(pathname)
	// ...
	
	// Read symbols from the .symtab section.
	syms, err := file.Symbols()
	// ...

	for _, sym := range syms {
		// The symbol is correct if the trace instruction pointer address
		// is within the symbol address range.
		if ip >= sym.Value && ip < (sym.Value+s.Size) {
			sym = s.Name
		}
	}

	return syms, nil
}
```

## Program executable path

To access the ELF binary we need the process's binary pathname. The pathname can be retrieved in kernel space from the `task_struct`'s user space memory mapping descriptor ([`task_struct`](https://elixir.bootlin.com/linux/v6.8.5/source/include/linux/sched.h#L748)->[`mm_struct`](https://elixir.bootlin.com/linux/v6.8.5/source/include/linux/mm_types.h#L734)->[`exe_file`](https://elixir.bootlin.com/linux/v6.8.5/source/include/linux/mm_types.h#L905)->[`f_path`](https://elixir.bootlin.com/linux/v6.8.5/source/include/linux/fs.h#L1016)) that we can pass through an eBPF map to userspace.

The userspace program will then access its `.symtab` ELF section.

```c
SEC("perf_event")
int sample_stack_trace(struct bpf_perf_event_data* ctx)
{
	// ...
	/* Get current task executable pathname */
	task = (struct task_struct *)bpf_get_current_task(); /* Current task struct */
	exe_path = get_task_exe_pathname(task);
	// ...
}

/*
 * get_task_exe_pathname returns the task exe_file pathname.
 * This does not apply to kernel threads as they share the same memory-mapped address space,
 * as opposed to user address space.
 */
static __always_inline void *get_task_exe_pathname(struct task_struct *task)
{
	/*
	 * Get ref file path from the task's user space memory mapping descriptor.
	 * exe_file->f_path could also be accessed from current task's binprm struct 
	 * (ctx->args[2]->file->f_path)
	 */
	struct path path = BPF_CORE_READ(task, mm, exe_file, f_path);

	buffer_t *string_buf = get_buffer(0);
	if (string_buf == NULL) {
		return NULL;
	}
	/* Write path string from path struct to the buffer */
	size_t buf_off = get_pathname_from_path(&path, string_buf);
	return &string_buf->data[buf_off];
}
```

To retrieve the pathname from the [`path`](https://elixir.bootlin.com/linux/v6.8.5/source/include/linux/path.h#L8) struct we need to walk the directory hierarchy until reaching the root directory of the same VFS mount. For the sake of simplicity, we don't go into the details of this part.

## The eBPF program trigger

To run the eBPF program with a fixed frequency the [Perf](https://perf.wiki.kernel.org/index.php/Main_Page) subsystem exposes a kernel software event of type CPU clock ([`PERF_COUNT_SW_CPU_CLOCK`](https://elixir.bootlin.com/linux/v6.8.5/source/include/uapi/linux/perf_event.h#L119)) with user APIs. Luckily, eBPF programs can be attached to those events.

So, after the program is loaded:

```go
import (
	bpf "github.com/aquasecurity/libbpfgo"
	"github.com/pkg/errors"
	"golang.org/x/sys/unix"
)

func loadAndAttach(probe []byte) error {
	bpfModule, err := bpf.NewModuleFromBuffer(probe, "sample_stack_trace")
	// ...
	defer bpfModule.Close()

	if err := bpfModule.BPFLoadObject(); err != nil {
		// ...
	}

	prog, err := bpfModule.GetProgram("sample_stack_trace")
	if err != nil {
		// ...
	}

	// ...
}
```

this Perf event can be leveraged to be able to trigger the sampler every x nanoseconds. Because Perf exposes user APIs, the userspace program can prepare the clock software events for all the CPUs and attach the eBPF program to them:

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
			Sample: 10 * 1000 * 1000,
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
		// ...
		
		// Attach the BPF program to the sampling perf event.
		if _, err = prog.AttachPerfEvent(evt); err != nil {
			return errors.Wrap(err, "error attaching the BPF probe to the sampling perf event")
		}
	}

	return nil
}
```

> In this example we're using the [libbpfgo](https://github.com/aquasecurity/libbpfgo) library.

## Wrapping up

The user program loads the eBPF program, attaches it to the Perf event in order to be triggered periodically, and samples stack traces. Trace instruction pointers are resolved into symbols and before returning, the statistics about residency fraction are calculated with data stored in the histogram.

The statistics are finally printed out like below:

```
80% main()->foo()->bar()
20% main()->foo()->baz()
```

You can see a full working example at [github.com/maxgio92/yap](https://github.com/maxgio92/yap). YAP is a sampling-based, low overhead kernel-assisted profiler.

## Next

I personally would like to use statistics to build graph structures, like [flamegraphs](https://github.com/brendangregg/FlameGraph).

Also, I'd like to investigate other ways to extend symbolization support for stripped binaries and collect traces when binaries are built without frame pointers.

## Thanks

Thanks for your time, I hope you enjoyed this blog.

Any form of feedback is more than welcome. Hear from you soon!
