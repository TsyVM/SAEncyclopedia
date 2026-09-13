# Front-End / Pause Menu (CMenuManager + aScreens)
*Source: `menu_structure.json`*
- **$schema:** menu_structure.v1
- **Generated:** tools/derive_menu.py

## Summary
| Field | Value |
| --- | --- |
| CMenuManager size | 0x1b78 |
| aScreens table | 0x8ce008 .. 0x8d06e0 |
| Screen count | 44 |
| tMenuScreen stride | 0xe2 |
| tMenuScreenItem stride | 0x12 (12/screen) |
| Menu actions | 70 |
| Frontend sprites | 25 |
| Checks passed | 6/6 |

## Screen titles (GXT keys, first 12)
'FEP_STA', 'FEH_LOA', 'FEH_BRI', 'FEH_AUD', 'FEH_DIS', 'FEH_MAP', 'FES_NGA', 'FES_NGA', 'FES_LMI', 'FET_LG', 'FES_DEL', 'FET_LG'

## Checks
| Check | Result |
| --- | --- |
| tmenuscreen_stride_0xE2 | PASS |
| tmenuscreen_tiles | PASS |
| tmenuitem_stride_0x12 | PASS |
| ascreens_titles_are_gxt | PASS |
| item_names_are_gxt | PASS |
| screen_count_44 | PASS |
