# object.dat Record Layout
*Source: `object_structure.json`*
- **Note:** Derived by tools/derive_object.py from GTA:SA 1.0 US object.dat + gta_sa.exe. All 15 checks pass. Do not hand-edit.

## Sscanf format
%s %f %f %f %f %f %f %f %d %d %d %d %d %f %f %f %s %f %f %f %f %f %d %d

## Sscanf format va
0x868dc8

## Fields
```
name, mass, turn_mass, air_resistance, elasticity, percent_submerged, uproot_limit, cdamage_multiplier, cdamage_effect, special_cdr, camera_avoid, causes_explosion, fx_type, fx_offset_x, fx_offset_y, fx_offset_z, fx_name, break_smash_mult, break_vx, break_vy, break_vz, break_v_rand, break_gun_mode, break_sparks
```

## Variants
| Field | Value |
| --- | --- |
| Basic fields | 17 |
| Breakable fields | 24 |
| Distinguished by | field count from sscanf, NOT field I |

## Census

### Basic
758

### Breakable
235

### Well formed
993

### Truncated
- JUD_LAN
- spraydoor_LAw2

### Markers
2

## Domains

### Cdamage effect
- 0
- 1
- 20
- 200
- 202

### Special cdr
- 0
- 1
- 2
- 4
- 6
- 7
- 8
- 9
- 20

### Fx type
- 0
- 1
- 2

### Break gun mode
- 0
- 1
- 2

## Named exceptions

### Special cdr out of range
- imy_bbox

### Camera avoid undocumented 2
- CARRIER_DOOR_SFSe
- portakabin

### Percent submerged over 120
- dump1

### Breakable declared but no params
- sec_keypad

### Mass over documented 50000
514

### Truncated records
- JUD_LAN
- spraydoor_LAw2
