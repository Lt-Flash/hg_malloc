# HG_MALLOC design

How HG_MALLOC works, what it guarantees, and how to run and monitor it. For the configuration reference see [configuration.md](configuration.md); for measurements, [benchmarks.md](benchmarks.md).

## 1. Allocation

**Size classes.** Requests up to 64 KB (including an 8-byte cell header) are served from 21 fixed size classes, 64 B to 64 KB. Every cell carries a small hidden header with its class, so a free needs no lookup. Larger requests go to a coalescing large tier that packs into whole pages.

**The common path takes no lock.** Each process (and each thread of a threaded process) keeps a private free list per class. An allocation pops from it and a free pushes to it - no lock, no shared cache line. When a private list runs dry it refills a batch from a shared per-class pool; when it grows past its threshold it donates a batch back. Only those batch transfers take the arena lock.

**Blocks and the buddy allocator.** The arena is a grid of huge pages - 2 MB on x86-64, larger on arm64 kernels with 16 KB or 64 KB base pages. A buddy allocator inside each page hands out blocks, and a block is carved into cells of one class. The shared pool keeps each block's free cells on that block and refills from the fullest partially-used block, so emptier blocks drain completely. A block whose cells are all free goes back to the buddy and can be re-cut for any class - memory used by a burst of one size is not stranded there.

**Small arenas.** An arena with an INIT under one huge page settles its page size by the backing it gets. On plain 4 KB memory it runs in small mode, on 256 KB pages: at least 256 KB with a cap, 512 KB without. On huge pages its INIT is raised to one huge page, and an arena whose cap leaves no room above that page is refused at startup.

**Idle caches are swept.** Every 30 s each process flushes what its private lists hold back to the shared pool (staggered, a process or two per second), keeping a small residue for the classes it is actively using. Without this, a worker that went idle would keep its cached cells forever.

**Integrity checks at the pool boundary.** Each block keeps a map of which of its cells are in the shared pool. A cell that reaches the pool twice is an exact double free: the second copy is refused and counted. Every link the pool follows is checked against the map, so a scribbled free-list link stops the walk instead of leading it off the arena. Both are counted in `hg_stats` (`corruption`) and logged at CRITICAL on each detection; the process keeps running. Detection happens when the second copy reaches the shared pool: a cell freed twice and taken again by the same process before its next flush is not caught.

## 2. Backing memory

The arena is acquired through four tiers, best first, each verified from the kernel's own counters rather than assumed:

| tier | mechanism | notes |
|---|---|---|
| 1 | `mmap(MAP_HUGETLB)` | huge pages from the hugetlb pool; unswappable |
| 2 | THP: `MADV_HUGEPAGE` before first touch | huge pages at fault, from general memory |
| 3 | THP: `MADV_COLLAPSE` after fill | pages collapsed into huge pages afterwards |
| 4 | plain 4 KB pages | always available |

Tiers 2-4 pin the committed arena with `mlock()`. The startup log names the tier achieved, and says why when THP is turned off by a host setting. Huge pages make the allocator's own work cheaper but are not required; an arena on 4 KB pages behaves the same.

**Each growth step negotiates its own backing.** A step may land on a different tier than the memory before it, so `hg_stats` reports `tier_bytes` - the split by tier - whenever more than one tier holds bytes.

**Tier 1 reserves the whole cap from the pool when the arena is created.** Growth on tier 1 therefore cannot fail half-way, and the pool must be sized for the caps (section 7). If the pool can hold the initial size but not the cap, the arena keeps huge pages at its initial size, cannot grow, and says so at startup. If it cannot hold even the initial size, the arena starts on THP with its full cap and stays elastic.

## 3. Elastic arenas

`-m INIT:CAP` (and `-M INIT:CAP` for the private arenas) makes an arena elastic. Without a cap it is fixed at its size. For the smallest INIT see Small arenas (section 1).

**One mapping, created before fork.** The whole cap is mapped once, shared, before any worker forks. Growth and shrink never create or change mappings - they populate and release ranges of that one shared object, which every process sees immediately. An untouched reserved range costs only page-table entries.

