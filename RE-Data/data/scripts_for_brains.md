# CScriptsForBrains Slot Table
*Source: `scripts_for_brains.json`*
- **$schema:** scripts_for_brains.v1
- **Generated:** tools/derive_scriptsforbrains.py

## Summary
| Field | Value |
| --- | --- |
| Slots | 70 |
| Stride | 20 |
| Table bytes | 1400 |
| Free sentinel | 0xFFFF (id word) |
| This relative | ✅ true |
| Facts checked | 3 |
| Facts passed | 3 |

## Record fields
| Offset | Size | Name |
| --- | --- | --- |
| 0 | 2 | model id (0xFFFF = free slot) |
| 2 | 1 | flag byte |
| 3 | 1 | flag byte |
| 4 | 1 | enabled/active byte (Init sets 1) |
| 8 | 4 | dword (from 0x859D9C template) |
| 12 | 2 | word |
| 14 | 2 | word |
| 16 | 4 | dword |

## Structural facts
| Entry va | Method | Fact | Verified |
| --- | --- | --- | --- |
| 0x0046A8C0 | CScriptsForBrains::Init | Init builds 0x46 (70) records of stride 0x14 (20 B); sets a byte @rec+4 = 1 | ✅ true |
| 0x0046A900 | CScriptsForBrains::SwitchAllObjectBrainsWithThisID | SwitchAllObjectBrainsWithThisID walks the same 70-slot x 20-byte table | ✅ true |
| 0x0046A9C0 | CScriptsForBrains::AddNewStreamedScriptBrainForCodeUse | AddNew…: slot = index*20 (lea *5 then *4); a free slot has id word == -1 (0xFFFF) | ✅ true |
