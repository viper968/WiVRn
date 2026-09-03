# Perf work progress

Findings: <https://claude.ai/code/artifact/e06b26da-01c2-4a0a-ace2-31056a349853>
Plan: `PERF-PLAN.md` · Baseline: `traces/BASELINE.md`

Baseline: **`SendData` mean 0.194 ms**, n=46,146 (nvenc h265, ~50 Mbit/s, 90 Hz, Quest 3S).

| # | Item | Commit | Measured result |
|---|------|--------|-----------------|
| 0 | Baseline capture | `92cc79d2` | SendData 0.194 ms, n=46,146 |
| 1 | E5 pacer lost wakeup + uninit `in_flight_frames` | `23c69eaa` | correctness; race reproduced and fixed |
| 2 | E2 AES key schedule per datagram | `5a44c202` | **~9 ns/datagram — negligible** |
| 3 | E6 DSCP never applied | `8c725dc0` | **confirmed on the wire: `tos 0xb8`** |
| 4 | E1 batch shard sends | `f9a9314a` + `0d6c6e0a` | **SendData −6% at matched load** |
| 5 | E3/E4 client-side | not started | bounded at 3.69% client CPU, unattributed |
| 6 | F6 loss + IDR telemetry (server) | `d162a460` | **3% packet loss = ~25% of frames + ~20 IDR/s** |

## Corrections to the original review

The review was written without the ability to build or measure. Three claims did not survive:

- **E2 was overstated.** The AES key schedule costs ~9 ns/datagram with AES-NI, not the
  meaningful per-packet cost assumed. Kept as a consistency cleanup, not a perf win.
- **E6's "wrong sockopt level" was wrong.** Linux accepts `IP_TOS` on an `AF_INET6` socket and
  applies it to v4-mapped traffic, which is what WiVRn actually uses. The real defect was only
  that `set_tos` had no callers. Now sets `IPV6_TCLASS` too, for native v6.
- **E1 was overstated, then partly vindicated.** The 24× syscall reduction is real but worth
  only ~0.08% of a core on its own. The first implementation was actually a **3% regression**,
  because clearing the batch vectors destroyed the packets and made every shard reallocate.
  With the packets reused (`0d6c6e0a`) it is **−6% on SendData** at matched load — a real win,
  but from fixing allocator churn, not from the syscall saving the review predicted.

## Outstanding

- **E3/E4** need a source-built client for symbols; the nightly APK ships a stripped
  `libwivrn.so`. Bounded at 3.69% of client CPU regardless.
- **Whether E6 helps** still needs a congested network to show up; the marking is confirmed
  present but an idle link shows no difference, which is the correct result.
- `encode` and `encoder_work iter` read +3% against the baseline in every post-change run
  alike, including the one where `SendData` improved. That offset tracks the session
  (52 vs 51 Mbit/s), not any of these commits — worth re-checking if it ever grows.

## Bug found along the way (not from this work)

- `wivrn-server` **segfaults** when per-connection setup fails: it logs
  `Client connection failed: Address already in use` and then dies on SIGSEGV instead of
  reporting the error and carrying on. Reproduced twice. Worth reporting upstream — a bind
  failure should not be fatal.

  The port conflict that triggered it was self-inflicted, though: `pkill -f
  ".../wivrn-server"` also matches the wrapper shell whose command line contains that path,
  including the shell running `pkill`, which then dies mid-cleanup and leaves the server
  alive. Use `pkill -x wivrn-server`. Likewise `pgrep -a wivrn wayvr` takes only one pattern
  and silently errors, so it reports "nothing running" regardless.

## What F6 revealed

Injecting 3% packet loss with netem, against a Quest 3S:

```
healthy    link: 1440 frames, no loss over 5.0s
3% loss    link: 26.36% of frames incomplete (364/1381),
                 26.36% lost in flight, 110 IDR recoveries over 5.0s
restored   link: 1494 frames, no loss over 5.0s
```

3% packet loss costs **~25% of frames** and **~20 forced keyframes per second**. That is the
cost of the current any-loss-means-IDR policy, now measurable — and the strongest argument yet
for F4 (FEC or reference-picture selection). It is also the instrument F1 would be tuned with.

The in-headset half of F6 still needs a client build.

## Rules

- One commit per finding, measurement in the commit message.
- Rebuild: `. deps/env.sh && cmake --build build-server-tracing -j16` (`-Werror` is on).
- Stop the Flatpak WiVRn first; both bind port 9757. Verify no stale `wivrn-server` survives
  a `pkill` — one did, and the next start crashed on the bound port.
