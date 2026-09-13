# RE-Data — machine-readable tables

Every structured table produced by the SA Encyclopedia's reverse-engineering passes, as JSON. All
addresses are **GTA: San Andreas 1.0 US (SecuROM + HOODLUM patch)** virtual addresses
(`ImageBase 0x400000`). Files are UTF-8. Do not hand-edit files whose headers say
`"generated": "tools/..."` — regenerate from the binary.

> **Conventions.** Confidence is tagged ✅ verified (disassembly spot-check or structural fact
> confirmed) / 🔷 external-derived (name from disassembly/RH_ScopedInstall, not yet confirmed by
> our own disassembly) / ⏳ unidentified. Where a file carries a `note` or `_note` field, read that
> first.

---

## Binary identity & relocation map

| File | Rows | What it is |
|---|--:|---|
| [`hoodlum_relocation_map.json`](hoodlum_relocation_map.json) | 492 | **The HOODLUM relocation table** — every `.text` stub (`E9 rel32` at 16-byte-aligned addresses) that jumps into the `.HOODLUM` section, establishing the `entry_va → body_va` pair for all relocated functions. Produced by a linear scan of raw `.text` bytes; 492 stubs confirmed, 9 unaligned veneer jumps noted separately. Median relocated body is 64 bytes; largest is 3,392 bytes (`CStreaming::AddImageToList`). 1,609 total direct call sites traced. **Single source of truth for the relocation map — all other tools consume this.** |

The `.text` section starts at `0x00401000` (raw offset 0x400) and spans 4,546,048 bytes. The
`.HOODLUM` section lives at `0x01556000` (raw offset 0xD98000) and covers 135,168 bytes. Every
function entry in `.text` is a 5-byte far jump; the real code runs in `.HOODLUM`.

---

## Symbol & class naming (the function map)

| File | Rows | What it is |
|---|--:|---|
| [`function_catalogue.json`](function_catalogue.json) | 492 | **The consolidated function map** — one record per relocated function: `entry_va`, `body_va`, `body_extent`, `direct_call_sites`, `entry_bytes`, `name`, `source`, and `confidence`. 368/492 functions named (74.8%); 124 remain ⏳ unidentified. Sources are disassembly `RH_ScopedInstall` hooks and internal RE work. |
| [`class_catalogue.json`](class_catalogue.json) | 367 | **Per-class method tables** — 367 named methods grouped under 74 classes. Includes 16 structural facts byte-verified against disassembly (all 16 pass). Heaviest classes: `CTheScripts` (34 methods), `CStreaming` (32), `CPathFind` (27). |
| [`name_verification.json`](name_verification.json) | 119 | **Disassembly-promoted names** — 195 external-only names considered; 119 promoted to ✅ where the method's own disassembly references the class's signature statics (name and data-flow both corroborate). 13 were already verified by prior work, giving 132 total structure-or-disassembly-verified names across 13 classes. |
| [`attribution.json`](attribution.json) | 15 | **Class attributions for the 124 unidentified stubs** — 15 stubs reach a signature static from a known class; 11 are STRONG (cluster membership + hit count), 4 are data-only leads. Attributed classes: `CStreaming`, `CColStore`, `CEntryExitManager`, `CGarages`, `CPathFind`. |

**Key structural facts confirmed in `class_catalogue.json`:**

- `CRunningScript` objects are **224 bytes** (`0xE0`); pool base `0xA8B430`. `GetScriptIndexFromPointer` divides by 224 using the reciprocal `0x92492493`.
- `CTheScripts` mission local-var block: **4 KB** (0x400 dwords) at `0xA48960`. `OnAMission` flag sits immediately after at `0xA49960`.
- Script-blocked vehicle-model list: 20 dwords of `−1` at `0xA448F0`. Script-suppressed car-model list: 40 dwords of `−1` at `0xA44940`. `WaitingForScriptBrain` array: 150 entries × 8 bytes at `0xA476B0`.
- `CStreaming` info record: stride **20 bytes** (`0x14`), flags byte at `base+6` (`0x8E4CC6`), `loadState` at `record+0x10`; IMG archive list is 8 entries × 48 bytes at `0x8E48D8`. Render list is a doubly-linked list with head sentinel at `0x8E48A0`.
- Path node stride: **28 bytes** (`0x1C`); link-count packed in the low nibble of `node+0x18`; link table at `0x96FA94`.

