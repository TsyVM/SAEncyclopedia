# C27.2 — The Catalogue and the Class Map

> **The one-sentence version:** the 368 named functions cluster hard into scripting, streaming, and
> pathfinding (`CTheScripts` 34, `CStreaming` 32, `CPathFind` 27 — together over a quarter of every
> name recovered), which is itself a finding about *why* these particular 492 functions, out of the
> thousands in `.text`, were the ones the HOODLUM crack chose to relocate.

**Subsystem category:** Binary substrate — catalogue
**Depends on:** [C27.1 — Methodology](01-methodology-the-hook-harvest.md)
**Data artifact:** [`RE-Data/data/function_catalogue.json`](../RE-Data/data/function_catalogue.json)
**RE status:** Documented
**Confidence:** 🔷 for the 356 matched-but-not-individually-disassembled entries · ✅ for the 12 in
[C27.3](03-verification-and-the-remaining-124.md)

---

## 1. The class map

75 distinct classes appear among the 368 named functions. The top 20 by count:

| Class | Named functions |
|---|---|
| `CTheScripts` | 34 |
| `CStreaming` | 32 |
| `CPathFind` | 27 |
| `CGarages` | 13 |
| `CReplay` | 13 |
| `CShopping` | 13 |
| `CIplStore` | 11 |
| `CVehicleRecording` | 11 |
| `CRunningScript` | 11 |
| `CColStore` | 10 |
| `CEntryExitManager` | 10 |
| `CGameLogic` | 10 |
| `CConversations` | 9 |
| `CGangWars` | 9 |
| `CPickups` | 9 |
| `CDarkel` | 7 |
| `CScriptsForBrains` | 7 |
| `CStreamedScripts` | 7 |
| `CCarAI` | 6 |
| `COnscreenTimer` | 6 |

The remaining 55 classes each contribute five or fewer, tailing off to a long list of single
`CTaskSimple*`/`CTaskComplex*`/`CEvent*` entries — individual AI task and event classes, each
represented by exactly one relocated method (usually a `Constructor`).

## 2. Reading the shape of the list

This distribution is not what you'd get by relocating 492 random functions out of `.text`. Two things
stand out:

- **Scripting and streaming dominate.** `CTheScripts`, `CRunningScript`, `CScriptsForBrains`,
  `CStreamedScripts`, `CDarkel` (the SCM `Darkel`/random-events opcode family) together contribute over
  60 named functions — the largest single cluster in the catalogue by a wide margin once grouped by
  subsystem rather than by individual class. `CStreaming`, `CIplStore`, `CColStore`,
  `CStreamingInfo`, `CdStream` add another cluster around the loader. These are precisely the systems a
  crack needs to touch: script execution and the streaming loader are where disc/CD-check gates and
  save-related validation historically live in this engine family.
- **Long single-method tail from AI tasks.** Dozens of `CTaskSimple*`/`CTaskComplex*`/`CEvent*` classes
  appear exactly once, almost always as `Constructor`. That's consistent with compiler/linker behavior
  rather than a deliberate crack target: constructors for small, frequently-instantiated task objects
  are natural candidates for whatever code-layout transformation produced the 492-function split in the
  first place (see [C0.1 §4](../C0-Binary-Identity/01-the-hoodlum-layer.md) for what is and isn't known
  about that transformation's cause).

This is presented as an observation, not a closed conclusion — C0.1 already flags *why* exactly these
492 were chosen as an open item, and this class distribution is a data point for that question, not an
answer to it.

## 3. A representative slice

Fifteen entries drawn across the spread of classes and extents, to show the shape of the full 492-row
table (all fields — `entry_va`, `body_va`, `body_extent`, `direct_call_sites`, `name`, `confidence` —
are in the data artifact for all 492):

| `entry_va` | Name | Body extent | Direct callers |
|---|---|---|---|
| `0x00447CB0` | `CGarages::DeActivateGarage` | 32 B | 1 |
| `0x00410800` | `CColStore::GetBoundingBox` | 32 B | 1 |
| `0x00452090` | `CPathFind::Find2NodesForCarCreation` | 224 B | 2 |
| `0x0048E970` | `CTaskSimpleHandsUp::Constructor` | 64 B | 5 |
| `0x00407480` | `CStreamingInfo::AddToList` | 96 B | 12 |
| `0x00407F00` | `CStreaming::HasSpecialCharLoaded` | 32 B | 1 |
| `0x00464D70` | `CRunningScript::IsPedDead` | 48 B | 6 |
| `0x0040A080` | `CStreaming::RequestFile` | 176 B | 3 |
| `0x0044D480` | `CPathFind::TestForPedTrafficLight` | 160 B | 1 |
| `0x00470620` | `CScriptResourceManager::HasResourceBeenRequested` | 64 B | 1 |
| `0x004076A0` | `CStreaming::IsVeryBusy` | 32 B | 7 |
| `0x00463C00` | `CStuckCarCheck::HasCarBeenStuckForAWhile` | 64 B | 1 |
| `0x0043A870` | `CConversations::SetUpConversationNode` | 256 B | 4 |
| `0x00406F50` | `CPopulation::DoesCarGroupHaveModelId` | 64 B | 0 |
| `0x00408C70` | `CStreaming::RequestVehicleUpgrade` | 64 B | 3 |

Note the last row's `0` direct callers alongside a real, plausible name — a reminder that
`direct_call_sites` (from `hoodlum_relocation_map.json`) only counts direct-call cross-references found
by linear/static scanning; a function reached solely through a vtable, a function pointer table, or an
inlined caller the scanner didn't resolve will show `0` there even when it is real, named, and correct.
This matters again in [C27.3 §3](03-verification-and-the-remaining-124.md#3) when reading the same field
for the *unmatched* 124.

---

### Key takeaways

- 368 named functions span 75 classes; the top 3 (`CTheScripts`, `CStreaming`, `CPathFind`) alone account
  for 93 — almost a quarter of every name recovered.
- Grouped by subsystem rather than class, scripting and streaming are the two largest clusters — both
  plausible crack-relevant targets, though this chapter does not claim that as proven, only as an
  observation consistent with C0.1's still-open question about the relocation's cause.
- The long tail of one-off `CTaskSimple*`/`CEvent*` constructors looks like a byproduct of code layout,
  not a deliberate target list.
- `direct_call_sites == 0` does not mean "not really called" — it means "not called *directly*, from
  what the linear scanner could see." Several named, real functions have it.

**Next:** [C27.3 — Verification, and the remaining 124](03-verification-and-the-remaining-124.md).
