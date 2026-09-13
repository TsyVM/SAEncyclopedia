# Chapter 25 — `object.dat`: Dynamic Object Physics

> **Goal of this chapter:** decode how San Andreas gives every dynamic object its physics — mass, buoyancy,
> collision-damage behaviour and, for breakable things, how they shatter — and pin the format down against
> the executable's own parser. The whole 126 KB file is read by **one `sscanf` format** compiled into
> `gta_sa.exe`, and the census of 993 objects it produces overflows several of the limits the file's own
> header documents — every overflow named, not rounded away.

**Subsystem category:** Physics / world
**Depends on:** [C24 — Surface Materials](../C24-Surfaces/C24-Surfaces.md) (surfaces and objects are the two
halves of the collision world) · [C21 — Particles](../C21-Particles/C21-Particles.md) (the FX system an
object names)
**Ties:** [C6](../C6-Collision/C6-Collision.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C21](../C21-Particles/C21-Particles.md), [C24](../C24-Surfaces/C24-Surfaces.md), [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md), [C45](../C45-Damage/C45-Damage.md), [C47](../C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md)
**RE status:** Verified
**Confidence:** ✅ Verified (the single-`sscanf` grammar, the two record variants, the census, the
enumerated domains, the named exceptions) / ⏳ (the physical units of the continuous scalars)

---

## Deep-dive pages

- [C25.1 — One `sscanf`, two record variants](01-the-record-format.md): the 24-field format string the
  executable parses with, why a "basic" object is 17 fields and a "breakable" one 24, and why the variant is
  decided by field *count* — not, despite the header's hint, by the collision-damage value.
- [C25.3 — Physics scalars and their runtime roles](03-physics-scalars-and-runtime.md): what mass, turn_mass, air_resistance, elasticity, percent_submerged, uproot_limit, and cdamage_multiplier actually do in the physics integrator — and why 99999.0 is the immovable sentinel.
- [C25.4 — Modding object.dat](04-modding-object-dat.md): the two record templates, the breakable-without-params trap, safe ranges, and step-by-step instructions for adding new entries.
- [C25.2 — The census and its named exceptions](02-census-and-exceptions.md): 993 objects, the enumerated
  domains they honour, and the handful of rows that break the file's own documented limits — a mass cap
  ignored 514 times, a buoyancy over 100 %, an out-of-range collision code, and one object marked breakable
  with no way to break.

---

## 25.1 The result first

| Claim | Evidence |
|---|---|
| The whole file is parsed by **one `sscanf` format** | `%s %f×7 %d×5 %f×3 %s %f×5 %d×2` at VA `0x868DC8` — **24** conversions ✅ |
| A **basic** object is **17 fields** | `758` records stop after the FX name ✅ |
| A **breakable** object is **24 fields** | `235` records add the 7 break-info fields ✅ |
| The variant is decided by **field count**, not field I | `13` breakable objects have `I < 200`; one `I ≥ 200` object has **no** break fields ✅ |
| Comment/section lines are **skipped** | the loader tests `';'` (`0x3B`) and `'*'` (`0x2A`); a stray `#` line fails `sscanf` and is dropped ✅ |
| Enumerated fields obey their **documented domains** | collision-effect, FX-type, explosion, gun-break all in range ✅ |
| The data **overflows several documented limits** | mass cap, buoyancy, a collision code, a breakable-without-params — each named ✅ |

## 25.2 One format string, the whole file

`object.dat` looks like a loose comma-and-tab table, but the executable reads every line with a single
`sscanf` call against one format string at VA `0x868DC8`:

```
%s %f %f %f %f %f %f %f %d %d %d %d %d %f %f %f %s %f %f %f %f %f %d %d
```

That is 24 conversions, and they map one-to-one onto the fields the header documents: a name, seven
floating-point physics scalars (mass, turn-mass, air resistance, elasticity, percent submerged, uproot
limit, collision-damage multiplier), five integer codes (collision-damage effect, special collision
response, camera-avoid, causes-explosion, FX type), a three-float FX offset, an FX name, and finally the
seven break-info fields (a smash multiplier, three break-velocity components, a randomness factor, a
gun-break mode and a spark flag). The field order in this chapter is not inferred from the columns — it is
read straight off the executable's format string.

