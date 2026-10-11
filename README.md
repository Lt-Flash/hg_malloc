# HG_MALLOC

A memory allocator for [OpenSIPS](https://github.com/OpenSIPS/opensips), for both shared and private memory, selected at runtime with `-a HG_MALLOC`. Built for busy SIP proxies, registrars and SBCs, where the allocator sits on every message.

## Why

- **Less CPU per call:** 10-30% less than F_MALLOC, F_PARALLEL_MALLOC and HP_MALLOC for the whole server at 20k-60k calls per second.
- **No drops under load:** 0 UDP drops at 60k cps, where every other allocator drops 160,000 or more; the workers keep the socket drained.
- **Tight latency:** 1.2 million registrations at 60k per second with a p99 of 4 ms, against 11-28 s for the others.
- **No more guessing `-m`:** `-m 128:1024` starts at 128 MB, grows as traffic arrives, and gives memory back when it goes quiet.
- **Safe and observable:** double frees that reach the shared pool and corrupted free-list links are caught and counted instead of followed; every grow, shrink and lock hold is visible through MI, statistics and events.
- **Runs anywhere:** huge pages when the host has them, plain 4 KB pages when it does not; builds on x86 and ARM, 32-bit and 64-bit, glibc and musl.

Measured on vanilla OpenSIPS, every allocator from the same binary - see [benchmarks](docs/benchmarks.md). In production on 11 SIP nodes (SBCs and registrars) since October 2026.

## How it works

- **Lock-free common path.** 21 size classes from 64 B to 64 KB, served from a private free list per process: an allocation and the free that returns it take no lock and touch no shared cache line. Cells move between processes through a shared per-class pool, in batches.
- **Memory is reused, not stranded.** Blocks are carved from a buddy allocator inside each huge page (2 MB on x86-64); a block whose cells are all free returns to the buddy and can be re-cut for any size class.
- **Huge pages when available.** The arena is acquired through a four-step ladder - `MAP_HUGETLB`, transparent huge pages by advice, by collapse, then plain 4 KB pages - each tier verified from the kernel's own counters.
- **Elastic arenas.** `-m INIT:CAP` reserves the cap and commits pages only as needed; a maintenance process grows ahead of demand and releases quiet pages with the arena lock dropped. Growth stops at the cap, at what the backing tier can give, and at a floor of host free memory - and when the backing or host memory keeps refusing, an alertable event is raised.

The full design, with measurements behind each decision: [docs/design.md](docs/design.md).

## Code

- **Branch:** [`feature/hg-malloc-v3-final`](https://github.com/Lt-Flash/opensips/tree/feature/hg-malloc-v3-final) on the Lt-Flash/opensips fork - current OpenSIPS master plus one commit; clone and build it as any OpenSIPS tree. Proposed upstream as [OpenSIPS/opensips#4327](https://github.com/OpenSIPS/opensips/pull/4327).
- **Patch:** [`patches/`](patches) - the same commit as `git format-patch` output, against OpenSIPS master [`6832fe4e66`](https://github.com/OpenSIPS/opensips/commit/6832fe4e66): the allocator (`mem/hg_*`), elastic-arena and autoscaling configuration, MI commands, statistics and events, and `modules/hgstress`, a soak and fault-injection test module excluded from the default build.

```bash
git clone https://github.com/OpenSIPS/opensips && cd opensips
git checkout 6832fe4e66
git am /path/to/hg_malloc/patches/0001-*.patch
```

## Quick start

Build: `HG_MALLOC` is in the `DEFS` of `Makefile.conf.template`; it is compiled in next to the other allocators and only used when selected.

```bash
# fixed 512 MB shared arena, 16 MB private arena per process
opensips -f opensips.cfg -a HG_MALLOC -m 512 -M 16

# elastic: start at 128 MB, allowed to grow to 1 GB; private arenas 16 -> 32 MB
opensips -f opensips.cfg -a HG_MALLOC -m 128:1024 -M 16:32
```

The service needs an unlimited memory-lock limit, because the committed arena is pinned and every grow is an `mlock()`:

```ini
# systemd unit
[Service]
LimitMEMLOCK=infinity
```

HG warns at startup, and under `opensips -C`, when the limit cannot cover the arena's growth.

**Enable huge pages on the host first.** HG uses only what the host offers. With no hugetlb pool and transparent huge pages turned off, every arena drops through the ladder to plain 4 KB pages. It still works, but without the huge-page gain. Configure one or both:

```bash
# Tier 1 - hugetlb pool: shm cap + pkg cap x processes, in 2 MB pages, ~12% margin.
# e.g. -m 128:1024 -M 16:32 with 20 processes: (1024 + 32*20) / 2 = 832 pages, +12% = 932
sysctl -w vm.nr_hugepages=932            # persist in /etc/sysctl.d/; reserve early, before memory fragments

# Tiers 2-3 - transparent huge pages, used when there is no pool or it is too small
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled         # private arenas (-M)
echo advise  > /sys/kernel/mm/transparent_hugepage/shmem_enabled   # shared arena (-m)
```

Many distributions ship `shmem_enabled` as `never`, which puts the shared arena on 4 KB pages even when THP is on for private memory. `MADV_COLLAPSE` (tier 3) needs Linux 6.1 or later. The startup log names the tier each arena got, and, when THP is off by a host setting, why it fell to 4 KB pages; `hg_stats` shows the split in `tier_bytes` when growth landed on more than one tier.

Grow ahead of demand and give memory back with a profile - [configuration](docs/configuration.md):

```
auto_scaling_profile = MEM_SHM
    scale up to 1024 on 80% for 3 cycles within 10
    scale down to 256 on 30% for 120 cycles

shm_auto_scaling_profile = MEM_SHM
```

## Monitoring

```bash
opensips-cli -x mi core:hg_stats         # per arena: tier, committed/reserved, grows, lock holds, corruption
opensips-cli -x mi core:hg_advise        # what -m and -M should be, from the peaks actually reached
opensips-cli -x mi statistics:get hgmem:
```

Events: `E_CORE_SHM_GROW_BLOCKED` (growth refused for a sustained period), `E_CORE_HG_LOCK_STALL` (an arena-lock hold above `hg_lock_stall_us`). Parameters, examples and what to alert on: [configuration](docs/configuration.md#events).

## Costs

- Memory: about the same as F_MALLOC for registrations (+2% live memory, measured on 400k contacts); more for workloads dominated by 1-4 KB objects, from size-class rounding.
- The committed arena is pinned, so it shows as resident from the start.
- The huge-page tiers are Linux-only; elsewhere the arena is plain pinned memory.

## Status

Not yet in upstream OpenSIPS. Proposed as [OpenSIPS/opensips#4327](https://github.com/OpenSIPS/opensips/pull/4327); available as a branch and as a patch (see [Code](#code)). Licensed under the GNU GPL v2 or later, like OpenSIPS.
