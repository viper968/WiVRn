# Perf work progress

Findings: <https://claude.ai/code/artifact/e06b26da-01c2-4a0a-ace2-31056a349853>
Plan: `PERF-PLAN.md` · Baseline: `traces/BASELINE.md`

Baseline: **`SendData` mean 0.194 ms**, n=46,146 (nvenc h265, ~50 Mbit/s, 90 Hz, Quest 3S).

| # | Item | Commit | Measured result |
|---|------|--------|-----------------|
| 0 | Baseline capture | `92cc79d2` | SendData 0.194 ms, n=46,146 |
| 1 | E5 pacer lost wakeup + uninit `in_flight_frames` | `23c69eaa` | correctness; race reproduced and fixed |
| 2 | E2 AES key schedule per datagram | `5a44c202` | **~9 ns/datagram — negligible** |
| 3 | E6 DSCP never applied | `8c725dc0` | now applied; wire check outstanding |
| 4 | E1 batch shard sends | `f9a9314a` | **24× fewer syscalls, ~0.08% of a core** |
| 5 | E3/E4 client-side | not started | bounded at 3.69% client CPU, unattributed |

## Corrections to the original review

The review was written without the ability to build or measure. Three claims did not survive:

- **E2 was overstated.** The AES key schedule costs ~9 ns/datagram with AES-NI, not the
  meaningful per-packet cost assumed. Kept as a consistency cleanup, not a perf win.
- **E6's "wrong sockopt level" was wrong.** Linux accepts `IP_TOS` on an `AF_INET6` socket and
  applies it to v4-mapped traffic, which is what WiVRn actually uses. The real defect was only
  that `set_tos` had no callers. Now sets `IPV6_TCLASS` too, for native v6.
- **E1 was overstated.** The 24× syscall reduction is real, but it is worth ~0.08% of one core,
  not the "largest single reduction in work" claimed. The kernel's per-packet UDP path dominates
  and `sendmmsg` still pays it.

## Outstanding

- **E6 wire check** needs root, which was not available:
  `sudo tcpdump -v -i enp44s0 udp port 9757 | grep -i 'tos\|class'` — expect `0xb8`.
  Only shows a benefit on a congested network.
- **E1 end-to-end effect** unmeasured. Needs a session reproducing `traces/BASELINE.md`
  conditions (~50 Mbit/s), which requires the headset worn and moving. Validation sessions
  only reached ~12 Mbit/s, where `SendData` mean is not comparable to the baseline.
- **E3/E4** need a source-built client for symbols; the nightly APK ships a stripped
  `libwivrn.so`.

## Pre-existing bugs found along the way (not from this work)

- `wivrn-server` **segfaults** when per-connection setup fails, e.g. when the port is already
  bound: it logs `Client connection failed: Address already in use` and then dies on SIGSEGV
  instead of reporting the error and carrying on. Reproduced twice.

## Rules

- One commit per finding, measurement in the commit message.
- Rebuild: `. deps/env.sh && cmake --build build-server-tracing -j16` (`-Werror` is on).
- Stop the Flatpak WiVRn first; both bind port 9757. Verify no stale `wivrn-server` survives
  a `pkill` — one did, and the next start crashed on the bound port.
