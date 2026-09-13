# C30.3 — CShopping and the 560-item Ledger

> **The one-sentence version:** the shop system tracks what the player has bought in a pair of parallel
> **560-entry** arrays — an id list and a flag list — and the size is proven twice, once by the lookup
> loop's `cmp …, 0x230` bound and once by the shutdown clearing `0x8C` dwords (= 560 bytes) of the flag
> array; alongside sit a 24-byte shop-section record and a 30-dword clothes-state save buffer.

**Subsystem category:** Gameplay features — shops and purchases
**Depends on:** [C30 hub](C30-Gameplay-Managers.md), [C27.3](../C27-Function-Catalogue/03-verification-and-the-remaining-124.md)
(which disassembled `CShopping::GetExtraInfo` as a bounds-checked array lookup)
**RE status:** Documented
**Confidence:** ✅ for the 560-item arrays, the record stride and the clothes buffer · 🔷 for methods not
disassembled here

---

## 1. The purchase ledger, sized two ways

`CShopping::HasPlayerBought` (`entry_va 0x0049B5E0`) answers whether an item id has been purchased by
scanning an id array, then reading a parallel flag byte:

```
0156B776  cmp  dword ptr [eax*4 + 0xA97D90], ecx  ; id array entry == target id?
0156B77E  je   found
0156B780  inc  eax
0156B784  cmp  eax, 0x230                          ; 560 entries
0156B78C  jl   loop
...
0156B78F  mov  al, byte ptr [eax + 0xA972A0]       ; the found item's bought flag
```

So there are **560 (`0x230`) shop items**: an id array of 560 dwords at `0xA97D90` and a parallel
**bought-flag byte array at `0xA972A0`**. The count is confirmed independently by
`CShopping::ShutdownForRestart` (`0x0049B640`), which resets the flags:

```
01562500  mov  ecx, 0x8C                   ; 140 dwords ...
01562507  mov  edi, 0xA972A0               ; ... of the flag array
0156250C  rep stosd                         ; = 560 bytes cleared
```

`0x8C × 4 = 560` — the clear covers exactly one byte per item, so the lookup bound (`0x230` = 560) and the
clear size (`0x8C` dwords = 560 bytes) agree to the item. Two independent numbers closing on 560 make the
item count ✅. `SetPlayerHasBought` (`0x0049B610`) writes the same flag array.

## 2. The 24-byte shop-section record — field layout

Shop contents are parsed from a loaded data structure of fixed-width records. `GetExtraInfo`
(`entry_va 0x0049ADE0`) — the method [C27.3](../C27-Function-Catalogue/03-verification-and-the-remaining-124.md)
already spot-checked as a bounds-checked array lookup — steps that structure with `add ecx, 0x18`, a
**24-byte record stride**; `GetNameTag` (`0x0049ADA0`) uses the same `add ecx, 0x18`. The RE-Data sweep
recovered the four fields inside that 24-byte record:

```cpp
struct CShoppingRecord {       // 24 bytes (0x18)
    uint32_t key;              // +0   item key hash — used to look up prices in ms_prices[]
    uint32_t price;            // +4   base item price in dollars (before zone multiplier)
    uint32_t extra_data[2];    // +8   item-type union:
                               //        weapon   → { ammo, 0 }
                               //        clothes  → { modelKey, textureKey }
                               //        tattoo   → { type1, texKey }
                               //        general  → { extra1, extra2 }
    char     nameTag[8];       // +16  null-terminated display name tag (max 8 chars)
};
static_assert(sizeof(CShoppingRecord) == 0x18);
```

`GetNameTag` reads `nameTag` at `+16`; `GetKey` reads `key` at `+0`; `GetExtraInfo` returns the
`extra_data` union at `+8`. The `price` field at `+4` is read by the purchase screen before the game
applies the zone-based `gPriceMultipliers` modifier (a float array indexed by interior zone ID). The
section walkers (`FindSection`, `GetNextSection`, `GetKey`, `GetItemIndex`) all navigate this same table —
the stride `0x18` is the single authoritative constant tying them together.

## 3. The clothes-state buffer

Buying clothes changes the player model's appearance, and that state is saved/restored around shop visits.
`StoreClothesState` (`entry_va 0x0049B200`):

```
0049B200  movzx eax, byte ptr [0xB7CD74]   ; player index
          imul eax, eax, 0x190             ; * 400  (player-info array stride)
          mov  ecx, [eax + 0xB7CD98]       ; player-info pointer
          ...  mov ecx, 0x1E               ; 30 dwords
          mov  edi, 0xA9A810               ; clothes-state buffer
          rep movsd                         ; copy 120 bytes out
```

The clothes-state buffer is **30 dwords (120 bytes) at `0xA9A810`**; `RestoreClothesState` (`0x0049B240`)
copies it back the other way. Incidentally this discloses a fact that belongs to another subsystem — the
player-info array at `0xB7CD98` has a **400-byte (`0x190`) stride** — noted here as a byproduct (the way
C27.3 got `CObject`'s stride for free), not claimed as a CShopping structure.

## 4. The rest of the class

`RemoveLoadedShop` and the section navigators complete the loaded-shop parser; `UpdateStats` folds
purchases into the stats system. Full list in
[`gameplay_managers.json`](../RE-Data/data/gameplay_managers.json).

---

### Key takeaways

- The shop tracks **560 items (`0x230`)** in parallel arrays — id list at `0xA97D90`, bought-flag bytes at
  `0xA972A0` — the count proven by both the lookup bound and the shutdown clear (560 bytes).
- A shop-section record is **24 bytes (`0x18`)**, the stride shared by `GetExtraInfo`/`GetNameTag`.
- The clothes-state save buffer is **30 dwords at `0xA9A810`**; the player-info array's 400-byte stride
  falls out as a byproduct.

**Next:** back to the [C30 hub](C30-Gameplay-Managers.md). With Pickups, Garages, Entry/Exits, Replay and
Shopping done (C29–C30), the gameplay-manager cluster from [C28.1 §3](../C28-Class-Catalogue/01-the-class-map-by-subsystem.md#3-which-classes-already-have-a-home-and-which-are-candidates)
is closed but for `CGangWars` (flag-only) — the next leads are C27's 124 unnamed functions, or the 356
🔷 names awaiting disassembly.
