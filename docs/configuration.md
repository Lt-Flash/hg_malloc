# Configuration

## Command line

| option | meaning |
|---|---|
| `-a HG_MALLOC` | use HG_MALLOC for shared and private memory (`-s` / `-k` select them separately) |
| `-m INIT[:CAP]` | shared arena: initial size, and the cap it may grow to |
| `-M INIT[:CAP]` | private (per-process) arena, same form |

Sizes are MB, or take a `k`/`m`/`g` suffix: `-m 128:1g`, `-M 512k:16m`. Without a cap the arena is fixed. With a cap, an allocation that would fail commits one more grow step and retries, up to the cap. The cap is on the command line because the reservation is made before the configuration is parsed.

## Memory autoscaling profiles

The worker autoscaler's `auto_scaling_profile` grammar also sizes memory arenas. For an arena the numbers are MB (or k/m/g):

```
auto_scaling_profile = MEM_SHM
    scale up to 1024 on 80% for 3 cycles within 10
    scale down to 256 on 30% for 120 cycles

auto_scaling_profile = MEM_PKG
    scale up to 64 on 80% for 2 cycles within 5
    scale down to 8 on 20% for 60 cycles

shm_auto_scaling_profile = MEM_SHM
pkg_auto_scaling_profile = MEM_PKG
```

| element | meaning for an arena |
|---|---|
| `up to N` | growth ceiling, within the `-m`/`-M` cap |
| `on P% for C cycles within W` | grow when live usage is at or above P% of the committed arena in C of the last W cycles |
| `down to M` | shrink floor - may be below the initial size |
| `on Q% for C cycles` | release memory after C consecutive cycles at or below Q% |

Usage is live usage - what is handed out - not the carved footprint. A pkg profile governs every worker's private arena separately, so its numbers are per-worker. A profile written with size suffixes, or with a target of 1000 or more, is refused if it is attached to a process group, so a memory profile cannot be mistaken for a worker count.

Choosing the numbers: `P` is the occupancy you want the arena to run at (70-80% is the usual shape) and `Q` should sit well below it (20-40%), so the band between them absorbs normal swings without growing and shrinking back and forth.

## Global parameters

All optional.

