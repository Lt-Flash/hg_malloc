# HG_MALLOC design

How HG_MALLOC works, what it guarantees, and how to run and monitor it. For the configuration reference see [configuration.md](configuration.md); for measurements, [benchmarks.md](benchmarks.md).

## 1. Allocation

**Size classes.** Requests up to 64 KB are served from 21 fixed size classes, 64 B to 64 KB. Every cell carries a small hidden header with its class, so a free needs no lookup. Larger requests go to a coalescing large tier that packs into whole pages.

**The common path takes no lock.** Each process (and each thread of a threaded process) keeps a private free list per class. An allocation pops from it and a free pushes to it - no lock, no shared cache line. When a private list runs dry it refills a batch from a shared per-class pool; when it grows past its threshold it donates a batch back. Only those batch transfers take the arena lock.

**Blocks and the buddy allocator.** The arena is a grid of 2 MB pages. A buddy allocator inside each page hands out blocks, and a block is carved into cells of one class. The shared pool keeps each block's free cells on that block and refills from the fullest partially-used block, so emptier blocks drain completely. A block whose cells are all free goes back to the buddy and can be re-cut for any class - memory used by a burst of one size is not stranded there.

**Idle caches are swept.** Every 30 s each process flushes what its private lists hold back to the shared pool (staggered, a process or two per second), keeping a small residue for the classes it is actively using. Without this, a worker that went idle would keep its cached cells forever.

**Integrity checks at the pool boundary.** Each block keeps a map of which of its cells are in the shared pool. A cell that reaches the pool twice is an exact double free: the second copy is refused and counted. Every link the pool follows is checked against the map, so a scribbled free-list link stops the walk instead of leading it off the arena. Both are counted in `hg_stats` (`corruption`) and logged once; the process keeps running.

## 2. Backing memory

The arena is acquired through four tiers, best first, each verified from the kernel's own counters rather than assumed:

| tier | mechanism | notes |
|---|---|---|
| 1 | `mmap(MAP_HUGETLB)` | huge pages from the hugetlb pool; unswappable |
| 2 | THP: `MADV_HUGEPAGE` before first touch | huge pages at fault, from general memory |
| 3 | THP: `MADV_COLLAPSE` after fill | pages collapsed into huge pages afterwards |
| 4 | plain 4 KB pages | always available |

Tiers 2-4 pin the committed arena with `mlock()`. The startup log names the tier achieved, and why when it is 4 KB. Huge pages make the allocator's own work cheaper but are not required; an arena on 4 KB pages behaves the same.

**Each growth step negotiates its own backing.** A step may land on a different tier than the memory before it, so `hg_stats` reports `tier_bytes` - the split by tier - whenever more than one tier holds bytes.

**Tier 1 reserves the whole cap from the pool when the arena is created.** Growth on tier 1 therefore cannot fail half-way, and the pool must be sized for the caps (section 7). If the pool cannot hold the cap, the arena keeps huge pages at its initial size, cannot grow, and says so at startup.

## 3. Elastic arenas

`-m INIT:CAP` (and `-M INIT:CAP` for the private arenas) makes an arena elastic. Without a cap it is fixed at its size.

**One mapping, created before fork.** The whole cap is mapped once, shared, before any worker forks. Growth and shrink never create or change mappings - they populate and release ranges of that one shared object, which every process sees immediately. An untouched reserved range costs only page-table entries.

**Committed and reserved.** `committed` is what is backed, pinned and usable; `cap` is the size of the reservation. The buddy's page descriptors are laid out for the whole cap at startup, so a grow only publishes pages - it never needs to find room for metadata in an arena that is full.

### 3.1 Growth

Three triggers, one mechanism:

- **Exhaustion** - always armed once a cap exists. An allocation that finds no memory grows the arena and retries once.
- **Headroom** (`hg_grow_ahead`, default on) - when the free part of the arena drops below twice its reserve floor, one step is grown ahead of demand.
- **Profile** - with an `auto_scaling_profile`, live usage above the profile's threshold for the configured number of cycles grows one step before anything fails.

**A grow is a two-phase commit.** The step is reserved under the arena lock; the expensive part - faulting the pages in, pinning them, verifying their backing - runs with the lock released; the new pages are published under the lock again, in constant time. Other allocations keep running throughout. One grow is in flight at a time: a process that runs out while it is in progress waits for the publish without holding the lock (up to 2 s, then the request fails and `grow_wait_timeouts` counts it).

