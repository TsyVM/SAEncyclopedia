# weapon.dat Record Layout
*Source: `weapon_structure.json`*

## Source
weapon.dat $ gun records + gta_sa.exe 1.0 US cross-check (C14.4)

## Dollar record

### Active rows
53

### Disabled rows
5

### Widths
| Field | Value |
| --- | --- |
| 25 | 43 |
| 29 | 10 |

### Fields 25
```
name, eFireType, targetRange, weaponRange, modelId1, modelId2, weaponslot, assocGroupId, ammoClip, damage, fireOffX, fireOffY, fireOffZ, skillLevel, reqStat, accuracy, moveSpeed, anim1start, anim1end, anim1fire, anim2start, anim2end, anim2fire, breakout, flags
```

### Fields 29 extra
- speed
- radius
- lifespan
- spread

## Skill level census
| Field | Value |
| --- | --- |
| Triple tier weapons | 10 |
| Single tier weapons | 19 |
| Pistol rows | 4 |
| Pistol undocumented tier | P=3 (colt_cop model, reqStat=5000) |

## Flags bit table
| 1 | 2 | 4 | 8 |
| --- | --- | --- | --- |
| CANAIM | AIMWITHARM | 1STPERSON | ONLYFREEAIM |
| MOVEAIM | MOVEFIRE |  |  |
| THROW | HEAVY | CONTINUOUSFIRE | TWIN_PISTOL |
| RELOAD | CROUCHFIRE | RELOAD2START | LONG_RELOAD |
| SLOWSDWN | RANDSPEED | EXPANDS |  |

## Undocumented bits

### Nibble2 bit8
- COUNTRYRIFLE
- RLAUNCHER
- RLAUNCHER_HS
- SNIPERRIFLE

### Nibble5 bit8
- COUNTRYRIFLE
- EXTINGUISHER
- FTHROWER
- MINIGUN

## Disabled rows

### COUNTRYRIFLE
- P=0
- P=2

### SNIPERRIFLE
- P=0
- P=2

### JETPACK
- fully cut (only row)

### SKATEBOARD (melee £)
- fully cut (only row)

## Exe crosscheck

### Exe md5
170b3a9108687b26da2d8901c6948a18

### Weapontype table
```
UNARMED, BRASSKNUCKLE, GOLFCLUB, NIGHTSTICK, KNIFE, BASEBALLBAT, SHOVEL, POOLCUE, KATANA, CHAINSAW, DILDO1, DILDO2, VIBE1, VIBE2, FLOWERS, CANE, GRENADE, TEARGAS, MOLOTOV, ROCKET, ROCKET_HS, FREEFALL_BOMB, PISTOL, PISTOL_SILENCED, DESERT_EAGLE, SHOTGUN, SAWNOFF, SPAS12, MICRO_UZI, MP5, AK47, M4, TEC9, COUNTRYRIFLE, SNIPERRIFLE, RLAUNCHER, RLAUNCHER_HS, FTHROWER, MINIGUN, SATCHEL_CHARGE, DETONATOR, SPRAYCAN, EXTINGUISHER, CAMERA, NIGHTVISION, INFRARED, PARACHUTE, , ARMOUR
```

### Weapontype table size
49

### Nameless slot index
47

### Names absent from exe table
- JETPACK
- SKATEBOARD

### Exe names absent from weapon dat
- ARMOUR

### Reload sample time string found
❌ false

### Flags bit names found in exe
_(empty)_

## Exe crosscheck checks
| Check | Passed | Detail |
| --- | --- | --- |
| gta_sa.exe MD5 matches the documented 1.0 US retail binary | ✅ true | 170b3a9108687b26da2d8901c6948a18 |
| WEAPONTYPE pointer table found and starts at UNARMED | ✅ true |  |
| WEAPONTYPE pointer table ends at ARMOUR | ✅ true |  |
| WEAPONTYPE pointer table has 49 entries (48 named + 1 blank slot) | ✅ true |  |
| exactly one nameless WEAPONTYPE slot, at id 47 (between PARACHUTE and ARMOUR) | ✅ true |  |
| every weapon.dat name is in the exe table EXCEPT JETPACK and SKATEBOARD | ✅ true |  |
| ARMOUR is the only exe-table name absent from weapon.dat (non-weapon pickup) | ✅ true |  |
| no trace of 'reloadSampleTime' anywhere in the executable (case-insensitive) | ✅ true |  |
| none of the 15 documented flags-bit names appear as standalone strings in the executable | ✅ true |  |
