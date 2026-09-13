# ped.dat / pedgrp.dat Layout
*Source: `ped_tables_structure.json`*
- **Note:** Derived by tools/derive_peds.py from GTA:SA 1.0 US ped.dat + pedgrp.dat + peds.ide + gta_sa.exe. All 10 checks pass.

## Ped dat

### Types
```
CIVMALE, CIVFEMALE, COP, GANG1, GANG2, GANG3, GANG4, GANG5, GANG6, GANG7, GANG8, GANG9, GANG10, MEDIC, FIREMAN, CRIMINAL, PROSTITUTE
```

### Count
17

### Verbs used
- Hate
- Respect

### Verbs documented
- Hate
- Dislike
- Like
- Respect

### Referenced not defined
- DEALER

### Defined not referenced
- PROSTITUTE

### All names in exe
18/18

### Gang clique
✅ true

### Relationships

#### CIVMALE

##### Respect
- CIVMALE
- CIVFEMALE

#### CIVFEMALE

##### Respect
- CIVFEMALE
- CIVMALE

#### COP

##### Hate
- CRIMINAL
- DEALER

##### Respect
- MEDIC
- FIREMAN
- COP

#### GANG1

##### Hate
- GANG2
- GANG3
- GANG4
- GANG5
- GANG6
- GANG7
- GANG8
- GANG9
- GANG10

##### Respect
- GANG1

#### GANG2

##### Hate
- GANG1
- GANG3
- GANG4
- GANG5
- GANG6
- GANG7
- GANG8
- GANG9
- GANG10

##### Respect
- GANG2

#### GANG3

##### Hate
- GANG1
- GANG2
- GANG4
- GANG5
- GANG6
- GANG7
- GANG8
- GANG9
- GANG10

##### Respect
- GANG3

#### GANG4

##### Hate
- GANG1
- GANG2
- GANG3
- GANG5
- GANG6
- GANG7
- GANG8
- GANG9
- GANG10

##### Respect
- GANG4

#### GANG5

##### Hate
- GANG1
- GANG2
- GANG3
- GANG4
- GANG6
- GANG7
- GANG8
- GANG9
- GANG10

##### Respect
- GANG5

#### GANG6

##### Hate
- GANG1
- GANG2
- GANG3
- GANG4
- GANG5
- GANG7
- GANG8
- GANG9
- GANG10

##### Respect
- GANG6

#### GANG7

##### Hate
- GANG1
- GANG2
- GANG3
- GANG4
- GANG5
- GANG6
- GANG8
- GANG9
- GANG10

##### Respect
- GANG7

#### GANG8

##### Hate
- GANG1
- GANG2
- GANG3
- GANG4
- GANG5
- GANG6
- GANG7
- GANG9
- GANG10

##### Respect
- GANG8

#### GANG9

##### Hate
- GANG1
- GANG2
- GANG3
- GANG4
- GANG5
- GANG6
- GANG7
- GANG8
- GANG10

##### Respect
- GANG9

#### GANG10

##### Hate
- GANG1
- GANG2
- GANG3
- GANG4
- GANG5
- GANG6
- GANG7
- GANG8
- GANG9

##### Respect
- GANG10

#### MEDIC

##### Respect
- COP
- FIREMAN

#### FIREMAN

##### Respect
- COP
- MEDIC

#### CRIMINAL

##### Hate
- COP

#### PROSTITUTE

##### Hate
- COP

### Tier
verified

## Pedgrp dat

### Groups
57

### Max peds per group data
21

### Engine cap
21

### Header claims
32

### Array base va
0xc0f358

### Index type
uint16 model index

### Distinct models
185

### Peds ide models
276

### Models resolve
185/185

### Popcycle labels
```
POPCYCLE_GROUP_AIRCREW, POPCYCLE_GROUP_AIRCREW_RUNWAY, POPCYCLE_GROUP_BEACHFOLK, POPCYCLE_GROUP_BUSINESS, POPCYCLE_GROUP_CASUAL_AVERAGE, POPCYCLE_GROUP_CASUAL_POOR, POPCYCLE_GROUP_CASUAL_RICH, POPCYCLE_GROUP_CLUBBERS, POPCYCLE_GROUP_CRIMINALS, POPCYCLE_GROUP_DESERT_FOLK, POPCYCLE_GROUP_ENTERTAINERS, POPCYCLE_GROUP_FARMERS, POPCYCLE_GROUP_GOLFERS, POPCYCLE_GROUP_OUT_OF_TOWN_FACTORY_WORKERS, POPCYCLE_GROUP_PARKFOLK, POPCYCLE_GROUP_PROSTITUTES, POPCYCLE_GROUP_SERVANTS, POPCYCLE_GROUP_WORKERS
```

### Shared loader with
CARGRP.DAT

### Tier
verified
