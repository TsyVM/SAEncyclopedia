# C32.1 — The 224-byte Object, Field by Field

> **The one-sentence version:** `CRunningScript::Init` constructs the object front to back, and reading its
> stores in order — with the accessors filling in what `Init` only zeroes — recovers the entire 224-byte
> layout: two list links, an 8-byte name, a base pointer and program counter, an 8-entry gosub stack, a
> 34-dword local-variable array, and a flag block that runs exactly to offset 0xE0.

**Subsystem category:** Scripting — object layout
**Depends on:** [C32 hub](C32-CRunningScript-Object.md), [C28.2](../C28-Class-Catalogue/02-cthescripts-and-the-script-object.md)
**RE status:** Documented
**Confidence:** ✅ for offsets read from instructions and the tiling · 🟡 for the individual flag-byte roles

---

## 1. Init builds the object

`CRunningScript::Init` (`entry_va 0x004648E0`, `this` in `edx`) writes every region of the object, so its
body is the layout in construction order:

```
name[8]:        mov [edx+8]  = [0x859D44]      ; 8-byte script name copied in
base ptr:       mov [edx+0x10] = 0             ; +0x10  script code base
                mov [edx+0x04] = 0 ; [edx] = 0 ; +0x00/+0x04 list links
PC:             mov [edx+0x14] = 0             ; +0x14  program counter
gosub stack:    lea edi,[edx+0x18]; ecx=8; rep stosd   ; +0x18  8 dwords
stack depth:    mov [edx+0x38] = 0 (word)      ; +0x38
locals:         lea edi,[edx+0x3C]; ecx=0x22; rep stosd ; +0x3C  34 dwords
flags:          mov [edx+0xC4..0xC8] = 0
                mov [edx+0xC9] = 0xFF
                mov [edx+0xCC] = 0 ; [edx+0xD0]=0(word); [edx+0xD2]=0
                mov [edx+0xD3] = 1
                mov [edx+0xD4]=0 ; [edx+0xD8]=0 ; [edx+0xDC]=0
```

## 2. The field map

Combining `Init` with the accessors (§3) gives the complete 224-byte object:

| Offset | Size | Field | Read from |
|---|---|---|---|
| `+0x00` | 4 | `next` (intrusive list) | `AddScriptToList` / `RemoveScriptFromList` |
| `+0x04` | 4 | `prev` (intrusive list) | same |
| `+0x08` | 8 | `name[8]` | `Init` (copies `0x859D44`) |
| `+0x10` | 4 | base pointer (script code base) | `UpdatePC` |
| `+0x14` | 4 | **program counter (current IP)** | `UpdatePC`, every var accessor |
| `+0x18` | 32 | **gosub/return stack** (8 dwords) | `Init` (`rep stosd` 8) |
| `+0x38` | 4 | stack depth (word) + pad | `Init` |
| `+0x3C` | 136 | **local variables** (34 dwords) | `Init` (`rep stosd` 0x22); accessors |
| `+0xC4` | 24 | condition/state flag bytes | `Init` (`+0xC9=0xFF`, `+0xD3=1`, …) |
| `+0xDC` | 4 | **mission-LVAR-mode flag** (byte) + tail | `GetPointerToLocalVariable` |

The tiling closes: the local array ends at `0x3C + 34×4 = 0xC4`, exactly where the flag bytes begin, and
the flag block runs to `0xE0` = **224**. No gap, no field without a home.

## 3. The accessors that pin the runtime fields

Two fields `Init` only zeroes are given their *meaning* by the methods that use them:

**Program counter (+0x14).** `CRunningScript::UpdatePC` (`0x00464DA0`) is the clearest:

```
015620B8  mov  edx, dword ptr [ecx + 0x10]   ; base pointer
015620BB  sub  edx, eax                        ; base - offset
015620BD  mov  dword ptr [ecx + 0x14], edx     ; -> new PC
```

`+0x10` is the base of the script's code and `+0x14` is the live instruction pointer; `UpdatePC` computes an
absolute address from a signed offset (the SCM `gosub`/`goto` operand form). Every variable accessor
(`GetIndexOfGlobalVariable`, `ReadArrayInformation`) reads the bytecode through `[ecx+0x14]` and advances it,
confirming `+0x14` as the PC.

**Intrusive list links (+0x00 / +0x04).** `AddScriptToList` (`0x00464C00`) and `RemoveScriptFromList`
(`0x00464BD0`) splice the object into a doubly-linked list using `[this]` (next) and `[this+4]` (prev) — the
same intrusive-list idiom C28.3 found in the streaming render list, here threading active scripts together
(the list heads `CTheScripts` manages in [C28.2](../C28-Class-Catalogue/02-cthescripts-and-the-script-object.md)).

## 4. The gosub stack and the timers

The **8-dword region at +0x18** is the `gosub`/`return` stack: SCM's `gosub` opcode pushes the return PC
here and `return` pops it, so a script can nest subroutine calls eight deep. Its depth counter is the word at
`+0x38`. The **34-dword local array at +0x3C** is the SCM convention in the flesh: 32 general local variables
followed by 2 timers (`TIMERA`/`TIMERB`, the auto-incrementing locals every mission uses) — `32 + 2 = 34 =
0x22`, exactly `Init`'s `rep stosd` count. C32.2 takes the variable system further.

---

### Key takeaways

- The 224-byte `CRunningScript` is fully mapped and **tiles exactly** (0x00→0xE0, `0x3C + 34×4 = 0xC4`).
- Runtime state: **base pointer +0x10, program counter +0x14, 8-entry gosub stack +0x18**, stack depth
  +0x38, **34-dword local array +0x3C**, flag block +0xC4→0xE0.
- The PC and list links, which `Init` only zeroes, are pinned by `UpdatePC` and the list splice methods.

**Next:** [C32.2 — The variable system, and why 0xA48960 exists](02-the-variable-system.md).
