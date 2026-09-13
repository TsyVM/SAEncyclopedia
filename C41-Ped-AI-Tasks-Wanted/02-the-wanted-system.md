# C41.2 — The wanted system

The wanted stars are the most-recognised number in the game, and they are computed by a single function from
a single running total. San Andreas does not track "stars" directly — it accumulates **chaos points**
(`m_ChaosLevel`) as you commit crimes, and derives the star count by crossing a fixed ladder of thresholds.
This page reads that ladder straight out of the executable.

## The ladder, proven cold

`CWanted::UpdateWantedLevel` at `0x561C90` compares `m_ChaosLevel` against six descending thresholds and
sets the star count accordingly. The thresholds appear as literal `cmp` immediates, in order:

| Address | `cmp` immediate | Chaos ≥ | Wanted level |
|---|---:|---:|:--:|
| `0x561CA5` | `0x11F8` | **4600** | ★★★★★★ (6) |
| `0x561CE7` | `0x0960` | **2400** | ★★★★★ (5) |
| `0x561D28` | `0x04B0` | **1200** | ★★★★ (4) |
| `0x561D69` | `0x0226` | **550** | ★★★ (3) |
| `0x561DAA` | `0x00B4` | **180** | ★★ (2) |
| `0x561DE5` | `0x0032` | **50** | ★ (1) |

Below 50 chaos points you are `WANTED_CLEAN` (0 stars). `derive_ai.py` asserts all six immediates are
present **and in descending code order** (`wanted_star_ladder`) — the ladder must read high-to-low or the
level assignment would be wrong. This is the wanted-star system in its entirety, recovered from the bytes:
not folklore, not a wiki table, but the six numbers the shipped code actually compares against.

## The cap

Before the ladder runs, the function clamps the running total:

```
m_ChaosLevel = min(m_ChaosLevel, MaximumChaosLevel)
```

`MaximumChaosLevel` is a global at `0x8CDEE8` holding **9200** (`derive_ai.py`: `max_chaos_9200`). So chaos
saturates at 9200 — well above the 4600 needed for six stars, giving headroom so that a six-star spree
doesn't instantly decay below the threshold when you stop offending. The star ceiling itself is **six**:
`eWantedLevel` runs `WANTED_CLEAN = 0` through `WANTED_LEVEL_6`, seven values, and the maximum is set to
`WANTED_LEVEL_6` at initialisation.

## Where chaos comes from, and where it goes

`m_ChaosLevel` is fed by **crimes**. `CWanted` keeps a fixed queue of the crimes you are currently suspected
of — `m_CrimesBeingQd[16]`, sixteen slots (🟡, from gta-reversed's declaration; the queue is why witnesses
"reporting" you raise the level with a delay rather than instantly). Each registered crime adds chaos; over
time, with no fresh crimes and no cops with line-of-sight, the level decays back down the ladder — crossing
the same thresholds in reverse drops your stars. The parole mechanic is visible in the struct too:
`m_ChaosLevelBeforeParole` preserves the pre-arrest level.

`CWanted` is **`0x29C` bytes** in full (🟡 — `gta-reversed`'s `VALIDATE_SIZE(CWanted, 0x29C)`; this pass
proved the ladder and the cap, not every field), and it caps the pursuit force at **`MAX_COPS_IN_PURSUIT` =
10** simultaneous cops (🟡).

## Why this answers an earlier question

When the coverage comparison first flagged the missing player-state systems, the "wanted stars" were the
headline example of something GTASA had never documented. This page closes that specific gap in the strongest
form the project offers: the star mechanic is a six-rung chaos-point ladder, and all six rungs are read
directly from `UpdateWantedLevel`'s comparison immediates. The stars you see on the HUD
([C23](../C23-Fonts-HUD/C23-Fonts-HUD.md)) are `m_WantedLevel`, and `m_WantedLevel` is exactly which of these
six thresholds your chaos total has crossed.

## Open items

- ⏳ The per-crime chaos values (how many points a car theft vs. a kill adds) — a separate crime table.
- ⏳ The decay rate (chaos lost per second when unobserved) and the observed/observing conditions.
- ⏳ The full `CWanted` field map beyond the ladder, cap, crime queue and cop cap.

## Key takeaways

- Wanted level is derived from **chaos points** via a fixed ladder — **50 / 180 / 550 / 1200 / 2400 / 4600**
  → **1…6 stars** — proven cold from six descending `cmp` immediates in `UpdateWantedLevel` @`0x561C90`.
- Chaos is capped at **9200** (`0x8CDEE8`) and the star ceiling is **6** (`eWantedLevel`, 7 values).
- Chaos comes from a **16-slot crime queue** and decays back down the same ladder; the cop cap is **10** (🟡).

**Continue:** [C41.3 — Dispatch and roadblocks →](03-dispatch-and-roadblocks.md)
