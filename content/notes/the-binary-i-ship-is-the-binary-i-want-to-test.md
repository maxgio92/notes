---
title: The Binary I Ship Is the Binary I Want to Test
tags: [Tech, eBPF, Testing]
---

This expands my OpenSouthCode 2026 talk:
[The Binary You Ship Is the Binary You Test](https://github.com/maxgio92/xcover/blob/main/docs/talks/opensouthcode-2026/slides.md).

I have a slightly awkward question for anyone who collects code coverage in CI:

**Are you testing the same binary that you ship?**

For many projects, the honest answer is no.

Traditional coverage starts at compile time. Go needs `-cover`, GCC needs `-fprofile-arcs`, and Clang needs its profiling and coverage-mapping flags. The compiler injects counters, the test suite runs against the instrumented result, and a report appears at the end.

That works well until the build output becomes the thing under test.

Linux distributions do not ship source trees. They ship packages containing compiled binaries. If I create a second, instrumented package for coverage, I now have two artefacts:

```text
package-foo-1.0.0.apk
package-foo-1.0.0-instrumented.apk
```

One goes to users. The other produces the reassuring percentage in CI.

Compiler flags, optimisation, link order, and build environments can all change the result. The coverage report may be accurate for the instrumented binary, but that binary is not what I publish.

At the scale of thousands of packages across Go, C, C++, Rust, and native extensions, the operational cost is also rather unpleasant. Every language needs different tooling. Every package needs another build path. CI time, storage, and maintenance all increase.

I wanted another option: collect coverage from the finished binary, without rebuilding it.

## Asking the kernel what ran

Linux already knows how to observe userspace functions through **uprobes**, or user-level dynamic probes.

A uprobe attaches to an offset in an executable. When execution reaches that address, the kernel temporarily replaces the instruction with an `INT3` breakpoint. The CPU traps into the kernel, a handler runs, and execution resumes.

With eBPF, that handler can be a small verifier-checked program. It can identify the function, record that the function ran, and send the result back to userspace through a BPF map or ring buffer.

The useful detail is that a uprobe needs a **binary path and an offset**. It does not require source-level instrumentation.

That led me to build [xcover](https://github.com/maxgio92/xcover), an eBPF uprobe-based profiler for function coverage.

The workflow looks like this:

```sh
xcover run --path ./myapp --detach
xcover wait

./myapp foo
./myapp bar
./myapp baz

xcover stop
```

The application and its tests need no coverage flags, wrapper, or special build. xcover runs beside them, attaches probes to function entry points, and records which functions execute.

The report is deliberately simple:

```json
{
  "exe_path": "./myapp",
  "funcs_traced": [
    "github.com/myorg/myapp/pkg/server.HandleRequest",
    "github.com/myorg/myapp/pkg/server.parseConfig",
    "github.com/myorg/myapp/pkg/db.Connect"
  ],
  "funcs_ack": [
    "github.com/myorg/myapp/pkg/server.HandleRequest",
    "github.com/myorg/myapp/pkg/db.Connect"
  ],
  "cov_by_func": 0.6667
}
```

This is **function coverage**, not line or branch coverage. It answers a narrower question: which functions in this binary did the test workload exercise?

That narrower signal has one valuable property. It comes from the actual executable.

## Keeping the probe set sensible

Attaching probes to every function in a large binary would include the language runtime, standard library, dependencies, and generated code. That produces a lot of noise and makes frequently called runtime functions rather expensive.

xcover can filter function names when it attaches probes:

```sh
xcover run \
  --path ./myapp \
  --include '^github\.com/myorg/myapp' \
  --exclude '\.pb\.go' \
  --detach
```

For Go binaries, project scoping can derive the module from the executable:

```sh
xcover run --path ./myapp --scope project --detach
```

Filtering improves the report and reduces runtime cost. As we shall see, that second benefit matters.

## Then somebody ran `strip`

There is a catch, of course. There is always a catch.

To attach a uprobe, xcover needs the offset of each function. An unstripped ELF binary usually provides those offsets through `.symtab`, along with function names.

Production packages commonly remove that data:

```sh
strip --strip-all ./myapp
```

The executable code remains, but its convenient table of contents disappears. `readelf --symbols` has nothing useful to report, and xcover cannot ask `.symtab` where each function begins.

The first version of the problem looked rather final: uprobes can attach to the code, but only if I already know where the code starts.

The binary still contains clues, though.

## Recovering functions with resurgo

I built [resurgo](https://github.com/maxgio92/resurgo) to recover function boundaries from stripped ELF binaries.

Its strongest source is DWARF Call Frame Information in `.eh_frame`. This data supports stack unwinding and exception handling, so it often survives stripping. It can describe function address ranges even when names and debug sections have gone.

Not every function appears cleanly in that data. Compiler optimisation, inlining, and unusual code generation leave gaps. resurgo therefore combines several signals:

1. **DWARF CFI ranges** provide high-confidence function boundaries.
2. **Architecture-specific prologues** reveal likely entry points.
3. **Direct call targets** identify addresses that must be callable entries.
4. **Alignment boundaries** add weaker supporting evidence.

No single heuristic gets to declare victory. resurgo cross-checks the signals and assigns confidence to each candidate.

xcover uses this automatically. If `.symtab` exists, it reads it. If not, it asks resurgo to recover the offsets.

```text
DBG .symtab not found, falling back to static recovery
DBG resolved 1791 functions via resurgo
```

The command and report format stay the same. Recovery can miss heavily inlined functions or unusual prologues, and stripped functions may lack their original names. This is not a perfect reconstruction of the source program.

It is still coverage from the production binary, which is the point.

## The bill arrives per function call

Kernel uprobes are useful, but they are not free.

Every call to a probed function takes an `INT3` trap into the kernel, runs the BPF program, and returns to userspace. The cost is mostly fixed per call. A tiny function called millions of times hurts far more than a large function called once.

I measured four cases with 100 uprobes on an AMD Ryzen 7 7840U. Each benchmark ran ten times:

| Scenario | Time per call |
|---|---:|
| Baseline, no tracing | 1.17 ns |
| Idle probe | 1.99 ns |
| Probe hit, function already seen | 1,230 ns |
| Probe miss, first observation | 2,911 ns |

The hit case is roughly 1,000 times slower than the tiny baseline call. The first observation, which updates the map and submits an event, is roughly 2,500 times slower.

Those multipliers look dramatic because the baseline function does almost nothing. The absolute cost is about 1.2 microseconds for a repeated hit and 2.9 microseconds for a first observation.

That can be acceptable in CI. It is not acceptable for a latency benchmark, nor would I enable broad tracing on a production hot path and hope for the best.

The memory cost is modest. A map with an eight-byte function identifier and an eight-byte value uses about 16 bytes per traced function:

| Functions | Approximate map storage |
|---:|---:|
| 1,000 | 16 KB |
| 10,000 | 160 KB |
| 50,000 | 800 KB |

The practical rule is simple: filter the probes to the code you care about, and use xcover for test workloads where the measured overhead fits the job.

## Can we remove the kernel trap?

The BPF program itself is small. The expensive part is crossing into the kernel for every hit.

So I tried moving BPF execution into the traced process.

[bpftime](https://github.com/eunomia-bpf/bpftime) is a userspace eBPF runtime. Its syscall server intercepts BPF-related calls, shares the resulting state with an injected agent, and installs in-process trampolines at function entry points. The BPF program runs as JIT-compiled userspace code.

The path changes from this:

```text
function call -> INT3 -> kernel -> BPF -> userspace
```

to this:

```text
function call -> in-process trampoline -> BPF -> continue
```

xcover has an experimental userspace mode built behind the `userspace` build tag:

```sh
make xcover-userspace

xcover-userspace run \
  --path ./myapp \
  --userspace-bpf \
  --detach

LD_PRELOAD=$(xcover-userspace agent extract) ./myapp
```

The coverage logic and report remain the same. The current experiment changes how the probe executes.

The early measurements are encouraging:

| Scenario | Kernel | Userspace | Reduction |
|---|---:|---:|---:|
| Idle probe | 1.99 ns | 1.13 ns | 43% |
| Repeated hit | 1,230 ns | 426 ns | 65% |
| First observation | 2,911 ns | 1,084 ns | 63% |

Removing the kernel transition cuts the repeated-hit cost by almost two thirds. It does not make tracing free, but it confirms where most of the time was going.

## Userspace mode has sharp edges

The userspace path is experimental for good reasons.

It currently uses single, perf-event-based uprobe attachment rather than `uprobe_multi`. The tracee must load the bpftime agent through `LD_PRELOAD`. Dynamically linked child processes inherit that environment across `execve`, but statically linked and musl binaries cannot use this injection method.

The Frida Gum interceptor also has gaps around aggressive compiler optimisation, including some tail-call and link-time optimisation cases.

Kernel mode remains the general path. Userspace mode is a useful experiment with a real performance result, not yet a universal replacement.

## What comes next

There are three parts of the work I want to push further.

First, I want to run xcover across thousands of real packages. Synthetic tests tell me the cost of a probe. A broad package corpus will expose odd toolchains, unusual ELF layouts, and the places where function recovery needs more evidence.

Second, I want to reduce overhead in both modes. Better filtering, fewer events, and cheaper bookkeeping all matter when a hot function sits inside a test workload.

Third, I want to stress the userspace runtime. The current results support the idea, but support for static binaries, aggressive optimisation, and less visible injection will decide how useful it can become.

Some work has already landed upstream. libbpfgo now supports single-uprobe attachment with a per-probe `bpf_cookie` through `AttachUprobeWithOpts`. Further bpftime changes still need upstream work.

## Coverage should describe the artefact

Build-time coverage remains the right tool when I need detailed source-level line or branch data and can test the instrumented build with confidence.

xcover addresses a different problem. It gives me language-neutral function coverage from a compiled ELF binary, including binaries whose symbols have been stripped. It trades per-call runtime cost for a simpler build and a stronger link between the test result and the artefact I publish.

That trade is measurable. It has limits. It is also useful today.

The principle behind it is the bit I care about most:

**The binary I ship should be the binary I test.**
