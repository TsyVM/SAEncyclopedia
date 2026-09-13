# Zone Record Layout & Census
*Source: `zone_structure.json`*

## Chapter
C22

## Subject
GTA:SA zones, radar grid and the map gridref

## Target
gta_sa.exe 1.0 US, md5 170b3a9108687b26da2d8901c6948a18

## Zone record

### Fields
- name
- type
- x1
- y1
- z1
- x2
- y2
- z2
- island
- textKey

### Exe scanf format
%s %d %f %f %f %f %f %f %d %s

### Exe format va
0x868d04

### Exe parser ref va
0x5b4ae9

### Loaded via
the IPL path; gta.dat lists `IPL DATA\MAP.ZON` and `IPL DATA\INFO.ZON`

### Tier
verified

## Info zon

### Records
378

### Types
| Field | Value |
| --- | --- |
| 0 | 378 |

### Islands
| Field | Value |
| --- | --- |
| 1 | 378 |

### Distinct names
377

### Distinct text keys
169

### Name equals text key
110

### Extent

#### X
- -2997.47
- 2997.06

#### Y
- -2892.97
- 2993.87

#### Z
- -242.99
- 900.0

### Gxt resolution

#### Resolved
169

#### Of
169

#### Misses
_(empty)_

### Tier
verified

## Map zon

### Records
6

### Types
| Field | Value |
| --- | --- |
| 3 | 6 |

### Islands
| Field | Value |
| --- | --- |
| 3 | 1 |
| 2 | 3 |
| 1 | 2 |

### Names
- Vegas
- SF01
- SF02
- SF03
- LA01
- LA02

### Text key
UNUSED (a sentinel; hashes to no GXT key)

### World span
6000.0

### Tier
verified

## Radar grid

### Tiles
144

### Grid
- 12
- 12

### Units per tile
500.0

### Naming
radar%02d.txd, in models/gta3.img

### Exe format va
0x866b98

### Exe loader va
0x588015

### Exe loop bound
0x90 (144)

### Exe handle array va
0xba8478

### Orientation
| Key | NW(-3000,+3000) | NE(+3000,+3000) | SW(-3000,-3000) | SE(+3000,-3000) |
| --- | --- | --- | --- | --- |
| Conversion va | 0x5858d0 |  |  |  |
| Col | floor((worldX + 3000) * (1/500))  -> 0 at west, 11 at east |  |  |  |
| Row | floor(11 - (worldY + 3000) * (1/500))  -> 0 at north, 11 at south |  |  |  |
| Index | col + row*12  (lea at 0x584b7b / 0x584bca) |  |  |  |
| Corners | 0 | 11 | 132 | 143 |
| Tier | verified |  |  |  |

### Tier
verified (count, naming, grid arithmetic, tile orientation)

## Gridref

### Grid
- 10
- 10

### Cells
100

### Units per cell
600.0

### Record bytes
32

### Exe path va
0x87295c

### Exe format
%c%d %s

### Exe loader va
0x71d500

### Index arithmetic
((letter - 'A') * 10 + (num - 1)) * 32

### Artists
| Field | Value |
| --- | --- |
| STUARTM | 34 |
| SCOTT | 14 |
| ANDREWSO | 12 |
| GARY | 9 |
| WAYLAND | 8 |
| STEVEM | 7 |
| NIK | 5 |
| ADAMC | 4 |
| JIMA | 4 |
| SIMONL | 3 |

### Note
a development ownership table - maps map-grid cells to Rockstar artist user IDs - shipped in retail and still parsed by the engine

### Consumers

#### Table base va
0xc72fb0

#### Loader va
0x71d4e0

#### Loader called from
0x5BA392 (init sequence)

#### Accessors va
- 0x71d5a0
- 0x71d5e0
- 0x71d650
- 0x71d670

#### Finding
The loader IS called at init, so the 100-record table at 0xC72FB0 is populated. But the four accessor functions (world->cell x2, a validity check, and the getter that returns atoi(name)) have ZERO call/jmp/pointer references anywhere in the executable. The table is WRITE-ONLY in retail: loaded and never read. The consuming feature was cut, leaving the reader API as unreferenced dead code.

#### Tier
verified

### Tier
verified (grid, completeness, record size, index arithmetic, consumers)

## Checks
| Name | Pass | Detail |
| --- | --- | --- |
| info.zon: every record has exactly 10 fields | ✅ true | 378 records |
| map.zon: every record has exactly 10 fields | ✅ true | 6 records |
| both files are a single `zone` ... `end` section | ✅ true |  |
| info.zon: every bounding box has min < max on all three axes | ✅ true | 378/378 |
| map.zon: every bounding box has min < max on all three axes | ✅ true | 6/6 |
| map.zon spans exactly -3000..+3000 on X and Y (the world square) | ✅ true | span 6000 |
| info.zon lies inside the world square | ✅ true | X -2997.47..2997.06  Y -2892.97..2993.87 |
| EVERY info.zon text key resolves to a GXT key (C19 cross-check) | ✅ true | 169/169, misses: none |
| map.zon's text key is the sentinel UNUSED and resolves to nothing | ✅ true |  |
| gta3.img holds exactly 144 radarNN.txd tiles, numbered 0..143 with no gap | ✅ true | 144 tiles |
| 144 tiles form a 12 x 12 grid over the 6000-unit world | ✅ true | 500 units per tile |
| exe world->tile conversion uses +3000, x(1/500) and an 11-row Y-flip | ✅ true | off=3000 inv=0.002000 flip=11 |
| exe conversion opcodes present at VA 0x5858D0 (fadd/fmul/fsubr) | ✅ true |  |
| exe tile index is col + row*12  (lea [ecx+ecx*2]; [eax+ecx*4]) | ✅ true |  |
| tile 0 = NW corner; 11 = NE; 132 = SW; 143 = SE | ✅ true | corners->tiles {'NW': 0, 'NE': 11, 'SW': 132, 'SE': 143} |
| gridref.dat is a complete 10 x 10 grid (A1..J10) | ✅ true | 100 cells, 10 artists |
| the exe's index arithmetic maps the 100 cells onto 0..3168 step 32, no collision | ✅ true | record size 32 bytes |
| gridref table base 0xC72FB0 is written by loader and getter | ✅ true |  |
| gridref loader (0x71D4E0) IS called (so the table is populated at init) | ✅ true | 1 call/jmp refs |
| the 4 gridref accessor functions are DEAD CODE (zero refs anywhere) | ✅ true | refs {'0x71d5a0': (0, 0), '0x71d5e0': (0, 0), '0x71d650': (0, 0), '0x71d670': (0, 0)} |