---

## Script engine (SCM)

| File | Rows | What it is |
|---|--:|---|
| [`scm_exe_arity.json`](scm_exe_arity.json) | 1,924 | **Opcode arities extracted from `gta_sa.exe`** — `opcode → parameter count`, derived from `ProcessCommands` dispatch handlers by summing `CollectParameters` / `StoreParameters` push-counts plus variable and string-pointer reads. Self-check: 96.0% agreement against the 1,374 closure/forced arities; 188/214 scripts decode to exact end with zero overshoot. See C18.6. |
| [`scm_opcode_arity.json`](scm_opcode_arity.json) | 1,474 | **Opcode arity oracle** — 1,449 validated entries (opcode→arity confirmed) and 25 unvalidated entries held separately. |
| [`runningscript_layout.json`](runningscript_layout.json) | — | **`CRunningScript` object layout** (224 bytes, 7 structural facts, all verified): `next/prev` intrusive list links at `+0/+4`; 8-char name at `+8`; code base pointer at `+0x10`; program counter at `+0x14`; 8-dword gosub/return stack at `+0x18`; 34 local-variable dwords (32 locals + 2 timers) at `+0x3C`; flag block at `+0xC4`; mission-LVAR-mode flag at `+0xDC`. The `+0xDC` flag redirects local-variable access to the shared 4 KB mission-LVAR block at `0xA48960`; global-variable space begins at `0xA49960`. |
| [`scripts_for_brains.json`](scripts_for_brains.json) | — | **`CScriptsForBrains` slot table** — 70 records × 20 bytes; free slot sentinel is `0xFFFF` in the id word. Per-record layout: model-id word at `+0`, two flag bytes at `+2/+3`, active byte at `+4` (Init sets to 1), then dword + two words + dword. 3 structural facts verified. |
| [`streaming_stores.json`](streaming_stores.json) | — | **Streaming store layouts** — `CColStore` slot is 44 bytes, pool object at `0x965560`, status byte at `slot+0x28`. `CIplStore` slot is 52 bytes, pool object at `0x8E3FB0`. `CStreamedScripts` table: 82 slots × 32 bytes, count word at `+2628`, flag byte at `slot+4`. 9 structural facts verified. |

**`ProcessCommands` helper functions (arity counting anchors):**

| VA | Role |
|---|---|
| `0x464080` | `CollectParameters(N)` — consumes N parameters |
| `0x464370` | `StoreParameters(N)` — stores N parameters |
| `0x464790` | `GetPointerToScriptVariable` — +1 parameter |
| `0x463D50` | `ReadStringFromScript` — +1 parameter |
| `0x464250` | `CollectNextParameterWithoutIncreasingPC` (peek, +0) |

---

## Gameplay systems

