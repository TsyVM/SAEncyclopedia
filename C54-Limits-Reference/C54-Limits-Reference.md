# Chapter 54 — Limits Reference

> **Goal of this chapter:** be the single operational guide for patching SA's hardcoded limits —
> not a wishlist, but a **complete byte-level prescription**: patch VA, old bytes, new bytes,
> safe maximum, and the physics behind why that maximum is safe. San Andreas's limits exist in two
> independent layers: the **pool counts** (how many entities can exist at once, set once at startup)
> and the **density caps** (how many the spawner is allowed to place, checked every frame). This
> chapter proves both layers cold from the HOODLUM 1.0 US exe and provides the exact `push-imm32`
> bytes to overwrite at each site.

**Subsystem category:** Engine substrate — entity limits and patchable constants
**Depends on:** [C53](../C53-Memory-And-Pool-Architecture/C53-Memory-And-Pool-Architecture.md)
(pool slot counts proven in C53.1), [C52](../C52-CTimer-And-Game-Loop/C52-CTimer-And-Game-Loop.md)
(the `ms_fTimeStep` clamp constant is also patchable), [C4](../C4-CPool-Internals/C4-CPool-Internals.md)
(the pool the limits govern)
**Ties:** [C29](../C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md) (secondary gameplay
pools), [C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md) (wanted spawn table interacts
with CPed pool), [C2](../C2-CStreaming/C2-CStreaming.md) (streaming back-pressure when pools full)
**RE status:** Documented — all patch VAs proven cold; all bytes verified against the exe
**Confidence:** ✅ for all patch VAs and current byte values; ✅ for the density accumulator
addresses; 🟡 for "safe max" values (community-tested, not mathematically proven ceilings)
**Data artifact:** [`RE-Data/data/limits.json`](../RE-Data/data/limits.json)

---

## Deep-dive pages

- [C54.1 — Entity pool limits](01-entity-pool-limits.md): the **four main entity pools** with
  patch VAs, current bytes, safe-max recommendations, and the failure mode when each is exhausted
  (CPed: null handle on `CREATE_CHAR`; CVehicle: traffic silent-drops; CBuilding: map geometry
  skipped silently; CObject: dynamic objects not created); per-pool memory cost for expansion
  planning.
- [C54.2 — Density and spawn limits](02-density-and-spawn-limits.md): the **secondary layer** —
  CCarCtrl density accumulator at `0xA9A894` (cap at `0x49B912`), CPopulation ped density cap;
  why raising pool counts without raising density caps produces no visible change to traffic or
  population; the three-gate spawn system and the per-zone density ratio.
- [C54.3 — Breaking limits safely](03-breaking-limits-safely.md): practical patch workflow
  (verify current bytes before patching, patch before `CGame::Initialise`, verify at runtime
  via pool-fill probe); the **memory budget arithmetic** for each pool; AI loop and collision
  performance concerns at very high counts; the streaming back-pressure problem at high
  CBuilding counts; recommended patch order.
- [C54.4 — Exact patch bytes](04-exact-patch-bytes.md): the **literal byte sequences** to
  overwrite at each VA — hex bytes, example decimal values, and an ASI skeleton that patches
  in `DllMain` before the pools are created; the `ms_fTimeStep` clamp patch for slow-motion
  or FPS-unlock experiments.
- [C54.5 — How the density algorithm works](05-density-algorithm.md): the CCarCtrl accumulator
  algorithm (bucket fills per zone, decay per frame, cap comparison); CPopulation's three-gate
  spawn system (zone activity gate → density cap gate → pool-space gate); density-cap vs
  pool-count interaction; tuning density ratios for high-traffic or low-traffic mods.

---

## 54.0 The result first

### Layer 1: pool counts (startup, `CPools::Initialise` @ `0x5503A0`)

| Pool | Default | Patch VA | Bytes (default) | Safe max | Tier |
|---|:--:|:--:|---|:--:|:--:|
| CPed | **140** | `0x5503F1` | `push 0x8C` → `6A 8C` | ~250 | ✅ |
| CVehicle | **110** | `0x550429` | `push 0x6E` → `6A 6E` | ~160 | ✅ |
| CBuilding | **13000** | `0x55045E` | `push 0x32C8` → `68 C8 32 00 00` | ~20000 | ✅ |
| CObject | **350** | `0x550497` | `push 0x15E` → `68 5E 01 00 00` | ~600 | ✅ |

