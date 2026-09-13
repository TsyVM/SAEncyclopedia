# Surface Type Tables
*Source: `surface_structure.json`*
- **Note:** Derived by tools/derive_surfaces.py from GTA:SA 1.0 US shipped files + gta_sa.exe. All 12 checks pass. Do not hand-edit.

## Namespace
| Field | Value |
| --- | --- |
| Surface types | 179 |
| Agreement | surfinfo.dat = surfaud.dat = exe string table = 179 |
| Identical order | ✅ true |

## Surfinfo dat

### Records
179

### Columns
37

### Fields
```
name, adhesion_group, tyre_grip, wet_grip, skidmark, friction_effect, softland, see_thro, shoot_t, sand, water, s_water, beach, steep_sl, glass, stairs, skateable, pavement, roughness, flame, sparks, sprint, footsteps, footdust, cardirt, carclean, w_grass, w_gravel, w_mud, w_dust, w_sand, w_spray, proc_plant, proc_obj, climbable, bullet_fx, name_end
```

### Name repeated last column
✅ true

### Adhesion groups
- HARD
- LOOSE
- ROAD
- RUBBER
- SAND
- WET

### Skidmark domain
- DEFAULT
- MUDDY
- SANDY

### Friction effect domain
- NONE
- SPARKS

### Bullet fx domain
- DUST
- SAND
- SPARKS
- WOOD

### Tier
verified

## Surfaud dat

### Records
179

### Columns
10

### Fields
- name
- con
- grs
- snd
- grv
- wod
- wtr
- mtl
- lgs
- til

### Flags boolean
✅ true

### Tier
verified

## Surface dat

### Shape
6x6 lower-triangular adhesion/friction matrix

### Groups
- RUBBER
- HARD
- ROAD
- LOOSE
- SAND
- WET

### Values
21

### Matrix

#### Rubber
- 6.0

#### Hard
- 3.6
- 2.0

#### Road
- 4.5
- 3.0
- 6.0

#### Loose
- 3.2
- 3.5
- 2.0
- 1.0

#### Sand
- 3.0
- 4.0
- 2.0
- 1.0
- 1.0

#### Wet
- 2.8
- 2.0
- 1.0
- 1.0
- 1.0
- 0.5

### Self adhesion diagonal
- 6.0
- 2.0
- 6.0
- 1.0
- 1.0
- 0.5

### Tier
verified

## Procedural
| Field | Value |
| --- | --- |
| Procobj surface refs | 17 |
| Plants surface refs | 42 |
| Both subset of 179 | ✅ true |
