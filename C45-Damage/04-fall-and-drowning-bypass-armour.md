# C45.4 — Fall and drowning bypass armour

[C45.2](02-armour-health-model.md) established the general rule: damage drains armour first, then health. This
page documents the two exceptions — **falling** and **drowning** — which skip armour entirely and go straight
to health. It is a small finding with a large gameplay consequence, and it is proven cold.

## The exception, in the code

The damage-response calculator at `0x4AD550` decides how much of a hit armour absorbs. It opens by loading
the ped's armour and checking the weapon type against two values:

```
0x4AD556: fld  [edi+0x548]        ; load m_fArmour
0x4AD572: cmp  eax, 0x35          ; WEAPON_DROWNING (53)?
0x4AD57B: cmp  eax, 0x36          ; WEAPON_FALL (54)?
          ... (if either) skip the armour block, apply straight to health ...
0x4AD5C1: fld  [edi+0x548]        ; else: the normal armour drain (C45.2)
0x4AD5F5: mov  [edi+0x548], 0     ;       armour consumed
```

The two `cmp` immediates are exactly the `eWeaponType` values from [C45.1](01-every-source-is-a-weapon.md):
`WEAPON_DROWNING` = **0x35** (53) and `WEAPON_FALL` = **0x36** (54). When the incoming damage is either, the
function **returns before the armour subtraction** — armour is never touched, and the full amount is applied
to health. `derive_damage.py` asserts both immediates gate the armour block (`fall_drown_bypass_armour`).

## Why it works this way

Armour in San Andreas is body armour — a vest. It is entirely sensible that a bulletproof vest does nothing
against **drowning** (you cannot armour your lungs) or a **fall** (a vest does not cushion an impact with the
ground). So the game models the intuition directly: the two damage sources that are not *impacts to the body
surface* ignore the body-surface protection. Every other source — bullets, melee, explosions, being hit by a
car — goes through armour normally ([C45.2](02-armour-health-model.md)).

The gameplay consequence is real and familiar: **you cannot survive a fatal fall or a drowning by wearing
armour.** A player at full health and full armour is effectively at 200 EHP against gunfire, but only 100
against a long fall or running out of breath underwater — the armour bar is simply not consulted. It is the
kind of asymmetry players feel without knowing why, and here it is, three `cmp` instructions deep.

## How the fall amount is produced

The *type* is `WEAPON_FALL`; the *amount* comes from the impact. When a ped's downward velocity is arrested
by the ground (`m_vecMoveSpeed.z` going from strongly negative to zero — the [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)/[C47](../C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md)
`CPhysical` collision), the game measures how fast it was falling and, above a threshold speed, converts the
excess into a `WEAPON_FALL` damage amount. Below the threshold, short drops do no damage; above it, damage
scales with the impact speed, which is why terminal-velocity falls are instantly fatal — the amount exceeds
100 and, with armour bypassed, empties health in one event. Drowning is the timed sibling
([C45.3](03-the-five-cases.md)): the breath timer emits repeated `WEAPON_FALL`-sized... rather,
`WEAPON_DROWNING` amounts, each bypassing armour, until the ped surfaces or dies.

## The magnitude source — narrowed

The fall *amount* is the collision **impact intensity**, not a bespoke fall formula. When a ped's downward
motion is arrested, the [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)/[C6](../C6-Collision/C6-Collision.md)
collision response accumulates an impact into the `CPhysical` field **`m_fDamageIntensity`** (the largest
impact felt this frame), and the same field gates whether the impact counts as damage — the engine compares
it against a **mass-relative threshold** (a `m_fDamageIntensity > k · m_fMass` test appears in the collision
damage path). Above the threshold, that intensity becomes the `WEAPON_FALL` amount handed to the
[C45.2](02-armour-health-model.md) subtraction (bypassing armour, per this page). So the "fall curve" is not
a special function — it is the **general collision-impact intensity**, thresholded on mass, routed through
`WEAPON_FALL`. `derive_openitems.py` records this mechanism (`fall_damage_mechanism`).

## Open items

- ⏳ The exact **mass-relative constant** `k` and the impact-intensity → damage scale (the mechanism is
  `m_fDamageIntensity` vs mass; the precise coefficients are the remaining narrow gap).
- ⏳ The **drown damage-per-tick** and the breath-timer duration.
- ⏳ Whether any other `eWeaponType` (e.g. explosion) has a similar armour interaction.

## Key takeaways

- **Falling and drowning bypass armour** — the damage-response calculator (`0x4AD550`) checks
  `WEAPON_DROWNING` (0x35) and `WEAPON_FALL` (0x36) and skips the armour subtraction, applying the full amount
  to health. Proven cold.
- The rationale is physical: a vest protects against surface impacts (bullets, melee, cars — which *do* use
  armour) but not against drowning or hitting the ground.
- So armour never saves you from a fatal fall or a drowning; the fall *amount* still comes from
  [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)/[C47](../C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md)
  impact velocity above a threshold.

**Continue:** [back to the C45 hub →](C45-Damage.md) · or [C45.2 — The armour → health model](02-armour-health-model.md).