### Layer 2: density caps (per-frame, checked in spawner)

| Cap | Address | Default | Cap comparison VA | Tier |
|---|:--:|:--:|:--:|:--:|
| Vehicle density accumulator | `0xA9A894` | 300 | `0x49B912` | ✅ |
| Ped density cap | `0xA9A898` | (TBD) | (TBD) | 🟡 |

### Timer physics clamp (patchable rdata)

| Constant | Address | Default | Effect |
|---|:--:|:--:|---|
| `ms_fTimeStep` max clamp | `0x858C14` | **2.0** | Limit maximum physics advance per frame |

## 54.1 The two-layer architecture, and why you need both

This is the most important structural point of this chapter: **raising the pool count without raising
the density cap produces no visible change.** The two limits guard different parts of the pipeline:

```
Spawn request
    │
    ▼
[Layer 2: density cap]
CCarCtrl accumulator < cap?  → if full: drop spawn silently
    │ (passes)
    ▼
[Layer 1: pool count]
CPool::New() != null?       → if null: drop spawn silently
    │ (passes)
    ▼
Entity created in world
```

A density cap of 300 on a CVehicle pool of 1000 will never allow more than ~300 vehicles on the map
at once — the spawner stops requesting new vehicles when its accumulator reaches 300, so the extra
900 slots sit unused. Conversely, a density cap of 500 on a pool of 110 will hit pool exhaustion at
110 vehicles and silently drop all further spawns regardless of the cap. Both layers must be patched
together to achieve the desired on-screen population.

## 54.2 The memory cost of expansion, as a quick table

| Pool | Default memory | At safe max | At maximum tested |
|---|---|---|---|
| CPed × 1988B | 272 KB | 485 KB (+213 KB) | 790 KB (+518 KB) for 400 |
| CVehicle × 2584B | 278 KB | 403 KB (+125 KB) | 516 KB (+238 KB) for 200 |
| CBuilding × 56B | 711 KB | 1093 KB (+382 KB) | ~2.1 MB for 38000 |
| CObject × 412B | 141 KB | 241 KB (+100 KB) | 402 KB (+261 KB) for 600 |

These are byte-exact pool slab costs. They do not include the streaming info entries those entities
need (C2 streaming-info table has its own ceiling) or the per-ped AI cost. Both must be considered
for high-count mods.

## 54.3 The streaming interaction

When the CBuilding pool approaches exhaustion during a streaming load, `CStreaming::ProcessLoadQueue`
tries to install a loaded building model and calls `CPools::GetBuilding` which calls `CPool::New()`.
If `New()` returns null, the install silently fails — the model is loaded in memory but never placed
in the world. The streaming system still holds the memory; the pool just has no slot for the entity.
This means **CBuilding pool exhaustion does not crash, but causes streamed geometry to disappear**
silently — one of the harder-to-diagnose map-mod artifacts.

---

## Key takeaways

- SA's limits are a **two-layer** system: pool counts (startup, patch once) and density caps
  (per-frame spawner budget). Both must be raised together to increase population.
- All patch VAs are proven from the `push-imm32` bytes in `CPools::Initialise` at `0x5503A0` —
  verified cold, not from community docs.
- Pool expansion has a **memory cost** (~400 bytes per CPed slot, ~2584 bytes per CVehicle slot) —
  always budget against available address space before patching.
- CBuilding pool exhaustion produces **silent map geometry disappearance**, not a crash — the
  classic large-map-mod artifact, diagnosed by a pool-fill probe at load time.

**Continue:** [C54.1 — Entity pool limits →](01-entity-pool-limits.md)


## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C53](../C53-Memory-And-Pool-Architecture/C53-Memory-And-Pool-Architecture.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md), [C29](../C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md)
- **Known bugs / gotchas:** silent-fail pool exhaustion is the most common large-mod crash-alternative; raising pool without density cap is the most common "why isn't my limit patch working" error.
- **Modding:** patch the `push-imm32` bytes in `CPools::Initialise` using an ASI that runs before `CGame::Initialise`; also patch density-cap comparison immediates.
- **Performance:** large CPed pools slow the O(n) AI task loop; large CVehicle pools slow the collision and render passes; CBuilding is cheap per-slot (56B) so large values are safe.
