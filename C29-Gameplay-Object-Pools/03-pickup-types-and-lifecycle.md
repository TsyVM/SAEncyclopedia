# C29.3 — Pickup types and the full pickup lifecycle

## The in-use byte at +0x18 encodes type, not just occupancy

C29.1 established that the byte at record `+0x18` is an in-use flag — `FindPickUpForThisObject` treats zero as "empty slot, skip." But the byte encodes more than occupancy. Per `CPickup`'s method disassembly, the type byte's non-zero values correspond to the different kinds of pickups:

| Value at +0x18 | Meaning |
|---|---|
| 0 | Empty slot — no pickup |
| 1 | On-foot weapon pickup (collect by walking over) |
| 2 | Floating / rotating pickup (collectibles, special items) |
| 3 | On-foot weapon pickup — collected once then destroyed |
| 4 | Money pickup (dollar-sign icon) |
| 5 | In-vehicle health / armour (enter vehicle zone to collect) |
| 6–9 | Various sub-types (sniper, armour, health variants) |
| 14 | Weapon pickup — requires the player to be on-foot and unarmed for that slot |

The exact canonical enum is ⏳ (not fully traced), but the zero-check pattern in `FindPickUpForThisObject` confirms `0 = empty` is the sentinel. All non-zero values are alive pickups of varying types.

## Additional per-slot fields

Beyond the three fields documented in C29.1, further accessor methods reveal:

| Offset | Size | Field | Evidence method |
|---|---|---|---|
| `+0x00` | 4 | World-object pointer (the CObject this pickup wraps) | `FindPickUpForThisObject` (compares `[ecx]` to target) |
| `+0x12` | 2 | Money-per-day (`uint16`) | `UpdateMoneyPerDay` |
| `+0x16` | 2 | Reuse counter (`uint16`) | `GetActualPickupIndex` (handle validation) |
| `+0x18` | 1 | Type/occupancy byte | `FindPickUpForThisObject` |
| `+0x1A` | 2 | Weapon model index (`uint16`) | `CPickup::ExtractAmmoFromPickup` (+0x22 on the object, +0x1C type, +0x1D flag) |

The full 32-byte record is partially mapped. The remaining bytes (`+0x01..0x11`, `+0x13..0x15`, `+0x19`, `+0x1B..0x1F`) are likely position data (world XYZ float), ammo count, and state flags.

## The full pickup lifecycle

### Spawn

`CPickups::GenerateNewOne_WeaponType` (one of the creation methods in the class, catalogued via `gameplay_pools.json`) allocates a new slot:

1. Scans the 620-slot pool for an empty slot (`+0x18 == 0`)
2. If none found → pool is full → no pickup created (silent fail)
3. Sets `+0x18` to the pickup type (non-zero)
4. Stores the weapon model index at `+0x1A`
5. Creates a CObject for the visual representation and stores its pointer at `+0x00`
6. Increments the reuse counter at `+0x16` (making any previous handles to this slot invalid)
7. Returns the packed handle `(counter << 16) | slot_index`

### Active state

Each frame, `CPickups::Update` walks the 620-slot pool and for each active slot:
- Checks player proximity (position from the wrapped CObject)
- If player is close enough and eligible (`PlayerCanPickUpThisWeaponTypeAtThisMoment`), triggers collection

The floating-rotation animation for pickups is also driven each frame by `Update` — it directly manipulates the wrapped CObject's rotation matrix.

### Collection

When the player collects a pickup:
1. `AddToCollectedPickupsArray` stores the handle in the 20-entry ring buffer at `0x978628`
2. The weapon or item is transferred to the player's inventory
3. The slot is cleared: `+0x18 = 0`, CObject is deleted
4. The reuse counter at `+0x16` is NOT reset — it will increment on the next allocation to this slot

### Respawn

Some pickups respawn after collection (e.g., hidden packages, weapon pickup points in missions). This is handled by the caller that originally created the pickup — `CPickups` itself has no respawn logic. The caller stores the slot handle, detects when `GetActualPickupIndex` returns -1 (slot was reused or freed), and creates a new pickup at the same position.

### Pool full: what breaks

When all 620 slots are occupied:
- New pickup-spawn requests fail silently
- The game world will not generate weapon drops, money bags, or special pickups beyond the cap
- Mission-critical pickups (health stars in missions, mission objectives) are created via the same pool — if the pool is full during a mission, the objective pickup will not appear

**The 620 limit in practice:** standard gameplay rarely approaches this limit. It becomes relevant in mods that spawn many weapon pickups simultaneously, or in multiplayer scenarios (SAMP) where many players have died leaving weapon drops. The fixed-array design means the limit cannot be raised without patching the pool base address and end-of-pool address in `FindPickUpForThisObject` (`0x97D644`) and `Init` (`0x97D65A`).

## The collected-pickups ring buffer in depth

The ring at `0x978628` (20 dwords, cursor at `0x978624`) is used by `IsPickUpPickedUp` to answer: "has this pickup already been collected this session?" This prevents some pickup types from being collected twice. The cursor wraps at 20, so after 20 collections the oldest entry is overwritten. For pickups that should only be collectible once per game session, the caller must use a more permanent flag (a global boolean or a save-game flag) rather than relying on the ring.

**Previous:** [C29.2 — CGarages and the 216-byte garage](02-cgarages-the-garage-pool.md)  
**Continue:** [C29.4 — Garage types and mission integration →](04-garage-types-and-mission-integration.md)
