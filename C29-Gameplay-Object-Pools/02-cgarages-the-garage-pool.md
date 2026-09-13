# C29.2 — CGarages and the 216-byte Garage

> **The one-sentence version:** `CGarages` owns **50 garage records of 216 bytes each at `0x96C048`** — the
> stride is written into five different methods and the count closes to the byte — and three adjacent bytes
> late in each record hold the garage's **type (+0x4C), door state (+0x4D) and flags (+0x4E)**, with the
> door state driven by a small machine (closed → opening → open → closing) that six accessor methods read
> and write consistently.

**Subsystem category:** Gameplay features — garage pool
**Depends on:** [C29 hub](C29-Gameplay-Object-Pools.md), [C29.1](01-cpickups-the-pickup-pool.md)
**RE status:** Documented
**Confidence:** ✅ for the pool geometry, the three field offsets and the door-state transitions (all
re-checked by the tool) · 🔷 for methods not disassembled here

---

## 1. The garage record, sized five ways

`CGarages::ChangeGarageType` (`entry_va 0x004476D0`) indexes the pool and writes two of its bytes:

```
01561699  imul eax, eax, 0xD8              ; index * 216
0156169F  add  eax, 0x96C048               ; + pool base
015616A7  mov  byte ptr [eax + 0x4C], cl   ; type byte
...
015616B8  mov  byte ptr [eax + 0x4D], cl   ; door-state byte
```

Stride `0xD8` = **216 bytes**, base `0x96C048`. The same `imul …, 0xD8` + base appears in
`DeActivateGarage` (`0x00447CB0`), `ActivateGarage` (`0x00447CD0`), `IsGarageOpen` (`0x00447D00`) and
`SetTargetCarForMissionGarage` (`0x00447C40`); and `Shutdown` (`0x004471B0`) walks the array with
`add esi, 0xD8` from `0x96C098` (= base + 0x50) to `0x96EAC8`. Five independent witnesses to the 216-byte
stride. The count is the walk length, and it closes:

```
(0x96EAC8 − 0x96C048) / 0xD8 = 0x2A30 / 0xD8 = 50 garages
```

So the engine reserves **50 garage slots**, the known limit — recovered here purely as `(end − base) /
stride` with no residue. How many are actually defined at runtime is a separate live count in the global
`0x96C024`, which `GetGarageNumberByName` (`0x00447680`) uses as its loop bound while linear-searching the
pool by name.

## 2. The type / state / flags byte triple

Three consecutive bytes near offset 0x4C carry the garage's mutable state, each pinned by the method that
touches it:

| Offset | Field | Evidence |
|---|---|---|
| **+0x4C** | garage **type** | `ChangeGarageType` writes it; `ActivateGarage` gates on `cmp [eax+0x4C], 0xB` (type 11) |
| **+0x4D** | **door state** | written by `ChangeGarageType`, `OpenThisGarage`, `CloseThisGarage`, `ActivateGarage`; read by `IsGarageOpen`, `IsGarageClosed` |
| **+0x4E** | **flags** | `DeActivateGarage` does `or [eax+0x4E], 2` (set bit 1); `ActivateGarage` does `and [eax+0x4E], 0xFD` (clear it) |

The flags bit is a clean pair: deactivating a garage *sets* bit 1 at +0x4E and activating one *clears* it —
the two methods are exact inverses on that bit, which is what makes the offset and the bit both ✅ rather
than guessed.

## 3. The door state machine

The door-state byte at +0x4D is a small finite-state machine. Reading the six methods that touch it gives
every transition and both terminal tests:

- **`IsGarageClosed`** (`0x00447D30`) reads the byte via its absolute address `0x96C095` (`= 0x96C048 +
  0x4D`, confirming base *and* offset a sixth way) and returns true when it is **0** — so **0 = fully
  closed**.
- **`IsGarageOpen`** (`0x00447D00`) returns true when the byte is **1 or 4** — so **1/4 = open**.
- **`CloseThisGarage`** (`0x00447D70`): from **{1, 3} → 2**. So **2 = closing** (a door that is open or
  opening begins to close).
- **`OpenThisGarage`** (`0x00447D50`): from **{0, 2, 5} → 3**. So **3 = opening** (a door that is closed or
  closing begins to open).
- **`ActivateGarage`** (`0x00447CD0`): if the door is at rest (**0**) and the type is 11, kick it to **3**
  (opening).

That assembles into the expected cycle — **0 closed → 3 opening → {1,4} open → 2 closing → 0 closed** — read
entirely from the transitions, with `IsGarageOpen`/`IsGarageClosed` as the two terminal predicates. The
exact roles of the minor states 4 and 5 (a second "open" variant and an "open-waiting" value) are 🟡: their
values are certain, their precise distinction is not traced here.

## 4. The rest of the class

The remaining methods (🔷) are garage gameplay behaviour on top of the pool: the respray/paint logic
(`IsCarSprayable`, which rejects a fixed list of model ids, and `AllRespraysCloseOrOpen`), mission-garage
control (`SetTargetCarForMissionGarage` writes the target-car pointer at garage+0x40, `0x96C088 = base +
0x40`), the on-screen garage-name message with its 500-byte (`0x1F4`) text region at `0x96C00C…0x96C020`
(`TriggerMessage`, the class's most-called method at 17 callers), hideout inventory
(`CountCarsInHideoutGarage`), and the arrest/death cleanup (`PlayerArrestedOrDied`). Full list in
[`gameplay_pools.json`](../RE-Data/data/gameplay_pools.json).

---

### Key takeaways

- The garage pool is **50 records × 216 bytes at `0x96C048`** — stride seen five ways, count `(end −
  base)/stride = 50` closing to the byte, matching the engine's known limit without being taken from it.
- Each garage's mutable state is three adjacent bytes: **type +0x4C, door-state +0x4D, flags +0x4E**, each
  pinned by the method that writes it; the +0x4E flag bit is proven by activate/deactivate being exact
  inverses on it.
- The **door state machine** reads out completely: **0 closed → 3 opening → {1,4} open → 2 closing → 0**,
  with `IsGarageClosed` (==0) and `IsGarageOpen` (==1/4) as the terminal tests.
- `IsGarageClosed` reaching +0x4D via the absolute address `0x96C095` confirms base and offset a sixth,
  independent way.

**Next:** back to the [C29 hub](C29-Gameplay-Object-Pools.md), or on to the rest of the gameplay-manager
cluster (`CReplay`, `CShopping`, `CGangWars`, `CEntryExitManager`) as further chapters.
