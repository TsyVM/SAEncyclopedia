# C45.1 — Every Source is a Weapon

> **The one-sentence version:** San Andreas's damage system unifies all 59 damage causes under
> one `eWeaponType` enum — from fists (0) to gravity (54) — so that one code path handles all
> damage: a `CEventDamage` carrying an amount, a type, and a body part flows through one response
> calculator into one task-reaction machine, and the type tag controls both the subtracted amount
> and the animation reaction it triggers.

**Subsystem category:** Gameplay — damage taxonomy and event dispatch
**Depends on:** [C28](../C28-Class-Catalogue/C28-Class-Catalogue.md) (disassembly method),
[C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md) (the gun subset cross-checked against exe)
**RE status:** Documented — 59-entry enum confirmed; gun entries (22–38) exe-anchored via C14.4's
`WEAPONTYPE` table; environmental entries (48–58) from gta-reversed (🟡)
**Confidence:** ✅ for the gun subset (22–38) via C14.4 cross-check · 🟡 for the environmental
entries (48–58) from gta-reversed enum names · ✅ for the three-step dispatch structure

---

## 1. The `eWeaponType` taxonomy

`eWeaponType` is a `uint32` enum with **59 entries** (0–58). It groups into four bands:

| Range | Band | Representative entries |
|---|---|---|
| 0–15 | **Melee** | `UNARMED` (0), `BRASSKNUCKLE` (1), `GOLFCLUB` (2), `NIGHTSTICK` (3), bats, knives, `CHAINSAW` (9), katana (8), `CANE` (15) |
| 16–21 | **Thrown / heavy** | `GRENADE` (16), `TEARGAS` (17), `MOLOTOV` (18), `ROCKET` (19), `FREEFALL_BOMB` (21) |
| 22–38 | **Guns** | `PISTOL` (22), `PISTOL_SILENCED` (23), `DESERT_EAGLE` (24), `SHOTGUN` (25)–`SDPISTOL` (30), `UZI` (28), `AK47` (32), `M4` (31), `SNIPERRIFLE` (34), `RLAUNCHER` (35), `FLAMETHROWER` (37), `MINIGUN` (38) |
| 39–46 | **Gadgets** | `DETONATOR` (40), `SPRAYCAN` (41), `EXTINGUISHER` (42), `CAMERA` (43), `NIGHTVISION` (44), `PARACHUTE` (46) |
| 47 | Marker | `LAST_WEAPON` — end of player-holdable entries |
| 48–58 | **Non-weapon damage sources** | the world-as-weapon band — below |

The gun subset (22–38) is exe-anchored: [C14.4](../C14-Peds-And-Weapons/04-the-gun-record-columns.md)
cross-checked these values against the `WEAPONTYPE` name table compiled into the executable, so
the values are ✅. The environmental entries (48–58) are from gta-reversed (🟡) — consistently
referenced in modding documentation but not yet individually confirmed against a disassembly
immediate.

### 1.1 The environmental band (48–58)

| # | `eWeaponType` | Cause |
|--:|---|---|
| 48 | `WEAPON_ARMOUR` | Armour pickup or scripted-armour restoration channel |
| 49 | `WEAPON_RAMMEDBYCAR` | Struck by a vehicle (side/front impact) |
| 50 | `WEAPON_RUNOVERBYCAR` | Run over (under a vehicle) |
| 51 | `WEAPON_EXPLOSION` | Caught in a blast radius |
| 52 | `WEAPON_UZI_DRIVEBY` | Drive-by fire (weapon flag variant) |
| 53 | `WEAPON_DROWNING` | Oxygen depleted underwater |
| 54 | `WEAPON_FALL` | Fell from height |
| 55 | `WEAPON_UNIDENTIFIED` | Generic / scripted damage (source unspecified) |
| 56 | `WEAPON_ANYMELEE` | Query wildcard: matches any melee type |
| 57 | `WEAPON_ANYWEAPON` | Query wildcard: matches any weapon type |
| 58 | `WEAPON_FLARE` | Flare projectile (light only; used in stunting mods) |

So gravity (fall damage) is type 54, being run over is type 50, drowning is type 53. These are not
special cases in the damage system — they are just numbers in the same table as `WEAPON_PISTOL`.

