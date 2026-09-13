# C28.1 — The Class Map by Subsystem

> **The one-sentence version:** the 74 named classes are not a random scatter — sorted into the subsystems
> they belong to, they fall into seven clusters (scripting, streaming, paths & traffic, collision &
> geometry, world & AI-tasks, and gameplay managers), and every cluster points back at a chapter that
> already owns that subsystem, so this page is as much an index into the rest of the book as it is a list.

**Subsystem category:** Binary substrate — catalogue index
**Depends on:** [C27.2 — The catalogue and the class map](../C27-Function-Catalogue/02-the-catalogue-and-class-map.md)
**RE status:** Documented
**Confidence:** ✅ for the grouping (class names are C27's) · the per-cluster *interpretation* is 🟡, an
observation about code layout, not a proof

---

## 1. Seven subsystems

C27.2 ranked the classes by how many relocated methods each contributes. That ranking is useful but flat.
Grouped instead by *what the class does*, the 74 classes and 367 named methods sort cleanly into seven
subsystems. Each row names the chapter that already documents that subsystem's data or engine side; C28 is
adding the function-level identity underneath it.

| Subsystem | Named methods | Principal classes (method count) | Owning chapter(s) |
|---|---:|---|---|
| **Scripting / missions** | ~99 | `CTheScripts` (34), `CRunningScript` (11), `CDarkel` (7), `CScriptsForBrains` (7), `CStreamedScripts` (7), `CScriptResourceManager` (4), `CConversations` (9), `CPedToPlayerConversations` (2), `CConversationForPed` (1), `CSetPieces` (2), `CGameLogic` (10), `CRestart` (4), `CCheat` (3) | [C18](../C18-SCM-Script/C18-SCM-Script.md), [C19](../C19-GXT-Text/C19-GXT-Text.md) |
| **Streaming / loading** | ~67 | `CStreaming` (32), `CStreamingInfo` (5), `CdStream` (4), `CIplStore` (11), `CColStore` (10), `CTxdStore` (2), `CIplDefPool` (2), `CTempColModels` (1) | [C1](../C1-Streaming/C1-Streaming.md), [C2](../C2-CStreaming/C2-CStreaming.md), [C3](../C3-Model-Stores/C3-Model-Stores.md) |
| **Paths / traffic / driving** | ~65 | `CPathFind` (27), `CVehicleRecording` (11), `CCarAI` (6), `CStuckCarCheck` (5), `CCarCtrl` (4), `CStuntJumpManager` (4), `CRoadBlocks` (3), `CUpsideDownCarCheck` (3), `CTrafficLights` (2) | [C12](../C12-Path-Network/C12-Path-Network.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md) |
| **Collision / geometry** | ~19 | `CCollision` (5), `CCollisionData` (4), `CColModel` (3), `CColLine` (1), `CBox` (1), `CCollisionPlugin` (1), `CObjectPool` (1) | [C6](../C6-Collision/C6-Collision.md), [C8](../C8-Geometry/C8-Geometry.md) |
| **World / entities / peds** | ~19 | `CWorld` (2), `CEntity` (3), `CPed` (4), `CPlaceable` (1), `CBaseModelInfo` (1), `CPopulation` (1), `CPlayerPed` (1), `CPedIntelligence` (1) | [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), [C5](../C5-CWorld/C5-CWorld.md), [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md) |
| **AI task / event tail** | ~20 | 19 `CTaskSimple*` / `CTaskComplex*` / `CEvent*` / `CTaskTimer` classes, each with **one** relocated method (usually a `Constructor`) | [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md) (AI) |
| **Gameplay features / managers** | ~78 | `CGarages` (13)+`CGarage` (2), `CReplay` (13), `CShopping` (13), `CGangWars` (9), `CEntryExitManager` (10)+`CEntryExit` (1), `CPickups` (9)+`CPickup` (4), `COnscreenTimer` (6), `CTagManager` (5) | (mostly undocumented — candidate chapters) |

The method counts per cluster are approximate at the margins — `CGameLogic` is scripting-adjacent but
also a two-player/game-mode manager, and `CObjectPool` is a template used by several subsystems — so a
class is placed by its dominant role, and one or two could reasonably move. The clusters themselves are
not marginal.

## 2. What the grouping says

Two readings, both consistent with what [C27.2 §2](../C27-Function-Catalogue/02-the-catalogue-and-class-map.md#2-reading-the-shape-of-the-list)
already noted from the ungrouped list, now sharper once grouped:

- **Scripting and streaming together are nearly half of everything named.** ~99 scripting methods plus
  ~67 streaming methods is 166 of 367 — 45 %. These are exactly the two subsystems a disc-check / crack
  has structural reason to touch: the streamer is where the CD is read, and the mission-script VM is where
  a game gates progress. C27.2 flagged this; grouping makes the split quantitative. It remains an
  **observation** — [C0.1](../C0-Binary-Identity/01-the-hoodlum-layer.md) still lists *why exactly these
  492* as open — not a claim that the crack deliberately chose these classes.
- **The AI-task tail is a code-layout artifact, not a target list.** Twenty classes contributing exactly
  one `Constructor` each is what you get when a compiler emits many small, similar objects into adjacent
  translation units and a whole-program layout pass relocates a slice of them; it is not plausibly a
  hand-picked set. This is why C28 gives *structure* only to the three heavy classes: a class represented
  by a single constructor has almost no internal structure to recover from one function.

## 3. Which classes already have a home, and which are candidates

Most clusters point at an existing chapter, and C28.2–4 wire the three heavy classes into theirs
(`CTheScripts`→C18, `CStreaming`→C2, `CPathFind`→C12). The **gameplay-managers** cluster is the
conspicuous exception: `CGarages`, `CReplay`, `CShopping`, `CGangWars`, `CEntryExitManager` and `CPickups`
together carry ~78 named methods and **no** chapter documents them yet. That makes this row the natural
seed list for future chapters — each is a self-contained gameplay feature with a double-digit method
count and a known set of addresses to start from, which is a far warmer start than a cold data file.

---

### Key takeaways

- The 74 classes sort into seven subsystems; six of the seven already have an owning chapter, and this
  page is the index that connects C27's addresses to them.
- Scripting + streaming = 45 % of all named methods — the quantitative form of C27.2's observation,
  consistent with (but not proof of) a crack-driven relocation.
- The one-method `CTask*`/`CEvent*` tail is a layout byproduct; only classes with real method counts have
  recoverable structure, which is why C28 deep-dives exactly three.
- The undocumented **gameplay-managers** cluster (`CGarages`, `CReplay`, `CShopping`, `CGangWars`,
  `CEntryExitManager`, `CPickups`, ~78 methods) is the strongest seed list for future chapters.

**Next:** [C28.2 — CTheScripts and the script object](02-cthescripts-and-the-script-object.md).
