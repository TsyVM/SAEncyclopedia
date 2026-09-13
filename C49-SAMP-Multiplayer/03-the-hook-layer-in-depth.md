# C49.3 — The SA:MP Hook Layer in Depth

> **The one-sentence version:** `samp.dll` is a hook layer that attaches at three injection points
> — the `IDirect3DDevice9` vtable for rendering, `GetAsyncKeyState` for input interception, and
> `RakNet`'s UDP transport for network synchronization — and all three are confirmed by string
> evidence in the binary; the internal mechanics beyond the hook points are closed-source.

**Subsystem category:** Multiplayer — hook architecture
**Depends on:** [C49.1](01-reuses-the-base-engine.md), [C49.2](02-the-network-client.md),
[C44](../C44-Shaders/C44-Shaders.md) (the D3D9 pipeline the rendering hook intercepts),
[C46](../C46-Input-Devices/C46-Input-Devices.md) (the input system SA:MP intercepts)
**RE status:** Documented — all three hook types confirmed by string evidence; mechanics beyond
hook points are closed-source
**Confidence:** ✅ for the three hook types (D3D9 vtable, GetAsyncKeyState, RakNet/UDP) from
string evidence · 🔷 for the exact vtable slot number and hook-chain protocol (community-documented)
· ⏳ for internal SA:MP data structures

---

## 1. The Direct3D 9 rendering hook

SA:MP draws its HUD (player name tags, scoreboard, chat box, radar overlay) on top of the base
game's rendered frame. It achieves this by **hooking the `IDirect3DDevice9` vtable** — the same
D3D9 device object that the base game's RenderWare pipeline ([C44](../C44-Shaders/C44-Shaders.md))
holds.

### 1.1 The vtable hook mechanism

The `IDirect3DDevice9` COM interface is implemented as a vtable — an array of function pointers.
The vtable slots of interest to SA:MP are those that fire at the end of each rendered frame:

- **`Present` (vtable slot 17):** called once per frame to flip the backbuffer to screen
- **`EndScene` (vtable slot 42):** called at the end of the scene before `Present`

SA:MP writes a new function pointer into one or both of these vtable slots. Its patched function:
1. Calls SA:MP's own render code (name tags, chat, HUD)
2. Calls the original `Present`/`EndScene` to complete the base game's frame

This is a **vtable pointer replacement**, not a code patch — SA:MP writes to the vtable in
memory, not to the function itself. The base game's code calls through the vtable normally;
it just now calls SA:MP's hook function instead of D3D9's original implementation.

### 1.2 The hook-chain conflict

Because the vtable exists once in process memory and SA:MP writes directly to its slot, any other
mod that also hooks the same slot by overwriting it will silently break the chain. The two-hook
scenario:

```
Base game calls through vtable → calls SA:MP's hook → SA:MP calls original
                                                              ↑
                                                  [slot was original here]

After a second mod overwrites the same slot:
Base game → calls Mod B's hook → Mod B calls original  [SA:MP is bypassed]
```

The correct resolution is **hook chaining**: each hook stores the pointer it found in the slot
before writing its own, and calls the stored pointer at the end of its code. SA:MP's hook stores
the original `Present` pointer; Mod B should store SA:MP's hook pointer (not the original), so
the call sequence is `Base → Mod B → SA:MP → Original`.

For this to work, the mod must hook **after** SA:MP's `DllMain` runs (so it finds SA:MP's hook in
the slot, not the original). A mod loaded before SA:MP via `LoadLibrary` ordering or a proxy DLL
gets the original and chains independently; SA:MP then overwrites it and chains onto it — the
resulting order is `Base → SA:MP → Mod B → Original`, which is fine as long as both chain.

### 1.3 The rendering evidence

The string `"IDirect3DDevice9"` appears in `samp.dll`'s string table alongside the vtable-slot
indices being patched. This is the string-level evidence for the D3D9 hook; the exact mechanism
is inferred from the standard SA:MP community plugin SDK (🔷) which exposes `SAMP_RegisterEvent`
for `eOnSAMPRender` events that fire inside the patched `EndScene`.

---

## 2. The input hook

### 2.1 `GetAsyncKeyState` interception

SA:MP intercepts keyboard input by patching the call to `GetAsyncKeyState` (the Windows API that
reads key state) within the base game's input processing loop ([C46](../C46-Input-Devices/C46-Input-Devices.md)).
The string `"GetAsyncKeyState"` in `samp.dll`'s import table confirms this.

The mechanism: SA:MP's hook function:
1. Checks if the requested key is one SA:MP has claimed (T for chat, F6 for scoreboard, etc.)
2. If yes: returns "not pressed" to the base game, hiding the keypress from the base game's input
   system, and processes it internally for SA:MP's own use
3. If no: calls the real `GetAsyncKeyState` and returns its result unchanged

This is a **function replacement hook** at a specific call site, not a vtable hook. SA:MP replaces
the `call GetAsyncKeyState` instruction bytes at the base game's input call site with a `call`
to SA:MP's wrapper — the standard import address table (IAT) hook pattern.

### 2.2 Keys claimed by SA:MP

The following keys are claimed by SA:MP's input hook and cannot be rebound in the base game's
control settings while SA:MP is running:

| Key | SA:MP use |
|---|---|
| T | Open chat box for typing |
| F6 | Toggle scoreboard |
| Insert | Toggle spectator mode |
| K | Enter vehicle (in some SA:MP versions) |
| Y | Quick message (in some SA:MP versions) |

