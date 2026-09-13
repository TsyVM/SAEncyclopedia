# C53.1 — Pool slot counts and the CREATE table

## The CREATE function

All 13 pools are created in a single function: `CPools::Initialise` at `0x5503A0`. This function is called during `CGame::Initialise` at startup and is **not** called again at runtime — the pools are permanent for the lifetime of the process.

The pattern for each pool within this function is:

```
push <count>             ; hardcoded slot count — the patchable limit
push <string_ptr>        ; pool name string in .rdata (for debug)
mov  ecx, <alloc_result> ; this = pool global pointer
call CPool<T>::Ctor      ; typed constructor
```

Every count proven below was read directly from the `push-imm32` byte in the executable at the given `evidence_va`.

## The complete pool table

| # | Global VA | Count | Stride (bytes) | Ctor VA | Type |
|---|---|---|---|---|---|
| 0 | `0xB7448C` | 500 | 20 | `0x54F720` | Unknown (20-byte element) |
| 1 | `0xB74490` | **140** | **1988** | `0x54F7F0` | **CPed** |
| 2 | `0xB74494` | **110** | **2584** | `0x54F8C0` | **CVehicle** |
| 3 | `0xB74498` | **13000** | **56** | `0x54F970` | **CBuilding** |
| 4 | `0xB7449C` | **350** | **412** | `0x54FA40` | **CObject** |
| 5 | `0xB744A0` | 2500 | TBD | `0x54FAF0` | Unknown (2500) |
| 6 | `0xB744A4` | 10150 | TBD | `0x54FBC0` | Unknown (10150) |
| 7 | `0xB744A8` | 500 | TBD | `0x54FC90` | Unknown (500b) |
| 8 | `0xB744AC` | 200 | TBD | `0x54FD60` | Unknown (200) |
| 9 | `0xB744B0` | 64 | TBD | `0x54FE30` | Unknown (64a) |
| 10 | `0xB744B4` | 32 | TBD | `0x54FF00` | Unknown (32) |
| 11 | `0xB744B8` | 64 | TBD | `0x54FFD0` | Unknown (64b) |
| 12 | `0xB744BC` | 16 | TBD | `0x5500A0` | Unknown (16) |

All counts: `verified_by_disassembly`. Strides for indices 5–12 are TBD — the constructors for these pools use register calculations that were not decoded in the initial probe pass.

## The CPool<T> structure

Each pool global holds a pointer to a 20-byte `CPool<T>` struct:

| Offset | Name | Type | Notes |
|---|---|---|---|
| +0 | `m_pObjects` | `void*` | Heap block: `count × stride` bytes |
| +4 | `m_byteMap` | `uint8_t*` | Parallel flag array: `count` bytes |
| +8 | `m_nSize` | `int32_t` | Slot count (written once at construction) |
| +12 | `m_nFirstFree` | `int32_t` | -1 = no cached free slot |
| +16 | `m_bOwnsMemory` | `uint8_t` | 1 = pool frees its own blocks on Shutdown |

Handle encoding: `handle = (slot_index << 8) | byteMap[slot_index]`. A slot is in-use when `byteMap[slot] & 0x80 == 0`. Documented in detail in [C4 — CPool Internals](../C4-CPool-Internals/C4-CPool-Internals.md).

## The four main pools in detail

### CPed — pool 1

- **Global VA:** `0xB74490`
- **Default count:** 140 (patch VA: `0x5503F1`)
- **Stride:** 1988 bytes (`imul eax,eax,0x7C4` in ctor at `0x54F7F0`)
- **Memory for 140 slots:** 140 × 1988 + 140 (byteMap) = 278,260 bytes ≈ 272KB
- **Hard limit effect:** When the pool is full, `CPopulation::AddPed` silently fails — no ped is created, no crash.

### CVehicle — pool 2

- **Global VA:** `0xB74494`
- **Default count:** 110 (patch VA: `0x550429`)
- **Stride:** 2584 bytes (`imul eax,eax,0xA18` in ctor at `0x54F8C0`)
- **Memory for 110 slots:** 110 × 2584 + 110 = 284,350 bytes ≈ 278KB
- **Note:** The stride 2584 > CAutomobile measured size 2440 — CBike, CPlane, CHeli variants are larger. The pool stride is the maximum vehicle subtype size.

### CBuilding — pool 3

- **Global VA:** `0xB74498`
- **Default count:** 13,000 (patch VA: `0x55045E`)
- **Stride:** 56 bytes (`imul eax,eax,0x38` in ctor at `0x54F970`)
- **Memory for 13,000 slots:** 13,000 × 56 + 13,000 = 741,000 bytes ≈ 724KB
- **Usage:** All static world geometry in the currently loaded streaming cells fills this pool. Custom modded maps with dense static geometry routinely exhaust it.

### CObject — pool 4

- **Global VA:** `0xB7449C`
- **Default count:** 350 (patch VA: `0x550496`)
- **Stride:** 412 bytes (`imul eax,eax,0x19C` in ctor at `0x54FA40`)
- **Memory for 350 slots:** 350 × 412 + 350 = 144,550 bytes ≈ 141KB
- **Usage:** Breakable props, script-spawned objects, weapon pickups with collision. When full, `CREATE_OBJECT` and `CREATE_OBJECT_NO_OFFSET` SCM opcodes silently return handle 0.

## CPools::Shutdown

`CPools::Shutdown` at `0x550F10` is the mirror function. It reads each global pool pointer, calls the destructor (which frees `m_pObjects` and `m_byteMap` via `operator delete` at `0x8207AE`), and nulls the pointer. It runs during `CGame::ShutDown`.

**This was previously labeled "CPools::Initialise" in an older version of this encyclopedia — that was incorrect. `0x550F10` is the DESTROY function. `0x5503A0` is the CREATE function.**

**Continue:** [C53.2 — Memory budget and safe expansion →](02-memory-budget.md)
