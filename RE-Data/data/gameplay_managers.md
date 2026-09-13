# Gameplay Manager Struct Layouts
*Source: `gameplay_managers.json`*
- **$schema:** gameplay_managers.v1
- **Generated:** tools/derive_gameplay_managers.py

## Summary
| Field | Value |
| --- | --- |
| Replay blocks | 8 |
| Shop items | 560 |
| Entryexit record | 60 |
| Facts checked | 10 |
| Facts passed | 10 |

## Managers
| Key | Element | Record stride | Pool global | Visible objects slots | Visible objects base | Buffer base | Buffer end | Block size | Ped conv table | Ped conv entries | Ped packet stride | Num items | Bought id array | Has bought flags | Shop record stride | Clothes buffer | Clothes dwords |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| CEntryExitManager | CEntryExit | 60 | 9873368 | 32 | 9873208 |  |  |  |  |  |  |  |  |  |  |  |  |
| CReplay |  |  |  |  |  | 9960328 | 10760328 | 100000 | 9959480 | 140 | 1988 |  |  |  |  |  |  |
| CShopping |  |  |  |  |  |  |  |  |  |  |  | 560 | 11107728 | 11104928 | 24 | 11118608 | 30 |

## Structural facts
| Entry va | Method | Fact | Verified |
| --- | --- | --- | --- |
| 0x0043FD50 | CEntryExitManager::DeleteOne | CEntryExit record = 0x3C (60 B); DeleteOne divides ptr->index by 60 (reciprocal 0x88888889); pool object at 0x96a7d8 | ✅ true |
| 0x0043F180 | CEntryExitManager::EnableBurglaryHouses | pool layout: array ptr @ pool+0, flag-byte array @ pool+4, count @ pool+8 | ✅ true |
| 0x0043ECF0 | CEntryExitManager::SetAreaCodeForVisibleObjects | visible-objects array = 32 slots (cmp 0x20) at 0x96a738, count at 0x96a7dc | ✅ true |
| 0x0045D4B0 | CReplay::StreamAllNecessaryCarsAndPeds | replay buffer = 8 blocks x 0x186A0 (100,000 B) at 0x97FB88..0xA43088 | ✅ true |
| 0x0045D6C0 | CReplay::FindFirstFocusCoordinate | same 8 x 100,000 replay buffer walked in FindFirstFocusCoordinate | ✅ true |
| 0x0045EF20 | CReplay::InitialisePedPoolConversionTable | ped-pool conversion table = 0x8C (140) entries at 0x97F838; ped packet stride 0x7C4 | ✅ true |
| 0x0049B5E0 | CShopping::HasPlayerBought | shop items = 0x230 (560); bought-id array @ 0xA97D90, has-bought byte array @ 0xA972A0 | ✅ true |
| 0x0049B640 | CShopping::ShutdownForRestart | ShutdownForRestart clears 0x8C (140) dwords = 560 bytes of the flag array at 0xA972A0 | ✅ true |
| 0x0049B200 | CShopping::StoreClothesState | clothes-state buffer = 0x1E (30) dwords (120 B) at 0xA9A810; player-info stride 0x190 | ✅ true |
| 0x0049ADE0 | CShopping::GetExtraInfo | shop-section record stride = 0x18 (24 B) (GetExtraInfo; C27.3-verified method) | ✅ true |