**The maintenance process.** An elastic shared arena gets a dedicated core process, `HG maintenance`. It serves no requests; every second it applies the headroom rule, every `hg_scaling_cycle` seconds it evaluates the profile and the shrink policy, and it does the populate of any step it grows itself - so no SIP worker waits on a proactive grow.

**Step size.** `shm_grow_granule` / `pkg_grow_granule`, default 16 MB, rounded to the arena page. Smaller steps track demand more closely and make each commit shorter.

### 3.2 Limits

Growth stops at the smallest of three limits:

| limit | what | refusal |
|---|---|---|
| admin | the profile's scale-up target, else the cap | logged as a notice - a limit doing its job |
| backing | the hugetlb pool (tier 1, checked at startup), or `mlock()` | resource refusal |
| host memory | `MemAvailable` must stay above `hg_ram_floor_mb` (default max(256 MB, MemTotal/20)) | resource refusal |

The host-memory check charges a private-arena step once per worker process, since every worker grows its own under the same load. Tier 1 skips it: hugetlb pages were taken from host memory when the pool was created.

Every grow is an `mlock()` of the new range, and `mlock()` checks `RLIMIT_MEMLOCK` on hugetlb mappings too. An elastic arena warns at startup when the limit cannot cover its growth, and a refused grow names the resource that ran out - the limit, or the hugetlb pool.

### 3.3 GROW-BLOCKED

A resource refusal that persists is an incident; a single one is not. A refusal arms the state; it latches when refusals are still accumulating a full sweep interval later (or after a reclaim pass did not help), and clears on the next successful grow. Isolated refusals never latch.

When latched: the `hg_shm_grow_blocked` statistic is 1, one warning is logged, and the `E_CORE_SHM_GROW_BLOCKED` event is raised - from the sweep timer, never under the arena lock - with `arena`, `committed_mb`, `cap_mb` and `grow_refused`.

```
event_route[E_CORE_SHM_GROW_BLOCKED] {
    xlog("L_CRIT", "shm arena cannot grow: $param(committed_mb) of $param(cap_mb) MB committed, $param(grow_refused) refusals\n");
}
```

Refusal details are logged once per episode; `grow_refused` carries the count.

### 3.4 Shrink

**Only whole free pages are released.** A page is released only when the buddy has merged it back into one free top-level block - which means no cell in it is in use or sitting in any process's private cache. No coordination with other processes is needed: nothing can hold a pointer into it.

**The drain.** The shrink decision is made per policy cycle: after the profile's quiet window (live usage at or below its down-threshold for the configured cycles), the arena starts draining, releasing up to `hg_shrink_step` (default 64 MB) per maintenance tick, first from the top of the arena and, with `hg_interior_release = 1`, also from anywhere inside it. The gates are re-checked every tick, and the drain stops at the headroom line, so a release never triggers the grow that would undo it. Any grow ends the drain, and shrink counting resumes only after a cool-off of ten profile cycles.

**The release.** Pages leave the free list first, under the lock; the release itself (`MADV_REMOVE` on the shared arena, `MADV_DONTNEED` on a private one) runs with the lock dropped; if the kernel refuses, the pages go straight back. On tier 1 released pages return to the hugetlb pool. A kernel that does not support the advice leaves the arena grown and is noted once; no further release is attempted.

## 4. Private (pkg) arenas

- Each worker creates its own private arena at fork, after the configuration is parsed, so `-M INIT:CAP` and `pkg_auto_scaling_profile` apply to every worker individually. The pre-fork parent arena stays fixed at the initial size and never uses hugetlb, so no child can fault on the pool through a copy-on-write page.
- A worker grows and shrinks only its own arena; its policy runs in its own sweep.
- Host cost multiplies with the worker count - and on tier 1, so does the pool reservation, since each worker's arena reserves its own cap.

## 5. Module arenas

A module can ask for its own HG-managed arena with `mem/mem_arena.h` - for a cache whose size and lifetime have nothing to do with transactions. It gets the same size classes, caches, reclaim, elastic growth and statistics, independent of the core `-a` choice. When HG_MALLOC is not built, the API compiles to stubs and the module falls back to ordinary shm.

## 6. Monitoring

