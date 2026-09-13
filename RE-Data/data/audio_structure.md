# Audio Subsystem Layout
*Source: `audio_structure.json`*

## Chapter
C20

## Subject
GTA:SA audio subsystem

## Target
gta_sa.exe 1.0 US, md5 170b3a9108687b26da2d8901c6948a18

## Files

### PakFiles.dat

#### Bytes
468

#### Records
9

#### Width
52

#### Layout
char name[12] (NUL-terminated, 0xCD-padded) + 40 zero bytes

#### Tier
verified

#### Names
- FEET
- GENRL
- PAIN_A
- SCRIPT
- SPC_EA
- SPC_FA
- SPC_GA
- SPC_NA
- SPC_PA

### StrmPaks.dat

#### Bytes
272

#### Records
17

#### Width
16

#### Layout
char name[16], NUL-terminated, zero-padded

#### Tier
verified

#### Names
```
AA, ADVERTS, , AMBIENCE, BEATS, CH, CO, CR, CUTSCENE, DS, HC, MH, MR, NJ, RE, RG, TK
```

#### Note
index 2 is blank and is referenced by no track

### BankLkup.dat
| Key | FEET | GENRL | PAIN A | SCRIPT | SPC EA | SPC FA | SPC GA | SPC NA | SPC PA |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Bytes | 8520 |  |  |  |  |  |  |  |  |
| Records | 710 |  |  |  |  |  |  |  |  |
| Width | 12 |  |  |  |  |  |  |  |  |
| Layout | uint8 packIndex; uint8 pad[3] = 0xCC; uint32 offset; uint32 size |  |  |  |  |  |  |  |  |
| Semantics | offset = start of the 4804-byte bank header; size = PCM bytes after it |  |  |  |  |  |  |  |  |
| Tier | verified |  |  |  |  |  |  |  |  |
| Banks per pack | 7 | 137 | 3 | 218 | 46 | 18 | 209 | 52 | 20 |

### TrakLkup.dat
| Key | Size==oggLength | Size==oggLength+8068 |
| --- | --- | --- |
| Bytes | 23064 |  |
| Records | 1922 |  |
| Width | 12 |  |
| Layout | uint8 packIndex; uint8 pad[3] = 0xCD; uint32 offset; uint32 size |  |
| Semantics | offset = start of the 8068-byte track header; size field is inconsistent |  |
| Size convention | 804 | 1118 |
| Tier | verified |  |

### BankSlot.dat
| Field | Value |
| --- | --- |
| Bytes | 216902 |
| Header | uint16 numSlots = 45 |
| Records | 45 |
| Width | 4820 |
| Layout | uint32 bufferOffset; uint32 bufferSize; int32 = -1; int32 = -1; SoundEntry entries[400]; uint32 pad |
| Tier | verified (arithmetic, width, entry layout); open (field meanings) |

### EventVol.dat

#### Bytes
45402

#### Bytes read by game
45401

#### Layout
int8 volume[45401], indexed directly by audio event ID

#### Element
signed byte (movsx), no index scaling in any of 56 accesses

#### Sentinel
| Field | Value |
| --- | --- |
| Value | -128 |
| Raw | 0x80 |
| Meaning | no value set for this event |
| Count | 44804 |

#### Events with a value
597

#### Value range
- -28
- 20

#### Base pointer va
0xbd00f8

#### Exe literal event ids
```
52, 53, 76, 77, 78, 90, 91, 92, 93, 99, 107, 108, 109, 110, 111, 115, 116, 117, 119, 125, 128, 141, 144, 145, 151, 153, 158, 164, 1113
```

#### Trailing unread byte
present, and is itself 0x80

#### Tier
verified (element type, flat indexing, extent, sentinel); reasoned (values are decibel trims); open (the audio-event ID namespace itself)

## Sfx bank

### Header bytes
4804

### Max sounds
400

### Layout
uint32 numSounds; SoundEntry entries[400]

### Sound entry

#### Width
12

#### Fields
- uint32 dataOffset (relative to the end of the header)
- int32 loopOffset (-1 = no loop)
- uint16 sampleRate
- int16 trim (authored, not measured; meaning open)

### Banks
710

### Sounds
61993

### Looping sounds
351