| File | Rows | What it is |
|---|--:|---|
| [`gameplay_managers.json`](gameplay_managers.json) | — | **Three manager struct layouts** — `CEntryExitManager`: `CEntryExit` record is 60 bytes, pool object at `0x96A7D8` (array @ `pool+0`, flag-bytes @ `pool+4`, count @ `pool+8`), visible-objects array is 32 slots at `0x96A738`. `CReplay`: 8 blocks × 100,000 bytes at `0x97FB88`–`0xA43088`; ped-conv table 140 entries × 1,988-byte packet stride at `0x97F630`. `CShopping`: 560 items, shop record stride 24 bytes, bought-id array at `0xA98F50`, has-bought flags at `0xA98290`, clothes buffer (30 dwords) at `0xA9B990`. 10 structural facts verified. |
| [`gameplay_pools.json`](gameplay_pools.json) | — | **`CPickups` and `CGarages` pool layouts** — Pickup pool: 620 slots × 32 bytes at `0x9788C4`; in-use byte at `slot+0x18`; handle packs `{counter<<16 \| index}` with counter word at `slot+0x16`; collected-pickups ring is 20 dwords at `0x978628` with write-cursor at `0x978624`. Garage pool: 50 slots × 216 bytes at `0x96C048`; type byte at `garage+0x4C`, door-state at `+0x4D`, flags at `+0x4E`; count global at `0x96C020`. 10 structural facts verified. |
| [`conversations.json`](conversations.json) | — | **`CConversations` three-array layout** — per-ped state: 14 slots × 28 bytes at `0x9691D8`; conversation-node table: 12 slots × 44 bytes at `0x969360`; dialogue-line array: 50 slots × 24 bytes at `0x969570`. Build index global at `0x9691C8`; active flag at `0x9691D0`. Text-key copies are 6 characters. 4 structural facts verified. |
| [`vehicle_recording.json`](vehicle_recording.json) | — | **`CVehicleRecording` table layout** — 475 recording slots × 16 bytes at `0x97D880`; fields: recording-id at `+0`, loaded-data pointer at `+4`, status byte at `+0xC`; live count global at `0x97F630`. Playback state: 16 flag bytes at `0x97D6F0`, car-pointer array (dword-indexed) at `0x97D840`. 4 structural facts verified. |

---

## World data

| File | Rows | What it is |
|---|--:|---|
| [`zone_structure.json`](zone_structure.json) | 384 | **Zone record layout and census** — zone record has 10 fields; parsed via `sscanf` with format `%s %d %f %f %f %f %f %f %d %s` at `0x868D04`. `INFO.ZON`: 378 records, all type 0 (navigation zones), all island 1, 377 distinct names, 169 distinct GXT text keys. `MAP.ZON`: 6 records (type 3 area markers). Radar grid: 144 tiles, 12×12, 500 units/tile, named `radar%02d.txd` in `models/gta3.img`. Gridref: 10×10, 600 units/cell, 32-byte records, path string at `0x87295C`. 20 checks, all pass. |
| [`object_structure.json`](object_structure.json) | 993 | **`object.dat` record layout** — two variants distinguished by field count from `sscanf` (not by a flag field): basic (17 fields, 758 records) and breakable (24 fields, 235 records). Format string at `0x868DC8`. Notable exceptions: 514 objects exceed the documented 50,000-kg mass cap, 1 has special-CDR out of range, 1 reports >120% submerged. |
| [`surface_structure.json`](surface_structure.json) | 179 | **Surface type tables** — 179 surface types defined identically across `surfinfo.dat`, `surfaud.dat`, and the exe string table. `surfinfo.dat`: 37 columns per record. `surfaud.dat`: 10 columns, boolean flags, verified. `surface.dat`: 6×6 lower-triangular adhesion/friction matrix (21 values). 17 procedural-object surface references and 42 plant surface references are a subset of the 179. |
| [`ped_tables_structure.json`](ped_tables_structure.json) | — | **`ped.dat` and `pedgrp.dat` layout** — `ped.dat`: 17 ped-type codes; verbs used are a subset of the 4 documented verbs; one type is referenced in the exe but not defined in the file. `pedgrp.dat`: 57 groups, max 21 peds per group (matching the engine cap); header claims 32 but the data uses 21; ped-group array base `0xC0F358`. |
| [`gxt_structure.json`](gxt_structure.json) | 16,588 | **GXT container format** — 4-byte header (`version_word = 4`, `bits_per_char = 8`). Structure: `TABL` block (12-byte entries: `name[8]` + `uint32 offset`) → per-table block (`name[8]` if not MAIN, then `TKEY` + `TDAT`). `TKEY` entries sorted ascending by hash for binary search; hash is CRC-32 (reflected, poly `0xEDB88320`, init `0xFFFFFFFF`, **without** final complement) of the uppercased key name — confirmed 1,180/1,186 (99.5%) of SCM text-opcode GXT keys. `american.gxt`: 127 tables, 16,588 keys total (5,428 in MAIN). |
| [`weapon_structure.json`](weapon_structure.json) | 53 | **`weapon.dat` record layout** — 53 active rows (5 disabled: `COUNTRYRIFLE`, `SNIPERRIFLE`, `JETPACK`, `SKATEBOARD`, one more). Two row widths: 25-field (43 records) and 29-field (10 records, adding `ANIM2`, `RANGE2`, `SPEED2`, `ADDFX`). 10 triple-tier weapons (standard/pro/gangster stat tiers), 19 single-tier. The pistol has an undocumented fourth row (`P=3`, cop model, reqStat=5000). Exe weapontype table cross-checked: 49 entries, all 9 checks pass. |
| [`fonts_structure.json`](fonts_structure.json) | — | **Font system layout** — `fonts.dat`: 2 fonts, each with 208 proportional-width entries. Exe parser at `0x7187C0`; width-lookup at `0x7196F4`; char-remap at `0x718770`/`0x7192C0`; UV draw at `0x718B30`. Atlas: 16×16 cell grid of 32×32 px glyphs; glyph index maps directly to atlas cell. `fonts.txd`: 2 textures (`font1`/`font2`) at 512×512, 16-bit depth. `hud.txd`: 69 textures. Mask variants `font1m`/`font2m` are requested by the exe but absent from the TXD. 21 checks, all pass. |

