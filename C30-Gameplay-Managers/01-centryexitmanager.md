# C30.1 — CEntryExitManager and the 60-byte Marker

> **The one-sentence version:** the interior entry/exit markers (the doorways that teleport the player
> between the outside world and an interior) are **60-byte records** held in a templated pool, and the size
> is proven twice over — the index→address multiply is `× 0x3C`, and the address→index divide in `DeleteOne`
> uses the compiler's reciprocal for ÷60 — while a separate 32-slot array holds the markers currently
> visible around the player.

**Subsystem category:** Gameplay features — interior markers
**Depends on:** [C30 hub](C30-Gameplay-Managers.md), [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md)
**RE status:** Documented
**Confidence:** ✅ for the record size, the pool layout and the visible-set size · 🔷 for methods not
disassembled here

---

## 1. The 60-byte record, proven both ways

Six `CEntryExitManager` methods index the entry/exit array with the same multiply — `PostEntryExitsCreation`,
`EnableBurglaryHouses`, `DeleteOne`, `Shutdown`, `ShutdownForRestart` all contain `imul …, 0x3C`. Stride
`0x3C` = **60 bytes**. `DeleteOne` (`entry_va 0x0043FD50`) then does the *inverse* — recover an index from a
record pointer — and the constant it uses confirms the same 60:

```
01560F06  sub  esi, dword ptr [ecx]        ; ptr - array base = byte offset
01560F0C  mov  eax, 0x88888889             ; reciprocal of 60
01560F11  imul esi                          ; 64-bit multiply
01560F13  ...  sar 5 / shr 0x1F             ; -> index
```

`0x88888889` with the trailing `sar 5` is exactly the code a compiler emits to divide a signed integer by
**60**. A `× 60` to address a record and a magic `÷ 60` to invert it — the same two-independent-derivations
proof used for C28.2's 224-byte script object and C29's pool strides. The record size is ✅.

## 2. The pool object

Entry/exits are not a bare array but a **templated pool object** whose address is the global `0x96A7D8`.
`DeleteOne` and `EnableBurglaryHouses` (`0x0043F180`) read its three fields:

```
mov  ecx, dword ptr [0x96A7D8]   ; the pool object
mov  eax, dword ptr [ecx]        ; +0  = array base pointer (records)
mov  eax, dword ptr [ecx + 4]    ; +4  = per-slot flag-byte array
mov  esi, dword ptr [ecx + 8]    ; +8  = slot count
```

So the pool is the familiar `CPool` shape — a pointer to the packed 60-byte record array, a parallel array
of one status byte per slot (the high bit marks a free slot; `DeleteOne` tests `js` on it), and a count.
This is the same pool template `CObjectPool` uses in [C27.3 §3](../C27-Function-Catalogue/03-verification-and-the-remaining-124.md#3),
here instantiated for `CEntryExit`. The number of markers is therefore dynamic (whatever the loaded map
defines), not a fixed compile-time array like C29's pickups and garages — a real structural distinction, and
one the disassembly makes unambiguous.

## 3. The 32-slot visible set

Separate from the pool, the manager keeps a small working set of the markers currently near the player.
`SetAreaCodeForVisibleObjects` (`entry_va 0x0043ECF0`) fills it:

```
0156095D  ...                              ; for each nearby object
0156096A  cmp  edx, 0x20                   ; stop at 32
0156097A  mov  dword ptr [edx*4 + 0x96A738], ecx  ; store into the array
01560981  inc  edx
01560982  mov  dword ptr [0x96A7DC], edx   ; live count
```

A **32-entry pointer array at `0x96A738`**, with its live count at `0x96A7DC`. The `cmp …, 0x20` bound caps
it at 32 — a fixed-size scratch list refilled each time the player's area code changes, distinct from the
unbounded pool. `ResetAreaCodeForVisibleObjects` (`0x0043ED80`) clears it back out.

## 4. The rest of the class

The remaining methods (🔷) are the marker lifecycle and geometry: creation and teardown (`Init`,
`PostEntryExitsCreation`, `Shutdown`, `ShutdownForRestart`), the outside-world coordinate transform
(`GetPositionRelativeToOutsideWorld`), the linked stack of nested interiors (`AddEntryExitToStack`), and the
single `CEntryExit` element method (`GetEntryExitToDisplayNameOf`, the marker's GXT display name). Full list
in [`gameplay_managers.json`](../RE-Data/data/gameplay_managers.json).

---

### Key takeaways

- The `CEntryExit` record is **60 bytes (`0x3C`)** — proven by the index-multiply in six methods and the
  ÷60 reciprocal (`0x88888889`) in `DeleteOne`.
- Markers live in a **templated pool** at `0x96A7D8` (record-array pointer +0, per-slot flag bytes +4,
  count +8) — a dynamic count, unlike C29's fixed pickup/garage arrays.
- A separate **32-slot visible-objects array** at `0x96A738` (count at `0x96A7DC`) holds the markers near
  the player, refilled per area-code change.

**Next:** [C30.2 — CReplay and the 800 KB buffer](02-creplay.md).
