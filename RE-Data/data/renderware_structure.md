# RenderWare 3.6 Reference
*Source: `renderware_structure.json`*
- **$schema:** renderware_structure.v1
- **Generated:** tools/derive_renderware.py

## Summary
| Field | Value |
| --- | --- |
| Version | RenderWare 3.6 (RW36Active) |
| Modules linked | 88 |
| Plugins | anisot, hanim, matfx, skin2, uvanim |
| D3D9 modules | 21 |
| Checks | 5/5 |

## Modules by subsystem
| Subsystem | # |
| --- | --- |
| core (plcore) | 14 |
| core objects | 23 |
| device driver | 9 |
| os layer | 1 |
| plugins | 14 |
| tools | 5 |
| world / BSP | 22 |

## Object model
| Object | Role |
| --- | --- |
| RwEngine | the singleton device/plugin registry (plcore) |
| RpWorld | the BSP world sectors atomics are drawn against (world) |
| RpClump/RpAtomic | a model (clump) of drawable pieces (atomics) - the DFF (C7) |
| RpGeometry | vertex/index/material data of an atomic (C8) |
| RpMaterial/RwTexture/RwRaster | material -> texture -> GPU raster (C9) |
| RwFrame | the frame hierarchy (transform tree) atomics attach to |
| RwCamera | the view/projection (C39 CCamera wraps it) |

## Checks
| Check | Result |
| --- | --- |
| renderware_36 | PASS |
| module_count | PASS |
| core_present | PASS |
| d3d9_backend | PASS |
| plugins_present | PASS |
