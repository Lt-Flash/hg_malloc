# Benchmarks

All figures from one OpenSIPS binary built from master plus HG_MALLOC. The allocator is chosen on the command line, so it is the only difference between arms:

| arm | options |
|---|---|
| F_MALLOC | `-a F_MALLOC` |
| F_PARALLEL_MALLOC | `-a F_MALLOC -s F_PARALLEL_MALLOC` (F_PARALLEL_MALLOC is shared-memory only) |
| HP_MALLOC | `-a HP_MALLOC` |
| HG_MALLOC | `-a HG_MALLOC` |

## Setup

- **Server:** 16-vCPU KVM guest, Debian 13, gcc 14, kernel 6.12. 8 MB UDP socket buffers, `maxbuffer` 8 MB, RPS across all CPUs, a hugetlb pool large enough for the caps, `LimitMEMLOCK` unlimited.
- **Load:** sipp, up to 60 instances spread over three other hosts on the same LAN, 1,000-2,000 calls per second each.
- **CPU:** user plus system time of every OpenSIPS process over a fixed window while the load runs.
- **Latency:** sipp's per-call response times, plus a separate probe sending 100 requests per second and timing each round trip in microseconds.
- **Drops:** the kernel's UDP receive-buffer errors on the server.

## Proxy load: INVITE answered by tm

Vanilla config: proto_udp, tm, sl, signaling, sipmsgops; 8 UDP workers; each INVITE answered 486 by tm, ACK absorbed. 60 s per point.

All four arms measured in one session, `-m 2048 -M 32`; HG_MALLOC elastic `-m 128:2048 -M 16:32`.

| calls/s | | F_MALLOC | F_PARALLEL_MALLOC | HP_MALLOC | HG_MALLOC |
|---|---|---|---|---|---|
| 20,000 | CPU, cores | 2.20 | 2.07 | 2.09 | **1.87** |
| | UDP drops | 0 | 0 | 0 | **0** |
| | INVITE p99 | 0.85 ms | 0.89 ms | 0.79 ms | **0.75 ms** |
| 45,000 | CPU, cores | 4.15 | 3.92 | 3.91 | **3.32** |
| | UDP drops | 16,578 | 0 | 5,090 | **0** |
| | INVITE p99 | 6.8 ms | 4.1 ms | 5.7 ms | **2.4 ms** |
| 60,000 | CPU, cores | 5.06 | 5.13 | 5.69 | **3.99** |
| | UDP drops | 250,185 | 178,585 | 160,448 | **0** |
| | INVITE p99 | 10.3 ms | 10.6 ms | 12.1 ms | **5.7 ms** |

Every call completed in every arm; at 60,000 the probe saw INVITE timeouts with F_PARALLEL_MALLOC (4) and HP_MALLOC (4), none with HG_MALLOC. The elastic arena grew from 128 MB to 528 MB, 1,136 MB and 1,504 MB at the three rates. The longest arena-lock hold was 319 us, 497 us and 1,262 us; the 60,000 run recorded 2 holds over the 1 ms stall threshold.

A second session, F_MALLOC against HG_MALLOC only, gave the same picture: 17-21% less CPU, and 0 drops against 6,322 at 45,000 and 133,549 at 60,000.

## Registrar under a cold start: 1.2 million REGISTERs at 60,000 per second

usrloc without a database, registrar, 4 UDP workers. The elastic arena starts at 128 MB and has to grow to hold every registration.

| | F_MALLOC | F_PARALLEL_MALLOC | HP_MALLOC | HG_MALLOC `-m 128:4096` |
|---|---|---|---|---|
| shm size | `-m 4096` | `-m 4096` | `-m 4096` | starts at 128 MB |
| runs | 2 | 2 | 2 | 3 |
| registrations stored | 1,200,000 | 1,200,000 | 588,293 and 554,791 | **1,200,000** |
| UDP receive drops | 1.19 M and 1.65 M | 1.36 M and 1.54 M | 7.25 M and 7.34 M | **0 in every run** |
| sipp retransmissions | 1.21 M and 1.74 M | 1.41 M and 1.60 M | 8.36 M and 8.40 M | **0 in every run** |
| response p50 | 92 ms and 581 ms | 572 ms and 574 ms | 11.9 s | **under 1 ms** |
| response p99 | 11.6 s and 19.5 s | 15.5 s | 27.9 s and 28.0 s | **4 ms** |
| worst response | 23.5 s and 27.6 s | 19.6 s and 23.5 s | 28.0 s | **12-20 ms** |
| probe p99.9 | 78-82 ms | 74-79 ms | no reply (20 of 20 timed out) | **4.5-5.5 ms** |
| CPU, cores | 3.86-4.00 | 3.80-3.82 | 4.00 (saturated) | **1.74-1.83** |

With HP_MALLOC about half the registrations failed: the 4 workers were saturated, and a profile put 81% of their CPU inside `hp_shm_malloc`.

In every HG run the arena grew from 128 MB to 1,584 MB in 91 steps with no lock stalls; the longest lock hold was 330 us, and each grow's commit (up to 7 ms) ran with the lock released.

## Integrity

- **Soak:** hgstress, 9 workers each doing 2 million stamped alloc/free pairs across every size class and the large tier, verifying stamps while the others churn: 0 torn words, 0 corruption, under fixed and elastic arenas.
- **Fault injection:** a double free and a scribbled free-list link injected into a running process are both caught and counted (`double_free`, `foreign_ptr`), and the process keeps serving.
- **Memory lock limit:** with `RLIMIT_MEMLOCK` at 64 KB and `CAP_IPC_LOCK` dropped, an elastic arena warns at startup, and its refused grows name the limit; with the limit lifted the same arena grows normally.

## Builds

| target | result |
|---|---|
| x86_64, gcc 14 | clean |
| x86_64, clang, `-Werror` | clean |
| HG_MALLOC compiled out | clean |
| arm32, arm64 (cross) | compile and link |
| i386 (container) | clean; hgstress soak passes |
| Alpine / musl (container) | clean; hgstress soak passes |
