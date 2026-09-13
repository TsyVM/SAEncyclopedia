# Font System Layout
*Source: `fonts_structure.json`*
- **Note:** Derived by tools/derive_fonts.py from GTA:SA 1.0 US shipped files + gta_sa.exe. All 21 checks pass. Do not hand-edit.

## Fonts dat

### Format
plain text; '#' comment; bracketed section tags

### Total fonts
2

### Record
| Field | Value |
| --- | --- |
| Prop count | 208 |
| Prop rows | 26 |
| Prop per row | 8 |
| Stride bytes | 210 |
| Layout | byte prop[208]; byte replacement_space; byte unprop |
| Storage base va | 0xC718B0 |

### Fonts
| Id | Prop | Unprop | Replacement space |
| --- | --- | --- | --- |
| 0 | `[12, 13, 13, 28, 28, 28, 28, 8, 17, 17, 30, 28, 28, 12, 9, 21, 28, 14, 28, 28, 28, 28, 28, 28, 28, 28, 13, 13, 30, 30, 30, 30, 10, 25, 23, 21, 24, 22, 20, 24, 24, 17, 20, 22, 20, 30, 27, 27, 26, 26...` | 27 | 10 |
| 1 | `[15, 9, 17, 27, 20, 34, 23, 12, 12, 12, 21, 20, 12, 14, 12, 15, 23, 15, 21, 21, 21, 21, 21, 21, 20, 21, 12, 12, 24, 24, 24, 19, 10, 22, 19, 19, 22, 16, 19, 24, 22, 11, 16, 21, 15, 28, 24, 27, 20, 2...` | 20 | 10 |

## Exe
| Key | Cols | Rows | Cell px | Col | Row | Tile uv |
| --- | --- | --- | --- | --- | --- | --- |
| Parser va | 0x7187C0 |  |  |  |  |  |
| Width lookup va | 0x7196F4 |  |  |  |  |  |
| Char remap va | 0x718770 / 0x7192C0 |  |  |  |  |  |
| Uv draw va | 0x718B30 |  |  |  |  |  |
| Record stride | 0xD2 (imul, both parser and consumer) |  |  |  |  |  |
| Outer rows | 26 |  |  |  |  |  |
| Inner per row | 8 |  |  |  |  |  |
| Char direct range | 0x00..0x9B (0x91->0x40 special; >0x9B->glyph 0) |  |  |  |  |  |
| Uv grid | 16 | 16 | 32 | idx & 0x0F | idx >> 4 | 0.0625 |

## Atlas

### Grid
16 x 16 cells of 32 x 32 px, visually confirmed against the executable's UV math

### Char mapping
glyph index = ASCII codepoint - 0x20 for the printable block (rows 0-5): idx 0 = space, idx 33 = 'A', etc.

### Extended block
rows 6+ hold accented Latin and a secondary alphabet

### Atlas exceeds width table
the atlas populates ALL 16 rows (~256 glyphs), but fonts.dat's proportional table covers only the first 208 (rows 0-12); glyphs 208-255 have no proportional width. (Corrects an earlier 'rows 13-15 unused' inference.)

### Row occupancy

#### Font2
```
11, 15, 16, 16, 14, 15, 13, 16, 16, 16, 15, 15, 16, 16, 16, 14
```

#### Font1
```
16, 16, 16, 16, 15, 15, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16
```

### Renders
- C23-Fonts-HUD/atlas_font1.png
- C23-Fonts-HUD/atlas_font2.png

## Fonts txd

### Num declared
2

### Textures
| Name | Mask | Width | Height | Depth | Levels | D3d | Raster format |
| --- | --- | --- | --- | --- | --- | --- | --- |
| font2 |  | 512 | 512 | 16 | 1 | DXT3 | 768 |
| font1 |  | 512 | 512 | 16 | 1 | DXT3 | 768 |

## Hud txd

### Num declared
69

### Textures
```
radardisc, skipicon, siterocket, siteM16, radar_ZERO, radar_WOOZIE, radar_waypoint, radar_tshirt, radar_truck, radar_triadsCasino, radar_triads, radar_TorenoRanch, radar_TORENO, radar_THETRUTH, radar_tattoo, radar_SWEET, radar_spray, radar_school, radar_saveGame, radar_RYDER, radar_runway, radar_race, radar_qmark, radar_propertyR, radar_propertyG, radar_police, radar_pizza, radar_OGLOC, radar_north, radar_modGarage, radar_MCSTRAP, radar_mafiaCasino, radar_MADDOG, radar_LocoSyndicate, radar_light, radar_impound, radar_hostpital, radar_gym, radar_girlfriend, radar_gangY, radar_gangP, radar_gangN, radar_gangG, radar_gangB, radar_Flag, radar_fire, radar_enemyAttack, radar_emmetGun, radar_diner, radar_dateFood, radar_dateDrink, radar_dateDisco, radar_CRASH1, radar_CJ, radar_chicken, radar_CESARVIAPANDO, radar_centre, radar_CATALINAPINK, radar_cash, radar_burgerShot, radar_bulldozer, radar_boatyard, radar_BIGSMOKE, radar_barbers, radar_ammugun, radar_airYard, radarRingPlane, fist, arrow
```

## Notes
| Field | Value |
| --- | --- |
| Mask variants | font1m/font2m requested by the exe but absent from fonts.txd (capability without data). |
| Hud extra | exe also references MODELS\HUD.TXD and 'ps2btns'. |
