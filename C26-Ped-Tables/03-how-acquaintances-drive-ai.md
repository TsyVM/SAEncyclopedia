# C26.3 — How acquaintances drive ped AI

## The chain from data to behaviour

`ped.dat` defines relationships as a static table. The engine reads this table and uses it to drive real-time AI decision-making. This page traces that chain — from the stored `Hate`/`Respect` values through the AI decision tree to the behaviours a player experiences (cops chasing criminals, gang members attacking each other, civilians fleeing violence).

## The CTask-level integration

When a ped's AI evaluates a potential threat or interaction, it calls `CPed::GetPedGroupIntelligence` or the acquaintance accessor to query: "do I hate, respect, or feel neutral toward this other ped's type?" The result feeds directly into task selection:

- **Hate → CTaskSimpleAttack path:** a ped that hates another ped's type will select an attack task (melee or ranged depending on weapon state) when proximity and line-of-sight conditions are met. This is the mechanism behind `GANG1` attacking `GANG2`: GANG1 Hates GANG2, the proximity check passes, and an attack task is queued.
- **Respect → neutral/deferential path:** a ped that Respects another type will not initiate combat with them and may back away or defer in a crowd.
- **Neither → civilian neutral path:** peds that have no relationship to each other use threat-radius checks and flee distance (from `pedstats.dat`) to decide whether to flee or ignore.

## The PEDTYPE lookup at runtime

`ped.dat`'s type names map to the engine's `PEDTYPE` enum (the 18 strings in the exe — C26.1§3). At runtime, each ped instance has a `PEDTYPE` byte in its `CPed` struct. The acquaintance lookup is `acquaintance_table[my_type][other_type]` — a two-dimensional array indexed by both parties' types. The result is the stored verb (Hate, Respect, or nothing) which branches AI behaviour.

The PEDTYPE values map approximately to:
- `CIVMALE / CIVFEMALE` = ambient civilians (type 0 / 1)
- `COP` = police, FBI, SWAT, army (type 2)
- `GANG1–GANG10` = the ten gang types (types 3–12)
- `MEDIC / FIREMAN` = emergency services (types 13–14)
- `CRIMINAL` = criminally-behaving non-gang peds (type 15)
- `DEALER` = drug dealer class (type 16 — exists in exe but no block in ped.dat)
- `PROSTITUTE` = prostitute peds (type 17 — has block but not referenced by others)

## Gang wars: the clique in motion

The mutual-hatred clique (C26.1§4) produces the observed gang-territory behaviour:
1. A GROVE STREET (GANG1) ped spawns in BALLAS (GANG2) territory.
2. Each BALLAS ped evaluates the threat — `acquaintance[GANG2][GANG1]` returns **Hate**.
3. Each BALLAS ped within detection range enters attack-task state.
4. The GROVE ped similarly evaluates — `acquaintance[GANG1][GANG2]` returns **Hate** — and counter-attacks.

The all-versus-all clique means any two gang members from different gangs will fight on sight. There is no neutral or allied gang pairing in `ped.dat` — the data encodes pure inter-gang hostility. An alliance would require adding `Respect GANG3` to GANG1's block and `Respect GANG1` to GANG3's block.

## COP vs CRIMINAL: the law enforcement mechanism

The cop-criminal dynamic is the cleanest two-sided relationship in the table:
- `COP Hate CRIMINAL DEALER` — cops actively pursue criminals and dealers
- `CRIMINAL` has no Respect for COP (its block hates back or is neutral)

At runtime this means cops autonomously pursue criminally-classified peds. The player's wanted-level system classifies the player as `CRIMINAL` when wanted, which is why police peds immediately enter pursuit mode — the acquaintance table returns Hate for `[COP][CRIMINAL]`, and the proximity/line-of-sight check passes for any cop that can see the player.

## The two unused verbs as a modding opportunity

`Dislike` and `Like` are parsed and stored by the acquaintance loader but are never authored in the shipped file. If you author `Dislike` relationships, the AI will route them differently from `Hate` — presumably to a lower-priority threat response (verbal warnings or backing away rather than immediately attacking). Testing community-authored `Dislike` entries shows the verb is functional in the engine even without shipped data.

This means modders can create a **five-tier** relationship spectrum (Hate, Dislike, neutral, Like, Respect) rather than the two-tier (Hate, Respect) the shipped data uses. The two missing middle tiers are live engine features.

## The `DEALER` asymmetry at runtime

`COP Hate CRIMINAL DEALER` means cops pursue dealers. But `DEALER` has no block — dealers feel nothing toward anyone. A dealer ped therefore does not initiate any relationship-based action toward cops or other peds. In practice, dealer peds only enter combat if attacked directly (the generic self-defence trigger), because their acquaintance row has no Hate entries.

This is the shipped data encoding a real game design choice: dealers don't pick fights, but cops pursue them anyway.

## The `PROSTITUTE` asymmetry at runtime

`PROSTITUTE Hate COP` means prostitute peds flee from police presence (the flee-distance check in pedstats.dat governs how far they run). But no type hates or dislikes `PROSTITUTE`. The one-directional fear relationship means prostitutes are the only ped type whose interactions are entirely player-driven — nobody hunts them, nobody helps them. They simply flee cops and otherwise interact neutrally.

## Flee and headchange rate from pedstats.dat

The acquaintance table determines *whether* a ped will fight or flee. The *how fast* and *how far* parts come from `pedstats.dat` (C14.3):
- `flee_distance` — how far a ped runs before stopping and looking back
- `heading_change_rate` — how quickly a fleeing ped changes direction

These two files are complementary: ped.dat says "I hate you, I'll fight" or "I'm scared of you, I'll flee"; pedstats.dat says "when I flee, I run this far at this agility." The full behaviour is the product of both tables.

**Previous:** [C26.2 — pedgrp.dat: the population groups](02-pedgrp-population-groups.md)  
**Continue:** [C26.4 — Modding the ped tables →](04-modding-ped-tables.md)