---

## Audio & effects

| File | Rows | What it is |
|---|--:|---|
| [`audio_structure.json`](audio_structure.json) | — | **Audio subsystem layout** — `PakFiles.dat`: 9 records × 52 bytes each (`char name[12]` NUL-terminated and `0xCD`-padded, plus 40 zero bytes). SFX bank header: 4,804 bytes; max 400 sounds per bank; `SoundEntry` layout derived from exe divisors. Stream pack XOR key: `ea3ac4a19aa814f348b0d7239de8fff1` (16-byte period, keyed by `absoluteFileOffset % 16`). `BankLkup` stride 12 confirmed by the `0xAAAAAAAB` multiply-shift-3 reciprocal at `0x4DFBD7`. |
| [`fxp_structure.json`](fxp_structure.json) | 82 | **`effects.fxp` particle format** — plain text, CRLF, 616,708 bytes, 43,617 lines. Grammar: `FX_PROJECT_DATA:` block containing `FX_SYSTEM_DATA:` entries, each with a version word (constant `109`), `FILENAME`, `NAME`, and emitter records. 82 particle systems, 161 emitters, 1,470 info blocks, 4,120 curves, 6,563 keyframes. |
| [`eventvol_events.json`](eventvol_events.json) | 597 | **EventVol audio-volume event table** — 45,401-byte file at base pointer `0xBD00F8`; sentinel `0x80` (−128); 44,804 default entries, 597 configured entries. 89 exe references. 44 events are hardcoded in the exe with fixed volume deltas (all 44 also appear as configured entries). Event IDs are bare integer immediates — no id-to-name table exists in the binary; names are not recoverable from the exe. |

---

## Regeneration pipeline

Files marked `"generated": "tools/<script>.py"` are derived outputs. After any binary or source-data
change, re-run in dependency order:

```
tools/derive_attribution.py
tools/derive_class_catalogue.py
tools/derive_conversations.py
tools/derive_eventvol.py
tools/derive_function_catalogue.py
tools/derive_gameplay_managers.py
tools/derive_gameplay_pools.py
tools/derive_name_verification.py
tools/derive_runningscript.py
tools/derive_scriptsforbrains.py
tools/derive_streaming_stores.py
tools/derive_vehiclerecording.py
```

Files without a `generated` header (`audio_structure.json`, `fxp_structure.json`,
`gxt_structure.json`, `hoodlum_relocation_map.json`, `object_structure.json`,
`ped_tables_structure.json`, `scm_exe_arity.json`, `scm_opcode_arity.json`,
`surface_structure.json`, `weapon_structure.json`, `zone_structure.json`,
`eventvol_events.json`, `fonts_structure.json`) are either hand-authored from analysis
or produced by tools not listed in the manifest above — do not overwrite them without
cross-checking the source analysis.