### Sample rates
| Field | Value |
| --- | --- |
| 12000 | 54273 |
| 15000 | 5908 |
| 8000 | 1082 |
| 18000 | 286 |
| 22050 | 89 |
| 16000 | 36 |
| 26000 | 30 |
| 24000 | 24 |
| 44100 | 22 |
| 11025 | 18 |
| 23000 | 17 |
| 20000 | 16 |
| 32000 | 14 |
| 22100 | 12 |
| 10000 | 12 |
| 28000 | 10 |
| 21000 | 8 |
| 17600 | 8 |
| 22000 | 7 |
| 27000 | 6 |
| 17000 | 5 |
| 19000 | 4 |
| 26600 | 3 |
| 17974 | 3 |
| 16500 | 3 |
| 18800 | 3 |
| 14364 | 3 |
| 15010 | 2 |
| 16950 | 2 |
| 21600 | 2 |
| 17924 | 2 |
| 14000 | 2 |
| 24500 | 2 |
| 25400 | 2 |
| 23600 | 2 |
| 11048 | 2 |
| 18021 | 2 |
| 17968 | 2 |
| 25600 | 2 |
| 22400 | 2 |
| 8500 | 2 |
| 9000 | 2 |
| 28300 | 1 |
| 16946 | 1 |
| 20200 | 1 |
| 11000 | 1 |
| 16800 | 1 |
| 16700 | 1 |
| 13600 | 1 |
| 13000 | 1 |
| 21561 | 1 |
| 18900 | 1 |
| 11007 | 1 |
| 12500 | 1 |
| 34000 | 1 |
| 17825 | 1 |
| 10162 | 1 |
| 16499 | 1 |
| 21746 | 1 |
| 15375 | 1 |
| 26513 | 1 |
| 11615 | 1 |
| 11473 | 1 |
| 29711 | 1 |
| 11993 | 1 |
| 10653 | 1 |
| 9400 | 1 |
| 18017 | 1 |
| 22044 | 1 |
| 21950 | 1 |
| 17958 | 1 |
| 11983 | 1 |
| 22114 | 1 |
| 11984 | 1 |
| 14474 | 1 |
| 14489 | 1 |
| 14986 | 1 |
| 18007 | 1 |
| 12010 | 1 |
| 12918 | 1 |
| 12811 | 1 |
| 2021 | 1 |
| 25000 | 1 |
| 20900 | 1 |
| 12800 | 1 |
| 18569 | 1 |
| 19200 | 1 |
| 26400 | 1 |
| 24600 | 1 |
| 19700 | 1 |
| 16600 | 1 |
| 17800 | 1 |
| 11963 | 1 |
| 15571 | 1 |
| 11987 | 1 |
| 17143 | 1 |
| 22038 | 1 |
| 13548 | 1 |
| 11517 | 1 |
| 13462 | 1 |
| 18700 | 1 |
| 19993 | 1 |
| 23300 | 1 |

### Trim range
- -200
- 6229

### Exceptions
| Bank index | Pack | Sound | Rate | Loop | Trim |
| --- | --- | --- | --- | --- | --- |
| 138 | GENRL | 27 | 2021 | 0 | 400 |

### Pcm
16-bit signed mono, little-endian

### Tier
verified (container, entry layout, tiling, PCM width); open (trim field)

### Peak survey
| Field | Value |
| --- | --- |
| Sampled | 120 |
| Within 2dBFS | 104 |

## Stream pack

### Xor key
ea3ac4a19aa814f348b0d7239de8fff1

### Xor period
16

### Xor indexing
key[absoluteFileOffset % 16]

### Header bytes
8068

### Max beats
1000

### Layout
Beat beats[1000] {int32 timeMs; int32 type}, unused = {-1,0}; 68-byte trailer

### Trailer
| Field | Value |
| --- | --- |
| OggLength at | 8000 |
| RateField at | 8004 |
| Displaced by 8 on beat annotated tracks | ✅ true |
| Constant uint16 at 8064 | 1 |
| Fill | 0xCD |

### Tracks
1922

### Packs
16

### Codec
Ogg Vorbis, 2 channels

### Vorbis sample rates
| Field | Value |
| --- | --- |
| 32000 | 1882 |
| 24000 | 40 |

### Rate field values
| Field | Value |
| --- | --- |
| 48000 | 1744 |
| 0 | 137 |
| 24000 | 40 |
| 25137 | 1 |

### Rate field note
does NOT equal the Vorbis sample rate except on AMBIENCE; meaning open

### Beat tracks
| Track | Pack | Offset | Beats | Trailer at |
| --- | --- | --- | --- | --- |
| 175 | BEATS | 0 | 174 | 8008 |
| 177 | BEATS | 9979717 | 152 | 8008 |
| 178 | BEATS | 14115027 | 144 | 8008 |
| 179 | BEATS | 18033228 | 174 | 8008 |
| 180 | BEATS | 21743595 | 76 | 8008 |
| 181 | BEATS | 25474541 | 136 | 8008 |

