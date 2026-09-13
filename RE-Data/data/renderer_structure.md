# Render Pipeline (CRenderer)
*Source: `renderer_structure.json`*
- **$schema:** renderer_structure.v1
- **Generated:** tools/derive_renderer.py

## Summary
| Field | Value |
| --- | --- |
| Render lists | 4 (tile 0xb745d8 -> 0xb76838) |
| Camera matrix global | 0xb6f03c |
| Checks passed | 5/5 |

## Render lists (CEntity* [count], tiling)
| List | Base | Count | Bytes | End |
| --- | --- | --- | --- | --- |
| ms_aInVisibleEntityPtrs | 0xb745d8 | 150 | 0x258 | 0xb74830 |
| ms_aVisibleSuperLodPtrs | 0xb74830 | 50 | 0xc8 | 0xb748f8 |
| ms_aVisibleLodPtrs | 0xb748f8 | 1000 | 0xfa0 | 0xb75898 |
| ms_aVisibleEntityPtrs | 0xb75898 | 1000 | 0xfa0 | 0xb76838 |

## Pass order
ScanWorld -> ConstructRenderList -> RenderRoads -> RenderEverythingBarRoads -> RenderFadingInEntities

## Functions
| Name | VA |
| --- | --- |
| ScanWorld | 0x554fe0 |
| ConstructRenderList | 0x5556e0 |
| SetupScanLists | 0x553540 |
| PreRender | 0x553910 |
| RenderRoads | 0x553a10 |
| RenderEverythingBarRoads | 0x553aa0 |
| RenderFadingInEntities | 0x5531e0 |
| RenderOneRoad | 0x553230 |
| RenderOneNonRoad | 0x553260 |

## Checks
| Check | Result |
| --- | --- |
| render_lists_referenced_in_text | PASS |
| render_lists_tile | PASS |
| pass_reads_visible_count | PASS |
| construct_reads_camera_matrix | PASS |
| render_functions_in_text | PASS |
