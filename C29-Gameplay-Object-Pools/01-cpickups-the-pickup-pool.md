# C29.1 — CPickups and the 620-slot Pool

> **The one-sentence version:** `CPickups` owns a flat array of **620 pickup records, 32 bytes each, at
> `0x9788C4`** — the stride, base and count are all read from its own methods and the count closes to the
> byte — and the class's cleverness is a **packed handle** that carries a per-slot *reuse counter* in its
> high word so a stale reference to a recycled slot can be detected, plus a small **ring buffer** that
> remembers the last 20 pickups the player collected.

**Subsystem category:** Gameplay features — pickup pool
**Depends on:** [C29 hub](C29-Gameplay-Object-Pools.md), [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md)
(disassembly method)
**RE status:** Documented
**Confidence:** ✅ for the pool geometry and the handle scheme (re-checked by the tool) · 🟡 for individual
field *meanings* read from a single use site · 🔷 for methods not disassembled here

---

## 1. The pool, sized four ways

`CPickups::FindPickUpForThisObject` (`entry_va 0x004551C0`) scans the whole pool for the slot holding a
given world object, and its loop names every geometric constant at once:

```
0156D876  xor  eax, eax
0156D879  mov  ecx, 0x9788C4               ; pool base
0156D87F  mov  bl, byte ptr [ecx + 0x18]   ; slot's in-use byte
0156D882  test bl, bl
0156D884  je   +6                          ; empty slot -> skip
0156D886  cmp  dword ptr [ecx], edx        ; slot's object ptr == target?
0156D88A  add  ecx, 0x20                    ; next record = 32 bytes
0156D88E  cmp  ecx, 0x97D644               ; end of pool
0156D894  jl   loop
```

Base `0x9788C4`, stride `0x20` = **32 bytes**, end `0x97D644`. The count is the division, and it closes:

```
(0x97D644 − 0x9788C4) / 0x20 = 0x4D80 / 0x20 = 620 pickups
```

The same 32-byte stride shows up three more times, in three unrelated forms — `Init` (`0x00454A70`) walks
the array with `add eax, 0x20` up to `0x97D65A`; `GetActualPickupIndex` (`0x004552A0`) and
`UpdateMoneyPerDay` (`0x00455680`) both convert an index with `shl …, 5` (multiply by 32). Four independent
witnesses to one stride: the width is ✅, not inferred. The **in-use byte at record +0x18** is the slot's
occupancy/type flag — `FindPickUpForThisObject` treats a zero there as an empty slot to skip.

## 2. The packed handle, and why it carries a counter

A pickup is referred to from elsewhere (script, object) not by a bare index but by a **packed 32-bit
handle**. `CPickups::GetActualPickupIndex` (`entry_va 0x004552A0`) unpacks and *validates* one:

```
01564BBB  and  eax, 0xFFFF                 ; low word  = slot index
01564BC2  shl  edx, 5                        ; index * 32
01564BC5  shr  ecx, 0x10                     ; high word = expected counter
01564BC8  cmp  cx, word ptr [edx + 0x9788DA] ; == slot's stored counter?
01564BCF  je   valid                         ; else return -1
```

So a handle is `{high word = counter, low word = index}`, and each slot stores a **`uint16` counter at
record +0x16** (`0x9788DA − 0x9788C4 = 0x16`). When a slot is reused for a new pickup its counter is
bumped, so an old handle whose high word no longer matches the slot's current counter resolves to −1
instead of silently pointing at the wrong pickup. This is the classic *generational handle* pattern, read
straight from the compare: the low word indexes the array, the high word guards against slot reuse. It is
the same defensive idea as C28.2's `-1` "no id" sentinels, one level more sophisticated.

`AddToCollectedPickupsArray` (`0x00455240`) builds a handle the other way — `shl ecx, 5` to reach the slot,
`movzx …, word [slot + 0x9788DA]` to read the counter, `shl edx, 0x10` to put it in the high word, `or` in
the index — confirming the packing from the encode side. A second `uint16` field, at **record +0x12**
(`0x9788D6`), is the pickup's money-per-day value, written by `UpdateMoneyPerDay`.

## 3. The collected-pickups ring buffer

The game remembers the last few pickups the player grabbed, in a small wrap-around buffer.
`AddToCollectedPickupsArray` (`entry_va 0x00455240`):

```
01561595  mov  ax, word ptr [0x978624]      ; write cursor
0156159E  inc  ax
015615A0  cmp  ax, 0x14                       ; == 20 ?
015615A4  mov  dword ptr [ecx*4 + 0x978628], edx  ; store handle at cursor
015615AB  mov  word ptr [0x978624], ax
015615B1  jb   done
015615B3  mov  word ptr [0x978624], 0         ; wrap back to 0
```

A **20-entry (`0x14`) ring of dwords at `0x978628`**, with a `uint16` write cursor at `0x978624` that wraps
at 20. `CPickups::Init` (`0x00454A70`) zeroes exactly this — `mov ecx, 0x14; mov edi, 0x978628; rep stosd`
— so the buffer's size is asserted from both its writer and its initialiser. `IsPickUpPickedUp`
(`0x00454B40`) scans the same 20 slots (`cmp eax, 0x14`) to answer whether a given pickup handle is in the
recently-collected set.

## 4. The rest of the class

The remaining methods (🔷, C27-inherited) are the pickup lifecycle and weapon-pickup rules, coherent with
the pool above: creation (`GenerateNewOne_WeaponType`), the weapon→model and model→ammo mappings
(`ModelForWeapon`, and `CPickup::ExtractAmmoFromPickup`, which reads a pickup's model word at object+0x22,
type byte at +0x1C and flag byte at +0x1D), and the pickup-eligibility gate
(`PlayerCanPickUpThisWeaponTypeAtThisMoment`). Full list in
[`gameplay_pools.json`](../RE-Data/data/gameplay_pools.json) and
[`class_catalogue.json`](../RE-Data/data/class_catalogue.json).

---

### Key takeaways

- The pickup pool is **620 records × 32 bytes at `0x9788C4`** — stride seen four ways, count `(end −
  base)/stride = 620` closing to the byte, matching the engine's known limit without being taken from it.
- A pickup **handle packs `{counter<<16 | index}`**; each slot stores a `uint16` reuse counter at +0x16, so
  a stale handle to a recycled slot resolves to −1 — a generational-handle guard read from the validating
  compare.
- Per-slot fields recovered: in-use byte **+0x18**, money-per-day word **+0x12**, counter word **+0x16**.
- The **collected-pickups ring** is 20 dwords at `0x978628` with a wrapping cursor at `0x978624`, size
  agreed by its writer and its initialiser.

**Next:** [C29.2 — CGarages and the 216-byte garage](02-cgarages-the-garage-pool.md).
