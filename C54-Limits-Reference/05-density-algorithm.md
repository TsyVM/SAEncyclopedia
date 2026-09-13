# C54.5 — How the density algorithm works

## Why pool size alone does not control population

Raising a pool count does not increase the number of entities you see in the game world. The pool count is a **ceiling** — the absolute maximum that can exist simultaneously. The **density caps** are the floor-above-that-ceiling — the spawner checks them before allocating a new entity and backs off when the cap is reached.

If you raise CPed to 250 slots but leave ped density cap A at 63, the spawner will still stop creating peds after it has 63 active peds in the area — 187 pool slots sit empty. Conversely, if you raise cap A to 150 without raising the pool, the spawner will try to create more peds than the pool can hold and silently fail for every request beyond slot 140.

The two must be raised together.

## CCarCtrl: the vehicle density accumulator

The vehicle density system in `CCarCtrl` works through an **accumulator** at `0xA9A894`, not a simple count. Here is the algorithm as recovered from the disassembly:

```
for each traffic spawn attempt {
    score = 0
    for each vehicle in CVehicle pool {
        d = distance(vehicle.pos, player.pos)
        if d < TRAFFIC_RADIUS:
            score += weight(d)   // closer = higher weight
    }
    [0xA9A894] = score
    if [0xA9A894] >= 300:        // comparison at 0x49B912
        skip this spawn attempt
    else:
        allocate CVehicle slot, spawn vehicle
}
```

The key insight is that `0xA9A894` is not a raw count — it is a **distance-weighted score**. A single vehicle right next to the player contributes more to the score than one at the edge of the traffic radius. This is why raising the cap from 300 to 600 does not double the number of visible traffic vehicles — far-away vehicles contribute very little to the score. The score is dominated by the few near vehicles.

**What raising the density cap actually does:** it allows more low-score vehicles (far from the player, on parallel streets, at the edge of the traffic radius) to spawn. The visible increase is mostly distant traffic, not cars-next-to-the-player density.

**Practical implication for limit patching:** to see a meaningful increase in traffic density from a raised CVehicle pool, you need to raise both the pool (110 → 160) AND the density cap proportionally (300 → ~435). But the visible result is more vehicles on the edges of the draw distance, not a noticeably denser immediate environment.

## CPopulation: three-gate ped spawn system

`CPopulation` (the ped spawner) uses three density comparisons in sequence, each blocking the spawn if exceeded. Understanding all three is required to raise ped density correctly.

### Gate A — The base ped count gate

```
comparison VA: 0x52953B
global VA:     0xB70138
current value: 63
```

`CPopulation` maintains a per-frame count in the global at `0xB70138`. This is incremented as peds are tracked in the spawn area around the player and decremented as they exit range or die. Gate A compares this count against 63.

If the count ≥ 63, `CPopulation` skips all further spawn logic for this frame. This is the **primary traffic light** — raising it allows more peds to be alive in the surrounding area simultaneously.

### Gate B — Zone-specific density gate

```
comparison VA: 0x529C11
global VA:     0xB70140
current value: 53
```

Gate B is a secondary count that applies specifically to zone-type-restricted spawning (indoor areas, mission zones, dense urban grids). The exact zone-type conditions gated by this path are not fully traced, but the comparison appears during `CPopulation`'s zone-conditioned spawn branch. If your raised gate A is not having the expected effect in specific areas (dense city blocks, interiors), gate B is likely the binding constraint there.

### Gate C — Near-player ped gate

```
comparison VAs: 0x575227 (site 1), 0x582F0C (site 2)
global VA:      0xBA372C
current value:  80
```

Gate C applies the most restrictive constraint: the maximum number of peds that can be **near the player** simultaneously. Even if gates A and B permit spawning, gate C prevents it if there are already 80 peds within close range. This is the anti-crowding measure that keeps the player from being buried in NPCs.

The 80 limit is used at two callsites — both must be active for gate C to function. Both sites compare against the same global `0xBA372C`, so a patch to the comparison value must be applied at both.

## The spawn decision tree

The full spawn decision for a new ped, simplified:

```
CPopulation::Update() {
    if [0xB70138] >= 63:         // Gate A
        return  // too many peds globally
    
    if zone_density_rules:
        if [0xB70140] >= 53:     // Gate B
            return  // too many peds in this zone type
    
    if [0xBA372C] >= 80:         // Gate C (site 1)
        return  // too many peds near player
    
    // all gates pass: attempt allocation
    slot = CPed_pool.New()
    if slot == null:
        return  // pool exhausted (silent fail)
    
    // spawn ped at suitable position
    place_ped(slot, spawn_pos)
    [0xB70138]++
}
```

The interaction between the three gates means that raising only gate A (the most commonly known cap) may not produce the expected result if gate B or C is the actual binding constraint in your scenario. When tuning ped density:

1. Raise gate A proportionally with pool count (primary cap)
2. Raise gate B proportionally (secondary/zone cap)
3. Raise gate C carefully — it controls the directly-visible crowd density. Raising it too high creates ped crowds that impede player movement and confuse the AI pathfinder.

## Density vs distance

Both the vehicle and ped density systems are proximity-sensitive. Vehicles and peds far from the player are less likely to be spawned by the system, and the density caps are checked in terms of "entities near the player" rather than "entities anywhere in the world." This is why you can have 140 CPed slots but only see ~30-40 peds at any given moment — many slots are occupied by entities in the 3D world that are loaded but outside the visible density radius.

Raising the density caps increases the **near-player density** ceiling, not just the background world population.

## Recommended tuning ratios

When raising pool counts, use these ratios to keep the density system consistent:

| Pool change | Gate to raise | Ratio | Example |
|---|---|---|---|
| CPed 140 → N | Gate A (0x52953B) | (N/140) × 63 | 200→90 |
| CPed 140 → N | Gate B (0x529C11) | (N/140) × 53 | 200→76 |
| CPed 140 → N | Gate C (sites × 2) | (N/140) × 80 | 200→114 |
| CVehicle 110 → N | Density accum (0x49B912) | (N/110) × 300 | 160→436 |

These are starting points. The actual feel depends on the map layout — a city with narrow streets benefits less from raised gate C than an open suburb.

**Previous:** [C54.4 — Exact patch bytes](04-exact-patch-bytes.md)  
**Up:** [C54 — Limits Reference](C54-Limits-Reference.md)
