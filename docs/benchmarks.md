# Benchmarks

All figures from one OpenSIPS binary built from master plus HG_MALLOC, with F_MALLOC and HG_MALLOC selected by `-a`, so the allocator is the only difference between arms.

## Setup

- **Server:** 16-vCPU KVM guest, Debian 13, gcc 14, kernel 6.12. 8 MB UDP socket buffers, `maxbuffer` 8 MB, RPS across all CPUs, a hugetlb pool large enough for the caps, `LimitMEMLOCK` unlimited.
- **Load:** sipp, up to 60 instances spread over three other hosts on the same LAN, 1,000-2,000 calls per second each.
- **CPU:** user plus system time of every OpenSIPS process over a fixed window while the load runs.
- **Latency:** sipp's per-call response times, plus a separate probe sending 100 requests per second and timing each round trip in microseconds.
- **Drops:** the kernel's UDP receive-buffer errors on the server.

## Proxy load: INVITE answered by tm

Vanilla config: proto_udp, tm, sl, signaling, sipmsgops; 8 UDP workers; each INVITE answered 486 by tm, ACK absorbed. 60 s per point.

| calls/s | | F_MALLOC `-m 2048` | HG fixed `-m 2048` | HG elastic `-m 128:2048` |
|---|---|---|---|---|
| 20,000 | CPU, cores | 2.31 | 1.84 | 1.88 |
| | UDP drops | 0 | 0 | 0 |
| | INVITE p99 | 0.83 ms | 0.79 ms | 0.73 ms |
| 45,000 | CPU, cores | 4.13 | 3.34 | 3.42 |
| | UDP drops | 6,322 | 0 | 0 |
| | INVITE p99 | 6.6 ms | 1.7 ms | 1.8 ms |
| 60,000 | CPU, cores | 5.04 | 3.96 | 3.96 |
| | UDP drops | 133,549 | 0 | 0 |
| | INVITE p99 | 13.1 ms | 5.5 ms | 4.7 ms |

Every call completed in every arm. The elastic arena grew from 128 MB to 528 MB, 1,136 MB and 1,520 MB at the three rates. Arena-lock holds stayed under 1 ms in every HG run (worst 963 us), with no stalls.

## Registrar under a cold start: 1.2 million REGISTERs at 60,000 per second

usrloc without a database, registrar, 4 UDP workers. The elastic arena starts at 128 MB and has to grow to hold every registration.

| | F_MALLOC `-m 4096` | HG elastic `-m 128:4096` |
|---|---|---|
| runs | 2 | 3 |
| registrations stored | 1,200,000 | 1,200,000 |
| UDP receive drops | 1,186,214 and 1,645,819 | 0 in every run |
| sipp retransmissions | 1,212,770 and 1,741,181 | 0 in every run |
| response p50 | 92 ms and 581 ms | 0 ms (under 1 ms) |
| response p99 | 11.6 s and 19.5 s | 4 ms |
| worst response | 23.5 s and 27.6 s | 12-20 ms |
| probe p99.9 | 78-82 ms | 4.5-5.5 ms |
| CPU, cores | 3.86-4.00 | 1.74-1.83 |

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
