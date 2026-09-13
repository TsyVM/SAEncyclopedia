# C49.2 — The network client

`samp.dll` is where multiplayer actually happens: a client library injected into the running `gta_sa.exe` that
adds networking, an overlay, and input handling the retail game never had. This page reads its structure from
its own strings — version, network layer, the sync taxonomy, and the hook surface — while marking the
method-level internals ⏳, because it is closed source.

## Version and network layer

`samp.dll` carries the version string **`0.3.7-R5`** — the last major SA:MP release line. It networks over
**RakNet**, the same UDP game-networking library many titles of the era used: its fingerprints are all over
the binary (`RakNetTime`, `PlayerID`, and a full reliability-layer statistics block — packet loss, acks,
resends, good/bad packet counts). `derive_samp.py` asserts the version and the RakNet presence. RakNet gives
SA:MP ordered/reliable and unreliable channels over UDP, which is why fast-changing state (position) can go
unreliable while critical events (chat, spawns) go reliable.

## The sync-packet taxonomy

The heart of the client is how it packages a player's state for the wire. SA:MP defines a fixed set of
**sync packets**, one per situation the player can be in, all present as `ID_*_SYNC` identifiers:

| Packet | When |
|---|---|
| `ID_PLAYER_SYNC` | on foot |
| `ID_VEHICLE_SYNC` | driving |
| `ID_PASSENGER_SYNC` | a passenger in a vehicle |
| `ID_AIM_SYNC` | aiming a weapon (camera/aim vector) |
| `ID_TRAILER_SYNC` | towing a trailer |
| `ID_UNOCCUPIED_SYNC` | an empty vehicle being pushed/moved |
| `ID_SPECTATOR_SYNC` | spectating |

`derive_samp.py` confirms all **7** (`sync_packet_taxonomy`). Each frame the client picks the packet matching
the local player's state, fills it with position/velocity/rotation/health/weapon/keys, and sends it; incoming
packets of the same kinds drive the *other* players. Those remote players are ordinary
[C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md)-pool `CPed`/`CVehicle` entities — but their movement
comes from the network, not the [C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md) task tree. The
seven-way split exists because each situation has different state to send (a passenger needs no steering; an
aimer needs a camera vector), so a single fat packet would waste bandwidth.

## The hook surface

To do all this inside a game that knows nothing of networking, `samp.dll` hooks three of the engine
subsystems this encyclopedia documents:

- **Direct3D 9** — `D3DXCheckVersion` and the D3D init strings show SA:MP intercepts the device to draw its
  own chat box, scoreboard and nametags **over** the [C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)
  frame (the overlay that makes SA:MP recognisable). This is the same D3D9 the [C44](../C44-Shaders/C44-Shaders.md)
  RenderWare pipelines run on.
- **Input** — `GetAsyncKeyState` shows it reads the keyboard directly (for chat and its own keybinds),
  alongside the game's [C46](../C46-Input-Devices/C46-Input-Devices.md) `CPad`.
- **Audio** — `BASS_ChannelSetSync` shows it uses the **BASS** library (`bass.dll`, also in the tree) for its
  own audio streams, separate from the [C20](../C20-Audio/C20-Audio.md) engine audio.

Plus **RCON** (`ID_RCON_COMMAND`) — the remote-admin console, with a standalone `rcon.exe` client in the tree
for server operators.

## What is left open (and why)

The honest boundary: everything above is read from strings and headers the client *ships*. The parts that
would require disassembling a closed-source binary to method level are left ⏳:

- the **exact byte layout** of each sync struct (the compression/quantization of position and rotation);
- the **`samp.saa`** archive: now *identified* — magic **`SAA2`** ("SA:MP Archive v2"), a `uint32` entry
  count (**22**), then an **encrypted/high-entropy payload** (`derive_openitems.py`: `samp_saa_format`). The
  container is characterised; the entries are encrypted, so decoding them needs the client's key — the format
  is named, the contents stay ⏳;
- the **hook mechanism** (how the D3D9 device and game loop are detoured).

These are not "not derivable" in principle — they are "not derived here", because the project does not
reverse a third-party client to that depth. A future pass could disassemble `samp.dll` with the same capstone
tooling used on `gta_sa.exe`; it would just be documenting someone else's program, at the lead (🔷) tier.

## Key takeaways

- `samp.dll` is version **0.3.7-R5**, networks over **RakNet** (UDP + reliability layer), and drives remote
  players as network-fed [C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md) entities.
- Player state is sent via a **7-packet sync taxonomy** (player/vehicle/passenger/aim/trailer/unoccupied/
  spectator), one per situation to save bandwidth.
- It hooks **D3D9** ([C40](../C40-Render-Pipeline/C40-Render-Pipeline.md)/[C44](../C44-Shaders/C44-Shaders.md))
  for its overlay, **input** ([C46](../C46-Input-Devices/C46-Input-Devices.md)), and **BASS** for audio, with
  **RCON** for admins; the closed-source internals (sync layout, `.saa`, hook mechanism) are honestly ⏳.

**Continue:** [back to the C49 hub →](C49-SAMP-Multiplayer.md) · or [C7 — RenderWare Stream](../C7-RenderWare-Stream/C7-RenderWare-Stream.md) (the IMG format SA:MP reuses).
