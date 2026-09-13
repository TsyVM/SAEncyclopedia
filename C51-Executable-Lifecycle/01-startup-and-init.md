# C51.1 — Startup and initialisation

> **The one-sentence version:** execution enters the retail binary at the CRT entry point `0x824570`,
> runs the MSVC C runtime setup, reaches `WinMain` to create the window and D3D9 device, hands
> control to `RwEngineOpen`/`RwEngineStart` to bring up RenderWare, and then calls
> `CGame::Initialise` at `0x53BC80` — which fans out to **142** subsystem initialisers in strict
> dependency order before the first game-logic frame runs.

**Subsystem category:** Executable architecture — startup chain
**Depends on:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md) (the binary this runs in),
[C50](../C50-RenderWare-Reference/C50-RenderWare-Reference.md) (RenderWare bring-up)
**RE status:** Documented — entry point, three CGame functions, and their callee counts confirmed;
WinMain and RenderWare stage detail 🟡 from gta-reversed and string evidence
**Confidence:** ✅ for the entry VA `0x824570`, `CGame::Initialise` @`0x53BC80` (142 callees),
`CGame::Process` @`0x53BEE0` (81 callees), `CGame::Shutdown` @`0x53C900` ·
🟡 for WinMain internals (gta-reversed naming, corroborated by error strings and import evidence)

---

## 1. The full startup chain

```
0x824570  PE AddressOfEntryPoint — the CRT startup stub
    │
    ├─ MSVC CRT init: heap, atexit table, static-storage-duration constructors
    │    (every global C++ object with a constructor runs here, before WinMain)
    │
    ▼
WinMain   (🟡 approximate VA 0x748C50)
    │
    ├─ Read gta_sa.set  — stored video/control preferences
    ├─ Window class registration (WNDCLASSEX, "Grand Theft Auto San Andreas")
    ├─ CreateWindowEx — the game window
    ├─ D3D9 device creation (IDirect3D9::CreateDevice — see §2)
    │
    ▼
RwEngineOpen / RwEngineStart
    ├─ Platform-init: the D3D9 driver (RwD3D9DeviceSystemRequest) registers
    ├─ Plugin attaches: rpSkin, rpHAnim, rpMatFX, rpUVAnim, rpAnisot, rpWorld (see §3)
    └─ Texture dictionary store wired to RW
    │
    ▼
CGame::Initialise  @0x53BC80
    │  142 subsystem initialisers in dependency order (see §4)
    ▼
Main message loop / game loop
    └─ CGame::Process every frame (@0x53BEE0, 81 calls)  →  C51.2
```

---

## 2. The WinMain phase: window and D3D9 device

### 2.1 `gta_sa.set` — preferences before the window exists

Before the window is created, `WinMain` (🟡) reads `gta_sa.set`, the persistent settings file
stored in the user's My Documents. This file holds the previously-chosen resolution, anti-aliasing
level, control bindings, and audio volumes. On a fresh install it does not yet exist and the engine
falls back to hardcoded defaults. The settings are loaded first because the window must be created
at the right size; creating the window then changing its size causes a frame of incorrect rendering.

### 2.2 Window creation

A Win32 window class named `"Grand Theft Auto San Andreas"` (the class name that tools like Spy++
see) is registered and `CreateWindowEx` is called with the stored dimensions. The window style
depends on the fullscreen setting: fullscreen uses `WS_POPUP` (no border, covers the whole screen);
windowed mode uses `WS_OVERLAPPEDWINDOW`. Either way, the HWND the window returns is passed
directly to D3D9.

### 2.3 D3D9 device creation: the 800×600×32 hard minimum

`IDirect3D9::CreateDevice` is the call that creates the hardware rendering context. The exact
parameters are 🟡 (not individually traced to instruction-level), but the error string
`"GTA SA requires 800x600x32"` in the executable's string table confirms the minimum validation
check: if the user's selected mode is below 800×600 at 32 bpp, SA refuses to start with that
error rather than creating a degraded device. The device is created with:

- **D3DDEVTYPE_HAL** (hardware acceleration — no REF fallback)
- **CREATE_FPU_PRESERVE** flag: this flag tells D3D9 to not change the x87 FPU control word. SA
  uses x87 for all floating-point ([C52.4](../C52-CTimer-And-Game-Loop/04-assembly-and-disassembly.md))
  — without `FPU_PRESERVE`, D3D9's initialization would set the FPU to 64-bit precision, breaking
  the engine's 80-bit extended-precision x87 operations.
- A `D3DPRESENT_PARAMETERS` struct built from the settings read from `gta_sa.set`.

The `IDirect3DDevice9*` returned is the device that the entire RenderWare pipeline (C50) and the
modding hook layer (C44.4, C49.3) operate on. It lives for the process lifetime.

### 2.4 The CRT static-constructor phase (before WinMain)

