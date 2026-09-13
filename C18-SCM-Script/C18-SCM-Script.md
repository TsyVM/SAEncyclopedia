# Chapter 18 — SCM: the Mission Script Format

> **Goal of this chapter:** decode the container that holds San Andreas's entire mission logic —
> a 3,079,744-byte `main.scm` plus 79 streamed scripts — with **every one of the six segment sizes
> predicted to the byte**, and four independent fields that confirm the decode without being asked to.

**Subsystem category:** Scripting / mission logic
**Depends on:** [C1.1 — The IMG VER2 archive model](../C1-Streaming/01-img-ver2-archive-model.md) ·
[C11 — IDE and IPL](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C10](../C10-2dEffect/C10-2dEffect.md), [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md), [C12](../C12-Path-Network/C12-Path-Network.md)
**RE status:** Container Verified · opcode stream Partially Reversed
**Confidence:** ✅ Verified (container) / ⏳ (instruction semantics)

---

## Deep-dive pages

- [C18.1 — The segment chain](01-the-segment-chain.md): six `GOTO`s, six exact size predictions, and
  43,800 bytes of globals that are entirely zero.
- [C18.2 — The mission and external tables](02-mission-and-external-tables.md): 135 missions that tile
  the file exactly, and 79 streamed scripts addressed in a virtual space that begins where `main.scm` ends.
- [C18.3 — The code stream](03-the-code-stream.md): what can be proved without an opcode table, and the
  three things that let a reader bootstrap one.
- [C18.4 — Deriving the opcode table](04-deriving-the-opcode-table.md): 1,225 arities validated by exact
  closure, and the four attempts that failed first.
- [C18.5 — The opcode ceiling](05-the-opcode-ceiling.md): one already-proven constraint lifts the table to
  1,374 arities with zero conflicts, and 124 closures turns out to be the data's honest ceiling.
- [C18.6 — The dispatch table](06-the-dispatch-table.md): reading the argument counts out of the
  interpreter's own handlers carries the corpus to 188 exact decodes and 1,448 validated arities.
- [C18.7 — The floor of static analysis](07-the-floor-of-static-analysis.md): an exhaustive forcer proves
  **no** arity assignment can walk 24 of the 26 open scripts — the blocker is not a missing arity — and
  forces one more opcode out of the two scripts that do admit a walk.

---

## 18.1 The result first

Every structural claim rests on arithmetic that had to come out exact:

| Segment | ID | Predicted size | Actual | |
|---|---:|---:|---:|---|
| 1 — globals | `0x73` | `1 + 43,800` | **43,801** | ✅ |
| 2 — model list | `0` | `1 + 4 + 389×24` | **9,341** | ✅ |
| 3 — missions | `1` | `1 + 16 + 135×4` | **557** | ✅ |
| 4 — external scripts | `2` | `1 + 8 + 79×28` | **2,221** | ✅ |
| 5 — unknown | `3` | `1 + 4` | **5** | ✅ |
| 6 — trailer | `4` | `1 + 8` | **9** | ✅ |

✅ **Six for six, no residue.** Each segment's length is fully accounted for by a record count multiplied
by a record width — which is the same class of evidence as the array-end proofs in
[C2.1](../C2-CStreaming/01-the-id-space.md) and [C4.5](../C4-Entities-And-Pools/05-the-cpool-object.md).

## 18.2 The structure

```
main.scm
├── 02 00 01 <int32>   GOTO → segment 2      (7 B)
│   └── 0x73  +  43,800 bytes of globals      ← all zero
├── 02 00 01 <int32>   GOTO → segment 3
│   └── id 0  +  uint32 389  +  char[24] × 389        model list
├── 02 00 01 <int32>   GOTO → segment 4
│   └── id 1  +  4 dwords  +  uint32 × 135            mission offsets
├── 02 00 01 <int32>   GOTO → segment 5
│   └── id 2  +  2 dwords  +  28-byte record × 79     external scripts
├── 02 00 01 <int32>   GOTO → segment 6
│   └── id 3  +  uint32 0
├── 02 00 01 <int32>   GOTO → code
│   └── id 4  +  uint32 43800  +  uint32 569
└── 55,976 …  the code stream        starts 03A4 09 "MAIN"
```

**The header is written in the instruction set it prefixes.** Each segment boundary is a real `GOTO`
(opcode `0x0002`, parameter type `0x01` = int32), so a virtual machine that knows nothing about segments
still lands on the first instruction by simply executing the file from byte 0.

That is an elegant design and it is also why the format is self-delimiting: the jump target *is* the
segment end.

## 18.3 Four fields that confirm the decode

None of these had to agree with anything:

| Check | Result |
|---|---:|
| Segment 3's `mainSize` == the first mission offset | **194,125 == 194,125** ✅ |
| Segment 3's `largestMission` == the largest gap between offsets | **68,439 == 68,439** ✅ |
| Segment 4's first base address == `len(main.scm)` | **3,079,744 == 3,079,744** ✅ |
| Segment 4's `largestExternal` == the largest declared size | **35,122 == 35,122** ✅ |

