# C28.2 — CTheScripts and the Script Object

> **The one-sentence version:** the largest class in the catalogue is the mission-script manager, and its
> methods hand up the one number [C18](../C18-SCM-Script/C18-SCM-Script.md) never had — the *in-memory*
> `CRunningScript` object is **224 bytes (`0xE0`)**, proven both by the multiply that indexes the pool and
> by the reciprocal-division that inverts it — along with the 4 KB mission local-variable block, the
> OnAMission flag that sits immediately after it, and the fixed-size model lists a running script uses to
> suppress and block vehicles.

**Subsystem category:** Scripting — in-memory structure
**Depends on:** [C28.1](01-the-class-map-by-subsystem.md), [C18](../C18-SCM-Script/C18-SCM-Script.md) (the
`main.scm` container and opcode model)
**RE status:** Documented
**Confidence:** ✅ for every byte-fact below (each re-checked by `derive_class_catalogue.py`) · 🔷 for the
methods listed but not individually disassembled here

---

## 1. Why this class matters to C18

[C18](../C18-SCM-Script/C18-SCM-Script.md) decoded `main.scm` as a *file* — the container, the mission
table, the opcode arities. What it could not see from the file is what the engine builds in memory when a
script runs: how big a running-script object is, where the pool of them lives, how a script's index is
recovered from its pointer. `CTheScripts` is the manager that owns exactly that machinery, and it is the
single heaviest class in C27's catalogue (34 relocated methods). Reading four of its methods recovers the
in-memory model that sits directly above C18's file model.

## 2. The 224-byte script object, proven twice

The clean result. `CTheScripts::StartNewScript[indexed]` (`entry_va 0x00464C90`) walks the script pool by
index:

```
0156E210  movzx eax, word ptr [esp + 8]     ; index argument
0156E215  imul  eax, eax, 0xE0              ; * 224
0156E21B  push  esi
0156E222  add   eax, 0xA8B430               ; + pool base
```

So a `CRunningScript` object is **`0xE0` = 224 bytes**, and the pool of them begins at `0xA8B430`. That is
one instruction saying it. The *inverse* operation says the same thing independently —
`CTheScripts::GetScriptIndexFromPointer` (`entry_va 0x00464D20`) turns a script pointer back into its
index:

```
01562034  sub   ecx, 0xA8B430              ; pointer - pool base = byte offset
0156203A  mov   eax, 0x92492493            ; signed reciprocal of 224
0156203F  imul  ecx                        ; 64-bit multiply
01562041  mov   eax, edx                   ; take the high dword
...       sar / shr                        ; normalize -> index
```

`0x92492493` is the constant a C compiler emits to divide a signed integer by **224** (it is
`round(2^39 / 224)` used as the magic multiplier, with the trailing `sar 7`). A multiply by 224 to go
index→offset and a magic-divide by 224 to go offset→index, in two different methods that never reference
each other, is the same two-independent-derivations standard the rest of the project runs on. The stride
is ✅, not inferred.

