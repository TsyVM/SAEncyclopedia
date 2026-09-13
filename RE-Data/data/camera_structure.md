# Camera Structure (CCamera / CCam)
*Source: `camera_structure.json`*
- **$schema:** camera_structure.v1
- **Generated:** tools/derive_camera.py

## Summary
| Field | Value |
| --- | --- |
| TheCamera singleton | 0xb6f028 |
| CCam stride | 0x238 (568) |
| CCam stride sites (imul) | 27 |
| CCamera size | 0xd78 (3448) |
| CCamera size tier | reasoned (bracketed 0x81C..0xE18; disassembly VALIDATE_SIZE 0xD78) |
| m_aCams offset | 0x174 |
| m_aCams count | 3 |
| m_nActiveCam offset | 0x59 |
| Camera modes | 66 (0..65) |
| Checks passed | 6/6 |

## GetActiveCam indexing
`m_aCams[m_nActiveCam] : movzx idx,[0xB6F081]; imul idx,0x238; add idx,0xB6F19C`

## Key functions
| Name | VA |
| --- | --- |
| Init | 0x5BC520 |
| Process | 0x52B730 |
| TakeControl | 0x50C7C0 |
| GetGameCamPosition | 0x50AE50 |
| GetLookDirection | 0x50AE90 |

## Checks
| Check | Result |
| --- | --- |
| ccam_stride_0x238 | PASS |
| m_acams_base_0xB6F19C | PASS |
| m_activecam_read_0xB6F081 | PASS |
| acams_offset_arithmetic | PASS |
| ccamera_size_bracketed | PASS |
| c38_sky_reads_thecamera | PASS |