The CRT entry at `0x824570` runs **before** any of the above. Critically, the MSVC CRT entry
stub walks the `.CRT$XI` and `.CRT$XC` linker sections, calling every C++ static storage duration
constructor before control reaches `WinMain`. This means any global C++ object in the binary whose
constructor does significant work (registering something, allocating memory, touching globals) runs
here — outside the debugger's normal `WinMain` breakpoint. ASI mods loaded via a `d3d9.dll` proxy
also have their `DllMain` called during this phase, because the proxy DLL is implicitly linked
and its DLL_PROCESS_ATTACH fires during the CRT's `_initterm` walk. This is why early-injection
ASI mods can run before `CGame::Initialise` — they execute during the static-init phase.

---

## 3. RenderWare bring-up: `RwEngineOpen` / `RwEngineStart`

After the D3D9 device is created, the code calls two RenderWare functions to connect RW to D3D9:

### 3.1 `RwEngineOpen`

`RwEngineOpen` initializes the RenderWare core library — allocating the internal RW heap, setting
up the stream subsystem, and preparing the plugin table. It does not yet touch D3D9.

### 3.2 Plugin attaches — the RW plugin table

Before `RwEngineStart`, a set of **plugin attach** calls registers the RW extensions SA uses
([C50.2](../C50-RenderWare-Reference/02-plugins-driver-pipelines.md)):

| Plugin | Call | Purpose |
|---|---|---|
| `rpWorldPluginAttach` | geometry/world system | `RpGeometry`, `RpWorld` |
| `rpSkinPluginAttach` | skin plugin | per-vertex bone weights |
| `rpHAnimPluginAttach` | hierarchical animation | bone-to-frame maps |
| `rpMatFXPluginAttach` | material effects | env-map, dual-texture |
| `rpUVAnimPluginAttach` | UV animation | scrolling texture coords |
| `rwD3D9DevicePluginAttach` | the D3D9 driver | the device `this` |

Each `Attach` call registers the plugin's per-object data allocation size and its stream-read
callbacks. After all attaches, the plugin table is fixed for the process lifetime — no
`Detach`/`Reattach` is possible at runtime. This is why custom RW plugins for mods must be
attached here, before `RwEngineStart`.

### 3.3 `RwEngineStart`

`RwEngineStart` completes the bring-up: it calls `RwD3D9DeviceSystemRequest` to drive the D3D9
device creation through the RW driver, making the earlier `IDirect3DDevice9*` accessible to all
RW rendering calls. After `RwEngineStart` returns, `RpAtomicRender` can issue draw calls — the
pipeline is live.

---

## 4. `CGame::Initialise` @`0x53BC80` — the 142-call fan-out

`CGame::Initialise` is the single largest init hub in the binary: `derive_lifecycle.py` counts
**142 direct callees** (`initialise_is_init_hub`). It is not a sequential list — it is a
**topological sort of the subsystem dependency graph**.

### 4.1 The dependency invariant

The core constraint:

> A subsystem X must initialise before any subsystem Y that depends on X.

Violations of this rule produce crashes or silent corruption — they cannot be caught by a compiler.
The 142 ordering is SA's engineers' hand-maintained solution to the dependency graph. Reading it
backwards from the crash site of a newly-added hook is the canonical debugging technique for
"why does my ASI crash during init."

### 4.2 Phase breakdown (from C51.3)

The 142 callees fall into four dependency phases, each depending on the phases before it:

| Phase | Name | Examples |
|---|---|---|
| 1 | **Foundation** | Memory allocator, CRT heap, `CdStreamInit` (IO thread), D3D9 device, `CTxdStore::Initialise`, `CColStore::Initialise` |
| 2 | **Data infrastructure** | `CModelInfo::Initialise`, `CStreaming::Initialise`, `CPools::Initialise` @`0x5503A0` (all 13 pools) |
| 3 | **World** | `CFileLoader::LoadLevel("DATA/DEFAULT.DAT")` (reads all IDE/IPL/COL), `CWorld::Initialise`, `CPopulation::Initialise` |
| 4 | **Game systems** | `CTheScripts::Init`, `CPad::Initialise`, `CTimer::Initialise`, `CPickups::Initialise`, `CGarages::Init`, audio, HUD |

The critical early calls:
- **`CdStreamInit`** (Phase 1): starts the streaming IO worker thread. Without it, all subsequent
  `CdStreamRead` calls block forever — the streaming system must be live before data files are
  loaded.
- **`CPools::Initialise`** @`0x5503A0` (Phase 2): allocates all 13 entity pools ([C53](../C53-Memory-And-Pool-Architecture/C53-Memory-And-Pool-Architecture.md)).
  No entity (`CPed`, `CVehicle`, `CBuilding`, `CObject`) can be allocated before this call. The
  pool counts are hardcoded `push-imm32` immediates immediately before the call — this is the
  patch site for pool-limit mods.