Two of these are *derived* quantities — the largest gap and the file length — so a wrong record width or
a wrong field order would break them immediately. Together they pin the mission table and the external
table independently of each other.

## 18.4 Two cross-domain confirmations

**Every model the scripts name exists in the IDE files.**

| Check | Result |
|---|---:|
| Segment 2 names found in `data/**/*.ide` | **388 / 388 (100 %)** ✅ |

Segment 2 is a list of the models the mission scripts request by name; C11 catalogued 14,915 model
definitions from an entirely separate file family. **They agree perfectly** — no typos, no orphans, in
either direction that matters. Compare the 21 placed-but-undefined objects found in
[C11.3](../C11-IDE-And-IPL/03-what-placements-prove.md): the script table is *cleaner than the placement
data*, 🟡 plausibly because a missing script model crashes a mission where a missing prop does not.

**Every external script named in segment 4 exists in `script.img`.**

| Check | Result |
|---|---:|
| Segment 4 names matching a `script.img` entry | **79 / 79** ✅ |
| `script.img` entries named in segment 4 | **79 / 79** ✅ |

A bijection, both ways. Details in [C18.2](02-mission-and-external-tables.md).

## 18.5 The globals are empty

Segment 1 is **43,800 bytes — 10,950 four-byte global variables — and every single byte is zero.**

✅ Verified: `0` non-zero bytes in `[8, 43808)`.

🟡 *Reasoned:* the compiler emits the block at full size but does not initialise it. Every global San
Andreas uses is assigned by the script itself at startup, which is consistent with the eight external
scripts that open with `SET_VAR_INT` writes ([C18.3 §4](03-the-code-stream.md)).

The one byte that is *not* zero is byte 7 — the segment's ID slot — holding **`0x73`**. ⏳ Its meaning is
open; §18.1's arithmetic proves only that it is exactly one byte wide.

## 18.6 What remains

🟡 **The opcode table — largely derived.** [C18.4](04-deriving-the-opcode-table.md) validated 1,225 arities
against 124 exact closures; [C18.5](05-the-opcode-ceiling.md) added the sound opcode-ceiling prune to reach
**1,374 with zero conflicts**, and showed 124 was the ceiling of what `main.scm` proves alone.
[C18.6](06-the-dispatch-table.md) then took the executable route the earlier pages named — reading the
argument counts out of the interpreter's `ProcessCommands` handlers in the HOODLUM-wrapped `gta_sa.exe`
([C0.1](../C0-Binary-Identity/01-the-hoodlum-layer.md)). The extracted arities agree with the 1,374 at
**96 %** and, confirmed by exact closure, carry the corpus to **188 / 214 scripts decoding to their exact
declared end** (75.4 % of script bytes) — a **verified table of 1,448 arities** with **zero overshoots and
zero conflicts**. [C18.7](07-the-floor-of-static-analysis.md) then took the table to **1,449** and, more
usefully, **refuted** the standing explanation for the remainder: an exhaustive search over every arity in
`0…16` plus variadic plus untagged inline strings finds **no complete walk at all** for 24 of the 26. ⏳
**25 opcodes and 26 scripts remain**, but not because of handlers that compute their argument
count at runtime — the static method's honest floor.

⏳ **Three constants: `964`, `569`, and `0x73`.** Their positions are verified by exact arithmetic; their
meanings are not derived. Community tables name them, and per
[C10.1 §4](../C10-2dEffect/01-the-record-and-corrections.md) those names are **not adopted here**.

---

### Key takeaways

- **All six segment sizes are predicted exactly** by a record count times a record width — zero residue
  on any of them.
- The container is **written in its own instruction set**: six real `GOTO`s, so an interpreter that knows
  nothing about segments still reaches the code.
- **Four self-validating fields agree** — `mainSize`, `largestMission`, the first external base address,
  and `largestExternal` — two of them against *derived* quantities.
- ✅ **388/388 script model names appear in the IDE files**, and **79/79 external script names match
  `script.img` in both directions.**
- **The 43,800-byte globals block is entirely zero** — 10,950 variables, none initialised in the file.
- ✅ The **opcode table is 1,449 arities** and **188 / 214 scripts decode to their exact declared end** —
  the file-only method reached 1,374 ([C18.5](05-the-opcode-ceiling.md)), the interpreter's own handlers
  ([C18.6](06-the-dispatch-table.md)) carried it the rest of the way with zero conflicts.
- ✅ [C18.7](07-the-floor-of-static-analysis.md) proves the remaining blockage is **not** a missing arity:
  no assignment whatsoever walks 24 of the 26 open scripts to their declared end. ⏳ 26 scripts, **25**
  opcodes, and three header constants remain — now with the search space for them correctly narrowed.

**Next:** [C18.1 — The segment chain](01-the-segment-chain.md)

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C7](../C7-RenderWare-Stream/C7-RenderWare-Stream.md), [C10](../C10-2dEffect/C10-2dEffect.md)
- **Known bugs / gotchas:** 26 scripts don't fully decode (C18.7 refutes the old explanation); AAA build artefact.
- **Modding:** main.scm is THE mission-mod file; the opcode table is the SCM-compiler contract.
- **Performance:** bytecode walked by C32 CRunningScript threads.
