# Chapter 45 — Damage: How Health Drains, and Why Everything Is a "Weapon"

> **Goal of this chapter:** answer a direct question — how does damage happen? Being shot, run over,
> drowning, falling from height? San Andreas has one strikingly uniform answer: **every damage source is an
> `eWeaponType`.** A pistol, a car, the sea and gravity are all "weapons" (`WEAPON_DROWNING`, `WEAPON_FALL`,
> `WEAPON_RAMMEDBYCAR`…), and each delivers a single number — an amount of damage — into one shared routine
> that drains **armour first, then health, both floored at zero**. This chapter proves that armour→health
> model cold from the executable, lays out the 59-entry damage-source taxonomy, and walks each of the cases
> you asked about to its `eWeaponType`.

**Subsystem category:** Gameplay — the damage model (`CPed` health/armour, `eWeaponType`, `CEventDamage`)
**Depends on:** [C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md) (the `CEvent`/task system a
damage event feeds), [C14](../C14-Peds-And-Weapons/C14-Peds-And-Weapons.md) (`weapon.dat` gun damage
values) · corroborated by `gta-reversed` (names + `VALIDATE_SIZE`)
**Ties:** [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) (`CPhysical` impact velocity → fall/vehicle
damage), [C16](../C16-Popcycle-And-Water/C16-Popcycle-And-Water.md) (water → drowning)
**RE status:** Documented
**Confidence:** ✅ for the health/armour offsets, the armour→health absorption, and the zero-floor
(`derive_damage.py`, 6/6) · 🟡 for the `eWeaponType` enum values and the event-struct sizes (gta-reversed)
**Data artifact:** [`RE-Data/data/damage_structure.json`](../RE-Data/data/damage_structure.json) — generated
by [`tools/derive_damage.py`](../tools/derive_damage.py)

---

## Deep-dive pages

- [C45.1 — Every source is a weapon](01-every-source-is-a-weapon.md): the `eWeaponType` taxonomy — 59
  entries where guns and fists share the enum with `WEAPON_RAMMEDBYCAR`, `WEAPON_RUNOVERBYCAR`,
  `WEAPON_EXPLOSION`, `WEAPON_DROWNING` and `WEAPON_FALL`.
- [C45.2 — The armour → health model](02-armour-health-model.md): the routine at `0x47108A` proven cold —
  damage hits **armour (`+0x548`)** first, the overflow spills into **health (`+0x540`)**, both clamped at
  `0.0`; health at zero is death.
- [C45.3 — The five cases](03-the-five-cases.md): shooting, being hit by a vehicle, falling, drowning and
  explosions — each traced to its `eWeaponType` and its magnitude source, and the `CEventDamage` flow that
  makes the ped react.
- [C45.4 — Fall and drowning bypass armour](04-fall-and-drowning-bypass-armour.md): the two exceptions to the
  armour→health rule — `WEAPON_FALL` (0x36) and `WEAPON_DROWNING` (0x35) skip armour entirely (proven cold at
  `0x4AD550`), so a vest never saves you from a fall or a drowning.

---

## 45.0 The result first

| Claim | Value | Tier | Evidence |
|---|---|:--:|---|
| Health / max-health / armour offsets | `+0x540` / `+0x544` / `+0x548` | ✅ | the damage routine reads/writes them |
| Default max health & armour | **100.0** | ✅ / 🟡 | `100.0f` present (117×); default from gta-reversed |
| Absorption order | **armour first, then health** | ✅ | `fld/fstp [+0x548]` then `fsubr/fst [+0x540]` |
| Both floored at | **0.0** (`0x858B50`) | ✅ | `fcomp [0x858B50]`, then store 0 |
| Death | health reaches 0 | ✅ | health-zero branch clamps and flags kill |
| Every damage source is | an `eWeaponType` (59) | 🟡 | enum from gta-reversed; guns tie [C14.4](../C14-Peds-And-Weapons/04-the-gun-record-columns.md) |
| Drowning / fall / vehicle | `WEAPON_DROWNING` 53 / `WEAPON_FALL` 54 / `RAMMEDBYCAR` 49 · `RUNOVERBYCAR` 50 | 🟡 | enum |
| Damage event | `CEventDamage` `0x44` → `CPedDamageResponse` `0xC` | 🟡 | gta-reversed `VALIDATE_SIZE` |
| Automated checks | **6 / 6** | — | `tools/derive_damage.py` refuses to write otherwise |

## 45.1 One model, many sources

The reason San Andreas can treat a bullet and a fall the same way is that damage is factored into two
independent halves. The **source** decides *how much* damage and *what type* — a gun reads its value from
`weapon.dat` ([C14.4](../C14-Peds-And-Weapons/04-the-gun-record-columns.md)), a fall computes it from impact
speed ([C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)), drowning ticks it from a breath timer. The
**sink** is always the same: the amount is passed, tagged with its `eWeaponType`, into `CPed`'s
armour→health subtraction ([C45.2](02-armour-health-model.md)). So "how does damage happen" has a single
structural answer with a taxonomy of inputs — which is exactly what this chapter separates: the enum of
sources on one side ([C45.1](01-every-source-is-a-weapon.md)), the one absorption routine on the other
([C45.2](02-armour-health-model.md)), and the specific plumbing per case in between
([C45.3](03-the-five-cases.md)).

## 45.2 Why the enum is the elegant part

It would be easy to imagine separate code paths for "shot", "run over", "drowned", "fell". San Andreas has
essentially one, because `eWeaponType` reaches past actual weapons to name the environmental killers —
`WEAPON_DROWNING`, `WEAPON_FALL`, `WEAPON_RAMMEDBYCAR` — as if they were guns you can be shot with. That is
what lets the same `CEventDamage` carry any cause, the same response calculator weigh it, and the same task
reaction (flinch, stagger, ragdoll, die) play out. It is the same "the enum says more than you'd expect"
shape this project keeps finding — C20's audio-event IDs, C39's 66 camera modes — here turning the entire
concept of "harm" into a single dispatchable type.

---

## Key takeaways

- Damage is **armour then health**: the amount drains `m_fArmour` (`+0x548`) first, the overflow subtracts
  from `m_fHealth` (`+0x540`), both floored at `0.0`; health at zero is death — proven cold at `0x47108A`.
- **Every damage source is an `eWeaponType`** — guns and fists share the 59-value enum with
  `WEAPON_RAMMEDBYCAR`, `WEAPON_RUNOVERBYCAR`, `WEAPON_EXPLOSION`, `WEAPON_DROWNING` and `WEAPON_FALL`.
- The source sets *how much* (weapon.dat, impact velocity, breath timer); the sink is one shared routine, so
  shooting, vehicles, falls and drowning all resolve the same way.

**Continue:** [C45.1 — Every source is a weapon →](01-every-source-is-a-weapon.md)


## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Key functions:** `DamageResponseCalc` (0x4ad550)
- **Callers:** **1** `.text` call-sites reach this chapter's functions.
- **Callees:** **5** distinct functions called from within them.
- **Known bugs / gotchas:** —
- **Modding:** —
- **Performance:** —
