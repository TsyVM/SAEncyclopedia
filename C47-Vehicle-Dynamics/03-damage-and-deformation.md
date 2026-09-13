# C47.3 — Damage and deformation

A ped has one health bar ([C45](../C45-Damage/C45-Damage.md)); a vehicle has many. San Andreas tracks vehicle
damage **per component** — engine, each wheel, each door, each panel, each light — in a compact
`CDamageManager`, and progresses each from intact to damaged to gone. This page reads that structure and its
degradation model.

## CDamageManager — 24 bytes for the whole car

`CDamageManager` is just **`0x18`** (24) bytes, because each component's state is tiny:

| Field | Type | Meaning |
|---|---|---|
| `m_fWheelDamageEffect` | `float` | how much wheel damage skews the handling |
| `m_nEngineStatus` | `uint8` (**0–250**) | engine health; higher = more damaged (smoke, then fire, then stall) |
| `m_anWheelsStatus[4]` | `eCarWheelStatus[4]` | per-wheel: OK / burst / missing |
| `m_aDoorsStatus[6]` | `eDoorStatus[6]` | per-door: OK / damaged / swinging / missing |
| `m_nLightsStatus` | `uint32` bitfield | the four lights, packed |
| `m_nPanelsStatus` | `uint32` | the panels, indexed by `ePanels` |

The layout tiles: `4 + 1 + 4 + 6 + 4 + 4 = 23`, plus one padding byte = `0x18`
(`derive_vehicle_dynamics.py`: `cdamagemanager_tiles`). Two design choices stand out. First, **engine status
is 0–250, not 0–100** — a wider range that lets the engine pass through distinct stages (healthy → smoking →
on fire → dead) with room between them. Second, **doors and wheels are per-component arrays** (`[6]` and
`[4]`), so the game knows *which* door is hanging off and *which* wheel burst — the front-left door and the
rear-right wheel have independent state.

## The components

- **Wheels (4)** — `eCarWheelStatus`, indexed FL/RL/FR/RR ([C47.1](01-four-wheel-suspension.md)'s ordering). A
  wheel can burst (deflate — the car pulls to that side via `m_fWheelDamageEffect`) or come off entirely.
- **Doors (6)** — `DOOR_BONNET`, `DOOR_BOOT`, `DOOR_LEFT_FRONT`, `DOOR_RIGHT_FRONT`, `DOOR_LEFT_REAR`,
  `DOOR_RIGHT_REAR`. Each progresses OK → damaged (bent, opens loosely) → **missing** (torn off), the
  `ePanelDamageState` enum.
- **Panels** — the body panels (wings, bumpers), a `uint32` indexed by `ePanels`, each with the same
  OK/damaged/missing states; this is the visible crumpling.
- **Lights** — a `uint32` bitfield, one state per headlight/tail-light (working or smashed).
- **Engine** — the single `0–250` scalar.

## Progressive damage

Damage does not flip a component straight to broken; it *progresses*. `CDamageManager` exposes one function
per component type:

```
ProgressWheelDamage(wheel)   ProgressPanelDamage(panel)
ProgressDoorDamage(door)     ProgressEngineDamage()
```

Each call advances that component one stage toward destruction, so repeated impacts on the same door walk it
OK → damaged → missing. This is the vehicle mirror of [C45](../C45-Damage/C45-Damage.md)'s ped model: where a
ped's damage subtracts from one armour/health pool, a car's damage *advances a state machine per part*. A
collision ([C6](../C6-Collision/C6-Collision.md)) that hits the front-left corner progresses the front-left
panel, the bonnet and possibly the front-left wheel — each independently — which is why crash damage looks
localized: only the parts you hit degrade.

At the end of the ladder is the aptly-named **`FuckCarCompletely(bDetachWheel)`** — the "total it" routine
that maxes every component's damage at once (engine dead, all panels gone, optionally wheels detached), used
when a car is written off in one catastrophic event. Its blunt name is a genuine Rockstar function label
preserved in `gta-reversed`; it is the single call that takes a car from any state straight to wreck.

## What ties where

Vehicle damage sits between two arcs. Its **cause** is [C45](../C45-Damage/C45-Damage.md)/[C6](../C6-Collision/C6-Collision.md):
a collision impulse or a `WEAPON_EXPLOSION` supplies the damage, exactly as it does to a ped, but the sink is
this per-component manager rather than a health float. Its **effect** feeds back into
[C47.1](01-four-wheel-suspension.md)/[C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md): a burst wheel
changes the suspension and pulls the steering (`m_fWheelDamageEffect`), a dead engine kills the drive force. So
damage is not cosmetic — it loops back into the dynamics of the same chapter.

## Open items

- ⏳ The exact `m_nEngineStatus` thresholds for smoke / fire / stall.
- ⏳ The **deformation** geometry — how a damaged panel's *mesh* is bent (the visual crumple, distinct from
  the status byte).
- ⏳ The `ePanels` bit layout in `m_nPanelsStatus` and the light bit layout in `m_nLightsStatus`.

## Key takeaways

- `CDamageManager` (**`0x18`**) tracks vehicle damage **per component**: engine (0–250), 4 wheels, 6 doors, a
  light bitfield and a panel bitfield — 23 bytes + 1 pad, tiling exactly.
- Damage **progresses** per part (OK → damaged → missing) via `Progress*Damage`, so crash damage is localized
  to what you hit — the state-machine-per-part counterpart to [C45](../C45-Damage/C45-Damage.md)'s single ped
  health pool.
- Damage loops back into dynamics — a burst wheel skews handling, a dead engine kills drive — and
  `FuckCarCompletely` is the one call that totals every component at once.

**Continue:** [back to the C47 hub →](C47-Vehicle-Dynamics.md) · or [C42 — Vehicle Physics](../C42-Vehicle-Physics/C42-Vehicle-Physics.md) · [C45 — Damage](../C45-Damage/C45-Damage.md).
