# CPickups / CGarages Pool Layouts
*Source: `gameplay_pools.json`*
- **$schema:** gameplay_pools.v1
- **Generated:** tools/derive_gameplay_pools.py

## Summary
| Field | Value |
| --- | --- |
| Pickup slots | 620 |
| Garage slots | 50 |
| Facts checked | 10 |
| Facts passed | 10 |

## Pools

### CPickups
| Key | Base | Cursor | Slots |
| --- | --- | --- | --- |
| Element | CPickup |  |  |
| Base | 9930948 |  |  |
| End | 9950788 |  |  |
| Stride | 32 |  |  |
| In use field | 24 |  |  |
| Handle | counter<<16 \| index |  |  |
| Collected ring | 9930280 | 9930276 | 20 |

### CGarages
| Key | Type | Door state | Flags |
| --- | --- | --- | --- |
| Element | CGarage |  |  |
| Base | 9879624 |  |  |
| End | 9890504 |  |  |
| Stride | 216 |  |  |
| Count global | 9879588 |  |  |
| Fields | 76 | 77 | 78 |

## Structural facts
| Entry va | Method | Fact | Verified |
| --- | --- | --- | --- |
| 0x004551C0 | CPickups::FindPickUpForThisObject | pickup pool = 620 slots x 0x20 (32 B), base 0x9788c4; slot-in-use byte at +0x18 | ✅ true |
| 0x00454A70 | CPickups::Init | Init: same 0x20 stride; collected-pickups ring = 0x14 (20) dwords at 0x978628 | ✅ true |
| 0x004552A0 | CPickups::GetActualPickupIndex | pickup handle packs {counter<<16 \| index}; counter word at slot+0x16 (0x9788da) | ✅ true |
| 0x00455240 | CPickups::AddToCollectedPickupsArray | collected ring write-cursor at 0x978624 wraps at 0x14 (20) | ✅ true |
| 0x004471B0 | CGarages::Shutdown | garage pool = 50 slots x 0xD8 (216 B); walked base+0x50 (0x96c098)..0x96eac8 | ✅ true |
| 0x004476D0 | CGarages::ChangeGarageType | garage base 0x96c048; type byte at garage+0x4c, door-state byte at +0x4d | ✅ true |
| 0x00447680 | CGarages::GetGarageNumberByName | garage count is a runtime global at 0x96c024 (loop bound) | ✅ true |
| 0x00447CB0 | CGarages::DeActivateGarage | DeActivateGarage sets flag bit 2 at garage+0x4e | ✅ true |
| 0x00447D50 | CGarage::OpenThisGarage | door state machine on garage+0x4d: from {0,2,5} -> 3 (OpenThisGarage) | ✅ true |
| 0x00447D70 | CGarage::CloseThisGarage | door state machine on garage+0x4d: from {1,3} -> 2 (CloseThisGarage) | ✅ true |