**Committed and reserved.** `committed` is what is backed, pinned and usable; `cap` is the size of the reservation. The buddy's page descriptors are laid out for the whole cap at startup, so a grow only publishes pages - it never needs to find room for metadata in an arena that is full.

### 3.1 Growth

Three triggers, one mechanism:

- **Exhaustion** - always armed once a cap exists. An allocation that finds no memory grows the arena and retries once.
- **Headroom** (`hg_grow_ahead`, default on) - when the free part of the arena drops below twice its reserve floor, one step is grown ahead of demand.
- **Profile** - with an `auto_scaling_profile`, usage is sampled once per cycle; at the end of each block of W cycles, if it was at or above the up-threshold in at least C of them, one step is grown before anything fails.

**A grow is a three-phase commit.** The step is reserved under the arena lock; the expensive part - faulting the pages in, pinning them, verifying their backing - runs with the lock released; the new pages are published under the lock again, in constant time. Other allocations keep running throughout. One grow is in flight at a time: a process that runs out while it is in progress waits for the publish without holding the lock (up to 2 s, then the request fails and `grow_wait_timeouts` counts it).

**The maintenance process.** Elastic shared arenas (the shm arena and module arenas) get a dedicated core process, `HG maintenance`. It serves no requests; every second it applies the headroom rule, every `hg_scaling_cycle` seconds it evaluates the profile and the shrink policy, and it does the populate of any step it grows itself - so no SIP worker waits on a proactive grow. Private arenas have no maintenance process: each process runs its own arena's policy in its 30 s cache sweep.

**Step size.** `shm_grow_granule` / `pkg_grow_granule`, default 16 MB, rounded to the arena page. Smaller steps track demand more closely and make each commit shorter.

### 3.2 Limits

Growth stops at the smallest of three limits:

| limit | what | refusal |
|---|---|---|
| admin | the profile's scale-up target, else the cap | logged as a notice - a limit doing its job |
| backing | the hugetlb pool (tier 1, checked at startup), or `mlock()` | resource refusal |
| host memory | `MemAvailable` must stay above `hg_ram_floor_mb` (default max(256 MB, MemTotal/20)) | resource refusal |

The host-memory check charges a private-arena step once per OpenSIPS process, since every process grows its own under the same load. Tier 1 skips it: hugetlb pages were taken from host memory when the pool was created.

