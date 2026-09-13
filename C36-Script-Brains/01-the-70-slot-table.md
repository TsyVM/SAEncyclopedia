# C36.1 — The 70-slot Table

> **The one-sentence version:** the attractor-brain registry is a fixed array of **70 records of 20 bytes**
> embedded in the manager object — the count `0x46` appears in three separate methods and the 20-byte stride
> in all of them — with a free slot marked by a `0xFFFF` model id, and `Init` lays out the record field by
> field.

**Subsystem category:** Scripting — table layout
**Depends on:** [C36 hub](C36-Script-Brains.md), [C32.1](../C32-CRunningScript-Object/01-the-object-layout.md)
(the this-relative reading style)
**RE status:** Documented
**Confidence:** ✅ for the count, stride and sentinel · 🟡 for the individual record fields

---

## 1. 70 slots, 20 bytes — proven three ways

`CScriptsForBrains::Init` (`entry_va 0x0046A8C0`) constructs the table:

```
0156BB71  lea  eax, [ecx + 0xE]             ; ecx = this; eax -> first record + 0xE
0156BB78  mov  edx, 0x46                     ; 70 records
0156BB80  ...  (write the record's fields)   ; via [eax-0xE]..[eax+2]
0156BBA1  add  eax, 0x14                      ; next record = 20 bytes
0156BBA4  dec  edx
0156BBA5  jne  0x156BB80
```

**70 (`0x46`) records, stride `0x14` = 20 bytes.** `SwitchAllObjectBrainsWithThisID` (`0x0046A900`) walks
the same array — `mov ecx, 0x46; … add eax, 0x14` — and `AddNewStreamedScriptBrainForCodeUse` (`0x0046A9C0`)
bounds its search at the same `cmp dl, 0x46`. Three independent methods agree on 70, so the count is ✅. The
20-byte stride shows up in the search methods as an address computation rather than a bare `add`:

```
015628A6  lea  eax, [eax + eax*4]            ; index * 5
015628AC  cmp  word ptr [ecx + eax*4], -1    ; index*5*4 = index*20; id word == -1?
```

`index × 5 × 4 = index × 20` reaches record `index` from the object base `ecx`, and the compare reads the
record's first word — its **model id** — against `-1`.

## 2. The 0xFFFF free-slot convention

`AddNewStreamedScriptBrainForCodeUse` finds a slot by scanning for an id word of `-1` (`0xFFFF`):

```
for index in 0..70:
    if word[this + index*20] == -1:   ; free slot
        use it
```

So a slot is **free when its id word is `0xFFFF`**, the same "no model id" sentinel the pickup pool
([C29.1](../C29-Gameplay-Object-Pools/01-cpickups-the-pickup-pool.md)) and the script-blocked model lists
([C28.2](../C28-Class-Catalogue/02-cthescripts-and-the-script-object.md)) use — an engine-wide idiom. `Init`
sets every slot's id to `-1` (`or ecx, 0xFFFFFFFF; mov word [eax-0xE], cx`) to start the table empty.

## 3. The 20-byte record

`Init`'s stores, taken relative to the record base (`eax = record + 0xE`), give the field layout:

| Offset | Size | Field | Init value |
|---|---|---|---|
| `+0x00` | 2 | **model id** (`0xFFFF` = free) | `-1` |
| `+0x02` | 1 | flag byte | `-1` |
| `+0x03` | 1 | flag byte | `-1` |
| `+0x04` | 1 | enabled/active byte | `1` |
| `+0x08` | 4 | dword (from the `0x859D9C` template) | template |
| `+0x0C` | 2 | word | `0` |
| `+0x0E` | 2 | word | `0` |
| `+0x10` | 4 | dword | `0` |

The `+0` id and the `+4` enabled byte are the load-bearing fields (the search keys); the middle dword copied
from `0x859D9C` is a per-record default the class stamps in at init. The exact meaning of the two flag bytes
and the trailing word/dword fields is left 🟡 — their positions are certain from `Init`, their runtime roles
not traced here.

---

### Key takeaways

- The attractor-brain table is **70 records × 20 bytes**, count `0x46` proven in three methods, stride `×20`
  reached as `index×5×4`.
- A slot is **free when its id word is `0xFFFF`** — the engine-wide "no model id" sentinel; `Init` empties
  the table by setting all 70 to `-1`.
- The 20-byte record's fields are mapped from `Init` (id +0, enabled +4, plus flag/word/dword fields), with
  the two search keys ✅ and the remainder 🟡.

**Next:** [C36.2 — Object brains vs streamed brains](02-brains-and-scripts.md).
