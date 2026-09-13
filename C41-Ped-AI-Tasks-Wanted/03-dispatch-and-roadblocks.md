# C41.3 — Dispatch and roadblocks

The wanted level ([C41.2](02-the-wanted-system.md)) is an input, not an output. What it drives is
**dispatch**: which branch of law enforcement is sent after you, and whether the roads ahead are blocked.
This page reads the escalation ladder and the roadblock arrays out of the executable.

## The escalation tiers

`CWanted` exposes a set of predicate functions, each answering "at the current wanted level, is *this* force
required?" They sit consecutively in the binary:

| Function | VA | Sent in when… |
|---|---|---|
| `AreMiamiViceRequired` | `0x561F30` | (the two-cop "Vice" spawn condition) |
| `AreSwatRequired` | `0x561F40` | mid wanted levels — SWAT on foot |
| `AreFbiRequired` | `0x561F60` | high wanted levels — FBI |
| `AreArmyRequired` | `0x561F80` | maximum wanted — the Army with tanks |
| `NumOfHelisRequired` | `0x561FA0` | returns how many police helicopters to dispatch |

`derive_ai.py` confirms all five resolve in `.text` (`dispatch_tiers_in_text`). The escalation is the
familiar one — street cops at one or two stars, SWAT around three or four, FBI at five, the Army at six, with
helicopters layered on as the level climbs — and each tier is a wanted-level threshold test, the dispatch
analogue of [C41.2](02-the-wanted-system.md)'s star ladder. The number of helicopters is a *count*, not a
boolean, which is why higher heat can put multiple choppers over you at once.

## Roadblocks — a fixed array of 325

When the chase is on, the game throws roadblocks across the road network. `CRoadBlocks` manages them with a
set of parallel fixed arrays, all referenced directly by the roadblock code:

| Global | VA | Size | Holds |
|---|---|---:|---|
| `InOrOut` | `0xA43438` | `bool[325]` | is roadblock *i* currently active |
| `RoadBlockNodes` | `0xA435A0` | `CNodeAddress[325]` | which path node ([C12](../C12-Path-Network/C12-Path-Network.md)) each sits on |
| `NumRoadBlocks` | `0xA43580` | `int32` | live count |
| `aScriptRoadBlocks` | `0xA43AB8` | `CScriptRoadBlock[16]` | the 16 script-placed roadblocks |

`derive_ai.py` confirms all four are referenced from `.text` (`roadblock_arrays_referenced`). The **325**
dynamic roadblock cap and the **16** script cap come from `gta-reversed`'s constants (🟡), but the layout is
consistent with them: `RoadBlockNodes` is `CNodeAddress[325]` and `CNodeAddress` is **4 bytes**
([C12](../C12-Path-Network/C12-Path-Network.md)'s `{area, node}` link), so the node array spans
`0xA435A0 + 325 × 4 = 0xA43AB4`, landing just before `aScriptRoadBlocks` at `0xA43AB8` (a 4-byte alignment
gap) — the two arrays sit exactly where 325 nodes would place them. `CScriptRoadBlock` is `0x1C` bytes, so
the script array is `16 × 0x1C = 0x1C0` bytes.

That roadblocks are keyed by **path node** is the tie to [C12](../C12-Path-Network/C12-Path-Network.md): the
dispatch system doesn't place blocks in free space, it selects nodes on the vehicle path graph ahead of you
and spawns the block there — which is why roadblocks always appear across actual roads, never mid-field.

## The whole loop, in one picture

Putting the chapter together, the police response is a closed loop entirely made of this chapter's pieces:

1. You commit a crime → `CWanted` adds **chaos points** ([C41.2](02-the-wanted-system.md)).
2. Chaos crosses a **star threshold** → `m_WantedLevel` rises.
3. The wanted level trips the **dispatch tiers** (`AreSwat/Fbi/ArmyRequired`, `NumOfHelisRequired`) and the
   **roadblock** generator.
4. Dispatched cops are `CPed`s whose [C41.1](01-the-task-tree.md) `CTaskManager` is handed pursuit tasks
   aimed at you — an event-response primary task, exactly slot 1–2 of the ladder.
5. On a bust, the [C39](../C39-Camera/C39-Camera.md) arrest-cam modes (`MODE_ARRESTCAM_ONE/TWO`) fire.

So "the police" is not a separate system — it is the **task tree** (C41.1) driven by the **wanted system**
(C41.2) as its event source, with dispatch and roadblocks as the spawn logic. The AI cluster the handoff
kept setting aside turns out to be one coherent machine.

## Open items

- ⏳ The exact wanted-level thresholds inside each `AreXRequired` predicate (which star count trips SWAT vs.
  FBI vs. Army).
- ⏳ `NumOfHelisRequired`'s formula (helis per wanted level).
- ⏳ Re-close `MAX_ROADBLOCKS = 325` as an exe immediate (a bound compare in `GenerateRoadBlocks`
  @`0x4629E0`), promoting it from 🟡 to ✅.

## Key takeaways

- The wanted level drives a **dispatch escalation** — Vice, SWAT, FBI, Army, and a *count* of helicopters —
  each a wanted-level predicate at `0x561F30…0x561FA0`.
- Roadblocks use fixed parallel arrays capped at **325** dynamic (+**16** script), keyed by
  [C12](../C12-Path-Network/C12-Path-Network.md) path nodes so they always land on real roads.
- Dispatch closes the loop: crime → chaos → stars → dispatched cops, whose behaviour is
  [C41.1](01-the-task-tree.md) pursuit **tasks** — the police are the task tree pointed at the player.

**Continue:** [back to the C41 hub →](C41-Ped-AI-Tasks-Wanted.md) · or [C12 — Path Network](../C12-Path-Network/C12-Path-Network.md) (the nodes roadblocks use).
