# C54.3 — Breaking limits safely

## The core principle

Every limit in SA was set to fit a specific hardware target (PS2 / 2004 PC). The limits are not there to make modding hard — they are there to prevent the PS2 from running out of memory. On modern hardware, almost all of them can be raised substantially without issue. The danger is not the limit itself; it is exceeding the **pool stride × count** budget or patching a density cap without raising the pool alongside it.

## Step-by-step procedure

### Step 1 — Decide which limit to raise

Identify whether you need:
- **More entities** (pool count limit) → patch the `push-imm32` in `CPools::Initialise`
- **More population density** (spawner limit) → patch the `cmp` immediate in the spawner
- **Both** (almost always — see [C54.2](02-density-and-spawn-limits.md))

### Step 2 — Calculate memory cost

```
extra_memory = (new_count - old_count) × stride_bytes
```

You must know the stride. For the four typed pools:
- CPed: 1988 bytes
- CVehicle: 2584 bytes
- CBuilding: 56 bytes
- CObject: 412 bytes

For pools 5–12 where stride is TBD, do not patch without first probing the constructor with Capstone to determine the stride.

Verify: sum all pool memory costs (default + your increases). If total > ~50MB extra, you are in unusual territory — test more carefully.

### Step 3 — Apply the patch

**Timing:** Patches to pool slot counts must be applied **before** `CPools::Initialise` at `0x5503A0` is called. This function is called from `CGame::Initialise`. The practical approach is:

1. Hook `CGame::Initialise` at its prologue
2. In the hook, write your patched count to the `push-imm` location **before** calling the original function
3. The pool is then allocated at the larger size for the lifetime of the process

Alternatively, use a code cave that runs before the original `CPools::Initialise` and patches the push arguments in-place before the call.

**Never patch pool counts after `CPools::Initialise` has already run.** The `m_pObjects` and `m_byteMap` pointers are already set to the old size — a count-only patch on a live pool creates a size mismatch and will crash when a slot beyond the original count is accessed.

### Step 4 — Patch density caps proportionally

If you raised a pool count, raise the matching density comparison:

```
new_density_cap = floor(old_density_cap × (new_count / old_count))
```

Apply this to each `cmp` instruction site for the relevant spawner. The `cmp` immediate is typically a 1- or 2-byte value in the instruction encoding — verify the instruction bytes before writing.

### Step 5 — Verify

1. Load the game with the new limits active.
2. Test in a dense scenario: a crowded city block for peds, a highway for vehicles, a heavily IPL-modded interior for buildings.
3. Monitor pool occupancy if you have a trainer or ASI that can read pool usage.
4. Watch for: missing entities (pool full, silent fail), crashes at game start (OOM on pool allocation), crashes mid-session (handle range overflow if you have scripts that rely on specific handle value ranges).

## What NOT to do

### Do not decrease pool counts

Pool counts should never go **below** their default values. Scripts compiled against the original counts may encode handle values that assume a minimum handle range. The streaming system similarly assumes minimum pool sizes. Decreasing a count below default can cause handle collisions or index-out-of-bounds reads.

### Do not patch pool counts at runtime (after init)

As stated above — the `m_pObjects` allocation is already sized. A count-only runtime patch creates a dangling-size mismatch.

### Do not raise pools 5–12 without identifying the stride

These pools have verified slot counts but TBD strides. Raising a count on a pool with an unknown stride means you cannot calculate the memory cost, and you cannot verify that the allocation will succeed. Probe the constructor first.

### Do not raise building pool above 25,000

The IPL loading loop that fills the CBuilding pool iterates through all IPL entries and calls placement code for each one. Beyond ~25,000 buildings, the load time becomes very long and fragmentation inside the pool's byteMap can cause degraded allocation performance. The safe maximum of 20,000 is well below the degradation threshold.

## The safe modification table (complete)

| Limit | Patch VA | Default | Safe max | Stride (bytes) | Extra memory at max |
|---|---|---|---|---|---|
| CPed pool | `0x5503F1` | 140 | 250 | 1988 | ~220KB |
| CVehicle pool | `0x550429` | 110 | 160 | 2584 | ~129KB |
| CBuilding pool | `0x55045E` | 13,000 | 20,000 | 56 | ~392KB |
| CObject pool | `0x550496` | 350 | 1,000 | 412 | ~272KB |
| Vehicle density cap | `0x49B912` | 300 | scale with pool | N/A | N/A |
| Ped density A | `0x52953B` | 63 | scale with pool | N/A | N/A |
| Ped density B | `0x529C11` | 53 | scale with pool | N/A | N/A |
| Ped density C | `0x575227`, `0x582F0C` | 80 | scale with pool | N/A | N/A |
| ms_fTimeStep max | rdata `0x858C14` | 2.0 | 3.0–4.0 | N/A | N/A |
| Frame limiter | main loop | 25fps | remove | N/A | N/A |

## Community tools that implement these patches

**SA Limit Adjuster** (community tool) patches the `push-imm32` arguments in `CPools::Initialise` at startup. Its implementation is consistent with the addresses documented here. It is useful as a reference for expected patch sites — when verifying a new limit, checking against SA Limit Adjuster's known addresses is a reasonable cross-check (not a substitute for disassembly proof).

**SilentPatch** additionally patches some of the non-timestep-scaled FPS hazards documented in [C52.3](../C52-CTimer-And-Game-Loop/03-fps-unlock-guide.md).

## Summary

The limits exist for hardware reasons, not game design reasons. Every pool limit and density cap documented in this chapter can be raised on modern hardware without breaking the game, provided you:

1. Calculate memory cost from stride × count
2. Keep total pool memory in the tens-of-MB range
3. Patch density caps proportionally with pool counts
4. Apply pool-count patches before `CPools::Initialise` runs
5. Never decrease counts below default

**Up:** [C54 — Limits Reference](C54-Limits-Reference.md)