### Tier
verified (key, container, tiling, codec); open (trailer rate field)

## Executable confirmation
| Key | Ref va | Divisor | Evidence | Record | Read bytes | BANKLKUP | PAKFILES | BANKSLOT | STRMPAKS | TRAKLKUP | EVENTVOL |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| BankLkup | 0x4dfbd7 | 12 | mov eax,0xAAAAAAAB; mul esi; shr edx,3; lea eax,[eax+eax*2]; shl eax,2 |  |  |  |  |  |  |  |  |
| PakFiles | 0x4dfc7d | 52 | mov eax,0x4EC4EC4F; mul esi; shr edx,4; imul eax,eax,0x34 |  |  |  |  |  |  |  |  |
| BankSlot | 0x4e0597 |  | imul eax,eax,0x12D4 after reading the 2-byte count | 4820 |  |  |  |  |  |  |  |
| StrmPaks | 0x4e0982 | 16 | shr eax,4 |  |  |  |  |  |  |  |  |
| TrakLkup | 0x4e0a02 | 12 | mov eax,0xAAAAAAAB; mul edi; shr edx,3 |  |  |  |  |  |  |  |  |
| EventVol | 0x5b9d68 |  | push 0xB159; cmp eax,0xB159 |  | 45401 |  |  |  |  |  |  |
| String vas |  |  |  |  |  | 0x85f118 | 0x85f140 | 0x85f15c | 0x85f184 | 0x85f1a0 | 0x86a440 |

## Checks
| Name | Pass | Detail |
| --- | --- | --- |
| PakFiles.dat = 9 x 52, no residue | ✅ true |  |
| PakFiles names == SFX filenames | ✅ true | ['FEET', 'GENRL', 'PAIN_A', 'SCRIPT', 'SPC_EA', 'SPC_FA', 'SPC_GA', 'SPC_NA', 'SPC_PA'] |
| StrmPaks.dat = 17 x 16, no residue | ✅ true |  |
| StrmPaks slot 2 blank; other 16 == streams filenames | ✅ true |  |
| BankLkup.dat divides by 12 with no residue | ✅ true | 710 banks |
| BankLkup pad bytes are all 0xCC | ✅ true |  |
| every SFX pack tiles exactly as sum(4804 + size) | ✅ true | {'FEET': 7, 'GENRL': 137, 'PAIN_A': 3, 'SCRIPT': 218, 'SPC_EA': 46, 'SPC_FA': 18, 'SPC_GA': 209, 'SPC_NA': 52, 'SPC_PA': 20} |
| all banks: sound offsets ascending and unique | ✅ true | 710/710 |
| all banks: first offset 0, last offset < size | ✅ true | 710/710 |
| all banks: unused entries are all-zero | ✅ true | 710/710 |
| all sounds: data length even (16-bit samples) | ✅ true | 61993/61993 |
| all sounds: loop offset is -1 or inside the sound | ✅ true | 61993/61993 |
| TrakLkup.dat divides by 12 with no residue | ✅ true | 1922 tracks |
| TrakLkup pad bytes are all 0xCD | ✅ true |  |
| TrakLkup never references pack index 2 (the blank StrmPaks slot) | ✅ true |  |
| XOR key decrypts to OggS at offset+8068 on every track | ✅ true | 1922/1922 |
| every stream pack tiles exactly as sum(8068 + oggLength) | ✅ true | 16 packs |
| all tracks are 2-channel Vorbis | ✅ true | {32000: 1882, 24000: 40} |
| exactly 6 tracks carry beat data, all in BEATS | ✅ true |  |
| TrakLkup size field is inconsistent across tracks | ✅ true | {'size==oggLength': 804, 'size==oggLength+8068': 1118} |
| sampled sounds peak at -2.00 dBFS | ✅ true | 104/120 = 86.7% |
| BankSlot.dat = 2 + 45 x 4820 | ✅ true | 45 slots |
| BankSlot words 2 and 3 are -1 in every slot | ✅ true |  |
| EventVol.dat is 45,402 bytes; executable reads 45,401 (0xB159) | ✅ true |  |
| EventVol: the one unread trailing byte is itself the 0x80 default | ✅ true |  |
| EventVol: 597 of 45,401 events carry a value | ✅ true | 1.31% of entries |
| EventVol: every event ID hard-coded in gta_sa.exe has a value set | ✅ true | 29/29, against a 1.31% base rate |
| EventVol: values are small signed integers (a trim, not a raw level) | ✅ true | range -28..20 |
