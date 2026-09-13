# Data-Folder Sweep (remaining retail files)
*Source: `datasweep.json`*
- **$schema:** datasweep.v1
- **Generated:** tools/derive_datasweep.py

## Files
| File | Kind | Key figure |
| --- | --- | --- |
| clothes.dat | clothing-rule language | 87 SETC rules |
| shopping.dat | economy/pricing tree | 37 sections |
| furnitur.dat | interior furnishing | 6 groups |
| numplate.dat | carplate charset colour palettes (JASC .PAL) |  |
| animgrp.dat | animation groups | 42 groups |
| water1.dat | water quads | 267 quads |
| timecycp.dat | alternate timecycle (same format as timecyc.dat) | 437 lines |
| default.dat | boot load list (IDE/IMG/COLFILE) |  |
| gta_quick.dat | quick-boot load list variant |  |
| default.ide | always-loaded object/weapon definitions (IDE) |  |
| txdcut.ide | cutscene TXD parent assignments (txdp section) |  |
| animviewer.dat | dev anim-viewer IDE load list |  |
| polydensity.dat | binary uint32 map polygon-density grid | 72005 u32 cells |

## Checks
| Check | Result |
| --- | --- |
| clothes_rule_verbs | PASS |
| shopping_sections | PASS |
| furnitur_groups | PASS |
| animgrp_groups | PASS |
| water1_quads | PASS |
| timecycp_is_timecycle | PASS |
| load_config_files | PASS |
| polydensity_binary_u32 | PASS |
