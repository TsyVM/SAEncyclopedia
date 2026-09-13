# C45.2 — The armour → health model

Whatever the source ([C45.1](01-every-source-is-a-weapon.md)), damage ends in one place: a short arithmetic
routine on the `CPed` that drains armour, spills the overflow into health, and floors both at zero. This page
reads that routine out of the executable instruction by instruction — it is the core of the whole chapter,
and it closes cleanly.

## The three fields

A `CPed`'s survival state is three consecutive floats. Their offsets are fixed by the field just before them
(`field_53C` at `+0x53C`), and the damage routine confirms them by use:

| Offset | Field | Default |
|---|---|---|
| `+0x540` | `m_fHealth` | 100.0 |
| `+0x544` | `m_fMaxHealth` | 100.0 |
| `+0x548` | `m_fArmour` | 0 (max 100.0) |

`derive_damage.py` asserts the damage code references both `[esi+0x540]` and `[esi+0x548]`
(`health_armour_offsets`), and the `100.0f` default is present in the image 117 times
(`max_health_100_present`).

## The absorption, read from the bytes

The routine at `0x47108A` applies a damage value (in a floating-point register) to a ped pointed to by
`esi`. Annotated:

```
0x47108A: fcom  [esi+0x548]          ; compare damage against ARMOUR
0x471092: test  ah, 0x41
0x471095: jp    0x4710AF             ; if damage <= armour -> "armour absorbs all"
                                     ; --- damage > armour: armour is used up ---
0x471097: fld   [esi+0x548]          ;   load armour
0x47109D: fsub  st(1)                ;   armour - damage  (remaining, negative)
0x47109F: fstp  [esi+0x548]          ;   (transient)
0x4710A5: fstp  st(0)
0x4710A7: fld   [0x858B50]           ;   push 0.0
0x4710AD: jmp   0x4710BB
                                     ; --- damage <= armour ---
0x4710AF: fsub  [esi+0x548]          ;   damage - armour  (<= 0, no health loss)
0x4710B5: mov   [esi+0x548], ebx     ;   armour = 0? (this path zeroes on overflow)
                                     ; --- apply the remainder to HEALTH ---
0x4710BB: fsubr [esi+0x540]          ;   health = health - remaining_damage
0x4710C1: fst   [esi+0x540]          ;   store health
0x4710C7: fcomp [0x858B50]           ;   compare health against 0.0
0x4710CF: test  ah, 0x41
0x4710D2: jp    0x472154             ;   health > 0 -> alive, done
0x4710D8: push  0x28
0x4710DA: mov   [esi+0x540], ebx     ;   health = 0  (clamp: the ped is dead)
```

Read as behaviour: **damage is applied to armour first.** If it exceeds the armour, the armour is consumed and
the *leftover* (`damage − armour`) is subtracted from health; if it does not, armour takes it all and health
is untouched. Then health is compared against `0.0` (the constant at `0x858B50`, which
`derive_damage.py` confirms is exactly `0.0` — `floor_constant_is_zero`) and, if it has gone negative, snapped
back to `0` — the moment the ped dies. `derive_damage.py` asserts the armour drain, the health spill, and the
zero-clamp as three separate checks (`armour_absorbs_first`, `remainder_spills_to_health`,
`health_clamped_at_zero`).

## Why armour "feels" like a second health bar

This is the mechanic every player knows — armour soaks damage until it runs out, then your health starts
dropping — expressed in six floating-point instructions. There is no separate armour-damage system and no
per-hit split ratio; armour is simply subtracted first and the remainder cascades. That is why 100 armour
plus 100 health behaves like 200 effective health against most sources, and why a single huge hit (an
explosion, a long fall) can blow through full armour *and* full health in one event: the overflow keeps
cascading in the same subtraction.

## Death is a threshold, not an event

Note what "death" is here: it is not a special call, it is the `health <= 0` branch that clamps health to
zero. The `CPedDamageResponse` (`0xC` bytes) that the calculator fills carries a `m_bHealthZero` flag set on
exactly this condition, and `CEventDamage::HasKilledPed()` reads it. So "was this the killing blow?" is
answered by the same subtraction that applied the damage — the response is computed once, and whether it
killed is just whether the floor clamp fired. Everything downstream (the death animation, the wanted-level
consequence via [C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md), the score) keys off that one
flag.

## Key takeaways

- `CPed` survival is three floats: `m_fHealth` `+0x540`, `m_fMaxHealth` `+0x544`, `m_fArmour` `+0x548`
  (defaults 100).
- The absorption at `0x47108A` is proven cold: **armour is subtracted first, the overflow cascades into
  health, and both are floored at `0.0`** (the constant at `0x858B50`).
- Death is the `health <= 0` clamp, surfaced as `CPedDamageResponse::m_bHealthZero` — computed by the same
  routine that applied the damage, so "did it kill?" is a by-product of the subtraction.

**Continue:** [C45.3 — The five cases →](03-the-five-cases.md)