The exact list varies by SA:MP version (0.3.7, 0.3.DL, etc.) and server-side configuration.
A mod that needs these keys must add itself to SA:MP's key event system
(`SAMP_RegisterEvent(eOnGetInputState, ...)`) rather than hooking `GetAsyncKeyState` independently.

### 2.3 Preventing key conflicts

For mods that do not need the above keys: the simplest strategy is to check whether `samp.dll` is
loaded before registering input bindings, and use alternative keys in multiplayer contexts:

```cpp
bool isSAMP = GetModuleHandleA("samp.dll") != nullptr;
if (isSAMP) {
    // Use F7/F8 instead of T/K for chat-adjacent actions
}
```

---

## 3. Network synchronization: scope and limits

### 3.1 The seven sync packet types (confirmed in C49.2)

SA:MP's 7 sync packet types define the boundary between what is networked and what is simulated
locally on each client:

| Packet type | What it syncs | What runs locally |
|---|---|---|
| Player sync | Position, heading, animation state, health | Ped AI, tasks, ragdoll |
| Vehicle sync | Position, velocity, driver input flags | Vehicle physics simulation |
| Aim sync | Camera direction, weapon in hand | Weapon recoil, animation |
| Passenger sync | Position, animation | Passenger AI |
| Trailer sync | Position, hitch state | Trailer swing physics |
| Unoccupied sync | Drifting vehicle position | Vehicle physics (server-driven) |
| Spectator sync | Observer position | Nothing (no collision) |

SA:MP transmits **position and state snapshots**, not physics forces. Each client runs the full SA
physics simulation locally; the server sends corrective position updates and the client lerps toward
them. The base game's physics engine, AI, streaming, and collision systems all run independently on
every client — two clients watching the same player will have slightly different simulations of all
the NPCs around them.

### 3.2 What this means for SA modding

An SA mod running in SA:MP will **run on the client** — it reads and writes the same game memory
that SA:MP syncs. A mod that teleports the player by writing to `CPlayer::m_vecPosition` will have
that position overwritten by SA:MP's next sync packet. Mods that need to persist changes in SA:MP
must work through SA:MP's plugin API (`samp.dll` export: `SAMP_RegisterEvent`,
`SAMP_SetLocalPlayerName`, etc.) to notify the server of any client-side state changes.

---

## 4. The RakNet reliability layer

The `RakNet version` string in `samp.dll` (C49.2) confirms RakNet is the transport library. RakNet
operates over UDP and provides:

- **Packet reliability:** reliable packets are retransmitted until acknowledged
- **Packet ordering:** ordered channels for state updates that must arrive in sequence
- **Connection management:** peer discovery, connection/disconnection handling, ping measurement

SA:MP uses unreliable packets for high-frequency position updates (best-effort, dropped if lost)
and reliable packets for state changes (money, health, vehicle entry/exit). The `ack/resend stats`
strings in `samp.dll` confirm the reliability protocol is active and monitored.

### 4.1 Latency effects on physics

Because SA:MP sends position snapshots rather than forces, network latency produces position
correction jumps. A player with 150ms latency will appear to "rubber-band" when their position
is corrected by the server's authoritative state. SA:MP's interpolation code (inside `samp.dll`,
closed-source) smooths these corrections but cannot eliminate the fundamental information delay.

---

## 5. The BASS audio integration

SA:MP uses the BASS audio library (`bass.dll`) for all custom audio — voice chat, server-streamed
audio, and audio from the SA:MP `.saa` archive. The base game's audio system
([C20](../C20-Audio/C20-Audio.md)) is completely separate.

The consequence: the two audio systems are independent in both code and resources. Modifying the
base game's audio (replacing MP3s, patching the audio engine) has no effect on SA:MP audio.
Conversely, SA:MP can play server-defined audio even if the player has the game audio muted.

A mod that wants to replace SA:MP audio must replace the `bass.dll` proxy or hook BASS API calls
(`BASS_ChannelPlay`, `BASS_SampleLoad`, etc.) — not the base game's audio system.

---

## 6. Hook discovery and the plugin SDK surface

The SA:MP community plugin SDK (🔷 community-documented) exposes the following hook points,
confirming by their existence what SA:MP intercepts:

| Hook event | SA:MP fires it when | Confirmed by |
|---|---|---|
| `eOnSAMPRender` | Inside the patched `EndScene` | D3D9 hook existence |
| `eOnGetInputState` | Before key state returned to game | GetAsyncKeyState hook |
| `eOnSendPacket` | Before an outgoing RakNet packet is sent | RakNet integration |
| `eOnReceivePacket` | When an incoming RakNet packet arrives | RakNet integration |
| `eOnScriptInitialise` | When SA:MP's internal script engine starts | String evidence |

These events are the surface area of the hook layer — any ASI or plugin that extends SA:MP's
functionality registers callbacks here rather than patching game memory directly.

---

### Key takeaways

- SA:MP hooks at three injection points: **D3D9 vtable** (rendering overlay), **`GetAsyncKeyState`**
  (input capture), and **RakNet/UDP** (network sync). All three are confirmed by string evidence.
- The D3D9 hook is a **vtable pointer replacement** — other mods that hook the same slot must chain
  properly to avoid silently bypassing SA:MP's overlay.
- SA:MP syncs **position/state snapshots**, not physics forces — the full simulation runs locally on
  each client independently, and position corrections arrive as lerp targets.
- BASS audio is completely separate from the base game's audio system; modifying game audio has no
  effect on SA:MP's voice/server-streamed audio.

**Previous:** [C49.2 — The network client](02-the-network-client.md)
**Continue:** [C49.4 — SA:MP data formats and limits →](04-samp-data-formats-and-limits.md)