**MI:** `core:hg_stats` returns, per arena (the shared arena and the calling process's private one):

| field | meaning |
|---|---|
| `tier`, `tier_bytes` | backing achieved at startup; per-tier split when growth mixed tiers |
| `committed`, `cap`, `grow_headroom` | elastic state |
| `grows`, `grows_proactive`, `grows_exhaustion`, `grow_refused`, `grow_blocked` | growth history and the latched gauge |
| `shrinks`, `punches`, `draining`, `whole_free_pages` | release history and what could be released now |
| `live_committed`, per-class carve and reuse | what is in use and where |
| `lock` | hold and wait histograms per reason (refill, return, flush, large, policy, stats), stalls, the worst hold and its reason, commit and release times |
| `corruption` | double frees, foreign pointers, bad classes, counted per kind |

**Statistics:** `hg_shm_*` counters next to `shmem:` (growth, refusals, the GROW-BLOCKED gauge), suitable for Prometheus.

**Events:** `E_CORE_SHM_GROW_BLOCKED` (section 3.3) and `E_CORE_HG_LOCK_STALL`, raised for an arena-lock hold at or above `hg_lock_stall_us` (default 1000 us), with `reason`, `hold_us`, `process` and `stalls`.

**Note on `shmem:free_size`.** Cells cached in private lists count as used, so free memory reads lower than what is reusable. Alert on `grow_blocked` and on the lock statistics, not on `free_size`.

## 7. Sizing

1. **Memory lock limit:** `LimitMEMLOCK=infinity` in the service unit (or `ulimit -l unlimited`). Required on every tier for an elastic arena.
2. **Tier 1 pool:** reserve `shm_cap + pkg_cap x workers`, in 2 MB pages, plus a margin of about 12%. A pool that cannot fit the caps gives fixed arenas on huge pages, or pushes late workers to THP.
3. **Private caps multiply:** `-M 16:64` on 30 workers is up to 1.9 GB of potential pinned growth.
4. **Tier 1 releases go back to the pool,** not to general memory; shrink `vm.nr_hugepages` to give them to the host.
5. **Measuring the pool:** `HugePages_Free` alone also moves with private hugetlb use; the shared arena's pages are `(HugePages_Total - HugePages_Free) - sum of Private_Hugetlb over all processes`.
6. **Memory per object:** size-class rounding costs about 30% more shared memory per stored object than F_MALLOC.

## 8. Troubleshooting

| symptom | cause | fix |
|---|---|---|
| startup: `RLIMIT_MEMLOCK is N KB, but growing needs M KB locked` | memory lock limit too low for the arena's growth | `LimitMEMLOCK=infinity` in the unit |
| `cannot grow ...: RLIMIT_MEMLOCK is N KB` | the same, at grow time | the same |
| `hugetlb pool cannot back a ... cap` at startup | `vm.nr_hugepages` smaller than the caps | grow the pool (section 7), restart |
| `the arena has no growth room` | a profile is attached but `-m`/`-M` has no cap | add the cap: `-m 128:1024` |
| `scale-up target ... exceeds the ... reservation` | the profile wants more than the cap | raise the cap or lower the target |
| arena runs unpinned (`continuing unpinned`) | memory lock limit too low on tiers 2-4 | `LimitMEMLOCK=infinity`; meanwhile the arena grows unpinned rather than refusing |
| grown pages on 4 KB on a THP host | each step negotiates its own backing | expected; see `tier_bytes`, or use tier 1 |
| never shrinks | usage above the down-threshold, inside the post-grow cool-off, or the top page in use | check `hg_stats`; enable `hg_interior_release` to release from inside the arena |
| `DRY RUN - would grow` and nothing happens | `hg_autoscale_dry_run = 1` | set 0 to act |
| testing under `ulimit -l` shows no refusals | running as root: `CAP_IPC_LOCK` lifts the limit | test as an unprivileged user (`setpriv`) |

## 9. Testing

`modules/hgstress` (excluded from the default build; `include_modules= hgstress`) is a multi-process soak: every worker churns stamped cells across all classes and the large tier, verifies the stamps while the others churn, and reports `PASS`/`FAIL` per worker. Its MI commands park and release stamped memory (`hgstress:hgs_hold`, `hgstress:hgs_release`, to drive grow and shrink) and inject a double free or a scribbled link (`hgstress:hgs_fault`, to exercise the integrity checks). It runs under any `-a` allocator.
