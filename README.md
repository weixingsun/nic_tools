# libnetacc — 0 code change, no recompiling, no special NIC driver.

**libnetacc** gives an existing Linux network program a faster TCP stack without
changing a line of its code. Load it in front of the program, and only the program's
connections run over a high-performance userspace TCP stack that talks to the NIC directly. 
Everything else on the host keeps using the normal kernel network stack. single NIC works

```
LD_PRELOAD=libnetacc.so  <elf-binary> <options>
```
---

## Why

Kernel TCP is general, safe and mature, and it is also where a fast service spends
much of its time: system calls, socket locks, copies, interrupts and wake-ups for every
message. At millions of messages a second, that overhead, not the application, sets
the ceiling and the tail latency.

libnetacc takes the traffic of the ports you choose off that path:

- **More throughput per core.** The same program on the same hardware moves more
  messages, or the same load on fewer cores.
- **Lower and steadier latency.** No interrupt-to-thread hand-off, and no system call
  per read or write.
- **Nothing to port.** The program keeps its own `read()`, `write()`, `epoll` and
  `accept()` calls. libnetacc answers them.
- **Contained.** Only the claimed ports go through it. SSH, monitoring and every other
  port stay on kernel TCP.

---

## Headline result: 9 million messages a second through one relay

A three-stage message chain — a producer, a relay and a consumer, each a separate
machine — carrying 1 KB messages, with **every byte of every message verified** at the
far end.

The relay is the hard part: it receives every message and sends it on, so it carries
the full load twice. Under libnetacc its application threads are almost idle at 9 M:
the work that used to sit in locks and system calls is gone.

<summary>How the 9 M pps with 1KB size packets figure</summary>

| Step | Messages/s | 2-hops latency (p50 / p99) | Data integrity |
|---|---|---|---|
| Kernel TCP (measured) | 2.0 M | 385 µs / 24.6 ms | |
| libnetacc + relay | **9.0 M** sustained (10 M peak) | **~105 µs / ~200 µs** | **0 errors** in 24 hours soak |

---

## Other workloads, in brief

| Program | Result |
|---|---|
| **nginx** (unmodified, master + workers) | **2x kernel TCP** on the same workers (1.45 M against 0.71 M requests/s). With 4 workers, 1.49 M requests/s, the cloud instance's packet limit. p99 1.6 ms at 190 k requests/s. Reloads under load without refused connections. |
| **valkey** (Redis protocol) | About the same throughput as kernel TCP (+2.5%) at the same p99 (0.135 ms), with far less CPU in its I/O threads. |
| **Bulk feeds** | A single TCP connection is capped by the cloud network at ~9.6 Gbps. libnetacc can stripe one logical stream over several connections and carried 3x more. |

---

## What it takes

- **Linux** most popular AWS instances (tested on x86/Arm).
- **One command to start:** start scripts pick the CPU layout, pin the network work to
  dedicated cores, and detect how the program is built: one process, or nginx-style
  master and workers.
- **The program unchanged.** Event-driven, threaded and pre-forked servers all work.

### Tools that come with it

Ordinary tools (`ss`, `tcpdump`) cannot see connections that bypass the kernel, so
libnetacc brings its own:

- **acc_diag** — one command that checks a running program is on the fast path and says
  what limits it.
- **acc_mon** — a live console: queues, NIC and cloud network limits, cores, every
  connection's throughput, CPU hot spots, and safe live tuning of a running program.
- **tcp_mon** — the same console for programs on kernel TCP, for side-by-side comparison.

---

## Quality

- Every build passes an automated suite with memory and undefined-behaviour checkers:
  100+ protocol test cases, fuzzing with millions of malformed frames, and
  multi-threaded race tests.
- Every result here was measured end to end with real traffic and full data
  verification.
- Settings that change behaviour ship off by default until they have been re-measured
  against earlier results. No release is allowed to make a past result worse.

## Status

A working proof of concept, validated in the lab for throughput, latency, data
integrity and 24-hour endurance. It's still developing features, please contact author
for more details.

---
