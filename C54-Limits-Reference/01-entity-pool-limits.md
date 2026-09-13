# C54.1 — Entity pool limits

## How to read this table

- **Patch VA:** the address of the `push-imm32` instruction in `CPools::Initialise` (`0x5503A0`) that pushes the slot count. Overwrite the immediate value here before `CGame::Initialise` calls the function.
- **Tier:** `verified_by_disassembly` — every value was read from the exe, not from community docs.
- **Safe max:** conservative community-tested value. See [C54.3](03-breaking-limits-safely.md) for how to calculate your own.

## Main entity pools

### CPed — 140 slots

| Field | Value |
|---|---|
| Global VA | `0xB74490` |
| Default count | 140 |
| Patch VA | `0x5503F1` |
| Push bytes | `push 0x8C` (0x8C = 140) |
| Element stride | 1988 bytes |
| Safe max | ~250 |

**What breaks beyond safe max:** Each CPed slot is 1988 bytes; raising to 400 = ~530KB extra pool memory, which is fine. The real constraint is the ped AI loop (`CTaskManager` in [C44](../C44-AI-And-Tasks/C44-AI-And-Tasks.md)) which runs O(n) over the pool — very large counts (~500+) measurably slow AI update times. Beyond ~400, watch for OOM on the `operator new` call at pool construction.

When the pool is exhausted at runtime: `CPopulation::AddPed` returns null — no crash, no ped. Scripts using `CREATE_CHAR` get handle 0.

---

### CVehicle — 110 slots

| Field | Value |
|---|---|
| Global VA | `0xB74494` |
| Default count | 110 |
| Patch VA | `0x550429` |
| Push bytes | `push 0x6E` (0x6E = 110) |
| Element stride | 2584 bytes |
| Safe max | ~160 |

**Note:** Pool expansion alone does not increase visible traffic. The CCarCtrl density accumulator at `0xA9A894` (compared against 300 at `0x49B912`) governs how many vehicles the spawner allows simultaneously. Raising the pool count without raising this cap will not produce more traffic.

---

### CBuilding — 13,000 slots

| Field | Value |
|---|---|
| Global VA | `0xB74498` |
| Default count | 13,000 |
| Patch VA | `0x55045E` |
| Push bytes | `push 0x32C8` (0x32C8 = 13000) |
| Element stride | 56 bytes |
| Safe max | ~20,000 |

**What breaks beyond safe max:** Static map geometry from IPL files fills this pool during streaming. Modded maps with dense static placement routinely hit this limit — entities beyond the cap are silently skipped (not a crash). Raising to 20,000 adds ~392KB of pool memory and is well-tested.

---

### CObject — 350 slots

| Field | Value |
|---|---|
| Global VA | `0xB7449C` |
| Default count | 350 |
| Patch VA | `0x550496` |
| Push bytes | `push 0x15E` (0x15E = 350) |
| Element stride | 412 bytes |
| Safe max | ~1,000 |

**What breaks at limit:** Script-created objects (`CREATE_OBJECT`, `CREATE_OBJECT_NO_OFFSET`) silently fail. The physics update loop for dynamic objects scales O(n) — above ~1,000 slots this becomes noticeable.

---

## Other pools (counts only — types TBD)

| Index | Global VA | Count | Patch VA | Notes |
|---|---|---|---|---|
| 0 | `0xB7448C` | 500 | `0x5503B9` | 20-byte element; type unknown |
| 5 | `0xB744A0` | 2,500 | `0x5504CE` | Likely CDummyObject or stream entry |
| 6 | `0xB744A4` | 10,150 | `0x550506` | ~model info slot count; likely CBaseModelInfo |
| 7 | `0xB744A8` | 500 | `0x55053E` | Type TBD |
| 8 | `0xB744AC` | 200 | `0x550576` | Type TBD; candidate: CTrain or CDamageManager |
| 9 | `0xB744B0` | 64 | `0x5505AE` | Type TBD; 64 = CActiveExplosion pool size |
| 10 | `0xB744B4` | 32 | `0x5505E3` | Type TBD; smallest named pool |
| 11 | `0xB744B8` | 64 | `0x550618` | Type TBD; second 64-slot pool |
| 12 | `0xB744BC` | 16 | `0x55064D` | Type TBD; 16 slots — rare entity class |

Patch VAs for pools 5–12 follow the same pattern: overwrite the immediate in the `push-imm32` at each address. **Do not patch these without identifying the type first** — the stride is unknown for pools 5–12, so memory cost calculations cannot be verified.

## Function VAs

| Function | VA |
|---|---|
| `CPools::Initialise` (CREATE) | `0x5503A0` |
| `CPools::Shutdown` (DESTROY) | `0x550F10` |
| `operator new` | `0x820595` |
| `operator delete` | `0x8207AE` |

**Continue:** [C54.2 — Density and spawn limits →](02-density-and-spawn-limits.md)
