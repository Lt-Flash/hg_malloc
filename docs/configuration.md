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
| `shm_grow_granule` | 16m | size of one grow step of the shared arena, rounded to the arena page |
| `pkg_grow_granule` | 16m | the same for every worker's private arena |
| `hg_ram_floor_mb` | 0 = max(256 MB, MemTotal/20) | host free memory a grow may never take the host below |
| `hg_autoscale_dry_run` | 0 | 1 = log what the policy would do, never act |
| `hg_grow_ahead` | 1 | keep free headroom by growing one step before an allocation has to |
| `hg_scaling_cycle` | 30 | seconds per policy cycle (the profile's "cycles") |
| `hg_shrink_step` | 64m | most an arena releases per maintenance tick |
| `hg_interior_release` | 0 | 1 = give back whole free pages from anywhere in the arena, not only the top |
| `hg_lock_stall_us` | 1000 | an arena-lock hold at or above this is a stall: counted, and raises `E_CORE_HG_LOCK_STALL`; 0 disables |

## Host requirements

- `LimitMEMLOCK=infinity` in the service unit, or `ulimit -l unlimited`: the committed arena is pinned and every grow is an `mlock()`, which checks the limit on hugetlb mappings too. HG warns at startup when the limit cannot cover growth, and a refused grow names the limit or the hugetlb pool, whichever ran out.
- Huge pages must be enabled on the host, or every arena falls through to plain 4 KB pages. Either or both of:
  - a hugetlb pool (tier 1): `vm.nr_hugepages` large enough for the shared cap plus the private arenas, in 2 MB pages. On this tier the whole cap is reserved from the pool when the arena is created; a pool too small for the cap gives a fixed arena on huge pages instead of an elastic one, and says so.
  - transparent huge pages (tiers 2-3): `/sys/kernel/mm/transparent_hugepage/enabled` set to `madvise` or `always` for the private arenas, and `/sys/kernel/mm/transparent_hugepage/shmem_enabled` set to `advise` or `always` for the shared arena. Many distributions ship `shmem_enabled=never`. `MADV_COLLAPSE` (tier 3) needs Linux 6.1 or later.

More detail - every tier, the growth and shrink mechanics, deployment recipes, troubleshooting: [design.md](design.md).