---

## 2. Why unify them: the `CEventDamage` dispatch

Making gravity a "weapon" sounds philosophical until you see the operational benefit. Because every
damage cause is a `uint32`, the entire damage pipeline is three steps regardless of cause:

```
[Any cause] → CEventDamage {amount, eWeaponType, ePedPieceTypes body_part, CEntity* attacker}
                    │
                    ▼
             Response calculator (C45.2)
             weighs: armour → health, head shot multiplier, fall immunity, water immunity
                    │
                    ▼
             Task tree reaction (C41)
             picks: flinch / stagger / ragdoll / death → correct anim group
```

A pistol shot and a two-storey fall differ only in how `CEventDamage` is constructed. From the
moment the event enters the ped's AI, both are indistinguishable in mechanism — only the `eWeaponType`
and `amount` differ.

### 2.1 The body part field

`CEventDamage` carries an **`ePedPieceTypes`** body-part enum alongside the weapon type. The body
part field:
- Selects the **hit reaction animation** (leg hit → limping, head hit → stagger-back, torso hit →
  standard hit-react)
- Applies the **headshot damage multiplier** (C45.3) — a bullet to the head of an unarmoured ped
  causes significantly more damage than the base `fDamage` from the weapon record
- Drives the **hit-flash FX** particle position (where the blood splatter spawns)
- Is the argument to `CPed::AddToPedChain` for ragdoll momentum direction

A weapon that doesn't logically have a body-part (explosion, drowning, fall) typically passes
`PED_PIECE_TORSO` or `PED_PIECE_UNKNOWN` — the response calculator handles these with a simple
health subtraction and a matching generic animation.

---

## 3. The type tag drives the reaction animation

The `eWeaponType` passed in `CEventDamage` is not just bookkeeping — the response calculator and
the task tree both read it to choose a reaction:

| Weapon band | Default reaction bias |
|---|---|
| Melee (0–15) | Knock-back in the direction of impact; stagger if force is moderate |
| Guns (22–38) | Hit-react animation keyed to the body part; can be single-frame or full stagger |
| `WEAPON_EXPLOSION` (51) | Launch: angular impulse is large, ped tumbles; ragdoll triggered |
| `WEAPON_FALL` (54) | Collapse: the ped's own fall animation plays; ragdoll on impact |
| `WEAPON_DROWNING` (53) | Drowning struggle animation; transitions to death if health reaches zero |
| `WEAPON_RAMMEDBYCAR` (49) | Car-hit reaction (faster than a normal hit-react, shorter window) |

This is the type tag's second duty: it is the **interface between "what hurt me" and "how I show it."**
The response calculator does not need to know whether a bullet, a bat, or a speeding bus caused the
event — the type tag already encodes enough information for the animation system to select the
appropriate response.

### 3.1 Script-accessible damage typing

The mission script engine ([C18](../C18-SCM-Script/C18-SCM-Script.md)) can apply scripted damage
to peds via `SET_CHAR_HEALTH` or `APPLY_FORCE_TO_CHAR`. For scripted kills and physics-triggered
responses, the script typically uses `WEAPON_UNIDENTIFIED` (55) — a generic damage type that
bypasses immunity logic (described below) and maps to the default torso hit-react.

---

## 4. Immunity system

Certain ped types are immune to specific damage sources. Immunity is governed by the `CPedStats`
data ([C26](../C26-Ped-Tables/C26-Ped-Tables.md)) and by scripted flags set on specific peds:

| Immunity type | Immune to `eWeaponType` | Notes |
|---|---|---|
| Water immunity | `WEAPON_DROWNING` (53) | Amphibious ped types; set by `bIgnoresWater` flag |
| Fall immunity | `WEAPON_FALL` (54) | Police helicopters; peds spawned at height who should not die on landing |
| Explosion immunity | `WEAPON_EXPLOSION` (51) | Script-set via `SET_CHAR_IMMUNITIES`; used for scripted bombs that shouldn't kill friendly peds |
| Vehicle immunity | `WEAPON_RAMMEDBYCAR` (49), `WEAPON_RUNOVERBYCAR` (50) | Script-set for escort peds and some mission targets |
| Fire immunity | (fire damage — maps to `WEAPON_FLAMETHROWER` (37)) | Set by `bOnFire` handling flags |

