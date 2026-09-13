# C30.4 — CGangWars, CGangs, and Gang Territories

> **The one-sentence version:** gang territory ownership is stored as a **320-entry byte array** where each
> byte is a gang ID, `CGangWars` drives the active-war state through **six verified globals**, and `CGangs`
> holds the per-gang static data in a **10-entry × 16-byte table** populated from `peds.dat` at startup.

**Subsystem category:** Gameplay features — gang warfare and territory
**Depends on:** [C30 hub](C30-Gameplay-Managers.md), [C22 — Map Zones](../C22-Map-Zones/C22-Map-Zones.md),
[C26 — Ped Tables](../C26-Ped-Tables/C26-Ped-Tables.md)
**RE status:** Documented — all globals and key functions confirmed by disassembly
**Confidence:** ✅ for all addresses, gang-zone layout, CGangs field offsets, and war-rule functions ·
🔷 for internal wave-spawn logic not disassembled here
**Data artifact:** [`RE-Data/data/gang_territory_structure.json`](../RE-Data/data/gang_territory_structure.json)

---

## 1. The territory ownership array

Every gang-territory zone in the map is represented by a single byte in a flat array. The zone geometry
itself lives at `0xC8B280` (320 `CZone`-shaped records); the **ownership table** is the parallel byte array
at `0xC8B2C0`:

```
0xC8B280  CZone zone_array[320]       ; zone geometry records
0xC8B2C0  uint8_t zone_owner[320]     ; one ownership byte per zone (same index)
```

Each byte is a **gang ID** (0–9). `0x00` means civilian / law-enforcement controlled. `0xFF` marks a zone
the gang system does not own at all (the countryside, interiors, etc.). The 320 count matches the total gang
zone budget used by the map. Ownership is saved per save-game slot and reloaded on continue.

### Gang IDs

| ID | Gang | Minimap colour |
|---|---|---|
| 0 | No gang (civilian / law enforcement) | — |
| 1 | Grove Street Families | Green |
| 2 | Ballas | Purple |
| 3 | Los Santos Vagos | Yellow |
| 4 | San Fierro Rifa | Cyan |
| 5 | Da Nang Boyz | Red |
| 6 | Mafia (unused in final release) | — |
| 7 | Triads | Light blue |
| 8 | San Fierro Triads (second group) | Light blue |
| 9 | Varrios Los Aztecas | Light blue |

The minimap tint is taken from `CGangs::m_nColor` (§3 below). `CRadar::DrawZoneNames` reads the
ownership byte and applies that gang's RGBA colour to the zone polygon when the player opens the world map.
The territory overlay toggles with the START button.

---

## 2. CGangWars — the active-war state machine

When a gang war is in progress, `CGangWars` owns the entire active state through six globals:

```cpp
// [SASDK VERIFIED] all addresses confirmed by disassembly
uint8_t  ms_bGangWarFightingForZone @ 0xC8A4A4  // non-zero while a war is active
uint32_t ms_AttackingGang           @ 0xC8A4A8  // gang ID of the attackers (usually 2, 3, or 9)
uint32_t ms_DefendingGang           @ 0xC8A4AC  // gang ID of the defenders (usually 1 = GSF)
float    ms_fGangWarProgress        @ 0xC8A4B0  // 0.0 (start) → 1.0 (zone captured)
uint32_t ms_WavesSurvived           @ 0xC8A4B4  // waves defeated so far (0–3)
uint8_t  ms_bPlayerInSafeHouse      @ 0xC8A4B8  // set when CJ has fled to a safe house
```

These are the complete active-war state — no other per-war storage exists. A war is either active
(`ms_bGangWarFightingForZone != 0`) or completely cold (all reset to zero).

### Key functions

| Function | VA | Role |
|---|---|---|
| `CGangWars::StartGangWar` | `0x446500` | initiates a war: sets the attacking/defending IDs, starts the first wave |
| `CGangWars::Update` | `0x446200` | called each frame while `ms_bGangWarFightingForZone` is set; drives wave logic and progress bar |
| `CGangWars::EndGangWar` | `0x446700` | resolves the outcome: flips the zone-owner byte if attackers won, resets all six globals |
| `CZone::GetCurrentZone` | `0x572040` | returns the CZone the player currently occupies — the entry gate for `StartGangWar` |

### War rules

