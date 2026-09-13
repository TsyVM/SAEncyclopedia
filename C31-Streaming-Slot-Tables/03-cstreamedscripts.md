# C31.3 — CStreamedScripts and the 82-slot Table

> **The one-sentence version:** the external/streamed mission scripts (the ones loaded on demand from
> `script.img`, not the resident `main.scm`) are held not in a `CPool` but in a **fixed 82-slot table of
> 32-byte records embedded directly in the manager object**, with the live count in a `uint16` at object
> +0xA44 — the compile-time counterpart to C31.1/31.2's dynamic stores, and the in-memory table behind
> [C18](../C18-SCM-Script/C18-SCM-Script.md)'s external scripts.

**Subsystem category:** Streaming — streamed scripts
**Depends on:** [C31 hub](C31-Streaming-Slot-Tables.md), [C18](../C18-SCM-Script/C18-SCM-Script.md) (the SCM
container and its external-script table)
**RE status:** Documented
**Confidence:** ✅ for the slot count, stride and count-field offset · 🟡 for the internal record fields ·
🔷 for methods not disassembled here

---

## 1. A fixed 82-slot table, not a pool

Where `CColStore` and `CIplStore` point their pool object at a heap array, `CStreamedScripts` carries its
table *inside itself*. `CStreamedScripts::ReInitialise` (`entry_va 0x004706A0`) clears it:

```
01565AD0  lea  eax, [ecx + 4]              ; first slot's flag byte (record + 4)
01565AD3  mov  ecx, 0x52                   ; 82 slots
01565AD8  mov  byte ptr [eax], 0           ; clear flag
01565ADA  add  eax, 0x20                    ; next slot = 32 bytes
01565ADD  dec  ecx
01565ADF  jne  0x1565AD8
```

**82 (`0x52`) slots, each 32 bytes (`0x20`)**, walked by a fixed count — no pool object, no dynamic size.
`RegisterScript` (`0x004706C0`) confirms the geometry and locates the live count:

```
0156B7A4  movzx eax, word ptr [ecx + 0xA44] ; live registered count
0156B7AB  shl  eax, 5                        ; count * 32
0156B7AF  lea  esi, [ecx + eax + 8]          ; next free slot = this + 8 + count*32
```

So the records begin at object **+8**, stride **32**, and the number registered so far is a `uint16` at
object **+0xA44**. `RegisterScript` then copies a NUL-terminated **script name** into the slot byte-by-byte,
so each 32-byte record holds (at least) the streamed script's name; `StartNewStreamedScript` (`0x00470890`)
reads a dword from the slot as the loaded-script pointer and hands it to `CTheScripts::StartNewScript`
([C28.2](../C28-Class-Catalogue/02-cthescripts-and-the-script-object.md)), while
`RemoveStreamedScriptFromMemory` (`0x004708E0`) frees it with the same `shl …, 5` indexing. The precise
split of the 32 bytes between name and pointer/handle fields is left 🟡 — the offsets touched are certain,
the full field map is not traced here.

## 2. 82 slots, and C18's external scripts

The 82-slot capacity sits directly beside [C18](../C18-SCM-Script/C18-SCM-Script.md). C18's container work
(handoff §7.1) found the SCM **external-script table** lists **79** entries in `script.img` (the `AAA` build
artefact excluded). This table's **82** slots is the in-memory *capacity* for those streamed externals —
comfortably above the 79 the retail game ships, the same "capacity ≥ shipped content" relationship seen with
C19's five language slots (two shipped) and C22's cut `NAVIG.ZON`. The two numbers are not in conflict: 79
is what `script.img` contains, 82 is how many the engine can hold registered at once. This chapter does not
adopt either number from the other — 82 is read from `ReInitialise`'s loop, 79 from C18's file parse.

`ReadStreamedScriptData` (`0x00470750`) ties the table to the script globals C28.2 mapped: it reads through
the `0xA49960`-region pointers (the same block as C28.2's OnAMission flag at `0xA49960` and the mission
local-var block at `0xA48960`), confirming the streamed-script system shares the resident script system's
global state.

## 3. The rest of the class

The remaining methods (🔷) complete the streamed-script lifecycle: `Initialise` (first-time setup),
`LoadStreamedScript` (the C27.3-verified loader that string-tags a request into the streamer), and the
register/start/remove trio detailed above. Full list in
[`streaming_stores.json`](../RE-Data/data/streaming_stores.json).

---

### Key takeaways

- Streamed scripts live in a **fixed 82-slot (`0x52`) × 32-byte (`0x20`) table embedded in the manager
  object** — records from +8, live count `uint16` at +0xA44 — a compile-time table, unlike C31.1/31.2's
  dynamic `CPool` stores.
- Each 32-byte record holds the streamed script's **name** (copied in by `RegisterScript`) and a
  loaded-script pointer read by `StartNewStreamedScript`; the exact field split is 🟡.
- The **82-slot capacity** is the in-memory counterpart to [C18](../C18-SCM-Script/C18-SCM-Script.md)'s
  **79** external scripts in `script.img` — capacity ≥ shipped content, each number derived independently.

**Next:** back to the [C31 hub](C31-Streaming-Slot-Tables.md). Remaining streaming leaves (`CTxdStore`,
`CModelInfo`) and the vehicle-recording table (`CVehicleRecording`, a sibling of C30's `CReplay`) are the
adjacent leads.
