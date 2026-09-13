# Vehicle Dynamics: Suspension, Tyres & Damage
*Source: `vehicle_dynamics.json`*
- **$schema:** vehicle_dynamics.v1
- **Generated:** tools/derive_vehicle_dynamics.py

## Summary
| Field | Value |
| --- | --- |
| Wheels | 4 |
| Suspension compression | 0x7d4 [0..1] (0=fully compressed, 1=relaxed) |
| Compression refs in code | 15 |
| CDamageManager | 0x18 |
| CAutomobile / CVehicle | 0x988 / 0x5a0 |
| Checks passed | 5/5 |

## CDamageManager fields
| Field | Type |
| --- | --- |
| m_fWheelDamageEffect | float |
| m_nEngineStatus | uint8 (0..250) |
| m_anWheelsStatus | eCarWheelStatus[4] |
| m_aDoorsStatus | eDoorStatus[6] |
| m_nLightsStatus | uint32 bitfield |
| m_nPanelsStatus | uint32 (per ePanels) |

## Checks
| Check | Result |
| --- | --- |
| suspension_compression_used | PASS |
| wheel_arrays_tile_float4 | PASS |
| four_wheels_six_doors | PASS |
| cdamagemanager_tiles | PASS |
| line_above_spring | PASS |