The same floor guards the INITs: at startup every process's private arena is checked against the pool and RAM before anything forks, and each later fork - the auto-scaler's included - is skipped if its INIT does not fit (see [Host memory checks](configuration.md#host-memory-checks)).

Every grow is an `mlock()` of the new range, and `mlock()` checks `RLIMIT_MEMLOCK` on hugetlb mappings too. An elastic arena warns at startup when the limit cannot cover its growth, and a refused grow names the resource that ran out - the limit, or the hugetlb pool. On tiers 2-4, an arena that could not pin its initial size runs unpinned, and its grows are populated without pinning instead of refused (section 8).

### 3.3 GROW-BLOCKED

Only resource refusals count: the memory-lock limit, the hugetlb pool, or the host floor. Reaching the cap or the profile's scale-up target is a limit doing its job: it is logged once as a notice and counted in `grow_refused`, and never latches.

A resource refusal that persists is an incident; a single one is not. The first refusal arms the state. It latches at the next 30 s sweep tick if refusals continued, or sooner if a reclaim pass did not help. It clears on the next successful grow, or when free space recovers above twice the reserve floor. Isolated refusals never latch.

When latched: the `hg_shm_grow_blocked` statistic is 1, one warning is logged, and the `E_CORE_SHM_GROW_BLOCKED` event is raised at the next sweep tick - from the sweep timer, never under the arena lock - with `arena`, `committed_mb`, `cap_mb` and `grow_refused`. While it lasts the event is raised again periodically.

```
event_route[E_CORE_SHM_GROW_BLOCKED] {
    xlog("L_CRIT", "shm arena cannot grow: $param(committed_mb) of $param(cap_mb) MB committed, $param(grow_refused) refusals\n");
}
```

Refusal details are logged once per episode; `grow_refused` carries the count.

### 3.4 Shrink

**Only whole free pages are released.** A page is released only when the buddy has merged it back into one free top-level block - which means no cell in it is in use or sitting in any process's private cache. No coordination with other processes is needed: nothing can hold a pointer into it.

**The drain.** The shrink decision is made per policy cycle. With a profile, the arena starts draining after usage has stayed at or below the down-threshold for the profile's C consecutive cycles; a profile without a `scale down` clause never shrinks. Without a profile, an elastic arena drains after 4 quiet sweeps (2 minutes) in which its free space exceeds one grow step plus four times the reserve floor. A shared arena then releases up to `hg_shrink_step` (default 64 MB) per maintenance tick from the top of the arena, and with `hg_interior_release = 1` up to another `hg_shrink_step` from inside it; a private arena releases one step per quiet window. The gates are re-checked every tick, and the drain stops at the headroom line, so a release never triggers the grow that would undo it. Any grow ends the drain, and shrink counting resumes only after a cool-off of 10 x C cycles (C = the profile's scale-down cycle count).

**The release.** Pages leave the free list first, under the lock; the release itself (`MADV_REMOVE` on the shared arena, `MADV_DONTNEED` on a private one) runs with the lock dropped; if the kernel refuses, the pages go straight back. On tier 1 released pages return to the hugetlb pool. A kernel that does not support the advice leaves the arena grown and is noted once; no further release is attempted.

## 4. Private (pkg) arenas

- Each process creates its own private arena at fork, after the configuration is parsed, so `-M INIT:CAP` and `pkg_auto_scaling_profile` apply to every process individually. The pre-fork parent arena never uses hugetlb, so no child can fault on the pool through a copy-on-write page, and never gets the profile; it keeps the `-M` cap and can still grow on exhaustion.
- A process grows and shrinks only its own arena; its policy runs in its own 30 s sweep, whatever `hg_scaling_cycle` is set to.
- Host cost multiplies with the process count - and on tier 1, so does the pool reservation, since each process's arena reserves its own cap.

## 5. Module arenas

A module can ask for its own HG-managed arena with `mem/mem_arena.h` - for a cache whose size and lifetime have nothing to do with transactions. It gets the same size classes, caches, reclaim and elastic growth, independent of the core `-a` choice, and appears in `core:hg_stats` (not in the `hgmem:` statistics, which describe the core shm arena). When HG_MALLOC is not built, the API compiles to stubs that return NULL, so the module must fall back to ordinary shm itself.

## 6. Monitoring

**MI:** `core:hg_stats` returns, per arena (the shared arena, every module arena, and the calling process's private one):

| field | meaning |
|---|---|
| `tier`, `tier_bytes` | backing achieved at startup; per-tier split when growth mixed tiers |
| `committed`, `cap`, `grow_headroom` | elastic state |
| `grows`, `grows_proactive`, `grows_exhaustion`, `grow_refused`, `grow_blocked` | growth history and the latched gauge |
| `shrinks`, `punches`, `draining`, `whole_free_pages` | release history and what could be released now |
| `live_committed`, per-class carve and reuse | what is in use and where |
| `lock` | hold and wait histograms per reason (refill, return, flush, large, policy, stats, other), stalls, the worst hold and its reason, commit and release times |
| `corruption` | double frees, foreign pointers, bad classes, counted per kind |

**Statistics:** `hg_shm_*` counters in the `hgmem:` group (growth, refusals, the GROW-BLOCKED gauge), suitable for Prometheus.

**Events:** `E_CORE_SHM_GROW_BLOCKED` (section 3.3) and `E_CORE_HG_LOCK_STALL`, raised for a shared-arena lock hold at or above `hg_lock_stall_us` (default 1000 us), with `reason`, `hold_us`, `process` and `stalls`.

**Note on `shmem:free_size`.** It counts only memory not yet carved into size-class blocks. Free cells already carved - in private caches or the shared pool - are reusable for their class but not counted as free, so free memory reads lower than what is reusable. Alert on `grow_blocked` and on the lock statistics, not on `free_size`.

## 7. Sizing

1. **Memory lock limit:** `LimitMEMLOCK=infinity` in the service unit (or `ulimit -l unlimited`). Required on every tier for an elastic arena.
2. **Tier 1 pool:** reserve `shm_cap + pkg_cap x processes`, in 2 MB pages, plus a margin of about 12%. A pool that cannot fit the caps gives fixed arenas on huge pages, or pushes late processes to THP.
   **Tiers 2-3 instead of a pool:** THP `enabled` = `madvise` or `always` (private arenas) and `shmem_enabled` = `advise` or `always` (shared arena). With neither a pool nor THP, every arena runs on 4 KB pages.
3. **Private caps multiply:** `-M 16:64` on 30 processes is up to 1.9 GB of potential pinned growth.
4. **Tier 1 releases go back to the pool,** not to general memory; shrink `vm.nr_hugepages` to give them to the host.
5. **Measuring the pool:** `HugePages_Free` alone also moves with private hugetlb use; the shared arena's pages are `(HugePages_Total - HugePages_Free) - sum of Private_Hugetlb over all processes`.
6. **Memory per object:** about the same as F_MALLOC for registrations (+2% live memory, measured on 400k contacts); more for workloads dominated by 1-4 KB objects, from size-class rounding. `used_size` counts each object at its class size, so it reads higher than the bytes requested.

## 8. Troubleshooting

| symptom | cause | fix |
|---|---|---|
| startup: `RLIMIT_MEMLOCK is N KB, but growing needs M KB locked` | memory lock limit too low for the arena's growth | `LimitMEMLOCK=infinity` in the unit |
| `cannot grow ...: RLIMIT_MEMLOCK is N KB` | the same, at grow time | the same |
| `hugetlb pool cannot back a ... cap` at startup | `vm.nr_hugepages` smaller than the caps | grow the pool (section 7), restart |
| startup: `below the 512 KB minimum viable arena` | a fixed arena (no cap) under 512 KB | give a cap (`-M 256k:2m`) or INIT of at least 512 KB |
| startup: `on huge pages INIT is at least one huge page ... no room to grow` | INIT under 2 MB on huge pages with a cap of 2 MB or less | raise the cap, or give INIT as 2 MB for a fixed arena |
| notice: `INIT raised from ... to 2 MB` | INIT under 2 MB on huge pages | expected; give INIT as 2 MB to silence it |
| startup: `the N processes started now need ... MB of RAM` | `-M` INIT x processes does not fit in the pool plus RAM above the floor | lower `-M` INIT or the worker counts, or add memory or huge pages |
| `not forking more processes until memory frees up` | the auto-scaler wants a process whose INIT does not fit | lower the scale-up maximum or `-M` INIT; OpenSIPS keeps running meanwhile |
| `the arena has no growth room` | a profile is attached but `-m`/`-M` has no cap | add the cap: `-m 128:1024` |
| `scale-up target ... exceeds the ... reservation` | the profile wants more than the cap | raise the cap or lower the target |
| arena runs unpinned (`continuing unpinned`) | memory lock limit too low on tiers 2-4 | `LimitMEMLOCK=infinity`; meanwhile the arena grows unpinned rather than refusing |
| grown pages on 4 KB on a THP host | each step negotiates its own backing | expected; see `tier_bytes`, or use tier 1 |
| never shrinks | usage above the down-threshold, inside the post-grow cool-off, a profile without `scale down`, or the top page in use | check `hg_stats`; enable `hg_interior_release` to release from inside the arena |
| `DRY RUN - would grow` and allocations fail past the initial size | `hg_autoscale_dry_run = 1` blocks all growth | set 0 to act |
| testing under `ulimit -l` shows no refusals | running as root: `CAP_IPC_LOCK` lifts the limit | test as an unprivileged user (`setpriv`) |

## 9. Testing

`modules/hgstress` (excluded from the default build; `include_modules= hgstress`) is a multi-process soak: every worker churns stamped cells across all classes and the large tier, verifies the stamps while the others churn, and reports `PASS`/`FAIL` per worker. Its MI commands park and release stamped memory (`hgstress:hgs_hold`, `hgstress:hgs_release`, to drive grow and shrink) and inject a double free or a scribbled link (`hgstress:hgs_fault`, to exercise the integrity checks). It runs under any `-a` allocator.
