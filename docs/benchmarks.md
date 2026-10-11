# Benchmarks

All figures from one OpenSIPS binary built from master plus HG_MALLOC. The allocator is chosen on the command line, so it is the only difference between arms:

| arm | options |
|---|---|
| F_MALLOC | `-a F_MALLOC` |
| F_PARALLEL_MALLOC | `-a F_MALLOC -s F_PARALLEL_MALLOC` (F_PARALLEL_MALLOC is shared-memory only) |
| HP_MALLOC | `-a HP_MALLOC` |
| HG_MALLOC | `-a HG_MALLOC` |

## Setup

- **Server:** 16-vCPU KVM guest (Xeon E5-2699 v4), Ubuntu 24.04, gcc 13, kernel 6.8. virtio NIC with 16 queues, 8 MB UDP socket buffers, `maxbuffer` 8 MB, a hugetlb pool large enough for the caps, `LimitMEMLOCK` unlimited.
- **Load:** sipp, up to 60 instances split evenly over two other 16-vCPU hosts with 16-queue NICs on the same LAN, 1,000-2,000 calls per second each.
- **CPU:** user plus system time of every OpenSIPS process over a fixed window while the load runs.
- **Latency:** sipp's per-call response times, plus a separate probe sending 100 requests per second and timing each round trip in microseconds.
- **Drops:** the kernel's UDP receive-buffer errors on the server.

## Proxy load: INVITE answered by tm

Vanilla config: proto_udp, tm, sl, signaling, sipmsgops; 8 UDP workers; each INVITE answered 486 by tm, ACK absorbed. 60 s per point.

All four arms measured in one session, `-m 2048 -M 32`; HG_MALLOC elastic `-m 128:2048 -M 16:32`.

| calls/s | | F_MALLOC | F_PARALLEL_MALLOC | HP_MALLOC | HG_MALLOC |
|---|---|---|---|---|---|
| 20,000 | CPU, cores | 2.40 | 2.30 | 2.25 | **2.04** |
| | UDP drops | 0 | 0 | 0 | **0** |
| | INVITE p99 (probe) | 0.94 ms | 0.82 ms | 0.73 ms | **0.78 ms** |
| 45,000 | CPU, cores | 4.41 | 4.14 | 4.22 | **3.62** |
| | UDP drops | 31,030 | 167 | 19,342 | **0** |
| | INVITE p99 (probe) | 6.1 ms | 3.3 ms | 4.6 ms | **1.4 ms** |
| 60,000 | CPU, cores | 8.56 | 8.13 | 5.40 | **4.69** |
| | UDP drops | 7,701,938 | 3,609,825 | 247,051 | **0** |
| | INVITE p99 (probe) | 36 ms | 38 ms | 8.4 ms | **3.3 ms** |
| | failed calls | 508,165 | 122,206 | 0 | **0** |

Up to 45,000 every call completed in every arm. At 60,000 F_MALLOC ran out of the 2 GB shared memory 16 s into the run (2,121 MB used), and F_PARALLEL_MALLOC ran out in one of its pools after 11 s (1,747 MB used in total); tm could no longer clone requests, calls failed, sipp retransmitted, and all 8 workers saturated. Both also lost 17 probe INVITEs. HP_MALLOC and HG_MALLOC completed every call.

The elastic arena grew from 128 MB to 528 MB, 1,136 MB and 1,536 MB at the three rates. The longest arena-lock hold was 550 us, 1,363 us and 1,143 us; 1 hold at 45,000 and 4 at 60,000 went over the 1 ms stall threshold.

The 60,000 point again with `-m 4096 -M 32` (HG_MALLOC `-m 128:4096 -M 16:32`), so that no arm runs out of memory:

| calls/s | | F_MALLOC | F_PARALLEL_MALLOC | HP_MALLOC | HG_MALLOC |
|---|---|---|---|---|---|
| 60,000 | CPU, cores | 5.98 | 5.43 | 6.31 | **4.59** |
| | UDP drops | 1,119,613 | 4,554 | 363,419 | **0** |
| | INVITE p99 (probe) | 13.8 ms | 6.5 ms | 13.4 ms | **1.9 ms** |
| | failed calls | 4,412 | 0 | 0 | **0** |

No arm logged a memory error. F_MALLOC peaked at 2,467 MB, more than the 2 GB of the first run, and still failed 4,412 calls whose retransmissions were all dropped; HP_MALLOC's slowest probe INVITE took 1.3 s. HG_MALLOC grew to the same 1,536 MB as with the 2 GB cap, with no lock stall (longest hold 873 us); one client host retransmitted 8,485 times, and six probe INVITEs got their answer only from tm's reply retransmission 500 ms later, with no drop and no send error on the server.

## Registrar under a cold start: 1.2 million REGISTERs at 60,000 per second

usrloc without a database, registrar, 4 UDP workers. The elastic arena starts at 128 MB and has to grow to hold every registration.

| | F_MALLOC | F_PARALLEL_MALLOC | HP_MALLOC | HG_MALLOC `-m 128:4096` |
|---|---|---|---|---|
| shm size | `-m 4096` | `-m 4096` | `-m 4096` | starts at 128 MB |
| runs | 2 | 2 | 2 | 2 |
| registrations stored | 1,200,000 | 1,200,000 | 520,033 and 515,904 | **1,200,000** |
| UDP receive drops | 2.22 M and 1.92 M | 1.65 M and 1.30 M | 6.90 M and 6.97 M | **0 in every run** |
| sipp retransmissions | 2.34 M and 2.00 M | 1.69 M and 1.33 M | 8.65 M and 8.71 M | **0 in every run** |
| response p50 | 596 ms and 588 ms | 580 ms and 84 ms | 11.9 s and 12.0 s | **under 1 ms** |
| response p99 | 19.6 s and 19.5 s | 15.6 s and 15.5 s | 28.0 s | **4 ms** |
| worst response | 27.5 s | 27.5 s and 23.5 s | 28.0 s | **8-12 ms** |
| probe p99.9 | 87-90 ms | 73-74 ms | no reply (stopped after 20 timeouts in a row) | **5.6-6.0 ms** |
| CPU, cores | 4.00 (saturated) | 3.84-3.97 | 4.00 (saturated) | **1.90-2.05** |

With HP_MALLOC more than half the registrations failed: the 4 workers were saturated, and a profile of this test put 81% of their CPU inside `hp_shm_malloc`.

In both HG runs the arena grew from 128 MB to 1,696 MB in 98 steps with no lock stalls; the longest lock hold was 334 us and 971 us, and each grow's commit (up to 7.1 ms) ran with the lock released.

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
| OpenSIPS CI on the PR (all compilers, architectures, unit tests, fuzzing) | passes |
