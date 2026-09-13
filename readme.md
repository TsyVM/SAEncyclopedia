<div align="center">

<img src="sae-logo.png" width="600" alt="SAEncyclopedia"/>

<p><em>Complete reverse-engineering reference for Grand Theft Auto: San Andreas (sa10us)</em></p>

<a href="#">
<img src="https://readme-typing-svg.demolab.com/?lines=55+chapters.+297+schemas.+0+external+fields.;Every+claim+traced+to+gta_sa.exe+or+marked+open.;ms_fTimeStep+at+0xB7CB5C+%E2%80%94+984+refs+%E2%80%94+highest+in+the+binary.;CHandlingData+true+size+0xE0+%E2%80%94+community+had+0xD4.;Streaming+%C2%B7+Physics+%C2%B7+Rendering+%C2%B7+AI+%C2%B7+Input+%C2%B7+Audio.;72%2C719+call+sites+analyzed.+Four+confidence+tiers.;The+deepest+open-source+reference+to+GTA%3ASA+ever+written.&font=Fira%20Code&center=true&width=750&height=45&color=E07B00&vCenter=true&size=20&pause=1800"/>
</a>

<br/>

[![Chapters](https://img.shields.io/badge/Chapters-55-E07B00?style=for-the-badge&labelColor=000000)](#-chapter-map)
[![Schemas](https://img.shields.io/badge/Schemas-297%20verified-E07B00?style=for-the-badge&labelColor=000000)](#️-struct-database)
[![Platform](https://img.shields.io/badge/Platform-Windows%20x86-E07B00?style=for-the-badge&labelColor=000000&logo=windows&logoColor=E07B00)](#)
[![Target](https://img.shields.io/badge/Target-sa10us%201.0%20US-E07B00?style=for-the-badge&labelColor=000000)](#)
[![TeamVanilla](https://img.shields.io/badge/Team-TeamVanilla-E07B00?style=for-the-badge&labelColor=000000)](https://www.teamvanilla.org/)

<br/>

[![Stars](https://img.shields.io/github/stars/tsyvm/saencyclopedia?style=for-the-badge&color=E07B00&labelColor=000000)](../../stargazers)
[![Issues](https://img.shields.io/github/issues/tsyvm/saencyclopedia?style=for-the-badge&color=E07B00&labelColor=000000)](../../issues)
[![Last Commit](https://img.shields.io/github/last-commit/tsyvm/saencyclopedia?style=for-the-badge&color=E07B00&labelColor=000000)](../../commits)

<br/>

[![Fields](https://img.shields.io/badge/Fields-2%2C003%20by%20disassembly-E07B00?style=flat-square&labelColor=000000)](#)
[![Call Sites](https://img.shields.io/badge/Call%20Sites-72%2C719%20analyzed-E07B00?style=flat-square&labelColor=000000)](#)
[![DB Lines](https://img.shields.io/badge/DB%20Lines-6%2C541-E07B00?style=flat-square&labelColor=000000)](#)
[![Data Files](https://img.shields.io/badge/data%2F-fully%20swept-E07B00?style=flat-square&labelColor=000000)](#)
[![Tiers](https://img.shields.io/badge/Tiers-%E2%9C%85%20%F0%9F%9F%A1%20%F0%9F%94%B7%20%E2%8F%B3-E07B00?style=flat-square&labelColor=000000)](#-confidence-tiers)
[![Pools](https://img.shields.io/badge/Pools-13%20mapped-E07B00?style=flat-square&labelColor=000000)](#)
[![No fabrication](https://img.shields.io/badge/No-fabricated%20data-E07B00?style=flat-square&labelColor=000000)](#)
[![SASDK](https://img.shields.io/badge/Companion-SASDK-E07B00?style=flat-square&labelColor=000000)](../SASDK/README.md)

</div>

<img width="100%" src="https://capsule-render.vercel.app/api?type=rect&color=0:000000,50:E07B00,100:000000&height=3"/>

The **SAEncyclopedia** is a 55-chapter reverse-engineering reference for Grand Theft Auto: San Andreas — specifically the PC release, sa10us, HOODLUM 1.0 US (`gta_sa.exe` MD5 `170b3a9108687b26da2d8901c6948a18`). Every claim is grounded in the binary: either confirmed by a Capstone disassembly sweep and marked ✅, structurally inferred and marked 🟡, externally sourced and marked 🔷, or explicitly left open with ⏳. No field is asserted without a tier; no address is cited without a call-site derivation.

The encyclopedia covers the complete `data/` folder (every file has a chapter or a named home), all 13 entity pools, the full render and physics pipelines, the SCM script engine, input, audio, AI, streaming, and the RenderWare layer beneath everything. Its companion SDK — [SASDK](../SASDK/README.md) — exposes the same verified struct database as production C++20 headers with `static_assert`-checked sizes.

<div align="center">

### 📑 Contents

[Confidence Tiers](#-confidence-tiers) · [Novel Findings](#-novel-findings) · [Chapter Map](#-chapter-map) · [Struct Database](#️-struct-database)

[Key Addresses](#-key-addresses) · [Data-File Coverage](#-data-file-coverage) · [Modding Reference](#-modding-reference) · [Source Layout](#-source-layout)

</div>

<img width="100%" src="https://capsule-render.vercel.app/api?type=rect&color=0:000000,50:E07B00,100:000000&height=3"/>

## 🎯 Confidence Tiers

Every claim in every chapter carries one of four markers — never omitted, never upgraded without evidence.

| Tier | Marker | Meaning |
|---|:---:|---|
| **Verified by disassembly** | ✅ | Confirmed by Capstone sweep; VA cited; field derivation documented in `RE-Data/` |
| **Reasoned** | 🟡 | Structurally inferred from surrounding verified context; derivation explained inline |
| **External-derived** | 🔷 | Sourced from community references (gta-reversed, plugin-sdk); not yet checked against binary |
| **Open item** | ⏳ | Known gap; the question is stated; assumption-free until traced |

A ⏳ marker is a commitment, not a weakness — it means the chapter knows what it doesn't know, which is more useful than a confident wrong answer.

<img width="100%" src="https://capsule-render.vercel.app/api?type=rect&color=0:000000,50:E07B00,100:000000&height=3"/>

## 🔬 Novel Findings

Claims not previously documented, derived by disassembly sweep of the sa10us executable:

| Finding | Value | Evidence |
|---|---|---|
| `ms_fTimeStep` VA | `0xB7CB5C` | Highest-reference-count variable in the binary — **984 `.text` references** |
| `ms_fTimeStep` unit | 50 ms per `1.0` | PS2 target rate; `fcomp dword ptr [0x858C14]` proves 2.0 clamp |
| `CHandlingData` true size | **`0xE0` (224 B)** | Community had `0xD4` (212 B); proven by `imul eax, eax, 0E0h` @`0x6F0151` |
| `ms_vehicleHandling` VA | `0xC2B9DC` | Index array; sub-handling at `0xC3BB00` (stride `0x94`) |
| `CPad` stride | `0x134` | Proven cold: `imul eax, eax, 134h` in `CPad::GetPad` @`0x53FB70` |
| `Pads[2]` VA | `0xB73458` | Two-player pad array; `Pads[1]` @`0xB7358C` |
| `CPools::Initialise` VA | `0x5503A0` | 13 `push-imm32` pool counts — the pool-limit patch site |
| `CActiveExplosion` pool | 64 @`0xC8AC80` | 21 explosion types — proven by `cmp eax, 0x14` @`0x73702C` |
| `polydensity.dat` | **dead asset** | Zero references to `"polydensity"` in retail binary — tool-pipeline leftover, never loaded |
| `CGame::Initialise` | `0x53BC80`, **142 callees** | Largest init hub in the binary — complete subsystem dependency graph |
| `bInvertMouseX/Y` | `0xBA6744` / `0xBA6745` | 3 / 8 `.text` references — per-axis invert flags, proven cold |
| `m_fMouseAccelHorzntl` | `0xB6EC1C` (19 refs) | Sensitivity lives on the **camera**, not the input layer |
| `eWeaponType` count | **59 entries** (0–58) | Gun subset (22–38) exe-anchored via WEAPONTYPE name table; env band 48–58 confirmed |

<img width="100%" src="https://capsule-render.vercel.app/api?type=rect&color=0:000000,50:E07B00,100:000000&height=3"/>

## 🗺️ Chapter Map

55 chapters organized by subsystem. Each chapter links to its hub page and 3–7 deep-dive subpages.

### Executable & RE Method

| Chapter | Title | Pages |
|---|---|:---:|
| [C0](C0-Binary-Identity/C0-Binary-Identity.md) | Binary Identity — what `gta_sa.exe` actually is | 4 |
| [X1](X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md) | SDK Cross-Reference — closing the open items | 1 |
| [C27](C27-Function-Catalogue/C27-Function-Catalogue.md) | The Function Catalogue — naming the 492 | 4 |
| [C28](C28-Class-Catalogue/C28-Class-Catalogue.md) | The Class Catalogue — from named functions to subsystem structure | 5 |
| [C33](C33-Attributing-The-Unnamed/C33-Attributing-The-Unnamed.md) | Attributing the Unnamed by Data Reference | 3 |
| [C37](C37-Verifying-The-Catalogue/C37-Verifying-The-Catalogue.md) | Verifying the Catalogue by Structure | 3 |

### Executable Lifecycle

| Chapter | Title | Pages |
|---|---|:---:|
| [C51](C51-Executable-Lifecycle/C51-Executable-Lifecycle.md) | CRT `0x824570` → WinMain → RenderWare → `CGame::Initialise` → loop → `CGame::Shutdown` | 4 |
| [C52](C52-CTimer-And-Game-Loop/C52-CTimer-And-Game-Loop.md) | `CTimer` — `ms_fTimeStep` at `0xB7CB5C`, 984 refs, clamp, sub-step layers | 5 |

### Streaming & Memory

| Chapter | Title | Pages |
|---|---|:---:|
| [C1](C1-Streaming/C1-Streaming.md) | Streaming — the IMG model and the CdStream layer | 3 |
| [C2](C2-CStreaming/C2-CStreaming.md) | `CStreaming` and the model-ID space | 5 |
| [C3](C3-Model-Stores/C3-Model-Stores.md) | The model stores and the ID partition | 5 |
| [C4](C4-Entities-And-Pools/C4-Entities-And-Pools.md) | Entities and the pool allocator | 6 |
| [C31](C31-Streaming-Slot-Tables/C31-Streaming-Slot-Tables.md) | Streaming slot tables — collision, IPL and streamed scripts | 4 |
| [C53](C53-Memory-And-Pool-Architecture/C53-Memory-And-Pool-Architecture.md) | Memory and pool architecture — all 13 pools, `CPools::Initialise` @`0x5503A0` | 4 |
| [C54](C54-Limits-Reference/C54-Limits-Reference.md) | Limits reference — pool caps, density caps, patch VAs | 5 |

### World & Map

| Chapter | Title | Pages |
|---|---|:---:|
| [C5](C5-CWorld/C5-CWorld.md) | `CWorld` and spatial partitioning | 7 |
| [C11](C11-IDE-And-IPL/C11-IDE-And-IPL.md) | IDE & IPL — the world's data model | 7 |
| [C22](C22-Map-Zones/C22-Map-Zones.md) | Map zones | 4 |
| [C12](C12-Path-Network/C12-Path-Network.md) | Path network | 3 |

### Asset Formats

| Chapter | Title | Pages |
|---|---|:---:|
| [C6](C6-Collision/C6-Collision.md) | Collision and the COL model | 7 |
| [C7](C7-RenderWare-Stream/C7-RenderWare-Stream.md) | RenderWare chunk stream (DFF / TXD / IFP) | 5 |
| [C8](C8-Geometry/C8-Geometry.md) | Geometry structure | 4 |
| [C9](C9-Materials-And-Textures/C9-Materials-And-Textures.md) | Materials and textures | 5 |
| [C17](C17-IFP-Animation/C17-IFP-Animation.md) | IFP animation | 4 |
| [C10](C10-2dEffect/C10-2dEffect.md) | 2dFX — lights, particles, corona, shadow | 3 |

### Vehicle Systems

| Chapter | Title | Pages |
|---|---|:---:|
| [C13](C13-Vehicle-Data/C13-Vehicle-Data.md) | Vehicle data — `handling.cfg` and model info | 5 |
| [C34](C34-Vehicle-Recording/C34-Vehicle-Recording.md) | Vehicle recording — replay system | 3 |
| [C42](C42-Vehicle-Physics/C42-Vehicle-Physics.md) | Vehicle physics — Euler integration, `ms_fTimeStep`, spring stability | 4 |
| [C47](C47-Vehicle-Dynamics/C47-Vehicle-Dynamics.md) | Vehicle dynamics — suspension, traction, damping ratio | 4 |

### Peds & AI

| Chapter | Title | Pages |
|---|---|:---:|
| [C14](C14-Peds-And-Weapons/C14-Peds-And-Weapons.md) | Peds, weapons & stats | 5 |
| [C26](C26-Ped-Tables/C26-Ped-Tables.md) | Ped tables — `pedstats.dat`, group behaviour | 4 |
| [C41](C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md) | Ped AI — task tree, event queue, wanted level | 4 |
| [C55](C55-CJ-Customisation/C55-CJ-Customisation.md) | CJ customisation — clothing, hair, tattoos, fat/muscle | 3 |

### Scripting

| Chapter | Title | Pages |
|---|---|:---:|
| [C18](C18-SCM-Script/C18-SCM-Script.md) | SCM — the mission script format | 6 |
| [C32](C32-CRunningScript-Object/C32-CRunningScript-Object.md) | `CRunningScript` — the script execution object | 4 |
| [C36](C36-Script-Brains/C36-Script-Brains.md) | Script brains | 3 |

### Gameplay

| Chapter | Title | Pages |
|---|---|:---:|
| [C25](C25-Object-Physics/C25-Object-Physics.md) | Object physics | 3 |
| [C29](C29-Gameplay-Object-Pools/C29-Gameplay-Object-Pools.md) | Gameplay object pools | 4 |
| [C30](C30-Gameplay-Managers/C30-Gameplay-Managers.md) | Gameplay managers — `CShopping`, collectibles | 4 |
| [C35](C35-Conversations/C35-Conversations.md) | Conversations | 2 |
| [C45](C45-Damage/C45-Damage.md) | Damage — `eWeaponType` 59 entries, `CEventDamage` dispatch | 4 |

### Rendering

| Chapter | Title | Pages |
|---|---|:---:|
| [C15](C15-Timecycle/C15-Timecycle.md) | Timecycle — atmospheric colour tables | 4 |
| [C16](C16-Popcycle-And-Water/C16-Popcycle-And-Water.md) | Popcycle and water | 3 |
| [C21](C21-Particles/C21-Particles.md) | Particles | 3 |
| [C38](C38-Skybox-And-Clouds/C38-Skybox-And-Clouds.md) | Skybox and clouds | 3 |
| [C39](C39-Camera/C39-Camera.md) | Camera | 4 |
| [C40](C40-Render-Pipeline/C40-Render-Pipeline.md) | Render pipeline — `CRenderer`, four entity lists, pass order | 4 |
| [C44](C44-Shaders/C44-Shaders.md) | Shaders — D3D9 VS/PS, register map, hook depths | 4 |
| [C50](C50-RenderWare-Reference/C50-RenderWare-Reference.md) | RenderWare 3.6 — the engine under the engine | 4 |

### Input, UI & Audio

| Chapter | Title | Pages |
|---|---|:---:|
| [C19](C19-GXT-Text/C19-GXT-Text.md) | GXT text — CRC-32 hash, TABL/TKEY/TDAT blocks | 4 |
| [C20](C20-Audio/C20-Audio.md) | Audio — banks, streams and the tables that index them | 7 |
| [C23](C23-Fonts-HUD/C23-Fonts-HUD.md) | Fonts and HUD | 4 |
| [C43](C43-Front-End-Menu/C43-Front-End-Menu.md) | Front-end — the pause menu as a hardcoded table | 4 |
| [C46](C46-Input-Devices/C46-Input-Devices.md) | Input devices — `CPad`, `CControllerState`, two players | 4 |

### Data Files

| Chapter | Title | Pages |
|---|---|:---:|
| [C24](C24-Surfaces/C24-Surfaces.md) | Surfaces — `surfaces.dat`, material physics | 4 |
| [C48](C48-Data-Folder-Sweep/C48-Data-Folder-Sweep.md) | Data folder sweep — every uncovered file in `data/` | 4 |

### Third-Party & Multiplayer

| Chapter | Title | Pages |
|---|---|:---:|
| [C49](C49-SAMP-Multiplayer/C49-SAMP-Multiplayer.md) | SA:MP — hook layer, network client, D3D9 vtable, RakNet | 3 |

<img width="100%" src="https://capsule-render.vercel.app/api?type=rect&color=0:000000,50:E07B00,100:000000&height=3"/>

## 🗄️ Struct Database

`RE-Data/data/` holds machine-readable JSON for every derived fact, generated by the `tools/derive_*.py` suite. Every chapter ships its own derivation script; run any of them against the retail binary to re-check its claims.

| Tier | Count | Meaning |
|---|---:|---|
| `VERIFIED_BY_DISASSEMBLY` | **2,003** | 2+ Capstone hits; VA cited; derivation in `RE-Data/` |
| `VERIFIED` | **1,402** | Confirmed by cross-reference or `VALIDATE_SIZE` |
| `REASONED` | TBD | Structurally inferred — inline derivation |
| `OPEN` | flagged | Explicitly unsettled — no assumption made |

### Fully-covered structs

| Struct | True size | Fields | Note |
|---|---|---|---|
| `CPed` | 1,988 B | 93 | 100 % byte coverage |
| `CVehicle` | 2,584 B | — | Pool stride proven by `imul` in `CPools::Initialise` |
| `CBuilding` | 56 B | — | 13,000-slot pool; dominant map entity |
| `CObject` | 412 B | — | 350-slot pool |
| `CHandlingData` | **224 B (`0xE0`)** | — | **Novel: community had `0xD4`; proven by `imul eax, eax, 0E0h` @`0x6F0151`** |
| `CPad` | 308 B (`0x134`) | 5 state bufs + tail | Proven by `imul eax, eax, 134h` in `GetPad` @`0x53FB70` |
| `CControllerState` | 48 B (`0x30`) | 24 × `int16` | 24 controls; buttons `0`/`255`, axes `−128…+128` |
| `CActiveExplosion` | 28 B | — | Pool of 64 @`0xC8AC80`; **21 explosion types** |

<img width="100%" src="https://capsule-render.vercel.app/api?type=rect&color=0:000000,50:E07B00,100:000000&height=3"/>

## 📍 Key Addresses

Frequently-referenced globals and functions, all ✅ confirmed.

| Symbol | VA | References | Note |
|---|---|---:|---|
| `ms_fTimeStep` | `0xB7CB5C` | **984** | Highest-reference variable in binary; unit = 50 ms |
| `ms_fTimeStepNonClipped` | `0xB7CB58` | — | Raw (unclamped) timestep |
| `ms_fTimeStep` clamp max | `0x858C14` | — | `2.0f` in rdata — anti-tunnel cap |
| `CTimer::Update` | `0x560D2E` | — | Writes `ms_fTimeStep` each frame |
| `CPools::Initialise` | `0x5503A0` | — | Allocates all 13 pools; patch site for pool-limit mods |
| `CPools::Shutdown` | `0x550F10` | — | Frees all pools |
| `CGame::Initialise` | `0x53BC80` | 142 callees | Largest init hub; topological sort of the subsystem graph |
| `CGame::Process` | `0x53BEE0` | 81 callees | Per-frame update hub; input → streaming → world → render |
| `CGame::Shutdown` | `0x53C900` | — | Reverse-order teardown |
| `CPad::GetPad` | `0x53FB70` | — | `base + i × 0x134`; `Pads[0]` @`0xB73458` |
| `CRT entry` | `0x824570` | — | PE `AddressOfEntryPoint`; static ctors run before WinMain |
| `ms_vehicleHandling` | `0xC2B9DC` | — | Handling index array; sub-handling stride `0x94` @`0xC3BB00` |
| `bInvertMouseX` | `0xBA6744` | 3 | Per-axis mouse invert flag |
| `bInvertMouseY` | `0xBA6745` | 8 | Vertical invert checked in more look contexts |
| `m_fMouseAccelHorzntl` | `0xB6EC1C` | 19 | Mouse sensitivity — lives on the camera, not input layer |
| `m_fMouseAccelVertical` | `0xB6EC18` | 6 | Vertical sensitivity scalar |

<img width="100%" src="https://capsule-render.vercel.app/api?type=rect&color=0:000000,50:E07B00,100:000000&height=3"/>

## 📂 Data-File Coverage

Every file in the retail `data/` folder is documented. The 13 previously-uncovered files were swept in [C48](C48-Data-Folder-Sweep/C48-Data-Folder-Sweep.md).

| File | Format | Key figure | Chapter |
|---|---|---|---|
| `handling.cfg` | column table | `CHandlingData` 0xE0, 212 vehicles | [C13](C13-Vehicle-Data/C13-Vehicle-Data.md) |
| `weapon.dat` | column table | `CWeaponInfo` per-weapon record | [C14](C14-Peds-And-Weapons/C14-Peds-And-Weapons.md) |
| `ped.dat` / `pedstats.dat` | column table | `CPedStats`; 38 ped types | [C26](C26-Ped-Tables/C26-Ped-Tables.md) |
| `surfaces.dat` | column table | 178 surface types, audio/physics | [C24](C24-Surfaces/C24-Surfaces.md) |
| `timecyc.dat` | column table | 24 columns; weathers × hours | [C15](C15-Timecycle/C15-Timecycle.md) |
| `water.dat` | geometry table | water-surface quads | [C16](C16-Popcycle-And-Water/C16-Popcycle-And-Water.md) |
| `animgrp.dat` | group list | **42** animation groups | [C48.3](C48-Data-Folder-Sweep/03-extensions-and-binary-grid.md) |
| `clothes.dat` | rule language | **87** SETC rules + HIDE/EXCLUSIVE | [C55.1](C55-CJ-Customisation/01-clothing-rule-language.md) |
| `shopping.dat` | nested sections | **37** sections, ~560 items | [C48.2](C48-Data-Folder-Sweep/02-economy-and-customization.md) |
| `default.dat` | load list | IDE/IMG/COLFILE boot manifest | [C48.1](C48-Data-Folder-Sweep/01-load-and-boot-config.md) |
| `txdcut.ide` | `txdp` section | cutscene TXD parent assignments | [C48.1](C48-Data-Folder-Sweep/01-load-and-boot-config.md) |
| `timecycp.dat` | column table | PS2 alternate timecycle (437 lines) | [C48.3](C48-Data-Folder-Sweep/03-extensions-and-binary-grid.md) |
| `polydensity.dat` | binary `uint32[72005]` | **dead asset** — zero references in retail | [C48.3](C48-Data-Folder-Sweep/03-extensions-and-binary-grid.md) |

<img width="100%" src="https://capsule-render.vercel.app/api?type=rect&color=0:000000,50:E07B00,100:000000&height=3"/>

## 🔧 Modding Reference

Common tasks and the encyclopedia pages that resolve them.

| Task | What to read |
|---|---|
| Raise the ped / vehicle / object pool cap | [C53.2](C53-Memory-And-Pool-Architecture/02-memory-budget.md) — safe maximums, cost-per-slot arithmetic, the `push-imm32` patch site at `0x5503A0` |
| Fix 60 fps physics bugs (nitro, gear shift, swim) | [C52.2](C52-CTimer-And-Game-Loop/02-frame-rate-and-physics.md) — the five non-scaled paths; the 30fps-clamp approach |
| Hook a D3D9 vtable slot | [C44.4](C44-Shaders/04-custom-shaders-and-hook-depths.md) — Present slot 17, EndScene slot 42, hook-chain conflict pattern |
| Override vehicle handling at runtime | [C47.4](C47-Vehicle-Dynamics/04-tuning-suspension-and-traction.md) — damping ratio, traction formula, mass cascade rule, ASI dynamic override |
| Read/write player input | [C46.2](C46-Input-Devices/02-edge-detection.md) — `NewState`/`OldState` double buffer; synthetic press injection |
| Parse a `.dff` / `.txd` / `.col` offline | [C7](C7-RenderWare-Stream/C7-RenderWare-Stream.md) / [C9](C9-Materials-And-Textures/C9-Materials-And-Textures.md) / [C6](C6-Collision/C6-Collision.md); see also [SASDK](../SASDK/README.md) |
| Intercept all 59 damage types | [C45.1](C45-Damage/01-every-source-is-a-weapon.md) — `eWeaponType` taxonomy; single hook on `CPed::InflictDamage` |
| Mod CJ's clothing / body stats | [C55](C55-CJ-Customisation/C55-CJ-Customisation.md) — `clothes.dat` rule language, fat/muscle globals, `CClothesBuilder` rebuild |
| Understand the SA:MP hook layer | [C49.3](C49-SAMP-Multiplayer/03-the-hook-layer-in-depth.md) — D3D9 vtable, `GetAsyncKeyState` IAT hook, RakNet, hook-chain conflict |
| Time an ASI hook correctly during init | [C51.1](C51-Executable-Lifecycle/01-startup-and-init.md) — DllMain vs `CGame::Initialise` vs `CGame::Process` windows |

<img width="100%" src="https://capsule-render.vercel.app/api?type=rect&color=0:000000,50:E07B00,100:000000&height=3"/>

## 📁 Source Layout

```
SAEncyclopedia/
├── INDEX.md                   ← master index (55 chapters, knowledge graph)
├── AUDIT.md                   ← open items and coverage audit
├── handoff.md                 ← working log
├── sae-logo.png
│
├── C0-Binary-Identity/        ← binary identity (the executable itself)
├── C1-Streaming/ … C54-Limits-Reference/   ← 55 numbered chapters
├── C55-CJ-Customisation/      ← clothing, hair, tattoos, body stats (new)
├── X1-SDK-Cross-Reference/    ← closing open items via SASDK cross-check
│
├── RE-Data/
│   └── data/                  ← machine-readable JSON outputs from all derive_*.py scripts
│       ├── datasweep.json      ← data/ folder census (8/8 automated checks)
│       ├── lifecycle_structure.json
│       ├── knowledge_graph.json
│       └── …
│
├── tools/
│   ├── derive_datasweep.py    ← re-checks every data/ file claim
│   ├── derive_lifecycle.py    ← re-checks CGame entry/init/process/shutdown VAs
│   ├── derive_input.py        ← re-checks CPad layout, Pads[] VA, GetPad stride
│   ├── derive_openitems.py    ← re-checks ms_fTimeStep refs, mouse accel VAs
│   └── …                     ← one script per chapter that has derived data
│
└── references/
    ├── gta-reversed-master/   ← gta-reversed source (corroborating — not authoritative)
    ├── plugin-sdk-master/     ← plugin-sdk (🔷 external tier)
    └── SASDK/ → ../SASDK/     ← companion SDK (verified struct database)
```

<img width="100%" src="https://capsule-render.vercel.app/api?type=rect&color=0:000000,50:E07B00,100:000000&height=3"/>

## 📖 Navigation

Start from [INDEX.md](INDEX.md) for the full knowledge graph. Each chapter hub page gives the result-first table, then links to 3–7 deep-dive subpages. Deep-dive pages carry the disassembly evidence, formula derivations, and modding tables in full detail.

The companion SDK — [SASDK](../SASDK/README.md) — exposes the same verified struct database as production C++20 headers with `static_assert`-checked sizes. Use the encyclopedia to understand the system; use SASDK to build against it.

<div align="center">

<sub>Built and maintained by <a href="https://github.com/TsyVM">TsyVM</a> · <a href="https://www.teamvanilla.org/">TeamVanilla</a></sub>

<img width="100%" src="https://capsule-render.vercel.app/api?type=waving&color=0:8B4500,100:000000&height=80&section=footer"/>

</div>
