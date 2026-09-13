# Chapter 30 — Gameplay Managers: Entry/Exits, Replay, Shopping

> **Goal of this chapter:** finish the [C28.1 §3](../C28-Class-Catalogue/01-the-class-map-by-subsystem.md#3-which-classes-already-have-a-home-and-which-are-candidates)
> gameplay-manager cluster that [C29](../C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md) began.
> Three more managers, three more structures read straight from `gta_sa.exe`: `CEntryExitManager`'s
> **60-byte pooled interior markers**, `CReplay`'s **8 × 100,000-byte** replay buffer, and `CShopping`'s
> **560-item** purchase-tracking arrays — each sized by arithmetic that closes, none taken from prior
> knowledge.

**Subsystem category:** Gameplay features — manager structures
**Depends on:** [C27](../C27-Function-Catalogue/C27-Function-Catalogue.md) (names + addresses),
[C28](../C28-Class-Catalogue/C28-Class-Catalogue.md) / [C29](../C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md)
(disassembly method), [C0.2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md) (resolver)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C27](../C27-Function-Catalogue/C27-Function-Catalogue.md), [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md), [C29](../C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md), [C34](../C34-Vehicle-Recording/C34-Vehicle-Recording.md), [C48](../C48-Data-Folder-Sweep/C48-Data-Folder-Sweep.md)
**RE status:** Documented — all four managers fully covered: CEntryExitManager, CReplay, CShopping,
and CGangWars (zone-ownership array, six active-war globals, CGangs static table)
**Confidence:** ✅ for every stride / count / offset below (re-checked by `derive_gameplay_managers.py`) ·
🔷 for methods listed but not individually disassembled
**Data artifact:** [`RE-Data/data/gameplay_managers.json`](../RE-Data/data/gameplay_managers.json) — generated
by [`tools/derive_gameplay_managers.py`](../tools/derive_gameplay_managers.py)

---

## Deep-dive pages

- [C30.1 — CEntryExitManager and the 60-byte marker](01-centryexitmanager.md): the interior entry/exit
  markers held in a templated pool (record 60 bytes, proven by the multiply *and* the ÷60 reciprocal), the
  pool object's array/flags/count layout, and the 32-slot visible-objects working set.
- [C30.2 — CReplay and the 800 KB buffer](02-creplay.md): the replay recording buffer — **8 blocks of
  100,000 bytes** walked identically by three methods — plus the 140-entry ped-pool conversion table, the
  per-ped packet stride (1,988 bytes = `sizeof(CCopPed)` — the replay reserves worst-case size per slot),
  and `tReplayPedUpdateBlock` packet access model.
- [C30.3 — CShopping and the 560-item ledger](03-cshopping.md): the purchase-tracking pair (a 560-entry
  bought-id array and its 560-byte flag array, sizes agreed by lookup and by clear), the fully-documented
  24-byte `CShoppingRecord` layout (key, price, extra_data union, nameTag), and the clothes-state save buffer.
- [C30.4 — CGangWars, CGangs, and Gang Territories](04-cgangwars.md): the 320-entry zone-ownership byte
  array, the six `CGangWars` active-war globals (fight flag, attacker/defender IDs, progress bar, wave count,
  safe-house flag), the four key functions, war rules, and the `CGangs` 10-entry × 16-byte static gang table
  populated from `peds.dat`.

---

## 30.0 The result first

| Claim | Evidence |
|---|---|
| **`CEntryExit` record = 60 B (`0x3C`)** | `imul …,0x3C` in six methods + ÷60 reciprocal `0x88888889` in `DeleteOne` ✅ |
| Entry/exits live in a **pool object** at `0x96A7D8` (array@+0, flag bytes@+4, count@+8) | `DeleteOne` / `EnableBurglaryHouses` ✅ |
| Visible-objects working set = **32 slots** at `0x96A738` | `SetAreaCodeForVisibleObjects` (`cmp …,0x20`) ✅ |
| **Replay buffer = 8 × 100,000 B (`0x186A0`)** at `0x97FB88…0xA43088` | `(0xA43088 − 0x97FB88)/0x186A0 = 8`; walked 3 ways ✅ |
| Replay ped-conversion table = **140 entries** (`0x8C`) at `0x97F838`; ped packet stride `0x7C4` = `sizeof(CCopPed)` | `InitialisePedPoolConversionTable` ✅ |
| **Shop items = 560 (`0x230`)**: id array `0xA97D90`, flag bytes `0xA972A0` | `HasPlayerBought` (`cmp …,0x230`) + `ShutdownForRestart` (clears `0x8C` dwords = 560 B) ✅ |
| `CShoppingRecord` = **24 B (`0x18`)**: key@+0, price@+4, extra_data[2]@+8, nameTag[8]@+16 | `GetKey` / `GetExtraInfo` / `GetNameTag` / `GetNextSection` ✅ |
| Clothes buffer = 30 dwords at `0xA9A810` | `StoreClothesState` ✅ |
| Gang zone ownership = **320-byte array at `0xC8B2C0`** | zone count × owner byte ✅ |
| **`CGangWars` globals** at `0xC8A4A4`–`0xC8A4B8` (fight flag, attacker, defender, progress, waves, safe-house) | disassembly ✅ |
| **`CGangs` table** = 10 × 16 B at `0xC091F0`; fields: model-sets@+0/+1, weapon-tier@+4, RGBA@+8 | `CGangs::Initialize` / `GetGangInfo` ✅ |
| **13 / 13** structural facts confirmed against `gta_sa.exe` | disassembly ✅ |

---

## The four managers

C29 took the two cluster members with the cleanest *fixed-array* structure. This chapter adds the remaining
four: `CEntryExitManager` (pooled 60-byte markers), `CReplay` (800 KB ring buffer), `CShopping` (560-item
purchase ledger), and `CGangWars` (active war state + the 320-zone ownership table and CGangs static table).
Together with C29 this closes the gameplay-manager cluster identified in
[C28.1 §3](../C28-Class-Catalogue/01-the-class-map-by-subsystem.md#3-which-classes-already-have-a-home-and-which-are-candidates).

As in C29, the sizes recovered here — 60-byte markers, an 800 KB replay buffer, 560 shop items — happen to
match the engine's known limits, but each is derived as `(end − base) / stride` or a bare loop bound read
from an instruction, and the match with community knowledge is the *confirmation*, never the source. The
project does not adopt community numbers; it re-derives them and notes when they agree.

**Next:** [C30.1 — CEntryExitManager and the 60-byte marker](01-centryexitmanager.md).

## See also (forward links)

The manager cluster extends the pools of [C29](../C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md); CReplay's recorded-camera sibling is [C34 — Vehicle Recording](../C34-Vehicle-Recording/C34-Vehicle-Recording.md), and CShopping is filled by shopping.dat ([C48.2](../C48-Data-Folder-Sweep/02-economy-and-customization.md)).

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C27](../C27-Function-Catalogue/C27-Function-Catalogue.md), [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md), [C29](../C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md), [C34](../C34-Vehicle-Recording/C34-Vehicle-Recording.md), [C48](../C48-Data-Folder-Sweep/C48-Data-Folder-Sweep.md)
- **Known bugs / gotchas:** CReplay buffer is a fixed 8x100k; CShopping ledger is 560 items (C48 shopping.dat fills it).
- **Modding:** entry/exit markers + replay + shopping are gameplay-mod surfaces.
- **Performance:** manager updates are bounded by their fixed caps.