The immunity check is a fast path in the response calculator (C45.2): if the ped has the immunity
flag for the incoming `eWeaponType`, the health subtraction is skipped entirely and the event is
still dispatched to the AI (so the ped can still react with a flinch animation) but does no damage.

---

## 5. Damage amounts and the weapon record

For the 22–38 gun range, the damage amount in `CEventDamage` comes from the weapon record's
`fDamage` field (C14.4 — the same record that supplies fire rate, range, and ammo type). For
environmental types, the damage amount is computed separately:

| Source | Amount computation |
|---|---|
| `WEAPON_FALL` (54) | Proportional to impact velocity squared: `dmg = k × v_impact²`; small falls do zero damage below a threshold velocity |
| `WEAPON_DROWNING` (53) | Constant per-tick drain (🟡 — exact rate not traced to address) |
| `WEAPON_EXPLOSION` (51) | Proportional to `(explosion_radius - distance) / explosion_radius × max_dmg`; falls off with distance |
| `WEAPON_RAMMEDBYCAR` (49) | Proportional to the vehicle's mass and relative velocity at impact |

The fall damage computation uses a velocity threshold below which no damage is applied — jumps
from two-storey heights typically just play the land animation without a damage event. The exact
formula is 🟡 (velocity threshold and `k` value not independently verified).

---

## 6. Modding the damage system

### 6.1 Script-level: `APPLY_FORCE_TO_CHAR` and health manipulation

The simplest damage-system mod uses SCM opcodes to set ped health or apply a force vector.
`SET_CHAR_HEALTH` bypasses `CEventDamage` entirely — it writes directly to `CPed::m_fHealth`.
For a mod that needs the proper animation and AI reaction, `APPLY_DAMAGE_TO_CHAR` (or the
equivalent via a scripted weapon pickup) is needed to go through the event path.

### 6.2 ASI hooks into `CPed::InflictDamage` or `CEventDamage`

An ASI mod can hook the `CPed::InflictDamage` function (🟡 — confirmed by name from gta-reversed)
to intercept all incoming damage events for a ped. The hook receives `{pPed, amount, weaponType,
bodyPart, attacker}` and can:
- Override the damage amount (e.g., difficulty scaling)
- Change the weapon type (e.g., make all vehicle damage resolve as `WEAPON_EXPLOSION` for dramatic
  ragdolls)
- Block damage entirely (god mode) by not calling the original

Because the system is unified under one event type, a single hook point on `CPed::InflictDamage`
covers all 59 damage causes — no need for separate hooks on "vehicle damage," "fall damage," etc.

### 6.3 Adding new damage types

The `eWeaponType` values above 58 are unused. A custom damage type can be added by using a value
in the 59–127 range (uint32 allows this even though the enum only defines 59 entries) and
registering its behaviour through hook points on the animation selection and immunity checks.
This is ⏳ — possible in principle but requires tracing the switch dispatch in the response
calculator to find where the animation group is selected based on weapon type.

---

### Key takeaways

- `eWeaponType` has **59 entries** (0–58) covering melee, guns, gadgets, and environmental killers;
  the gun subset (22–38) is ✅ exe-anchored via [C14.4](../C14-Peds-And-Weapons/04-the-gun-record-columns.md).
- The environmental band (48–58) makes gravity (`WEAPON_FALL`, 54), vehicles (`WEAPON_RAMMEDBYCAR`
  49, `WEAPON_RUNOVERBYCAR` 50), and drowning (`WEAPON_DROWNING`, 53) first-class weapon types.
- All 59 types flow through **one dispatch path**: `CEventDamage → response calculator → task
  reaction`; the type tag controls both damage amount selection and animation reaction.
- The **immunity system** skips health subtraction for specific (ped, weaponType) pairs but still
  dispatches the event to the AI.
- A single hook on `CPed::InflictDamage` covers all 59 damage sources — no per-source hooks needed.

**Continue:** [C45.2 — The armour → health model →](02-armour-health-model.md)
