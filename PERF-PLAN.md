# Perf work plan — read this first

Companion to the findings review: <https://claude.ai/code/artifact/e06b26da-01c2-4a0a-ace2-31056a349853>

The artifact says **what** to fix and **why**, with file:line anchors. It does not say how to build,
how to measure, or what "done" looks like — that is this file. Written from the web session at
`c7881fb`, where no build was possible. Everything below was read out of the repo's own
`docs/building.md`, `docs/profiling.md` and `CMakePresets.json`, not assumed.

---

## Do this before writing a single line of code

### 1. Establish environment facts

None of these are knowable from the source, and several change which findings even apply:

```bash
# Which encoder will actually run? (decides whether E1's syscall count is realistic
# and which encoder set_bitrate path F1/F2 would exercise)
lspci -nn | grep -Ei 'vga|3d|display'
ls /dev/dri/

# Is the Perfetto SDK present? Without it the tracing preset will not configure.
ls /usr/share/perfetto/sdk/perfetto.h /usr/share/perfetto/sdk/perfetto.cc

# Headset + adb
adb devices
adb shell getprop ro.product.model

# Link. E1/E3's whole premise is packets-per-second, which scales with bitrate.
# Read the bitrate actually configured on the headset before quoting any number.
```

Record the answers. The review's rates assume 100 Mbit/s at 90 Hz with two video streams; if the
real session is 50 Mbit/s at 72 Hz, halve everything before claiming a win.

### 2. Build a baseline and capture a baseline trace

**No fix gets written until there is a `baseline.pftrace` on disk.** `pftrace_summary.py` does
before/after diffs with Δmean, which is the entire verification story for the server-side work — and
it is worthless without the "before".

```bash
# NOTE: the presets set CMAKE_BUILD_TYPE=Debug. Profiling a Debug build is meaningless.
# Always override it.
cmake --preset server-tracing -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build-server-tracing

WIVRN_TRACING=inprocess WIVRN_TRACING_FILE=/tmp/baseline.pftrace \
  build-server-tracing/server/wivrn-server
# connect headset, stream a real app for ~60s, disconnect (the trace is written on disconnect)

tools/perfetto/pftrace_summary.py /tmp/baseline.pftrace
```

Keep the baseline trace and the session parameters (bitrate, refresh rate, codec, app) somewhere
durable. Every later comparison must use the **same** parameters or it is not a comparison.

---

## Constraints that will bite

| Fact | Consequence |
| --- | --- |
| Presets default to `CMAKE_BUILD_TYPE=Debug` | Override to `RelWithDebInfo` for anything measured. |
| `WIVRN_WERROR=ON` in the `base` preset | Every change must be warning-clean or the build fails. |
| **There is no test suite.** `common/wivrn_serialization_ut.cpp` is in no `CMakeLists.txt` — it is compile-time `static_assert`s that are never compiled. | Verification is: it builds, and it streams correctly in a real session. Budget for real sessions, not for `ctest`. |
| `protocol_version` (`common/protocol_version.h`) is a compile-time hash over the packet structs | Touching any packet struct forces a matched rebuild **and reinstall** of both server and APK. Mismatched versions are rejected at handshake with `incompatible_version`. |
| **Tracing is server-only.** `wivrn_trace.h` lives in `server/utils/`; `grep -rln perfetto client/` is empty. | The two client-side findings (E3, E4) have **no ready-made instrument**. See below. |

### Which side each change touches

- **Server binary only** — E1, E2, E5, E6, F1, F2. Fast loop: rebuild, restart server, re-trace.
- **Shared `common/`, so needs an APK rebuild to take effect on the headset** — E3.
- **Client only, needs APK rebuild + `adb install`** — E4, E7.
- **Both sides + protocol hash changes** — F3, if the payload size moves into
  `video_stream_description` rather than staying a constant.

The server-only set is where to start, purely because the loop is ~30 seconds instead of ~10 minutes.

---

## Measurement, per finding

### E1 (batch shard sends) and E2 (AES key schedule) — instrument already exists

`video_encoder::SendData` is already wrapped in a trace span:
`server/encoder/video_encoder.cpp:304` / `:346`, on `cpu_track::network`. Both findings live entirely
inside that span. So:

```bash
tools/perfetto/pftrace_summary.py /tmp/baseline.pftrace /tmp/after.pftrace
```

Δmean on `SendData` is the number. Do E2 first — it is a few lines and isolates cleanly — then E1,
so the two effects can be attributed separately rather than measured as one lump.

For E1 specifically, also confirm the syscall reduction directly, since Δmean alone won't prove the
mechanism:

```bash
sudo perf stat -e 'syscalls:sys_enter_writev,syscalls:sys_enter_sendmmsg' -p $(pidof wivrn-server) -- sleep 10
```

