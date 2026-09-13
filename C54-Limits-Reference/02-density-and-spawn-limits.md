# C54.2 — Density and spawn limits

## What density limits are

Pool limits control **how many** entities can exist. Density limits control **how many the spawner is allowed to create at once**. These are independent layers:

- Raising only the pool count does not increase visible population — the density cap stops the spawner first.
- Raising only the density cap without raising the pool count causes the spawner to fill the pool and run out of slots immediately.
- Correct limit expansion requires raising **both** layers together.

## Vehicle density — CCarCtrl

| Field | Value |
|---|---|
| Accumulator VA | `0xA9A894` |
| Comparison value | 300 |
| Comparison VA | `0x49B912` |
| Instruction | `cmp ecx, 0x12C` (0x12C = 300) |
| Tier | `verified_by_disassembly` |

**How it works:** `CCarCtrl` maintains a density accumulator at `0xA9A894`. Each spawned vehicle increments it; the spawner checks this value before creating a new vehicle. When the accumulator exceeds 300, spawning backs off. This is an internal weighting value — it is not a direct vehicle count, so the comparison value is not the same as the vehicle pool size (110).

**Safe range:** The comparison value of 300 can be raised proportionally with the CVehicle pool count. If raising CVehicle to 160, raising the density cap from 300 to ~435 (160/110 × 300) maintains the same per-slot spawn density ratio.

## Ped density — CPopulation

Three independent density comparisons govern ped spawning:

### Ped density cap A

| Field | Value |
|---|---|
| Global VA | `0xB70138` |
| Compared value | 63 |
| Comparison VA | `0x52953B` |
| Tier | `verified_by_disassembly` |
| Context | Medium-density ped count gate |

### Ped density cap B

| Field | Value |
|---|---|
| Global VA | `0xB70140` |
| Compared value | 53 |
| Comparison VA | `0x529C11` |
| Tier | `verified_by_disassembly` |
| Context | Indoor/zone-capped ped spawning; second CPopulation gate |

### Ped density cap C

| Field | Value |
|---|---|
| Global VA | `0xBA372C` |
| Compared value | 80 |
| Comparison VA (site 1) | `0x575227` |
| Comparison VA (site 2) | `0x582F0C` |
| Tier | `verified_by_disassembly` |
| Context | Max peds-on-foot near player; used at two callsites |

**Note on ped density:** The three density comparisons are all hardcoded `cmp` instruction immediates. To increase ped population, all three need to be raised proportionally. Each comparison is a `cmp [global], immediate` — the most straightforward patch is to change the immediate byte in each `cmp` instruction.

## Timer physics limit — ms_fTimeStep max clamp

| Field | Value |
|---|---|
| Float rdata VA | `0x858C14` |
| Current value | 2.0 (100ms max frame advance) |
| Evidence VA | `0x560DA2` (`fcomp dword ptr [0x858C14]`) |
| Tier | `verified_by_disassembly` |

This is the physics stall-protection clamp. See [C52.3](../C52-CTimer-And-Game-Loop/03-fps-unlock-guide.md) for full discussion.

## Streaming buffer sizes (informational)

These are intermediate streaming read buffers, not entity limits:

| Buffer | Size | VA |
|---|---|---|
| Small read buffer | 8,192 bytes (8KB) | `0x4C7066` |
| Large read buffer | 32,768 bytes (32KB) | `0x4C70C9` |

The streaming memory budget itself (`CStreaming::ms_memoryAvailable`) is set at runtime and is not a hardcoded limit in the exe.

## Interaction: raising both layers together

The recommended procedure when raising ped or vehicle limits:

1. Raise the **pool count** at the patch VA in `CPools::Initialise`.
2. Raise the **density comparison value(s)** for the corresponding spawner proportionally.
3. Test at the target density before releasing — spawn a full load of the new count in a dense area and verify no crashes.

Example: raising CPed 140→200 (ratio ≈ 1.43×):
- Patch ped pool count: `0x5503F1` → `push 0xC8` (200)
- Patch ped density A: `0x52953B` → compare against 90 (63 × 1.43 ≈ 90)
- Patch ped density B: `0x529C11` → compare against 76 (53 × 1.43 ≈ 76)
- Patch ped density C: `0x575227`, `0x582F0C` → compare against 114 (80 × 1.43 ≈ 114)

**Continue:** [C54.3 — Breaking limits safely →](03-breaking-limits-safely.md)
