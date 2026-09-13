# Executable Lifecycle
*Source: `lifecycle_structure.json`*
- **$schema:** lifecycle_structure.v1
- **Generated:** tools/derive_lifecycle.py

## Summary
| Field | Value |
| --- | --- |
| Entry point | 0x824570 |
| CGame::Process callees | 81 |
| CGame::Initialise callees | 142 |
| Checks | 4/4 |

## Per-frame update order (CGame::Process)
CPad::UpdatePads → CStreaming::Update → CCutsceneMgr::Update → CTheZones::Update → CCover::Update → CAudioZones::Update → CClock::Update → CWeather::Update → CTheScripts::Process → CCollision::Update → CTrain::UpdateTrains → CHeli::UpdateHelis → CDarkel::Update → CSkidmarks::Update → CGlass::Update → CWanted::UpdateEachFrame → CCreepingFire::Update → CSetPieces::Update → CPopulation::Update

## Checks
| Check | Result |
| --- | --- |
| entry_in_text | PASS |
| cgame_funcs_in_text | PASS |
| process_is_update_hub | PASS |
| initialise_is_init_hub | PASS |