Because it is one `sscanf`, the parser needs no schema branch: it hands the line the 24-conversion format
and takes back however many fields actually matched. A basic object supplies 17 and stops; a breakable one
supplies all 24. This is the same "let the standard-library parser count the fields" design the audio and
zone loaders used, and it explains both record variants without any second format.

## 25.3 Why the variant is not the collision code

The file's header hints that the break fields "need set if I ≥ 200" — `I` being the collision-damage effect,
whose values `200` and `202` mean *breakable* and *breakable-then-removed*. It is tempting to read that as
the rule the parser uses. It is not. The parser reads whatever fields are on the line; the header note is
guidance to the **authors**, and the authors did not follow it exactly. Thirteen objects carry full break
info while declaring `I < 200` (a petrol pump, some doors — things that break by gunfire or explosion rather
than by the generic breakable code), and — the mirror-image slip — one object, `sec_keypad`, declares
itself breakable with `I = 200` yet supplies **no** break fields at all. If the collision code drove the
parse, neither case could exist. The variant is a property of the line's field count, full stop, and the
data proves it by disagreeing with its own documentation in both directions.

## 25.4 What this chapter does not claim

⏳ The **physical units** of the continuous scalars are not derived. The header labels them (mass in
kilograms, air resistance a 0–1 scale, and so on) and the value distributions are consistent with those
labels, but how each feeds the rigid-body simulation is engine behaviour this chapter does not trace — it
belongs with the physics/handling model. What is ✅ is the grammar, the field identities and order (from the
executable), the two variants, the census, and the enumerated domains.

---

### Key takeaways

- ✅ `object.dat` is parsed by **one `sscanf` format** — `%s %f×7 %d×5 %f×3 %s %f×5 %d×2`, **24**
  conversions, at VA `0x868DC8` — so the field order and types come straight from the executable.
- ✅ Two record variants share that one format: **basic = 17 fields** (`758` objects), **breakable = 24**
  (`235`), told apart by how many fields `sscanf` matches.
- ✅ The variant is **not** keyed by the collision-damage code: 13 breakables declare `I < 200`, and one
  `I ≥ 200` object (`sec_keypad`) carries no break fields — the header's "if I ≥ 200" is authoring guidance.
- ✅ The loader **skips** `';'` and `'*'` lines; a stray `#` comment is dropped when `sscanf` fails.
- ⚠️ The data **overflows its own documented limits** in named, reproducible ways (see
  [C25.2](02-census-and-exceptions.md)) — the tolerant `sscanf` accepts them all.
- ⏳ The physical **units** of the scalars are left to the physics model.

**Continue:** [C25.1 — One `sscanf`, two record variants](01-the-record-format.md)

## See also (forward links)

The object.dat fields fill the shared [C42.2 — CPhysical rigid body](../C42-Vehicle-Physics/02-cphysical-rigid-body.md); object damage parallels [C45 — Damage](../C45-Damage/C45-Damage.md) and vehicle [C47 — damage/deformation](../C47-Vehicle-Dynamics/03-damage-and-deformation.md). Collision comes from [C6](../C6-Collision/C6-Collision.md).

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C6](../C6-Collision/C6-Collision.md), [C13](../C13-Vehicle-Data/C13-Vehicle-Data.md), [C21](../C21-Particles/C21-Particles.md), [C24](../C24-Surfaces/C24-Surfaces.md), [C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md), [C45](../C45-Damage/C45-Damage.md)
- **Known bugs / gotchas:** mass cap ignored 514x; sec_keypad declared breakable with no params (field-count parser can't detect).
- **Modding:** object.dat is the dynamic-object physics file; shares CPhysical with C42.
- **Performance:** one sscanf per line at load; rigid-body at runtime.
