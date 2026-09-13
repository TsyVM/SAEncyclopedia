# C45.3 — The five cases

With the taxonomy ([C45.1](01-every-source-is-a-weapon.md)) and the absorption model
([C45.2](02-armour-health-model.md)) in hand, each way of taking damage is just: *which `eWeaponType`, and
where does the number come from?* This page walks the cases you asked about — shooting, being hit by a
vehicle, falling, drowning, explosions — and the `CEventDamage` flow that ties them to the reaction.

## The common flow

Every case funnels through the same three steps before it reaches [C45.2](02-armour-health-model.md)'s
subtraction:

```
source → CEventDamage(inflictor, eWeaponType, bodyPart, amount)
       → CPedDamageResponseCalculator::ComputeDamageResponse  (0x14-byte calculator)
       → CPedDamageResponse (0xC: amount, body part, m_bHealthZero)
       → event enters the ped's C41 event group → task tree plays the reaction
```

So the only things that differ per case are the **`eWeaponType`** and **how the amount is produced**. The
sizes (`CEventDamage` `0x44`, `CPedDamageResponse` `0xC`, calculator `0x14`) are `gta-reversed`'s (🟡); the
rest of this page is which type + which magnitude source.

## Shooting / being shot

- **`eWeaponType`:** the gun's own type (22–38: `WEAPON_PISTOL` … `WEAPON_M4` …), or `WEAPON_UZI_DRIVEBY`
  (52) for drive-bys.
- **Amount:** the gun's **`damage` column in `weapon.dat`**
  ([C14.4](../C14-Peds-And-Weapons/04-the-gun-record-columns.md)) — a fixed per-weapon value, scaled by hit
  location. A headshot is the body-part field in `CEventDamage` selecting a lethal multiplier.
- The bullet is a hitscan trace; on hitting a ped it builds the damage event with the weapon's type and the
  bone it struck. Body armour ([C45.2](02-armour-health-model.md)) then absorbs before health.

## Being hit / run over by a vehicle

- **`eWeaponType`:** `WEAPON_RAMMEDBYCAR` (49) for an impact, `WEAPON_RUNOVERBYCAR` (50) for being driven
  over.
- **Amount:** derived from the vehicle's **impact speed** — the `CPhysical` collision response
  ([C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)) that turns relative velocity and mass into an
  impulse also yields the damage magnitude, so a fast car hurts more than a slow one. The two types differ in
  reaction: a ram launches the ped (knock-down), a run-over applies continuous crush damage while the wheels
  are over them.
- This is where the physics arc meets the damage arc: the same impact that moves the ragdoll
  ([C42.2](../C42-Vehicle-Physics/02-cphysical-rigid-body.md)) supplies the number the damage routine
  subtracts.

## Falling from height

- **`eWeaponType`:** `WEAPON_FALL` (54).
- **Amount:** proportional to **impact velocity** at landing — the faster the ped is moving downward when
  collision stops them, the larger the damage. Below a threshold speed there is no fall damage (short drops
  are free); above it, damage scales with the excess. The magnitude again comes from the
  [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) `CPhysical` velocity at the collision, not from a
  table.
- A high enough fall produces an amount that blows through armour and health in one event
  ([C45.2](02-armour-health-model.md)'s cascading overflow) — instant death from terminal velocity.

## Drowning

- **`eWeaponType`:** `WEAPON_DROWNING` (53).
- **Amount:** a **timed drain**, not an impact. While the ped is underwater
  ([C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md)'s water surface is the trigger), a breath/air
  timer counts down; once it expires the ped takes `WEAPON_DROWNING` damage repeatedly, per frame, until it
  surfaces or dies. This is the one case where the "weapon" fires continuously on a clock rather than once on
  contact — but it still routes through the exact same armour→health subtraction, so armour briefly delays
  drowning death just as it delays a bullet.

## Explosions

- **`eWeaponType`:** `WEAPON_EXPLOSION` (51).
- **Amount:** a radial falloff from the blast centre — full damage near the epicentre, tapering with
  distance. An explosion typically also applies a large `CPhysical` impulse (the launch), so peds are both
  damaged and thrown. Grenades, rockets, car explosions and the `WEAPON_FREEFALL_BOMB` all resolve to this
  type at the point of detonation, regardless of what launched them.

## The pattern

Across all five, the shape is identical: **a source computes an amount and tags it with an `eWeaponType`,
and the shared `CEventDamage` → response → armour→health path does the rest.** Shooting and explosions read
or compute a value at the point of contact; vehicles and falls borrow the number from the
[C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) physics impact; drowning meters it out on a timer. The
damage system's elegance is that none of that variety survives past the event — it all becomes one number and
one type.

## Open items

- ⏳ The exact **fall-damage curve** (threshold speed and scale) and the **explosion falloff** function.
- ⏳ The **body-part multiplier** table (headshot vs limb) inside the response calculator.
- ⏳ The **breath timer** duration and drown-damage-per-tick for `WEAPON_DROWNING`.
- ⏳ Regeneration / pickups (health & armour pickups, the armour cap enforcement).

## Key takeaways

- Each case is one `eWeaponType` plus a magnitude source: guns → `weapon.dat`
  ([C14.4](../C14-Peds-And-Weapons/04-the-gun-record-columns.md)); vehicles/falls →
  [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) impact velocity; drowning → a breath timer;
  explosions → radial falloff.
- All of them build a `CEventDamage`, get weighed by the response calculator, and enter the
  [C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md) event/task system for the reaction — one path.
- Drowning is the only continuous (timed) source; the rest are contact events — but every one ends in the
  same armour→health subtraction ([C45.2](02-armour-health-model.md)).

**Continue:** [back to the C45 hub →](C45-Damage.md) · or [C42 — Vehicle Physics](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) · [C41 — Ped AI](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md).
