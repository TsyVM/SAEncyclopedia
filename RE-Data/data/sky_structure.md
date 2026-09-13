# Sky / Skybox Rendering (CClouds)
*Source: `sky_structure.json`*
- **$schema:** sky_structure.v1
- **Generated:** tools/derive_sky.py

## Summary
| Field | Value |
| --- | --- |
| Sky is textured | False |
| Sky textures TXD | particle |
| Star positions | 9 |
| Stars drawn | 11 |
| Horizon bands | 6 |
| Moving-fog slots | 350 |
| Volumetric-cloud slots (disassembly) | 180 |
| Moon size default | 3 |
| Checks passed | 10/10 |

## CClouds functions
| Name | VA |
| --- | --- |
| Init | 0x7138d0 |
| Update | 0x712ff0 |
| Shutdown | 0x712fa0 |
| SetUpOneSkyPoly | 0x713060 |
| Render | 0x713950 |
| RenderSkyPolys | 0x714650 |
| MovingFog_GetFirstFreeSlot | 0x713710 |
| VolumetricClouds_GetFirstFreeSlot | 0x7135c0 |

## Sky textures (particle.txd)
| Texture | Global | Mask | Role |
| --- | --- | --- | --- |
| cloud1 | 0xC6AA78 |  | scrolling low cloud |
| cloudmasked | 0xC6AA7C |  | masked cloud |
| cloudhigh | 0xC6AA74 | cloudhighm | high/volumetric cloud |
| lunar | 0xC6BFC8 | lunarm | moon |

## Star tables (float[9], tile at 0x24)
- Y @ 0x8D55EC: [0.0, 0.05, 0.13, 0.4, 0.7, 0.6, 0.27, 0.55, 0.75]
- Z @ 0x8D5610: [0.0, 0.45, 0.9, 1.0, 0.85, 0.52, 0.48, 0.35, 0.2]
- size @ 0x8D5634: [1.0, 1.4, 0.9, 1.0, 0.6, 1.5, 1.3, 1.0, 0.8]

## Timecyc link (C15)
- **current_colours:** 0xB7C4A0
- **below_horizon_grey:** 0xB7CB10
- **sky_top_table:** 0xB7BF78
- **sky_bottom_table:** 0xB7BD50
- **note:** RenderSkyPolys blends m_BelowHorizonGrey (a lerp of m_CurrentColours.SkyBottom) - the C15 timecyc columns

## Checks
| Check | Result |
| --- | --- |
| cclouds_function_map | ✅ |
| sky_texture_names_present | ✅ |
| init_loads_sky_textures | ✅ |
| skybox_is_untextured_gradient | ✅ |
| skypolys_reads_timecyc_grey | ✅ |
| star_tables_tile | ✅ |
| star_table_values | ✅ |
| render_reads_star_tables | ✅ |
| moving_fog_count_350 | ✅ |
| moon_size_default_3 | ✅ |
