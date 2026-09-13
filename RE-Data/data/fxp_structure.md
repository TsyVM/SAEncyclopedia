# effects.fxp Particle Format
*Source: `fxp_structure.json`*

## Chapter
C21

## Subject
GTA:SA particle effects (models/effects.fxp)

## Target
gta_sa.exe 1.0 US, md5 170b3a9108687b26da2d8901c6948a18

## Encoding
| Field | Value |
| --- | --- |
| Form | plain text |
| Line ending | CRLF |
| Bytes | 616708 |
| Lines | 43617 |
| Tier | verified |

## Grammar
| Field | Value |
| --- | --- |
| Project | 'FX_PROJECT_DATA:' system* 'FX_PROJECT_DATA_END:' |
| System | 'FX_SYSTEM_DATA:' <version> FILENAME NAME LENGTH LOOPINTERVALMIN LENGTH PLAYMODE CULLDIST BOUNDINGSPHERE 'NUM_PRIMS: n' prim{n} OMITTEXTURES TXDNAME |
| Prim | 'FX_PRIM_EMITTER_DATA:' 'FX_PRIM_BASE_DATA:' NAME MATRIX TEXTURE TEXTURE2 TEXTURE3 TEXTURE4 ALPHAON SRCBLENDID DSTBLENDID 'NUM_INFOS: n' info{n} LODSTART LODEND |
| Info | 'FX_INFO_<TYPE>_DATA:' scalar* (curveName ':' interp)* |
| Interp | 'FX_INTERP_DATA:' LOOPED 'NUM_KEYS: n' keyframe{n} |
| Keyframe | 'FX_KEYFLOAT_DATA:' TIME VAL |
| Tier | verified - a count-driven parser consumes all 43617 lines with 0 left over |

## Counts
| Field | Value |
| --- | --- |
| Systems | 82 |
| Emitters | 161 |
| Info blocks | 1470 |
| Curves | 4120 |
| Keyframes | 6563 |

## System version word
| Field | Value |
| --- | --- |
| Value | 109 |
| Constant | ✅ true |
| Tier | open |

## Info types

### FX INFO SIZE DATA

#### Count
161

#### Scalars
- TIMEMODEPRT

#### Curves
- SIZEX
- SIZEY
- SIZEXBIAS
- SIZEYBIAS

### FX INFO EMLIFE DATA

#### Count
147

#### Scalars
_(empty)_

#### Curves
- LIFE
- BIAS

### FX INFO EMRATE DATA

#### Count
144

#### Scalars
_(empty)_

#### Curves
- RATE

### FX INFO EMSPEED DATA

#### Count
140

#### Scalars
_(empty)_

#### Curves
- SPEED
- BIAS

### FX INFO EMANGLE DATA

#### Count
122

#### Scalars
_(empty)_

#### Curves
- MIN
- MAX

### FX INFO FORCE DATA

#### Count
111

#### Scalars
- TIMEMODEPRT

#### Curves
- FORCEX
- FORCEY
- FORCEZ

### FX INFO ROTSPEED DATA

#### Count
108

#### Scalars
- TIMEMODEPRT

#### Curves
- MINCW
- MAXCW
- MINCCW
- MAXCCW

### FX INFO COLOURBRIGHT DATA

#### Count
102

#### Scalars
- TIMEMODEPRT

#### Curves
- RED
- GREEN
- BLUE
- ALPHA
- BIAS

### FX INFO EMROTATION DATA

#### Count
85

#### Scalars
_(empty)_

#### Curves
- ANGLEMIN
- ANGLEMAX

### FX INFO EMDIR DATA

#### Count
65

#### Scalars
_(empty)_

#### Curves
- DIRX
- DIRY
- DIRZ

### FX INFO EMSIZE DATA

#### Count
62

#### Scalars
_(empty)_

#### Curves
- RADIUS
- SIZEMINX
- SIZEMAXX
- SIZEMINY
- SIZEMAXY
- SIZEMINZ
- SIZEMAXZ

### FX INFO COLOUR DATA

#### Count
48

#### Scalars
- TIMEMODEPRT

#### Curves
- RED
- GREEN
- BLUE
- ALPHA

### FX INFO FRICTION DATA

#### Count
46

#### Scalars
- TIMEMODEPRT

#### Curves
- FRICTION

### FX INFO WIND DATA

#### Count
38

#### Scalars
- TIMEMODEPRT

#### Curves
- WINDFACTOR