This is the direct analogue of [C27.3 §3](../C27-Function-Catalogue/03-verification-and-the-remaining-124.md#3)'s
412-byte `CObject` stride, and it is a fact [C18](../C18-SCM-Script/C18-SCM-Script.md) needed but could not
reach: the file gives the script's *code*; this gives the object that *runs* it.

## 3. The mission local-variable block, and the flag right after it

`CTheScripts::WipeLocalVariableMemoryForMissionScript` (`entry_va 0x00464BB0`) is four instructions of pure
structure:

```
01569F11  mov  ecx, 0x400                  ; 1024 dwords
01569F16  xor  eax, eax
01569F18  mov  edi, 0xA48960               ; block base
01569F1D  rep stosd                        ; zero 1024 * 4 = 4096 bytes
```

The mission's local-variable memory is a flat **1024-dword (4 KB) block at `0xA48960`**. This is the RAM
backing the `LVAR` cells C18's opcode model reads and writes — the counterpart to the global-variable
space the container reserves. And the very next method pins what lives immediately after it:
`CTheScripts::IsPlayerOnAMission` (`entry_va 0x00464D50`) reads a flag at `0xA49960`:

```
0156AA19  cmp dword ptr [eax + 0xA49960], 1
```

`0xA49960 = 0xA48960 + 0x1000` — exactly 4096 bytes past the block's base, i.e. the dword sitting right at
the end of the 1024-dword block is the "on a mission" flag. Two methods, two adjacent globals, one
consistent 4 KB block: the base and the extent check each other. (`StartTestScript`, `0x00464D40`, corrobo­rates
the region — it `push 0xA49960` before starting a script, using the same OnAMission address as the test
script's control cell.)

## 4. The script-imposed model lists

A running script can force the streamer's hand — suppress certain car models from spawning, block others
outright. Two `CTheScripts` methods clear those lists, and in doing so give their exact sizes:

| Method | `entry_va` | Fill | Meaning |
|---|---|---|---|
| `ClearAllVehicleModelsBlockedByScript` | `0x0046A840` | `mov ecx,0x14` · `mov edi,0xA448F0` · `rep stosd` of `-1` | 20-entry blocked-model list at `0xA448F0`, sentinel −1 |
| `ClearAllSuppressedCarModels` | `0x0046A7C0` | `mov ecx,0x28` · `mov edi,0xA44940` · `rep stosd` of `-1` | 40-entry suppressed-model list at `0xA44940`, sentinel −1 |

The blocked list is the one [C27.3 §2](../C27-Function-Catalogue/03-verification-and-the-remaining-124.md#2-two-worked-examples)
disassembled as a name-verification example; here it is read for its *size* instead — 20 dwords — and set
beside its sibling, the 40-dword suppressed list. Both use `-1` ("no model id") as the empty sentinel,
the same convention `CPathFind::Init` uses in [C28.4](04-cpathfind-and-the-28-byte-node.md) and the audio
lookup tables use in [C20](../C20-Audio/C20-Audio.md) — an engine-wide idiom worth noting once.

## 5. The WaitingForScriptBrain array

Street peds and objects that carry a "script brain" (an external script attached on proximity) are tracked
in a fixed array. `CTheScripts::RemoveFromWaitingForScriptBrainArray` (`entry_va 0x0046ABC0`) walks it
whole:

```
01569CDD  mov  esi, 0xA476B0               ; array base
01569CE2  mov  ebx, 0x96                   ; 150 iterations
...
01569D09  add  esi, 8                      ; stride 8 bytes
01569D0C  dec  ebx
```

**150 entries × 8 bytes at `0xA476B0`** — each entry a `{uint32 modelOrPtr; uint16 id; uint16 pad}`-shaped
pair (the clear stores a dword `0` at +0 and `0xFFFF` at +4). `AddToWaitingForScriptBrainArray`
(`0x0046AB60`) scans the same base with the same `0x96` bound to find a free slot, confirming the count
from the other direction.

## 6. The rest of the class

The remaining 28 methods are C27-inherited names (🔷), not individually disassembled here, but they group
tidily and confirm the class's scope: script-object lifecycle (`StartNewScript[last-idle]`,
`GetUniqueScriptThingIndex`, `CleanUpThisVehicle`, `CleanUpThisObject`), the SCM `switch` opcode support
(`ReinitialiseSwitchStatementData`, `UseSwitchJumpTable` — a jump-table-backed multi-way branch that ties
to C18's control flow), script-controlled scene edits (`AddScriptSphere`, `DrawScriptSpheres`,
`AddToBuildingSwapArray`, `AddToInvisibilitySwapArray`, `AddScriptSearchLight`,
`AttachSearchlightToSearchlightObject`, `RemoveScriptCheckpoint`), LOD-object stitching
(`AddToListOfConnectedLodObjects`, `ScriptConnectLodsFunction`), and script-file ingestion
(`ReadObjectNamesFromScript`, `ReadMultiScriptFileOffsetsFromScript`, `UpdateObjectIndices`). Full list in
[`class_catalogue.json`](../RE-Data/data/class_catalogue.json).

---

### Key takeaways

- `CRunningScript` objects are **224 bytes (`0xE0`)**, pool base `0xA8B430` — proven by the index→offset
  multiply *and* the offset→index magic-divide, in two methods that never reference each other.
- The mission **local-variable block is 1024 dwords (4 KB) at `0xA48960`**, and the OnAMission flag sits at
  `0xA49960`, exactly one block-length past its base — the two facts cross-check.
- Script-imposed model control is two fixed lists — 20 blocked (`0xA448F0`) and 40 suppressed
  (`0xA44940`) — both `-1`-sentinel, the engine-wide "no id" convention.
- The WaitingForScriptBrain array is **150 × 8 bytes at `0xA476B0`**, size agreed by its add and remove
  methods.
- Together these put the in-memory layer directly on top of [C18](../C18-SCM-Script/C18-SCM-Script.md)'s
  file-level script model.

**Next:** [C28.3 — CStreaming, and re-deriving C2 from the other end](03-cstreaming-and-rederiving-c2.md).
