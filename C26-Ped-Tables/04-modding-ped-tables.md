# C26.4 — Modding the ped tables

## What you can change without touching the exe

Both `ped.dat` and `pedgrp.dat` are plain text files read at startup. Any change to these files takes effect on the next game load — no exe patching required. The constraints are:
- `ped.dat` type names must be PEDTYPE strings the exe knows (the 18 confirmed names — C26.1§3)
- `pedgrp.dat` model names must exist in `peds.ide` (the 276-model table — C26.2§2)
- `pedgrp.dat` groups are capped at **21 members** by the engine (`cmp ebp, 0x15` at `0x5BD10F` — C26.2§3)

## Modding ped.dat: the acquaintance matrix

### Adding a new relationship

To make `CIVMALE` flee from `GANG1` peds (for a "gang-territory fear" mod), add `Hate GANG1` to the CIVMALE block:

```
CIVMALE
    Hate GANG1 GANG2 GANG3 GANG4 GANG5 GANG6 GANG7 GANG8 GANG9 GANG10
    Respect MEDIC FIREMAN
```

**Hate does not mean attack** for civilian types — their `pedstats.dat` flee distance and low attack strength will cause them to flee rather than fight. The aggression level comes from pedstats, not from ped.dat. `Hate` is more accurately understood as "threat response required"; whether that response is attack or flee is a pedstats function.

### Creating gang alliances

To make GANG3 and GANG4 allied (neutral to each other, still enemies of others):

```
GANG3
    Hate GANG1 GANG2 GANG5 GANG6 GANG7 GANG8 GANG9 GANG10
    Respect GANG3 GANG4   ; self + ally

GANG4
    Hate GANG1 GANG2 GANG5 GANG6 GANG7 GANG8 GANG9 GANG10
    Respect GANG3 GANG4
```

This removes GANG3/GANG4 from each other's Hate list and adds each to the other's Respect list.

### Using the two unused verbs

To create a "wary but not hostile" relationship using `Dislike` (shipped but never authored):

```
CIVMALE
    Dislike CRIMINAL
    Respect MEDIC FIREMAN
```

With `Dislike CRIMINAL`, civilians will react to criminals with a lower-priority threat response than full Hate — likely backing away or giving a wide berth rather than fleeing at full speed. The exact behaviour is engine-defined; testing is required because no shipped examples exist.

### The 17-type limit

Only the 18 PEDTYPE names the exe knows are valid. You cannot add a 19th ped type without exe modification. If you need a new faction with unique acquaintances, reassign one of the ten gang slots (GANG6 through GANG10 are typically underused in the base game) to your new faction by changing which ped models use that PEDTYPE byte.

### Changing the PEDTYPE of a ped model

`peds.ide` assigns each ped model its PEDTYPE via the 6th column. To reassign `COP` peds to a custom faction, change their `PEDTYPE` string in `peds.ide`. Note: this affects all peds of that model — a single model cannot have two PEDTYPEs.

## Modding pedgrp.dat: population groups

### Adding models to a group

Simply add the model name (comma-separated) to an existing group line:

```
HMYRI, HMYST, WMYBMX, MY_NEW_PED    # POPCYCLE_GROUP_BUSINESS
```

The new model must exist in `peds.ide`. The group will cap at 21 members — models beyond position 21 are silently ignored by the loader at `0x5BD10F`. Keep group counts ≤ 21.

### Adding a new population group

Append a new line to the file:

```
MY_PED1, MY_PED2, MY_PED3    # POPCYCLE_GROUP_WORKERS
```

The comment label at the end of the line (`# POPCYCLE_GROUP_*`) identifies which popcycle slot this group fills. To connect it to the popcycle system ([C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md)), the label must match one of the 18 active group labels. If you add a group with a label that the popcycle system doesn't call, the group will be loaded into the array at `0xC0F358` but never sampled.

### Using repeats for spawn frequency

The shipped data uses model repeats to bias spawn probability. A business group might list `HMYRI, HMYRI, HMYST` to make HMYRI appear twice as often as HMYST. Since the engine's sampler reads the list as a flat array (the weighting algorithm is ⏳, not fully traced), repeating a model N times in a 21-slot group makes it appear in approximately N/21 of spawns.

Practical guidance: for a mod that wants 80% of a group to be `MY_PED_A` and 20% `MY_PED_B` in a 21-member group:
```
MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_A, MY_PED_B, MY_PED_B, MY_PED_B, MY_PED_B
```
17 × MY_PED_A + 4 × MY_PED_B = 21 slots, approximately 81/19% split.

### The 21-slot hard cap in detail

The loader at `0x5BD040` breaks from its per-group parse loop when `ebp == 21`. Any models after the 21st comma on a line are simply not loaded — no error, no warning. If you need a group with more than 21 distinct model possibilities, the only option is to split it into two groups and assign both to the same `POPCYCLE_GROUP_*` label. The popcycle system may sample both groups, effectively merging them at spawn time.

## The CARGRP.DAT connection

The vehicle-group loader uses the same function as the ped-group loader (C26.2§3). `CARGRP.DAT` is the vehicle equivalent of `pedgrp.dat` and has the same 21-slot cap and comma grammar. Changes to `CARGRP.DAT` affect which vehicles spawn in each zone — documented in [C13 — Vehicle Data](../C13-Vehicle-Data/C13-Vehicle-Data.md).

## Popcycle integration

Changes to `pedgrp.dat` group composition take effect automatically when the popcycle system samples a group ([C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md)). The popcycle file specifies which POPCYCLE_GROUP is active in each zone at each time of day — `pedgrp.dat` says what models are in that group. To change who spawns in a zone at night, either:
1. Edit the group's model list in `pedgrp.dat`, or
2. Remap which group label that zone uses in `popcycle.dat`

Both approaches are valid; (2) is more targeted when the goal is zone-specific changes without altering the group globally.

**Previous:** [C26.3 — How acquaintances drive AI](03-how-acquaintances-drive-ai.md)  
**Up:** [C26 — Ped Tables](C26-Ped-Tables.md)