### FX INFO SELFLIT DATA

#### Count
28

#### Scalars
- TIMEMODEPRT

#### Curves
_(empty)_

### FX INFO DIR DATA

#### Count
12

#### Scalars
- TIMEMODEPRT

#### Curves
- X
- Y
- Z

### FX INFO HEATHAZE DATA

#### Count
9

#### Scalars
- TIMEMODEPRT

#### Curves
_(empty)_

### FX INFO SPRITERECT DATA

#### Count
8

#### Scalars
- TIMEMODEPRT

#### Curves
- TOP
- BOTTOM
- LEFT
- RIGHT

### FX INFO JITTER DATA

#### Count
8

#### Scalars
- TIMEMODEPRT

#### Curves
- JITTERFACTOR

### FX INFO FLAT DATA

#### Count
6

#### Scalars
- TIMEMODEPRT

#### Curves
- RX
- RY
- RZ
- UX
- UY
- UZ
- AX
- AY
- AZ

### FX INFO EMPOS DATA

#### Count
5

#### Scalars
_(empty)_

#### Curves
- X
- Y
- Z

### FX INFO FLOAT DATA

#### Count
3

#### Scalars
- TIMEMODEPRT

#### Curves
_(empty)_

### FX INFO NOISE DATA

#### Count
3

#### Scalars
- TIMEMODEPRT

#### Curves
- NOISE

### FX INFO UNDERWATER DATA

#### Count
3

#### Scalars
- TIMEMODEPRT

#### Curves
_(empty)_

### FX INFO TRAIL DATA

#### Count
2

#### Scalars
- TIMEMODEPRT

#### Curves
- TRAILTIME
- SCREENSPACE

### FX INFO GROUNDCOLLIDE DATA

#### Count
1

#### Scalars
- TIMEMODEPRT

#### Curves
- BOUNCE
- SPEEDMULT
- BOUNCEERROR

### FX INFO ANIMTEX DATA

#### Count
1

#### Scalars
- TIMEMODEPRT

#### Curves
- TEXID

### FX INFO EMWEATHER DATA

#### Count
1

#### Scalars
_(empty)_

#### Curves
- WINDMIN
- WINDMAX
- RAINMIN
- RAINMAX

### FX INFO ATTRACTPT DATA

#### Count
1

#### Scalars
- TIMEMODEPRT

#### Curves
- POSX
- POSY
- POSZ
- FORCE

## Executable confirmation

### Fxp path string va
0x85a6d4

### Loader ref va
0x49ea9d

### Parser va
0x5c2420

### Tag match
repe cmpsb over 0x10 bytes at 0x5c253e - block tags are compared as literals

### Fx data tokens in exe
34

### Declared but unused
- FX_INFO_ATTRACTLINE_DATA
- FX_INFO_COLOURRANGE_DATA
- FX_INFO_SMOKE_DATA

### Structural tags absent
- FX_PROJECT_DATA
- FX_PRIM_BASE_DATA
- FX_INTERP_DATA
- FX_KEYFLOAT_DATA

### Note
scalar field names do not appear in the exe; fields are read positionally

## Checks
| Name | Pass | Detail |
| --- | --- | --- |
| file is pure printable text (CRLF, no control bytes) | ✅ true | 616708 bytes, 43617 lines |
| parser consumes every line, nothing left over | ✅ true | 43617 lines |
| declared NUM_PRIMS total == emitter blocks found | ✅ true | 161 prims |
| declared NUM_INFOS total == info blocks found | ✅ true | 1470 infos |
| declared NUM_KEYS total == keyframe blocks found | ✅ true | 6563 keys |
| every FX_INTERP_DATA carries exactly one NUM_KEYS | ✅ true | 4120 interps |
| every FX_INFO type has ONE constant child schema | ✅ true | 29 types |
| the FX_SYSTEM_DATA version word is constant | ✅ true | value = 109 on all 82 systems |
| every FX_INFO type in the file is a literal in gta_sa.exe | ✅ true | 29/29 |
| the exe declares 3 block types the shipped file never uses | ✅ true | FX_INFO_ATTRACTLINE_DATA, FX_INFO_COLOURRANGE_DATA, FX_INFO_SMOKE_DATA |
| structural block tags are ABSENT from the exe (parsed positionally) | ✅ true | FX_PROJECT_DATA, FX_PRIM_BASE_DATA, FX_INTERP_DATA, FX_KEYFLOAT_DATA |