- **`CFileLoader::LoadLevel`** (Phase 3): reads `default.dat` → triggers the cascade of IDE/IPL/COL
  loads that populate the model database, the collision store, and the map. This is by far the
  slowest init step. By the time it returns, the full object definition table is in memory.

### 4.3 What 142 callees means architecturally

The call-count is not incidental — it is the breadth of SA's subsystem graph. Each callee is a
module boundary: a subsystem with its own init/shutdown pair. The 142 count tells you how many
independent subsystems SA has, and the 81 count in `CGame::Process` ([C51.2](02-frame-loop-and-shutdown.md))
tells you how many of those require per-frame updates. The 61 difference (142 − 81) are subsystems
that initialize but do not need per-frame work — data stores, lookup tables, one-time pools.

---

## 5. Where data files are consumed

Startup is the **only** phase where `data/` files are parsed. The sequence inside `CGame::Initialise`:

```
CFileLoader::LoadLevel("DATA/DEFAULT.DAT")
    │
    ├─ IDE line: CFileLoader::LoadObjectTypes  → fills CModelInfo
    ├─ IMG line: mounts gta3.img/gta_int.img  → feeds CStreaming (C2)
    ├─ COLFILE line: loads .col archives      → fills CColStore
    │
    └─ After DEFAULT.DAT: each subsystem reads its own .dat
         handling.cfg    → CVehicleModelInfo (C13/C42)
         weapon.dat      → CWeaponInfo (C14)
         ped.dat         → CPedStats (C26)
         timecyc.dat     → CTimeCycle (C15)
         water.dat       → waterquads (C16)
         animgrp.dat     → animation groups (C17/C48.3)
         shopping.dat    → CShopping (C30/C48.2)
         clothes.dat     → CClothesBuilder (C48.2)
         surfaces.dat    → CSurfaceTable (C24)
```

By the time `CGame::Initialise` returns and the first frame runs, every table documented in the
data-file chapters is in memory. The per-frame loop ([C51.2](02-frame-loop-and-shutdown.md)) then
only **reads** those tables — there is no re-parsing at runtime.

---

## 6. Error paths during init

SA has minimal error recovery during startup. The observed failure modes:

| Failure | SA's response |
|---|---|
| D3D9 device creation fails | Message box, process exits — no fallback device |
| Video mode below 800×600×32 | `"GTA SA requires 800x600x32"` message, exits |
| IMG archive not found | Crash (unhandled null read from the archive mount) — no error dialog |
| Out of memory during pool alloc | Crash (operator new returns null → immediate deref) — see C53.2 |
| DFF/TXD load failure | Silent skip — entity is just absent from the world |

The lack of graceful error recovery for archive/pool failures is the reason pool-size modding
requires care: too large a pool count + fragmented address space = instant crash with no diagnostic.

---

## 7. ASI loader injection timing

The dominant injection mechanism — a `d3d9.dll` proxy loaded by the Windows DLL loader — fires
**before WinMain**, during the CRT's `_initterm` walk (§2.4 above). This gives ASI mods three
possible hook windows:

| Window | Timing | What's available |
|---|---|---|
| **DllMain (DLL_PROCESS_ATTACH)** | Before WinMain | CRT heap only; no game systems, no D3D9 |
| **Hook on `CGame::Initialise`** | Between WinMain and the 142-call fan-out | D3D9 live; pool patch must go here |
| **Hook on `CGame::Process`** | First frame | Everything initialized; safe for entity queries |

The "patch the pool count before `CPools::Initialise`" technique ([C53.2](../C53-Memory-And-Pool-Architecture/02-memory-budget.md), C51.3)
requires window 2: the hook installs at `CGame::Initialise`'s prologue, patches the `push-imm32`
immediate before `CPools::Initialise` is called, and jumps to the original code. A DllMain-window
hook cannot do this because `CGame::Initialise` has not been called yet; a `CGame::Process`-window
hook is too late because the pools are already allocated.

---

### Key takeaways

- Startup is: **CRT entry `0x824570`** → MSVC static constructors (ASI DllMain fires here) →
  **`WinMain`** (window + D3D9 device, `FPU_PRESERVE` required) → **RenderWare bring-up**
  (plugin attaches, then `RwEngineStart`) → **`CGame::Initialise`** @`0x53BC80` (142 inits, §4).
- The D3D9 device is created with **`D3DCREATE_FPU_PRESERVE`** — without it, D3D9 would corrupt
  SA's x87 80-bit precision arithmetic.
- **`CPools::Initialise`** @`0x5503A0` is the exact call that allocates the 13 entity pools; pool
  patches must target the `push-imm32` immediates immediately before this call.
- All `data/` files are parsed inside `CGame::Initialise` — by the first frame everything is in
  memory and there is no runtime re-parse.
- ASI mods must hook **at `CGame::Initialise`** (not `DllMain`, not `CGame::Process`) if they need
  to patch pool or streaming buffer sizes before allocation.

**Continue:** [C51.2 — The frame loop and shutdown →](02-frame-loop-and-shutdown.md)