**E1 correctness risk, do not skip:** `UDP::send_many_raw` (`common/wivrn_sockets.cpp:399`) calls
`sendmmsg` and ignores the returned count — messages the kernel did not accept are silently dropped.
That is defensible for UDP but it means a 50-datagram burst into a full send buffer loses the tail
*and* the loss shows up as an IDR storm, not as an error. Chunk at 16–32, and watch the client's
loss behaviour (E7's `debug_why_not_sent` lines in `adb logcat`) before and after. If IDR frequency
goes up, the batch size is too large for the socket buffer.

### E3 (40 KB buffer churn) and E4 (O(n²) reassembly) — no instrument, build one first

There is no client tracing. Two options, in order of preference:

1. **`simpleperf` on the client process.** Sampling, no instrumentation, and it will show
   `malloc`/`free` and the `try_submit_frame` prefix scan directly if the theory is right.
   ```bash
   WIVRN_PKG=$(adb shell pm list packages | grep wivrn | cut -d: -f2)
   adb shell simpleperf record -g -p $(adb shell pidof $WIVRN_PKG) --duration 20 -o /data/local/tmp/perf.data
   adb pull /data/local/tmp/perf.data && simpleperf report -g -i perf.data
   ```
   Capture this **before** touching either finding. If neither `operator new` nor
   `shard_accumulator::try_submit_frame` shows meaningfully in the profile, say so and drop or
   downgrade these two rather than fixing them on my say-so — they were derived from arithmetic,
   not measurement.

2. If simpleperf is awkward, add a counter to `global_metric` (`client/scenes/stream.h:296`) and
   read it off the existing in-headset performance panel. That is most of F6 anyway, so it is not
   wasted work.

### E5 (pacer shutdown race)

Not a performance fix — a hang. There is nothing to profile. Verify by repeatedly starting and
stopping sessions and confirming the server exits cleanly:

```bash
for i in $(seq 20); do
  timeout 20 build-server-tracing/server/wivrn-server & sleep 8; kill -TERM %1; wait
done
```

The race window is small, so absence of a hang in 20 runs is weak evidence. The stronger check is
reading the diff: `compute_cv.wait` must be a predicate wait or the `condition_variable_any`
stop-token overload, so that the stop request cannot be missed regardless of timing.

### E6 (DSCP)

Verify the option is actually set, not just that the code runs — the current function would fail on
these `AF_INET6` sockets, which is half the finding:

```bash
# on the server, while streaming
ss -uapn | grep wivrn                       # find the socket
sudo tcpdump -i any -n -v udp port 9757 -c 5 | grep -i 'tos\|class'
```

Look for traffic class `0xb8` (DSCP EF). If it reads `0x0`, the setsockopt silently did nothing —
which is exactly the bug being fixed, so it is worth confirming rather than assuming.

Whether it *helps* needs a congested link to show up at all. Load the network (a large download on
another machine on the same AP) and compare the client's decode-lateness plot with and without.
On an idle network, expect no measurable difference — that is the correct result, not a failure.

---

## Order of work

Follows the artifact's shortlist, resequenced for the build loop rather than for value.

1. **Baseline trace + simpleperf capture.** Nothing else starts first.
2. **E5** — correctness, no measurement needed, gets a real bug out of the tree.
3. **E2** — small, server-only, measurable on the existing `SendData` span.
4. **E6** — small, server-only, verified with `tcpdump` rather than a trace.
5. **E1** — the big one. Server-only, same span, but rewrites the slicing loop; do it alone so the
   diff is attributable, and watch IDR rate for the `sendmmsg` partial-send hazard.
6. **E3 + E4** — only if the simpleperf capture from step 1 supports them. Both need an APK cycle.
7. **F2** then **F6** — small feature work, and F6 is the instrument F1 and F4 would be judged with.

Commit each finding separately with its measurement in the commit message (Δmean, syscall counts,
whatever the instrument gave). One finding per commit — a squashed "perf improvements" commit makes
it impossible to bisect a regression back to the change that caused it.

Do not start F1 (adaptive bitrate), F3 (MTU) or F4 (loss recovery) in the same pass. They are design
work, they change behaviour rather than cost, and F4 in particular should not be attempted without
F6's loss telemetry in place to judge it.

---

## If a finding does not hold up

Several of these were derived from constants and arithmetic, not from a profile. E3 and E4 are the
most exposed — the allocator may already handle the 40 KB churn well, and the O(n²) scan may be
invisible next to the memcpy into the MediaCodec buffer. If the measurement says so, record that in
the commit history or here and move on. A finding that was wrong is worth knowing about; a finding
that was "fixed" without evidence is worse than one left alone.
