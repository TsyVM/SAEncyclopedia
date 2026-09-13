# C54.4 — Exact patch bytes for every limit

## How to read this page

Each patch is described as:
- **Target:** what the patch changes (the pool slot count or comparison value)
- **Patch VA:** the virtual address of the byte(s) to overwrite in `gta_sa.exe`
- **File offset:** the flat-file offset in the exe (VA − 0x400000 + file header offset; for HOODLUM 1.0 US, use `VA - 0x400000 + 0x1000`)
- **Original bytes:** what the byte sequence currently is at this address
- **Meaning:** what those bytes decode to in x86
- **How to change:** which bytes to overwrite and to what

All values verified by Capstone disassembly of the HOODLUM 1.0 US `gta_sa.exe` (MD5: `170b3a9108687b26da2d8901c6948a18`).

---

## CPed pool — 140 → custom count

**Patch VA:** `0x5503F1`  
**File offset:** `0x1503F1` (approx — verify against the exe: the byte at offset should be `8C 00 00 00`)

**Original instruction:**
```
0x5503EF:  68 8C 00 00 00   push 0x0000008C    ; push 140
0x5503F4:  68 ...           push string_ptr
```

The count is a 32-bit little-endian immediate in the `push imm32` instruction. The `68` opcode is followed by 4 bytes of the count value.

**To change to 200 (0xC8):**
```
Original: 68 8C 00 00 00
Patched:  68 C8 00 00 00
```

**To change to 250 (0xFA):**
```
Patched:  68 FA 00 00 00
```

Only change bytes 1–4 of the instruction (the 4-byte count). Leave the `68` opcode byte untouched.

---

## CVehicle pool — 110 → custom count

**Patch VA:** `0x550429`  
**Original instruction:**
```
0x550427:  68 6E 00 00 00   push 0x0000006E    ; push 110
```

**To change to 160 (0xA0):**
```
Patched:  68 A0 00 00 00
```

---

## CBuilding pool — 13000 → custom count

**Patch VA:** `0x55045E`  
**Original instruction:**
```
0x55045C:  68 C8 32 00 00   push 0x000032C8    ; push 13000
```

0x32C8 = 13000. Little-endian, so bytes are: `C8 32 00 00`.

**To change to 20000 (0x4E20):**
```
Patched:  68 20 4E 00 00
```

---

## CObject pool — 350 → custom count

**Patch VA:** `0x550496`  
**Original instruction:**
```
0x550494:  68 5E 01 00 00   push 0x0000015E    ; push 350
```

0x15E = 350. Bytes: `5E 01 00 00`.

**To change to 1000 (0x3E8):**
```
Patched:  68 E8 03 00 00
```

---

## ms_fTimeStep max clamp — 2.0 → custom

**Patch VA (rdata):** `0x858C14`  
This is a float constant in the `.rdata` section, not a code instruction. The `fcomp` at `0x560DA2` loads from this address:

```
0x560DA2:  D8 1D 14 8C 85 00   fcomp dword ptr [0x858C14]
```

The value at `0x858C14` is the IEEE-754 float `2.0`:
```
Original bytes: 00 00 00 40   (IEEE-754 LE = 2.0)
```

**To change to 3.0:**
```
Patched:  00 00 40 40   (IEEE-754 LE = 3.0)
```

**IEEE-754 reference for common values:**

| Value | Bytes (LE) |
|---|---|
| 0.5 | `00 00 00 3F` |
| 1.0 | `00 00 80 3F` |
| 2.0 | `00 00 00 40` |
| 3.0 | `00 00 40 40` |
| 4.0 | `00 00 80 40` |

---

## Vehicle density cap — 300 → custom

**Patch VA:** `0x49B912`  
**Original instruction:**
```
0x49B90F:  81 F9 2C 01 00 00   cmp ecx, 0x0000012C    ; cmp ecx, 300
```

The immediate is at bytes 2–5 of this 6-byte instruction: `2C 01 00 00` (300 LE).

**To change to 435 (proportional increase for pool 110→160):**
0x1B3 = 435. Bytes: `B3 01 00 00`.
```
Patched:  81 F9 B3 01 00 00
```

---

## Ped density cap A — 63 → custom

**Patch VA:** `0x52953B`  
This is an 8-bit `cmp` form:
```
0x52953B:  83 F8 3F   cmp eax, 0x3F    ; cmp eax, 63
```

The `83 F8` opcode is a `cmp reg32, imm8`. The immediate is the single byte `3F`.

**To change to 90 (proportional for 140→200):**
0x5A = 90.
```
Patched:  83 F8 5A
```

---

## Ped density cap B — 53 → custom

**Patch VA:** `0x529C11`  
```
0x529C11:  83 F8 35   cmp eax, 0x35    ; cmp eax, 53
```

**To change to 76:**
0x4C = 76.
```
Patched:  83 F8 4C
```

---

## Ped density cap C — 80 → custom (two sites)

**Site 1 — Patch VA:** `0x575227`  
**Site 2 — Patch VA:** `0x582F0C`  

Both sites use the same comparison value. Verify the instruction bytes at each site before patching — the immediate may be in a different position in the instruction depending on the comparison register and form.

**To change to 114 (proportional for 140→200):**
0x72 = 114.

Patch both sites. A missing patch at one site means the density cap is inconsistently applied across the two callsites.

---

## Applying patches as an ASI mod

The cleanest approach for limit patches is a startup hook that writes the new bytes before `CGame::Initialise` is called, which in turn calls `CPools::Initialise` at `0x5503A0`.

```cpp
// Example ASI hook skeleton (pseudocode)
void ApplyLimitPatches() {
    // Unprotect page containing CPools::Initialise
    DWORD old;
    VirtualProtect((void*)0x5503A0, 0x700, PAGE_EXECUTE_READWRITE, &old);

    // Raise CPed pool to 200
    *(uint32_t*)(0x5503F1) = 200;

    // Raise CVehicle pool to 160  
    *(uint32_t*)(0x550429) = 160;

    // Raise CBuilding pool to 20000
    *(uint32_t*)(0x55045E) = 20000;

    // Raise CObject pool to 1000
    *(uint32_t*)(0x550496) = 1000;

    // Restore page protection
    VirtualProtect((void*)0x5503A0, 0x700, old, &old);
}
```

Call `ApplyLimitPatches()` from `DllMain` at `DLL_PROCESS_ATTACH` — this runs before the game's `WinMain` begins, which guarantees it runs before `CGame::Initialise`.

For the density cap and float constant patches, same approach — just target the appropriate VAs.

**Previous:** [C54.3 — Breaking limits safely](03-breaking-limits-safely.md)  
**Continue:** [C54.5 — How the density algorithm works →](05-density-algorithm.md)