| parameter | default | meaning |
|---|---|---|
| `shm_auto_scaling_profile` | none | profile that drives the shared arena |
| `pkg_auto_scaling_profile` | none | profile that drives each worker's private arena |
| `shm_grow_granule` | 16m (one page on a small arena) | size of one grow step of the shared arena, rounded to the arena page |
| `pkg_grow_granule` | 16m (one page on a small arena) | the same for every worker's private arena |
| `hg_ram_floor_mb` | 0 = max(256 MB, MemTotal/20) | host free memory a grow may never take the host below |
| `hg_autoscale_dry_run` | 0 | 1 = log what the policy would do, never act |
| `hg_grow_ahead` | 1 | keep free headroom by growing one step before an allocation has to |
| `hg_scaling_cycle` | 30 | seconds per policy cycle (the profile's "cycles") |
| `hg_shrink_step` | 64m | most an arena releases per maintenance tick |
| `hg_interior_release` | 0 | 1 = give back whole free pages from anywhere in the arena, not only the top |
| `hg_lock_stall_us` | 1000 | an arena-lock hold at or above this is a stall: counted, and raises `E_CORE_HG_LOCK_STALL`; 0 disables |

Example:

```
shm_auto_scaling_profile = MEM_SHM
shm_grow_granule = 32m
hg_ram_floor_mb = 1024
hg_lock_stall_us = 2000
```

## Events

Both events describe the shared arena and are raised from the statistics timer, never while the arena lock is held. Catch them in the script with `event_route` or send them out with any event transport (`event_datagram`, `event_stream`, `event_rabbitmq`, ...).

### E_CORE_SHM_GROW_BLOCKED

The shared arena needs to grow and cannot - it is at its cap, the backing (memory-lock limit or hugetlb pool) ran out, or growing would take the host below `hg_ram_floor_mb`. A single refusal does not raise it: the state latches when refusals are still accumulating a full 30 s sweep later. It is raised when the state latches and again every 5 minutes while it lasts; the next successful grow clears it.

| parameter | meaning |
|---|---|
| `arena` | `shm` |
| `committed_mb` | memory the arena holds now |
| `cap_mb` | the `-m` cap |
| `grow_refused` | refused grows so far |

What to do: raise the `-m` cap, the memory-lock limit or the hugetlb pool - whichever the log names - or find what is filling the arena (`hg_stats`, `hg_advise`). With no subscriber the same moment is logged as a warning, and the `hgmem:hg_shm_grow_blocked` statistic reads 1 while it lasts.

### E_CORE_HG_LOCK_STALL

A process held the shared arena's lock for `hg_lock_stall_us` (default 1000 us) or longer. Raised at most once per 30 s, carrying the most recent stall; `hg_lock_stall_us = 0` turns it off.

| parameter | meaning |
|---|---|
| `arena` | `shm` |
| `reason` | what the lock was held for: `refill` (taking cells or a new block), `return` (giving cells or a block back), `flush` (a process emptying its private cache), `large` (an allocation over 64 KB), `policy` (a grow or shrink decision), `stats` (a statistics walk), `other` |
| `hold_us` | how long it was held |
| `process` | OpenSIPS process number that held it, as listed by `opensips-cli -x mi ps` |
| `stalls` | stalls since startup |

What to do: an occasional stall during a growth spurt is normal. A steady stream, or one `reason` dominating, is worth a look at the per-reason hold histogram in `hg_stats`.

### Examples

In the script:

```
loadmodule "event_route.so"

event_route[E_CORE_SHM_GROW_BLOCKED] {
    xlog("L_CRIT", "HG: shm arena cannot grow - $param(committed_mb) of $param(cap_mb) MB committed, $param(grow_refused) refusals\n");
}

event_route[E_CORE_HG_LOCK_STALL] {
    xlog("L_WARN", "HG: shm lock held $param(hold_us) us by process $param(process) for $param(reason), $param(stalls) stalls so far\n");
}
```

which logs, for example:

```
CRITICAL:HG: shm arena cannot grow - 1024 of 1024 MB committed, 37 refusals
WARNING:HG: shm lock held 1262 us by process 9 for return, 2 stalls so far
```

To another system, with `event_datagram` loaded:

```bash
opensips-cli -x mi event_subscribe E_CORE_SHM_GROW_BLOCKED udp:10.0.0.10:9999
```

## Monitoring

```bash
opensips-cli -x mi core:hg_stats      # per arena: tier, committed and cap, grows and shrinks, lock holds per reason, corruption counters
opensips-cli -x mi core:hg_advise     # what -m and -M should be, from the peak each arena actually reached
opensips-cli -x mi statistics:get hgmem:
```

`hg_stats` answers for the process that receives the MI command, so its `pkg` section is that process's private arena; `hg_advise` covers the worst process.

The `hgmem:` statistics group holds the shared arena's figures. The ones worth alerting on:

| statistic | alert when |
|---|---|
| `hg_shm_grow_blocked` | 1 - growth refused for a sustained period |
| `hg_shm_corruption` | above 0 - memory was corrupted and the allocator caught it |
| `hg_shm_repin_failed` | above 0 - memory taken back from the host could not be pinned again |
| `hg_shm_lock_stalls` | rising steadily |
| `hg_shm_committed` against `hg_shm_cap` | committed close to the cap |

Others for graphs: `hg_shm_grows`, `hg_shm_grows_proactive`, `hg_shm_grows_exhaustion`, `hg_shm_grow_refused`, `hg_shm_shrinks`, `hg_shm_shrink_bytes`, `hg_shm_live`, `hg_shm_live_peak`, `hg_shm_carved`, `hg_shm_free_to_carve`, `hg_shm_lock_hold_max_us`, `hg_shm_tier`.

Note that cells cached in private lists count as used, so `shmem:free_size` reads lower than what is reusable; alert on the figures above, not on `free_size`.

## Host requirements

- `LimitMEMLOCK=infinity` in the service unit, or `ulimit -l unlimited`: the committed arena is pinned and every grow is an `mlock()`, which checks the limit on hugetlb mappings too. HG warns at startup when the limit cannot cover growth, and a refused grow names the limit or the hugetlb pool, whichever ran out.
- Huge pages must be enabled on the host, or every arena falls through to plain 4 KB pages. Either or both of:
  - a hugetlb pool (tier 1): `vm.nr_hugepages` large enough for the shared cap plus the private arenas, in 2 MB pages. On this tier the whole cap is reserved from the pool when the arena is created; a pool too small for the cap gives a fixed arena on huge pages instead of an elastic one, and says so.
  - transparent huge pages (tiers 2-3): `/sys/kernel/mm/transparent_hugepage/enabled` set to `madvise` or `always` for the private arenas, and `/sys/kernel/mm/transparent_hugepage/shmem_enabled` set to `advise` or `always` for the shared arena. Many distributions ship `shmem_enabled=never`. `MADV_COLLAPSE` (tier 3) needs Linux 6.1 or later.

More detail - every tier, the growth and shrink mechanics, deployment recipes, troubleshooting: [design.md](design.md).
