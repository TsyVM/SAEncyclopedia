# The Damage Model (eWeaponType + armour/health)
*Source: `damage_structure.json`*
- **$schema:** damage_structure.v1
- **Generated:** tools/derive_damage.py

## Summary
| Field | Value |
| --- | --- |
| m_fHealth | 0x540 |
| m_fMaxHealth | 0x544 (default 100.0) |
| m_fArmour | 0x548 |
| Absorption | armour (+0x548) absorbs first; overflow spills into health (+0x540); both floored at 0.0 |
| Checks passed | 7/7 |

## How each damage happens (all eWeaponType)
| Cause | Mapping |
| --- | --- |
| shooting | gun eWeaponType (22..38) -> weapon.dat damage (C14.4) |
| hit_by_vehicle | WEAPON_RAMMEDBYCAR (49) / WEAPON_RUNOVERBYCAR (50); magnitude from CPhysical impact (C42) |
| fall_from_height | WEAPON_FALL (54); magnitude from impact velocity (C42) |
| drowning | WEAPON_DROWNING (53); triggered by the underwater breath timer |
| explosion | WEAPON_EXPLOSION (51) |

## Checks
| Check | Result |
| --- | --- |
| health_armour_offsets | PASS |
| armour_absorbs_first | PASS |
| remainder_spills_to_health | PASS |
| health_clamped_at_zero | PASS |
| floor_constant_is_zero | PASS |
| max_health_100_present | PASS |
| fall_drown_bypass_armour | PASS |
