---
title: From a frozen Telegram photo to the Linux graphics stack
date: 2026-08-30T13:14:00+02:00
tags: [Linux, Graphics, AMD, Wayland]
---

> I am new to Linux graphics. This post records what I learned.
> Our AI cats helped with the research and post structure.
> Any mistakes are mine. Corrections are welcome.

## 1. The fullscreen photo that froze a laptop

I opened a photo from Telegram Web in Brave and switched it to fullscreen. The display froze. After failed attempts to recover it, I forced the Framework Laptop to restart.

A TTY later showed Vulkan selecting my AMD Radeon 780M. Two warnings about dynamic-buffer limits followed. Their timestamp placed them more than ten minutes after the next boot began.

The previous boot told a different story. Its kernel journal pointed at AMD's display path: Display Core, DCN, DMCUB, MPCC and CRTC operations. That fault led to a larger question: how does a Linux application get pixels from its own code to a laptop panel?

{{< mermaid >}}
flowchart TB
    T1["11:15:04<br/>Mutter stacking assertion"] --> T2["11:15:08<br/>first DMCUB errors"]
    T2 --> T3["11:15:09<br/>MPCC idle timeout"]
    T3 --> T4["11:15:33<br/>DMUB command queue errors"]
{{< /mermaid >}}

The short answer is that the display stack wedged in AMD's DCN and DMCUB path. The logs leave the first bad state unattributed among kernel code, firmware, hardware and their interaction. To understand why that wording matters, we need to build the stack one layer at a time.

