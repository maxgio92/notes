# Incident evidence

Local facts and evidence limits.

## System

| Item | Value |
|---|---|
| Laptop | Framework Laptop 13, AMD Ryzen 7040 Series |
| GPU | AMD Radeon 780M, Phoenix1, PCI ID `1002:15bf` |
| Kernel | `7.1.6-201.fc44.x86_64` |
| Display Core | DCN 3.1.4, Display Core 3.2.378 |
| DMCUB firmware | `0x08005D00` |
| Mesa | 26.1.5 |
| Mutter | 50.3 |
| GNOME Shell | 50.3 |
| Brave | Flatpak `com.brave.Browser` 1.94.117 |
| BIOS | 03.18 |
| Panel | Internal `eDP-1`, 2256x1504 |
| Panel Self Refresh | Supported and enabled |

- Kernel command line: no AMD display override.
- `amdgpu.dcdebugmask` after reboot: `0`.

## Timeline

| Time on 2026-08-30 | Fact |
|---|---|
| 11:15:04.809 | Mutter logged `meta_window_set_stack_position_no_sync`. |
| 11:15:08.363 | The first repeated DMCUB error appeared. |
| 11:15:09.126 | `mpc2_assert_idle_mpcc` timed out. |
| 11:15:33 | `Error queueing DMUB command: status=2` began to repeat. |
| 11:16:18.661 | First power-key press. |
| 11:16:22.251 | KMS thread warning in `dmub_psr_get_state`. |
| 11:17:24.003 | Failed `user.slice` freeze. |
| 11:17:24.019 | `s2idle` entry. |
| 11:18:12.845 | More DMCUB and PSR timeouts after resume. |
| 11:18:17 and 11:18:19 | Failed CRTC blank. |
| 11:25:27.890 | Later power-key press. |
| 11:26:33.045 | Second failed `user.slice` freeze. |
| 11:28:14.832 | More display timeouts after the second resume. |
| 11:28:19 and 11:28:21 | Display Core again failed to blank the CRTC. |
| 11:28:47 | The old boot journal ended. |
| 11:29:35 | The next boot started. |
| 11:39:37.665 | The quoted Dawn and Vulkan warnings appeared. |

## Main kernel messages

```text
amdgpu 0000:c1:00.0: [drm] *ERROR* Error queueing DMUB command: status=2
amdgpu 0000:c1:00.0: [drm] *ERROR* dc_dmub_srv_log_diagnostic_data: DMCUB error - collecting diagnostic data
amdgpu 0000:c1:00.0: [drm] REG_WAIT timeout 1us * 100000 tries - mpc2_assert_idle_mpcc line:479
amdgpu 0000:c1:00.0: [drm] REG_WAIT timeout 1000us * 30 tries - dcn31_wait_for_det_apply line:118
amdgpu 0000:c1:00.0: [drm] REG_WAIT timeout 1us * 100000 tries - optc314_disable_crtc line:146
amdgpu 0000:c1:00.0: [drm] REG_WAIT timeout 1us * 100 tries - dcn31_program_compbuf_size line:141
[drm:dcn20_wait_for_blank_complete [amdgpu]] *ERROR* DC: failed to blank crtc!
```

- No graphics ring timeout.
- No GPU page fault.
- No Vulkan device loss.
- No successful GPU reset.

## Earlier signs

- 2026-08-24: GNOME Shell's KMS thread was inside an AMD atomic display commit.
- 2026-08-24: `dcn31_program_compbuf_size` timed out.
- 2026-08-27: the same timeout appeared again.

## Available updates

| Component | Installed | Cached update |
|---|---:|---:|
| Kernel | 7.1.6 | 7.1.10 |
| AMD firmware | 2026-06-22 | 2026-08-10 |
| Mesa | 26.1.5 | 26.1.8 |
| Mutter | 50.3 | 50.4 |

- Framework BIOS for this model: 3.20.
- AMD PhoenixPI change: `1.2.0.0e` to `1.2.0.0f`.

## Limits of the evidence

- Proven fault area: AMD Display Core and DMCUB command handling.
- Unproven cause: DMCUB firmware, kernel driver, or display power-state code.

- The Mutter assertion came before the DMCUB errors.
- The assertion also occurred after reboot without a display fault.
- The assertion does not prove cause.

- The Dawn and Vulkan warnings came after reboot.
- They did not cause this event.
