# Baseline — 2026-09-02

Reference measurement for the frame-path perf work. **Every later comparison must reproduce
these exact conditions** or it is not a comparison.

Primary trace: `baseline-20260902-184327.pftrace` (50 MB, n=46,146 encodes)

## Session parameters

| | |
| --- | --- |
| Server commit | `f9c28d1c` (branch `claude/wivrn-codebase-review-8m2mx7`) |
| Server build | `build-server-tracing`, RelWithDebInfo, `WIVRN_USE_PERFETTO=ON`, `WIVRN_USE_SYSTEM_BOOST=OFF` |
| Client | WiVRn-APK nightly `apk-42b3417ba92dff9aa45d4148639e3a354e1d0019`, pkg `org.meumeu.wivrn.github.nightly` |
| Headset | Meta Quest 3S, Android 14 |
| Encoder | **nvenc**, h265 10-bit, RTX 3050 Ti Mobile (driver 595.91.07) |
| Streams | 2 × 960×960 @ 24.6 Mbit/s + alpha 960×480 @ 0.6 Mbit/s |
| Total bitrate | ~50 Mbit/s (measured 51 Mbit/s on the wire) |
| Refresh rate | 90 Hz |
| Client decoder | `c2.qti.hevc.decoder` ×3 |
| Transport | server wired `192.168.3.2` (enp44s0) → headset Wi-Fi `192.168.3.186`, same /24 |
| VR app | WayVR 26.8.0-patched, launched from the headset app list |
| Encryption | **on** (default) — required, since E2 is about the per-datagram AES key schedule |

## Server-side slice stats (`pftrace_summary.py`)

| slice | n | mean ms | p50 ms | p99 ms | max ms |
| --- | --- | --- | --- | --- | --- |
| present_image | 46,150 | 0.020 | 0.016 | 0.043 | 3.763 |
| vk_copy_image_to_buffer | 46,147 | 0.080 | 0.079 | 0.081 | 1.021 |
| nvEncEncodePicture+Lock | 46,146 | 2.336 | 2.319 | 2.972 | 4.957 |
| encode | 46,146 | 3.738 | 3.088 | 6.256 | 7.369 |
| **SendData** | **46,146** | **0.194** | **0.184** | **0.401** | **2.948** |
| encoder_work iter | 23,074 | 7.484 | 7.483 | 8.800 | 10.455 |

`SendData` is the number E1 and E2 have to move — both live entirely inside that span.

At 24.6 Mbit/s per eye and 90 Hz, a frame is ~34 kB ≈ **24 shards**, so 0.194 ms covers ~24
`writev` calls ≈ **8 µs per shard**. Roughly 1.1M shards over this session.

Ignore the tool's `trace span` / `rate/s` columns — the span is computed across clock domains
and reads 5105 s for a ~4 minute session, so the derived rate is wrong. The per-slice
mean/p50/p99 are unaffected and are what the diff uses.

## Client-side profile

`simpleperf/baseline-client.data` — 10 s, cpu-clock, 17,463 samples.
Capture needs `--app org.meumeu.wivrn.github.nightly` (`-p <pid>` is refused even though the app
is PROFILEABLE and `perf_event_paranoid` is -1).

| shared object | % of client CPU |
| --- | --- |
| `[kernel.kallsyms]` | 38.85 |
| `libvrapiimpl.so` (Meta runtime) | 13.97 |
| `libc.so` | 13.14 |
| **`libwivrn.so`** | **3.69** |
| `vulkan.adreno.so` | 2.81 |
| `libsfplugin_ccodec.so` + stagefright + codec2 | ~7.5 |
| `libcrypto.so` | 1.37 |

Inside `libc.so`: `je_free` 12.07%, `je_malloc` 6.23%, `je_tcache_bin_flush_small` 3.88% —
so the allocator is ~2.9% of total client CPU, but **attribution is unresolved**: the release
APK ships a stripped `libwivrn.so`, so its symbols show as `libwivrn.so[+1a5b34]` and the
callgraph cannot be tied back to `try_submit_frame` or `receive_raw`.

**Consequence for E3/E4:** all of WiVRn's own client code is 3.69% of client CPU, which caps
what either finding can be worth. They are not disproven, but they are bounded and unattributed.
Resolving them needs a client built from source with symbols. Per PERF-PLAN.md they stay at
step 6 and should not be "fixed" on the strength of this.

## Reproducing

```bash
. /home/simon/Projects/WiVRn_eff/deps/env.sh
export XR_RUNTIME_JSON=$PWD/build-server-tracing/openxr_wivrn-dev.json
XRT_LOG=info WIVRN_TRACING=inprocess \
  WIVRN_TRACING_FILE=$PWD/traces/after-%t.pftrace \
  ./build-server-tracing/server/wivrn-server --no-manage-active-runtime
# connect: adb shell am start -a android.intent.action.VIEW \
#            -d "wivrn://192.168.3.2" org.meumeu.wivrn.github.nightly
# launch WayVR from the headset app list, stream ~4 min, then force-stop the client
python3 tools/perfetto/pftrace_summary.py traces/baseline-20260902-184327.pftrace traces/after-*.pftrace
```

The Flatpak WiVRn must be stopped first — both bind port 9757, and the local build also
removes `/run/user/1000/wivrn/comp_ipc` out from under it.

## Other traces here

- `baseline-20260902-183844.pftrace` (2 KB) — no app rendering, discard
- `baseline-20260902-183907.pftrace` (551 KB) — session start, too short
- `baseline-20260902-184454.pftrace` (344 KB) — tail after the ring buffer flushed, discard