New to Linux graphics? Read [sections 3 to 13](#3-the-smallest-useful-model) in order. Investigating a freeze? Start at [sections 14 to 16](#14-reconstructing-the-failure). Returning for a definition? Jump to the [type glossary](#type-glossary).

## 2. Read logs as a timeline

I reduced the failed boot to five decisive lines:

```text
11:15:04  Mutter stacking assertion
11:15:08  DMCUB error
11:15:09  MPCC idle timeout
11:15:33  Error queueing DMUB command
11:18:17  failed to blank CRTC
```

The first line came from Mutter, GNOME's compositor and window manager. The remaining lines came from `amdgpu`, the Linux kernel driver for the Radeon GPU. `DMCUB`, `MPCC` and `CRTC` all belong to the display side of the stack.

Order matters. The first DMCUB failure appeared at 11:15:08. I pressed the power key at 11:16:18. A kernel trace in the PSR path appeared four seconds after that press, during the suspend attempt. The suspend and PSR failures show that an already broken display path failed again during suspend. The initial cause remains earlier and unknown.

I use three labels for the rest of this account:

- **Fact**: a log or primary source states it directly.
- **Inference**: an explanation connects several facts beyond the explicit log statements.
- **Unknown**: a reproduction, trace, code fix or bisect is still needed.

## 3. The smallest useful model

Before naming every library and kernel object, I use three stages:

{{< mermaid >}}
flowchart TB
    A["1. App rendering<br/>(stage)"] --> W["Wayland commit<br/>(protocol exchange)"]
    W --> C["2. Compositor composition<br/>(optional stage)"]
    C --> P["3. KMS presentation<br/>(stage through kernel ABI)"]
    P --> D["Display scanout<br/>(hardware operation)"]
{{< /mermaid >}}

These boxes describe dependencies for one buffer. Each box names a stage, while the machine pipelines work across them.

1. **Rendering** produces pixels for an application's window.
2. **Composition** may combine several application buffers into an output buffer.
3. **Presentation** selects a buffer for display through Kernel Mode Setting (KMS).

The display engine then **scans out** that buffer. It reads pixels at the required pace and sends them towards the panel.

Composition is optional. A suitable fullscreen buffer may go to presentation through direct scanout. Across several frames, the stages overlap: an application can render a new buffer while the panel displays an older one.

## 4. A legend for components and rules

Linux graphics discussions often put a process, a protocol and a kernel interface in the same sentence. I clarify the stack by labelling what each name *is*.

| Group | Examples | What the group means |
|---|---|---|
| User-space components | Brave, Mutter, Sway | Running programs that perform work. |
| Frameworks and libraries | GTK, Qt, SDL, GLFW, wlroots, `libwayland-client`, `libdrm` | Reusable code used by applications or compositors. |
| Graphics project and runtime components | Mesa, RadeonSI, RADV | User-space implementations of graphics APIs and GPU support. |
| API specifications | OpenGL, Vulkan | Rules for asking a graphics implementation to render or compute. |
| Protocol specifications | Wayland, eDP | Rules for communication between parties. |
| Kernel framework and ABI | DRM and KMS | Kernel interfaces for graphics devices, buffers, command submission and displays. |
| Kernel driver component | `amdgpu` | AMD-specific code that implements DRM and KMS operations. |
| Protocol objects | `wl_surface`, `wl_buffer` | Named objects defined by Wayland. |
| Kernel objects | DRM framebuffer, plane, CRTC | Named objects exposed through the DRM/KMS ABI. |
| Sharing and sync objects | DMA-BUF file descriptor, fence, sync object | Handles to storage or completion state. |
| Firmware | DMCUB or DMUB firmware | Code run by AMD's display microcontroller. |
| Hardware | DMCUB microcontroller, graphics engine, display engine, eDP link, panel | Physical blocks that control, render, fetch and show pixels. |

Some names cover both a project and several runtime libraries. Mesa is one example. Ask both, "Where is Mesa?" and, "Which Mesa component and interface are we discussing?"

## 5. Stage one: how an application produces pixels

An application has several valid routes to a window buffer:

- GTK or Qt can choose a GPU or CPU renderer for it.
- A game can use SDL or GLFW, then call OpenGL or Vulkan.
- An application can use Wayland and a graphics API directly.
- A CPU renderer can write a shared-memory buffer and skip Mesa and the GPU graphics engine.

For GPU rendering on this Radeon 780M, the main route looks like this:

{{< mermaid >}}
flowchart TB
    APP["Application<br/>(component)"] --> TK["GTK, Qt, SDL or app renderer<br/>(optional library, framework or code)"]
    TK --> API["OpenGL or Vulkan<br/>(API)"]
    API --> MESA["Mesa plus RadeonSI or RADV<br/>(components)"]
    MESA --> DRM["DRM render node<br/>(device-file interface to kernel ABI)"]
    DRM --> AMD["amdgpu<br/>(kernel component)"]
    AMD --> GPU["Graphics engine<br/>(hardware)"]
    GPU --> BUF["Rendered pixel buffer<br/>(data object)"]
{{< /mermaid >}}

OpenGL and Vulkan are API specifications. Mesa supplies their running Linux implementations. For supported AMD GPUs, Mesa's **RadeonSI** driver implements OpenGL through its Gallium stack, while **RADV** implements Vulkan.

Mesa translates API work into AMD-specific commands. It submits those commands through DRM's rendering interface to `amdgpu`. The graphics engine executes them and writes pixels into buffer storage. A fence can signal when the asynchronous work has finished.

GTK and Qt are optional. They provide widgets, layout, input and window-system support, and they often choose a renderer. Applications can also reach Mesa directly.

## 6. The Wayland conversation

Rendering and Wayland communication are two parallel interfaces used by the same application. Rendering makes pixels. Wayland lets the client describe a surface and offer its content to the compositor.

{{< mermaid >}}
flowchart TB
    APP["Application<br/>(component)"]
    TK["GTK, Qt, SDL or direct code<br/>(optional library, framework or app code)"]
    API["OpenGL or Vulkan<br/>(API)"]
    MESA["Mesa plus RadeonSI or RADV<br/>(components)"]
    DRM["DRM render node<br/>(device-file interface to kernel ABI)"]
    GPU["Graphics engine<br/>(hardware)"]
    BUF["Rendered buffer<br/>(data object)"]
    WLC["libwayland-client or generated bindings<br/>(library or client code)"]
    WAY["Wayland<br/>(protocol)"]
    COMP["Mutter or Sway<br/>(component, Wayland server)"]

    APP --> TK
    TK --> API --> MESA --> DRM --> GPU --> BUF
    TK --> WLC --> WAY --> COMP
    BUF -->|"attached as wl_buffer"| WAY
{{< /mermaid >}}

The two routes meet at a reference to the buffer and its synchronisation state. OpenGL and Vulkan draw calls follow a separate route from Wayland messages.

A typical client update works like this:

1. The client creates a `wl_surface` and gives it a desktop role through an extension such as xdg-shell.
2. It renders content into storage.
3. It exposes that content as a `wl_buffer`, often through Linux DMA-BUF or `wl_shm`.
4. It attaches the buffer, marks damaged regions and commits the surface state.
5. The compositor releases the buffer when it no longer needs it.

Wayland defines asynchronous requests from client to server and events in the other direction. Input, window configuration, frame callbacks and buffer release return as events. GTK and Qt normally hide much of this through their Wayland backends and `libwayland-client`.

Mutter, KWin and Sway implement the server side in the compositor. The compositor acts as the Wayland display server.

## 7. Buffers are where the paths meet

We often say "the buffer" as if one object travels unchanged through every layer. In practice, each interface has its own view.

| View | Type | Meaning |
|---|---|---|
| Pixel storage | Data storage | Memory that contains image data. |
| DMA-BUF FD | Kernel sharing handle | A file descriptor that can export shared storage between processes or devices. |
| `wl_buffer` | Wayland protocol object | Content that a client can attach to a `wl_surface`. |
| DRM framebuffer | KMS kernel object | A description of storage that KMS can use for scanout. |
| Fence or sync object | Synchronisation object | Completion or ownership state. It contains no pixels. |

These views can refer to the same underlying storage while remaining distinct objects. A `wl_buffer` can come from DMA-BUF storage or `wl_shm`. A compositor can import storage and create a DRM framebuffer view suitable for KMS.

Zero-copy depends on compatible pixel formats, layout modifiers, colour handling, scaling and device limits. A mismatch may force the compositor to copy or redraw content.

This distinction also matters for the integrated Radeon 780M. It shares system memory with the CPU and has no separate dedicated VRAM. The stack still manages buffer objects and GPU addresses.

## 8. Stage two: how the compositor builds an output frame

Mutter acts as GNOME's Wayland server, compositor and window manager. Sway performs the same broad roles and uses **wlroots**, a library that supplies reusable backends, renderers, allocators, buffer support and Wayland protocol support.

The compositor receives committed surfaces. Its scene graph tracks their position, visibility, clipping, scale, effects and stacking. For normal composition, its renderer samples the current usable client buffers and draws an output buffer.

```text
committed client buffers
  -> compositor component and scene graph
  -> compositor renderer component
  -> OpenGL, OpenGL ES or Vulkan API plus Mesa, or a CPU renderer
  -> output buffer plus synchronisation state
```

A GPU compositor submits another render job. A software compositor can combine pixels on the CPU. Either route produces a buffer that can later be presented.

Wayland leaves the renderer and display backend to the compositor. A compositor can run nested inside another Wayland compositor, inside an X11 window, headlessly, or with software rendering. The compositor chooses how to render and where to present, which makes "Wayland renders the desktop" an inaccurate model.

## 9. The shortcut: direct scanout and hardware planes

Composition may be unnecessary. The compositor can sometimes assign a client's buffer directly to a display plane.

{{< mermaid >}}
flowchart TB
    B["Client wl_buffer<br/>(protocol object backed by storage)"] --> Q{"Compositor policy and<br/>hardware checks"}
    Q -->|"compose"| R["Compositor renderer<br/>(component)"]
    R --> O["Output buffer<br/>(data object)"]
    O --> K["DRM/KMS atomic commit<br/>(kernel ABI)"]
    Q -->|"direct scanout"| K
    K --> E["Display engine<br/>(hardware)"]
{{< /mermaid >}}

Direct scanout skips the compositor's render job while preserving compositor control and KMS presentation. The compositor receives the surface, checks policy and hardware limits, and chooses the KMS state.

A fullscreen window enables direct scanout when effects, scaling, cursor state, colour transforms, pixel formats and layout modifiers satisfy the requirements. Hardware overlay planes offer another option: KMS can place several buffers into the display pipeline as separate planes.

**Inference:** the Telegram fullscreen action may have changed composition or plane state.

**Unknown:** the journal leaves Mutter's route for that photo unclear: direct scanout, an overlay plane or normal composition.

## 10. Stage three: KMS presents

DRM has two related branches. Mesa uses its rendering ABI. The compositor separately uses its KMS display ABI.

{{< mermaid >}}
flowchart TB
    APP["Application or compositor renderer<br/>(user-space component)"]
    M["Mesa AMD driver<br/>(component)"]
    RN["renderD*<br/>(device-file interface to DRM ABI)"]
    CO["Compositor<br/>(component)"]
    LD["libdrm<br/>(library)"]
    PN["cardN<br/>(device-file interface to DRM/KMS ABI)"]
    AMD["amdgpu<br/>(kernel driver)"]
    GFX["Graphics engine<br/>(hardware)"]
    DISP["Display engine<br/>(hardware)"]

    APP --> M --> RN --> AMD --> GFX
    CO --> LD --> PN --> AMD --> DISP
{{< /mermaid >}}

Mesa rendering calls use rendering ioctls. KMS presentation uses display ioctls. Both often serve the same visible frame at different stages.

A DRM render node such as `/dev/dri/renderD128` allows unprivileged rendering while modesetting and DRM master remain unavailable. A primary node such as `/dev/dri/card1` exposes KMS and other controlled operations.

The compositor commonly uses **libdrm** to prepare DRM calls. `libdrm` is a user-space wrapper and helper library. The kernel driver remains `amdgpu`. Direct ioctls are possible.

KMS expresses display state with a stable object model:

{{< mermaid >}}
flowchart TB
    FB["DRM framebuffer<br/>(kernel object)"] --> PL["Plane<br/>(KMS object)"]
    PL --> CR["CRTC<br/>(KMS object)"]
    CR --> EN["Encoder<br/>(KMS object)"]
    EN --> CN["Connector eDP-1<br/>(KMS object)"]
    CN --> PA["Laptop panel<br/>(hardware)"]
{{< /mermaid >}}

- A **framebuffer** describes pixel storage for scanout.
- A **plane** supplies a pixel source, position and size.
- A **CRTC** controls an output scan sequence and mode. Its name is historical; it is still the stable KMS abstraction.
- An **encoder** represents conversion towards an output link.
- A **connector** represents the display sink, such as `eDP-1`.
- A **mode** defines dimensions, refresh and timings.

An atomic commit proposes the complete next state. The kernel validates it, then applies all accepted properties as one coherent update. OpenGL or Vulkan render the application's pixels. KMS controls their presentation. Display planes may also scale or blend during scanout.

## 11. Why the stages overlap

For one specific buffer, the dependencies remain ordered:

```text
render -> make available -> compose or select -> present -> scan out
```

The machine pipelines this chain with work on other frames.

{{< mermaid >}}
sequenceDiagram
    participant A as App component
    participant G as GPU graphics hardware
    participant C as Compositor component
    participant K as DRM/KMS plus amdgpu
    participant D as Display engine
    A->>G: Submit app render for buffer A2
    G-->>A: Signal render fence
    A->>C: Commit wl_surface with wl_buffer A2
    C->>G: Submit output composition O2
    G-->>C: Signal composition fence
    C->>K: Atomic commit O2 with fence state
    K->>D: Program next scanout state
    D-->>C: Page-flip or completion event
    Note over A,D: While O2 progresses, the display may scan O1 and the app may prepare A3.
{{< /mermaid >}}

Double or triple buffering lets an application prepare a new image while another buffer remains in use. Fences tell consumers when asynchronous work has completed. Wayland frame callbacks help clients pace new work. Buffer-release events tell a client when the compositor has finished with a buffer. KMS can accept fence state and later send a page-flip or completion event.

Output presentation follows its own cadence. The compositor builds monitor output frames from the latest usable buffers. It can reuse one client's buffer across many output frames, skip an intermediate client buffer, or present while another client remains unchanged.

## 12. Zooming into the Radeon 780M display side

The word "GPU" hides separate hardware blocks. A display engine can wedge while the graphics ring keeps running.

{{< mermaid >}}
flowchart TB
    K["DRM/KMS atomic state<br/>(kernel ABI)"] --> AD["amdgpu Display Manager and DC<br/>(kernel components)"]
    FW["DMCUB or DMUB firmware<br/>(firmware component)"] -. "runs on" .-> MCU["DMCUB or DMUB microcontroller<br/>(hardware)"]
    AD --> MCU
    AD --> DCN["DCN display engine<br/>(hardware)"]
    MCU <--> DCN
    DCN --> STREAM["eDP data stream<br/>(link protocol)"]
    STREAM --> EDP["eDP electrical link<br/>(hardware)"]
    EDP --> TCON["Panel timing controller<br/>(hardware)"]
    TCON -. "retains image during PSR" .-> RET["Retained frame<br/>(data and state)"]
    TCON --> LCD["LCD panel<br/>(hardware)"]
{{< /mermaid >}}

The Radeon 780M's **graphics engine** executes application and compositor render work. Its **DCN display engine** handles planes, composition blocks, output timing and links. This machine reported DCN 3.1.4.

**AMD Display Core**, often shortened to DC, is kernel display code inside `amdgpu`. It translates DRM display state into AMD display-pipeline state.

**DMCUB**, also called **DMUB** in interfaces and logs, is a display microcontroller block. It runs AMD display firmware. The driver uses this path for some display features and commands. Rendered pixels travel through buffer storage and display scanout, while DMCUB handles selected display work.

The final path uses the **eDP** protocol. The physical eDP link carries its data stream to the laptop panel.

This map gives the log terms a home. `MPCC` names a display composition block. CRTC messages concern the KMS output pipeline. DMCUB command failures concern the firmware-controlled display path. None of those names, by itself, reports a Vulkan render failure.

## 13. PSR: when the panel keeps the picture

Panel Self Refresh (PSR) lets a capable eDP panel retain an unchanged frame. The source can then reduce repeated memory fetches and link activity until content changes.

```text
unchanged output
  -> panel retains frame
  -> source reduces link and memory work

content changes
  -> source exits PSR
  -> new frame reaches panel
```

PSR has entry, steady and exit states. Entry and exit add transitions to the display path. On this Framework Laptop, the internal `eDP-1` panel advertised PSR support and PSR was enabled.

The incident needs careful wording:

- **Fact:** a warning at 11:16:22 ran through `dmub_psr_get_state`, `dmub_psr_enable`, `edp_set_psr_allow_active`, `amdgpu_dm_atomic_commit_tail` and `drm_mode_atomic_ioctl`.
- **Fact:** this trace appeared after the first DMCUB errors and after my power-key suspend request.
- **Inference:** the existing display fault affected later PSR control during an atomic KMS commit.
- **Unknown:** whether PSR was active at 11:15:08, whether fullscreen caused a PSR exit, and whether PSR initiated the fault.

Disabling PSR can provide useful evidence in a controlled test. Current mainline AMD source defines `0x10` in `amdgpu.dcdebugmask` as disabling PSR and PSR selective update. Anyone testing it should confirm the flag against the source for the kernel in use, test one change at a time and expect possible extra power use.

## 14. Reconstructing the failure

Here is the full sequence. The first display fault comes before the suspend attempt and the PSR trace.

{{< mermaid >}}
flowchart TB
    T1["11:15:04<br/>Mutter stacking assertion"] --> T2["11:15:08<br/>first DMCUB errors"]
    T2 --> T3["11:15:09<br/>MPCC idle timeout"]
    T3 --> T4["11:15:33<br/>DMUB command queue errors"]
    T4 --> T5["11:16:18<br/>power key starts suspend"]
    T5 --> T6["11:16:22<br/>PSR call trace in KMS thread"]
    T6 --> T7["11:17:24<br/>user.slice freeze times out"]
    T7 --> T8["11:18:17<br/>CRTC blank fails"]
    T8 --> T9["11:25 to 11:28<br/>more suspend and display failures"]
    T9 --> T10["11:29:35<br/>new boot"]
{{< /mermaid >}}

| Time on 2026-08-30 | Evidence | What I can conclude |
|---|---|---|
| 11:15:04.809 | Mutter logged `meta_window_set_stack_position_no_sync`. | **Fact:** a compositor warning occurred near the fullscreen action. **Unknown:** whether it caused, reflected or merely accompanied the fault. The same assertion later appeared during a healthy display session. |
| 11:15:08.363 | First `DMCUB error - collecting diagnostic data`. | **Fact:** AMD's display path detected a DMCUB failure about 3.6 seconds later. |
| 11:15:09.126 | `mpc2_assert_idle_mpcc` timed out. | **Fact:** a DCN display composition block failed to reach its expected idle state. |
| 11:15:33.059 | `Error queueing DMUB command: status=2` began repeating. | **Fact:** the driver could no longer queue display firmware commands normally. **Unknown:** the exact meaning of status 2 in this Fedora kernel build. |
| 11:16:18.661 | `Power key pressed short`; logind began suspend. | **Fact:** my first recovery attempt came after the display fault. |
| 11:16:22.251 | The KMS thread warned in `dmub_psr_get_state` during an atomic commit. | **Fact:** later suspend work reached the broken DMCUB PSR path. This PSR trace followed the first failure. |
| 11:17:24.003 | Freezing `user.slice` timed out. | **Fact:** suspend preparation stalled while the session was unhealthy. |
| 11:18:12.849 | More MPCC and CRTC-disable timeouts appeared around resume. | **Fact:** the display state remained broken across the suspend attempt. |
| 11:18:17.955 | `DC: failed to blank crtc!` | **Fact:** the driver failed a basic display disable operation. |
| 11:25:27 onwards | More power-key presses, suspend failures and display errors. | **Fact:** later recovery attempts added fallout after the original fault. |
| 11:28:47.874 | The old journal ended with repeated DMCUB errors. | **Fact:** no recovery appears in the stored log. |
| 11:29:35 | The next boot began. | **Fact:** the prior session ended. **Inference:** a forced restart or power cycle ended it. The journal ends before the physical act. |
| 11:39:37 | The supplied Dawn and Vulkan warnings appeared. | **Fact:** they occurred after the failed boot had ended. They cannot explain the earlier hang. |

The system was a Framework Laptop 13 with an AMD Ryzen 7040 Series processor and Radeon 780M integrated GPU. The failed boot ran Fedora kernel `7.1.6-201.fc44.x86_64`, Mesa 26.1.5, Mutter 50.3 and BIOS 03.18. `amdgpu` reported Display Core 3.2.378, DCN 3.1.4 and DMCUB firmware `0x08005D00`.

**Fact:** the journal held no graphics-ring timeout, GPU page fault, Vulkan device loss or successful GPU reset for this event. **Inference:** that absence weighs against those specific failure signatures. Rendering could still have triggered a display transition.

**Fact:** earlier dates also showed `dcn31_program_compbuf_size` timeouts, including a GNOME Shell KMS thread inside an AMD atomic display commit. **Inference:** those signs make the display fault less isolated. **Unknown:** whether they share a root cause with this incident.

The evidence supports this conclusion:

> The display stack wedged in the AMD DCN and DMCUB path after the fullscreen action. Later atomic commits, PSR queries, blanking and suspend all failed to recover it. The first bad command and its owner remain unknown across kernel code, firmware, hardware and their interaction.

The evidence establishes a failure in the DMCUB path and leaves firmware ownership unproven. The same symptoms could arise if kernel code sent a bad sequence, firmware stopped servicing valid commands, hardware wedged, or two parts violated a timing assumption.

## 15. Why the Vulkan warning was a red herring

The TTY showed these lines after reboot:

```text
Warning: maxDynamicUniformBuffersPerPipelineLayout artificially reduced from 500000 to 16 to fit dynamic offset allocation limit.
Warning: maxDynamicStorageBuffersPerPipelineLayout artificially reduced from 500000 to 16 to fit dynamic offset allocation limit.
```

It also said that a Dawn-based process had selected `AMD Radeon 780M Graphics (RADV PHOENIX)` with the Vulkan backend. [Dawn's current `Limits.cpp`](https://dawn.googlesource.com/dawn/+/refs/heads/main/src/dawn/native/Limits.cpp) emits those exact warnings when it clamps adapter limits to its internal maximum. The lines report an exposed WebGPU capability adjustment. Vulkan device loss produces different messages.

The timing settles this incident. They appeared at 11:39:37, about ten minutes after the new boot began. The failed boot ended before 11:29:35.

| Dawn capability warning | Kernel display failure |
|---|---|
| User-space warning | Kernel `[drm]` errors |
| Adjusts an exposed WebGPU limit | DMCUB command and DCN register timeouts |
| Reports a capability clamp | Blocks CRTC blanking and later atomic display work |
| Seen after reboot | Persisted during the failed prior boot |

I would assess each warning in context. This message reports a capability clamp, and its timestamp falls outside the crash sequence.

## 16. A repeatable way to investigate a graphics freeze

I start narrow and widen only when the evidence asks for it. First I identify the failed boot and read its kernel log:

```bash
journalctl --list-boots --no-pager
journalctl -k -b -1 --no-pager
```

Then I restrict the journal to the event window. Exact times keep later recovery attempts from masquerading as the cause:

```bash
journalctl -b -1 \
  --since '2026-08-30 11:14:00' \
  --until '2026-08-30 11:20:00' \
  --no-pager
```

Next I filter by the graphics and display terms already present in the log:

```bash
journalctl -k -b -1 --no-pager \
  | rg -i 'amdgpu|drm|dmub|dmcub|psr|crtc|mpcc|timeout|reset|fault'

journalctl -b -1 --no-pager \
  | rg -i 'mutter|gnome-shell|brave|suspend|power key'
```

I record the hardware and user-space renderer separately:

```bash
lspci -nnk | rg -A3 'VGA|Display|3D'
vulkaninfo --summary
```

`vulkaninfo` comes from Fedora's optional [`vulkan-tools`](https://packages.fedoraproject.org/pkgs/vulkan-tools/vulkan-tools/) package. It shows the current Vulkan state. The previous boot must come from its logs. Package, firmware and BIOS versions belong beside the incident log because an update can change any part of the path.

A stronger follow-up would collect:

- The exact wall-clock time and action for a safe reproduction.
- Kernel, Mesa, Mutter, firmware and BIOS versions.
- AMD firmware information and a DMUB trace, following the current kernel debug guide.
- A controlled PSR-disabled comparison after checking the flag against the running kernel source.
- Full logs and clear steps for the upstream project that owns the failing area.

Debugfs tracing can require root, produce a great deal of data and alter timing. I would enable only the trace groups needed for a planned reproduction. AMD's [Display Core debug guide](https://docs.kernel.org/gpu/amdgpu/display/dc-debug.html) lists the relevant DMCU, DMCUB and PSR tools.

At the time of the incident, cached Fedora metadata listed newer kernel, AMD firmware, Mesa and Mutter packages. Updating those distro packages was a sensible first test. Framework also listed BIOS 3.20 for this model. I would treat a BIOS update as separate work: read the current [Framework release guidance](https://community.frame.work/t/framework-laptop-13-ryzen-7040-bios-3-20-release-stable/83283) and recovery notes before deciding. Versions and advice age quickly.

## 17. The stack in four lines

This is the reference I wanted when I began:

```text
Rendering:    app or toolkit -> graphics API -> Mesa -> DRM render node -> amdgpu -> graphics engine -> buffer
Submission:   app or toolkit -> Wayland client support -> Wayland protocol -> compositor
Composition:  client buffers -> compositor renderer -> output buffer, unless direct scanout or planes avoid it
Presentation: compositor -> libdrm -> DRM/KMS -> amdgpu Display Core -> display engine -> eDP -> panel
```

The first two lines are parallel interfaces from a normal accelerated Wayland application. They meet when the client attaches rendered content as a Wayland buffer. Composition may follow. Presentation and scanout put a selected buffer on the panel.

### Type glossary

| Name | Type | What it does |
|---|---|---|
| Brave, GTK app, Qt app | User-space component | Runs application logic and produces window content. |
| GTK, Qt, SDL, GLFW | User-space library or framework | Provides UI, input, window integration and often a renderer. Each is optional. |
| OpenGL, Vulkan | Graphics API specification | Defines rendering and compute operations. Neither is a process or driver. |
| Mesa | User-space graphics project and runtime components | Supplies Linux graphics API implementations and GPU-specific user-space drivers. |
| RadeonSI | Mesa user-space driver component | Implements OpenGL for supported AMD GPUs through Mesa's Gallium stack. |
| RADV | Mesa user-space driver component | Implements Vulkan for supported AMD GCN and RDNA GPUs. |
| Wayland core and extensions | Protocol specification | Defines requests and events between clients and a compositor. |
| `libwayland-client`, `libwayland-server` | User-space library | Marshals and dispatches Wayland messages. |
| `wl_surface`, `wl_buffer` | Wayland protocol object | Represents surface state and attachable content. |
| Mutter, Sway | User-space component | Acts as Wayland server, compositor and window manager. |
| wlroots | User-space library | Supplies reusable compositor backends, renderers, buffer support and protocol support. Sway uses it. |
| DMA-BUF | Kernel sharing framework and exported object | Shares buffer storage through file descriptors. |
| `dma_fence`, sync file, DRM sync object | Synchronisation primitive or object | Signals completion and coordinates asynchronous buffer access. |
| `libdrm` | User-space library | Wraps many DRM ioctls and provides common definitions and helpers. |
| DRM | Linux kernel framework and user-space ABI | Provides device nodes, memory and buffer management, command submission, synchronisation and display interfaces. |
| DRM render node, such as `renderD128` | Device-file interface | Allows rendering while modesetting and DRM master remain unavailable. |
| DRM primary node, such as `card1` | Device-file interface | Exposes KMS and other primary-node operations under access control. |
| KMS | DRM display-control API and framework | Controls framebuffers, planes, CRTCs, connectors, modes and atomic state. |
| `amdgpu` | Linux kernel driver component | Implements AMD-specific DRM rendering and KMS operations. |
| AMD Display Core | Kernel subsystem inside `amdgpu` | Translates DRM display state into AMD display-pipeline state. |
| DCN 3.1.4 | AMD display hardware generation | The display controller generation reported on this system. |
| DMCUB or DMUB microcontroller | Hardware | Runs display firmware and handles commands for some display features. |
| DMCUB or DMUB firmware | Firmware component | Code that runs on the display microcontroller. |
| GPU graphics engine | Hardware | Executes drawing, shader and compute command streams. |
| Display engine | Hardware | Fetches scanout buffers, processes planes and drives output links. |
| eDP | Link protocol | Defines the data stream between the display source and panel. |
| eDP link | Hardware | Carries that stream from the display engine to the laptop panel. |
| PSR | eDP feature and state machine | Lets a capable panel retain an unchanged frame to save power. |
| Render target or pixel storage | Data object and storage interpretation | Holds pixels produced by rendering. |
| DRM framebuffer | KMS kernel object | Describes storage that KMS can select for scanout. |

### Primary references

For Wayland's client, server and buffer model, I used the [Wayland architecture guide](https://wayland.freedesktop.org/docs/book/Architecture.html), [protocol model](https://wayland.freedesktop.org/docs/book/Protocol.html), [reference documentation](https://wayland.freedesktop.org/docs/html/) and [core protocol](https://wayland.freedesktop.org/docs/html/apa.html). GTK documents its [Wayland backend](https://docs.gtk.org/gtk3/wayland.html), while Qt documents its [Wayland Client module](https://doc.qt.io/qt-6/qtwaylandclient-module.html).

For graphics APIs and Mesa, I used the [Vulkan Registry](https://registry.khronos.org/vulkan/), [OpenGL Registry](https://registry.khronos.org/OpenGL/index_gl.php), [Mesa platform overview](https://docs.mesa3d.org/systems.html), [RADV documentation](https://docs.mesa3d.org/drivers/radv.html) and [Mesa source-tree guide](https://docs.mesa3d.org/sourcetree.html).

For shared buffers and synchronisation, the kernel's [DMA-BUF documentation](https://www.kernel.org/doc/html/latest/driver-api/dma-buf.html) defines DMA-BUF, reservations and fences.

For the kernel graphics boundary, I used the [DRM user-space interfaces](https://www.kernel.org/doc/html/latest/gpu/drm-uapi.html), [DRM internals](https://www.kernel.org/doc/html/next/gpu/drm-internals.html) and [KMS documentation](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html).

For compositor roles, I used the [Mutter API overview](https://gnome.pages.gitlab.gnome.org/mutter/meta/), [Mutter backend documentation](https://gnome.pages.gitlab.gnome.org/mutter/meta/class.Backend.html), [Sway README](https://github.com/swaywm/sway), [wlroots documentation](https://wlroots.pages.freedesktop.org/wlroots/) and [Wayland compositor guide](https://wayland.freedesktop.org/docs/book/Compositors.html).

For the AMD path, I used the kernel's [`amdgpu` guide](https://docs.kernel.org/gpu/amdgpu/index.html), [driver-core guide](https://docs.kernel.org/gpu/amdgpu/driver-core.html), [DCN programming model](https://docs.kernel.org/gpu/amdgpu/display/programming-model-dcn.html) and [Display Core debug guide](https://docs.kernel.org/gpu/amdgpu/display/dc-debug.html). The current mainline [AMD debug-mask definitions](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/include/amd_shared.h) document the PSR and idle-power flags. VESA's [eDP presentation](https://www.vesa.org/wp-content/uploads/2010/12/DisplayPort-DevCon-Presentation-eDP-Dec-2010-v3.pdf) illustrates panel-side frame retention.

The crash began as a fullscreen photograph. The useful result is a map: APIs describe rendering, Wayland carries surface state and buffer references, Mesa implements graphics APIs, the compositor chooses an output plan, KMS presents it, and AMD's display hardware scans it out. When the screen next freezes, that map tells me which logs belong together and which merely arrived wearing a suspicious hat.
