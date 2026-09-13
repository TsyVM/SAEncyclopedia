# Vehicle Physics: Handling Manager + CPhysical
*Source: `physics_structure.json`*
- **$schema:** physics_structure.v1
- **Generated:** tools/derive_physics.py

## Summary
| Field | Value |
| --- | --- |
| Handling manager | 0xc2b9c8 (size 0xc624) |
| tHandlingData size | 0xe0 |
| Vehicle handling entries | 210 |
| VehicleNames stride | 14 |
| CPhysical size | 0x138 |
| Checks passed | 8/8 |

## Handling arrays
| Array | Count | Stride | Bytes |
| --- | --- | --- | --- |
| vehicle | 210 | 0xe0 | 0xb7c0 |
| bike | 13 | 0x40 | 0x340 |
| flying | 24 | 0x58 | 0x840 |
| boat | 12 | 0x3c | 0x2d0 |
| **total** |  |  | 0xc610 |

## Checks
| Check | Result |
| --- | --- |
| thandling_stride_0xE0 | PASS |
| drive_type_field_at_0x88 | PASS |
| vehicle_names_stride_14 | PASS |
| gethandlingid_uses_names | PASS |
| handling_mgr_referenced | PASS |
| handling_arrays_fit | PASS |
| c13_handling_count_tie | PASS |
| cphysical_functions_in_text | PASS |