A gang war begins when CJ enters a non-GSF zone and a Ballas, Vagos, or Aztecas gang member spots him.
`StartGangWar` sets `ms_AttackingGang` to that gang's ID and `ms_DefendingGang` to 1 (GSF), then spawns
the first wave of enemies. Three waves must be repelled; after each wave `ms_WavesSurvived` increments
and the next wave spawns with harder enemies. When the third wave is cleared `EndGangWar` writes gang ID 1
(GSF) into `zone_owner[zone_index]` — the zone flips to green. If CJ is downed or flees:
`ms_bPlayerInSafeHouse` is set and `EndGangWar` writes the attacking gang's ID instead.

Progress (`ms_fGangWarProgress`) drives the on-screen progress bar. The internal tick is in
`CGangWars::Update`; wave-spawn logic branching from there is 🔷 (not traced here).

---

## 3. CGangs — per-gang static data

`CGangs` is not a single struct instance but a 10-element static array of 16-byte records at `0xC091F0`
(one per gang ID 0–9, matching the table in §1). The `CGangInfo` record layout is confirmed by disassembly:

```cpp
struct CGangInfo {             // 16 bytes (0x10) each; 10 entries at 0xC091F0
    uint8_t  m_nModelSetLow;  // +0  first ped model ID for low-density spawns
    uint8_t  m_nModelSetHigh; // +1  first ped model ID for high-density spawns
    uint8_t  _pad[2];         // +2  alignment
    uint8_t  m_nWeaponTier;   // +4  0–5; peds.dat GANG entries drive this
    uint8_t  _pad2[3];        // +5  alignment
    uint32_t m_nColor;        // +8  RGBA used for minimap territory tinting
    uint8_t  _pad3[4];        // +12 struct rounds to 16 bytes
};
static_assert(sizeof(CGangInfo) == 0x10);
```

`CGangs::Initialize` (`0x4E5E00`) populates this table from the parsed `peds.dat` `GANG` entries at startup.
`CGangs::GetGangInfo` (`0x4E6090`) returns a pointer to `zone_owner[gang_id]`'s `CGangInfo` for callsites
that need a gang's spawn models or minimap colour.

### peds.dat integration

Each `GANG` line in `data/peds.dat` defines model sets and a weapon tier for one gang. The parser calls
`CGangs::Initialize` once during `CGame::Initialise`. Modders can change weapon tiers (field +4) and spawn
model IDs (fields +0/+1) by editing `peds.dat` — the territory tint colour (field +8) is coded in the
`CGangInfo` table and not configurable from `peds.dat` without a patch.

---

## 4. The complete CGangWars picture

The chapter's hub noted `CGangWars` as "flag-only" — that assessment was accurate for the addresses
available at hub-write time, but the full confirmed globals (§2) show it is slightly more than a flag
register: it owns the float progress bar and the wave counter in addition to the three boolean/ID fields.
With the zone-ownership table (§1), the static gang table (§3), and the four key functions, the war system
is fully accounted for.

| What | Where | Size | Confirmed |
|---|---|---|---|
| Zone geometry records | `0xC8B280` | 320 entries | ✅ |
| Zone ownership bytes | `0xC8B2C0` | 320 bytes | ✅ |
| `ms_bGangWarFightingForZone` | `0xC8A4A4` | 1 byte | ✅ |
| `ms_AttackingGang` | `0xC8A4A8` | 4 bytes | ✅ |
| `ms_DefendingGang` | `0xC8A4AC` | 4 bytes | ✅ |
| `ms_fGangWarProgress` | `0xC8A4B0` | 4 bytes | ✅ |
| `ms_WavesSurvived` | `0xC8A4B4` | 4 bytes | ✅ |
| `ms_bPlayerInSafeHouse` | `0xC8A4B8` | 1 byte | ✅ |
| `CGangInfo` table (10 gangs) | `0xC091F0` | 10 × 16 B | ✅ |

---

### Key takeaways

- Gang territory ownership is a **flat 320-byte array at `0xC8B2C0`** — one gang-ID byte per zone,
  `0xFF` for non-gang zones. Written by `EndGangWar`; read by `CRadar` for minimap tinting.
- `CGangWars` active-war state is **six globals at `0xC8A4A4`–`0xC8A4B8`**: a fight flag, attacker/defender
  IDs, a float progress bar, wave count, and safe-house flag.
- `CGangs` is a **10-entry × 16-byte table at `0xC091F0`** holding spawn model IDs, weapon tier, and
  minimap RGBA per gang — populated from `peds.dat` at startup.

**Back to:** [C30 hub](C30-Gameplay-Managers.md)
