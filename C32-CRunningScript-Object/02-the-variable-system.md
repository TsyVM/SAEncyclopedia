# C32.2 — The Variable System, and Why 0xA48960 Exists

> **The one-sentence version:** an SCM variable reference resolves three ways — a script's own local (its
> `+0x3C` array), a *mission* local (redirected into the shared 4 KB block at `0xA48960`), or a global (the
> `0xA49960` space) — and the switch between the first two is a single flag byte at object `+0xDC`, which is
> exactly what makes [C28.2](../C28-Class-Catalogue/02-cthescripts-and-the-script-object.md)'s mystery block
> at `0xA48960` do its job.

**Subsystem category:** Scripting — variable resolution
**Depends on:** [C32.1](01-the-object-layout.md), [C28.2](../C28-Class-Catalogue/02-cthescripts-and-the-script-object.md)
(the `0xA48960` block and `0xA49960` OnAMission flag), [C18](../C18-SCM-Script/C18-SCM-Script.md)
**RE status:** Documented
**Confidence:** ✅ for the three resolution paths and the flag · 🟡 for the exact TIMERA/TIMERB slot indices

---

## 1. Local variables: own array, or the shared block

`CRunningScript::GetPointerToLocalVariable` (`entry_va 0x00463CA0`) is four instructions that decide where a
"local" variable lives:

```
0156E2B0  mov  al, byte ptr [ecx + 0xDC]   ; the mission-LVAR-mode flag
0156E2B6  test al, al
0156E2B8  mov  eax, dword ptr [esp + 4]     ; variable index
0156E2BC  je   +local                        ; flag clear -> own array
0156E2BE  lea  eax, [eax*4 + 0xA48960]       ; flag set   -> shared block
```

If the object's `+0xDC` flag is **clear**, the variable is in the script's own `+0x3C` array (the "local"
branch resolves against `this`). If it is **set**, the accessor indexes the flat block at **`0xA48960`**
instead — [C28.2](../C28-Class-Catalogue/02-cthescripts-and-the-script-object.md)'s 4 KB mission
local-variable memory. That is the whole mechanism: **a mission script sets `+0xDC` and thereby shares one
global 4 KB LVAR block, while an ambient script leaves it clear and uses its own embedded 34-dword array.**

This closes the loose end C28.2 left open. C28.2 found the block at `0xA48960`, saw `WipeLocalVariableMemoryForMissionScript`
clear its 1024 dwords, and could say only that it was "the mission local-variable memory." Here is *why* it
exists and *how* it is reached: the per-object `+0xDC` flag rewrites local-variable resolution to point at
it. The block and the flag are two halves of one design — and note the block is 1024 dwords, far larger than
one script's 34, because it is the *shared* space a mission and its child threads read and write in common.

## 2. Global variables: the 0xA49960 space

`CRunningScript::ReadArrayInformation` (`0x00463CF0`) and `GetIndexOfGlobalVariable` (`0x00464700`) resolve
the *global* variable form. Both read an operand from the PC (`[ecx+0x14]`) and, for a global reference,
index the space at **`0xA49960`**:

```
0156E375  mov  edx, dword ptr [edx + 0xA49960]   ; global variable / array base
```

`0xA49960` is `0xA48960 + 0x1000` — precisely one 4 KB-block past the mission-local block, and the same
address [C28.2](../C28-Class-Catalogue/02-cthescripts-and-the-script-object.md) identified as the base of the
global region (its first dword is the OnAMission flag). So the two blocks are adjacent by design: mission
locals at `0xA48960`, globals immediately after at `0xA49960`, and a script's *own* locals inside the object
at `+0x3C`. Three variable scopes, three bases, one flag (`+0xDC`) selecting between the first and a shared
copy of it.

## 3. Arrays and the type-tagged operand

`ReadArrayInformation` also shows the SCM *array* access form: it reads a base variable, an index variable
and an element size from the bytecode, and the same `+0xDC` / global switch applies to whether the array
lives in the object's locals, the shared mission block, or the global space. The operand is **type-tagged** —
a leading byte (`GetIndexOfGlobalVariable`'s `sub eax, 2` / `sub eax, 5`) selects global-variable vs
local-variable-array forms — matching the typed-argument model C18 built for the opcode stream (handoff
§7.1). The variable system and the opcode reader share one encoding.

## 4. The 32 + 2 convention

The 34-dword local array (C32.1) is the SCM standard: **32 general locals + 2 timers**. The two timers
(`TIMERA`/`TIMERB`) are the auto-incrementing locals every mission uses for its `wait`/timeout logic; they
occupy the top of the array. The exact slot indices (32 and 33) are the natural reading — 🟡, since this
chapter did not trace an increment site — but the count is exact: `Init`'s `rep stosd 0x22` zeroes all 34 in
one go.

---

### Key takeaways

- A "local" variable resolves to the script's **own +0x3C array** when `+0xDC` is clear, or to the **shared
  `0xA48960` mission block** when it is set — which is *why* [C28.2](../C28-Class-Catalogue/02-cthescripts-and-the-script-object.md)'s
  4 KB block exists.
- **Global** variables live at **`0xA49960`** = `0xA48960 + 0x1000`, adjacent to the mission-local block by
  design (its first dword is C28.2's OnAMission flag).
- Variable operands are **type-tagged** in the bytecode, the same typed encoding C18 uses for opcode
  arguments — one encoding shared by the reader and the variable system.
- The local array is **32 locals + 2 timers = 34 dwords**, matching `Init`'s `rep stosd 0x22`.

**Next:** back to the [C32 hub](C32-CRunningScript-Object.md). With the object mapped, the adjacent leads are
the SCM opcode handlers that mutate these fields (C18's remaining 25 open opcodes) and the sibling
`CScriptsForBrains` / `CStreamedScripts` ([C31.3](../C31-Streaming-Slot-Tables/03-cstreamedscripts.md)).
