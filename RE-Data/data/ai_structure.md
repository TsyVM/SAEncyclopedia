# Ped AI: Tasks, Wanted & Dispatch
*Source: `ai_structure.json`*
- **$schema:** ai_structure.v1
- **Generated:** tools/derive_ai.py

## Summary
| Field | Value |
| --- | --- |
| CTaskManager size | 0x30 |
| Primary task slots | 5 |
| Secondary task slots | 6 |
| Max wanted level | 6 stars |
| Max chaos level | 9200 |
| Max cops in pursuit | 10 |
| Max roadblocks | 325 |
| Checks passed | 6/6 |

## Wanted-star ladder (chaos points -> stars)
| Chaos >= | Stars |
| --- | --- |
| 4600 | 6 |
| 2400 | 5 |
| 1200 | 4 |
| 550 | 3 |
| 180 | 2 |
| 50 | 1 |

## CTaskManager (size 0x30, tiling)
| Field | Offset | Count |
| --- | --- | --- |
| m_aPrimaryTasks | 0x0 | 5 |
| m_aSecondaryTasks | 0x14 | 6 |
| m_pPed | 0x2c | 1 |

## Checks
| Check | Result |
| --- | --- |
| wanted_star_ladder | PASS |
| max_chaos_9200 | PASS |
| taskmanager_layout | PASS |
| taskmanager_tiles | PASS |
| roadblock_arrays_referenced | PASS |
| dispatch_tiers_in_text | PASS |
