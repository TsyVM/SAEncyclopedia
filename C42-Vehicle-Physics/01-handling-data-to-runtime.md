# C42.1 — From handling.cfg to a runtime array

[C13](../C13-Vehicle-Data/C13-Vehicle-Data.md) proved `handling.cfg` is 210 rows of text. This page proves
what the engine does with them: parses each into a fixed **224-byte** struct and stores all 210 in a
contiguous array it indexes by vehicle. Every fact here is read from the executable's own indexing code.

## The struct stride, from the index multiply

The cleanest proof that `tHandlingData` is `0xE0` bytes is an accessor that reaches into the array.
`cHandlingDataMgr::HasFrontWheelDrive` at `0x6A0480` is short and total:

```
0x6A0480: movzx eax, byte [esp + 4]     ; handlingId
0x6A0485: imul  eax, eax, 0xE0          ; * sizeof(tHandlingData)  <-- stride = 0xE0
0x6A048C: mov   bl,  [eax + ecx + 0x88] ; tHandlingData.m_transmissionData.m_nDriveType
0x6A0495: cmp   bl, 0x52                ; 'R'  (rear-wheel drive)
0x6A0498: setne dl                      ; -> has front-wheel drive if NOT 'R'
```

The `imul eax, eax, 0xE0` multiplies the vehicle's `handlingId` by the struct size to index the array — so
`sizeof(tHandlingData) = 0xE0` (224 bytes), read straight from the multiply (`derive_physics.py`:
`thandling_stride_0xE0`). The follow-on reads the **drive type** at offset `+0x88` and compares `'R'`:
rear-wheel drive is stored as the ASCII byte `'R'` (`0x52`), and the function returns "has front-wheel drive"
when it is anything else — exactly `gta-reversed`'s `GetTransmission().m_nDriveType != 'R'`. So two facts fall
out of one function: the struct is `0xE0` bytes, and its transmission drive-type sits at `+0x88`.

## The array and its parallel name table

The 210 structs live in `cHandlingDataMgr` (`gHandlingDataMgr` @`0xC2B9C8`), referenced from 14 places in
`.text`. Alongside them is a **parallel name table**, `VehicleNames` at `0x8D3978` — `char[210][14]`, one
14-byte name per handling entry. Reading it at stride 14 gives real handling names:

```
VehicleNames[0] @0x8D3978 = "LANDSTAL"
VehicleNames[1] @0x8D3986 = "BRAVURA"
VehicleNames[2] @0x8D3994 = "BUFFALO"
```

`GetHandlingId` at `0x6F4FD0` — how a name becomes an index — confirms the stride: it loads `0x8D3978`
(`VehicleNames`), pushes the entry size **`0xE` (14)**, and calls a string-table search:

```
0x6F4FD9: mov  esi, 0x8D3978     ; VehicleNames
0x6F4FE0: push 0xE               ; entry size = 14
0x6F4FE2: push esi
0x6F4FE3: push ebx               ; the name to look up
0x6F4FE4: call 0x8214D0          ; FindExactWord(name, VehicleNames, 14, ...)
```

So a `handling.cfg` line's name is matched against `VehicleNames` to get a `handlingId`, and that id then
indexes `m_aVehicleHandling` at `× 0xE0`. `derive_physics.py` verifies the three names and the `0xE` stride
push (`vehicle_names_stride_14`, `gethandlingid_uses_names`).

## The four handling arrays

`cHandlingDataMgr` holds four arrays, one per vehicle physics class, and they account for almost the whole
object:

| Array | Count | Element | Bytes |
|---|---:|---|---:|
| `m_aVehicleHandling` (`tHandlingData`) | **210** | `0xE0` | `0xB7C0` |
| `m_aBikeHandling` (`tBikeHandlingData`) | **13** | `0x40` | `0x340` |
| `m_aFlyingHandling` (`tFlyingHandlingData`) | **24** | `0x58` | `0x840` |
| `m_aBoatHandling` (`tBoatHandlingData`) | **12** | `0x3C` | `0x2D0` |
| | | **sum** | **`0xC610`** |

The four arrays sum to `0xC610`, and the class is `0xC624` (`gta-reversed`'s `VALIDATE_SIZE`) — the arrays
fill all but the last ~20 bytes (a small header/tail of scalar members). `derive_physics.py` asserts the sum
fits (`handling_arrays_fit`). The vehicle count **210** is the same number [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md)
counted in `handling.cfg`, so the file's row count and the runtime array's element count agree — the
cross-subsystem check this project treats as its strongest evidence (`c13_handling_count_tie`).

## The tHandlingData fields (head of the struct)

From the parse order, the struct opens with the core rigid-body parameters, then transmission, then
brakes/steering/traction/suspension:

| Offset | Field | Range (from the file legend) |
|---|---|---|
| `+0x00` | `m_nVehicleId` | the handling id |
| `+0x04` | `m_fMass` | 1.0 – 50000.0 |
| `+0x08` | `m_fMassRecpr` | `1 / mass` (precomputed) |
| `+0x0C` | `m_fTurnMass` | moment of inertia |
| `+0x10` | `m_fDragMult` | aerodynamic drag |
| `+0x14` | `m_vecCentreOfMass` | x/y/z, 12 bytes |
| `+0x88` | `m_transmissionData.m_nDriveType` | `'F'` / `'R'` / `'4'` |

`m_fMassRecpr` being stored *next to* `m_fMass` is a classic physics-engine micro-optimisation — dividing by
mass is done every time a force is applied, so the reciprocal is cached once at load ([C42.2](02-cphysical-rigid-body.md)
uses it).

## Key takeaways

- `tHandlingData` is **`0xE0` (224 B)**, proven by `imul eax,eax,0xE0` in `HasFrontWheelDrive`; the drive
  type is at `+0x88` (`'R'` = rear-wheel drive).
- The 210 structs are indexed via a parallel `VehicleNames[210][14]` table (`LANDSTAL`/`BRAVURA`/`BUFFALO`…)
  at stride 14, and the manager also holds bike (13), flying (24) and boat (12) arrays summing to `0xC610`.
- The **210** vehicle count equals [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md)'s `handling.cfg` row count
  — the data file and the runtime array meet at the same number.

**Continue:** [C42.2 — The CPhysical rigid body →](02-cphysical-rigid-body.md)
