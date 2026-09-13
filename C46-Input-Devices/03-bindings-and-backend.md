# C46.3 — Bindings and the backend

Two pieces remain to close the input path: the **binding layer** that decides *which* physical key fills
*which* control, and the **device backend** that reads the hardware in the first place. This page covers both
— and corrects a common assumption about the second.

## The binding layer

The game logic never asks "is the F key down?" — it asks "is the *fire* action active?" The translation is
`CControllerConfigManager` (**`0x12E4`** bytes), which holds the player's key map. Its actions are an
enumeration, `eControllerAction`, of about **53** bindable gameplay controls:

| Band | Examples |
|---|---|
| On-foot | `PED_FIRE_WEAPON`, `PED_JUMPING`, `PED_SPRINT`, `PED_DUCK`, `PED_LOOKBEHIND`, `PED_ANSWER_PHONE` |
| Movement | `GO_FORWARD`, `GO_BACK`, `GO_LEFT`, `GO_RIGHT` |
| Vehicle | `VEHICLE_ACCELERATE`, `VEHICLE_BRAKE`, `VEHICLE_HANDBRAKE`, `VEHICLE_HORN`, `VEHICLE_STEERLEFT/RIGHT` |
| Targeting/camera | `PED_LOCK_TARGET`, `PED_CYCLE_TARGET_LEFT/RIGHT`, `CAMERA_CHANGE_VIEW_ALL_SITUATIONS` |
| Radio / misc | `VEHICLE_RADIO_STATION_UP/DOWN`, `VEHICLE_RADIO_TRACK_SKIP`, `NETWORK_TALK`, group-control |

The marker `NUM_OF_1ST_PERSON_ACTIONS` sits at **53**, after which a handful of debug/screenshot actions
follow (🟡 — the count is `gta-reversed`'s enum). The lookup is
`GetControllerKeyAssociatedWithAction(action, type)`: given an abstract action and an input *type* (keyboard,
mouse, joystick), it returns the physical key bound to it, from a per-action record (`CControllerAction`,
`0x20` bytes). So the pipeline is three stages:

```
device  →  CControllerState (raw)  →  config lookup (action↔key)  →  "is action X active?"
```

The binding layer is why the controls screen ([C43](../C43-Front-End-Menu/C43-Front-End-Menu.md)) can rebind
any action: changing a binding rewrites the `CControllerAction` record, and every future
`GetControllerKeyAssociatedWithAction` call returns the new key — the game logic, asking only about *actions*,
never notices.

## The backend — DirectInput 8, not SDL

Here is the correction. Reading `gta-reversed` alone would suggest San Andreas reads input through **SDL** —
its `CPad` methods take `SDL_Event`s (`ProcessGamepadEvent(const SDL_Event&)`). They do not, in the shipped
game. That SDL path is `gta-reversed`'s **reimplementation** of the input backend, swapped in to make the
reversed project portable. The retail `gta_sa.exe` reads devices through **DirectInput 8**: the imports
`DINPUT8.dll` and `DirectInput8Create` are present in the executable (`derive_input.py`:
`directinput_backend`), and there is no SDL anywhere in the retail binary.

This is exactly the kind of trap the house rules warn about — `gta-reversed` is a *corroborating* source for
structure (its `VALIDATE_SIZE` for `CPad` `0x134` and `CControllerState` `0x30` are gold), but it is a
*reimplementation* for behaviour, and its device layer was rewritten. The struct layout is the original's; the
backend is not. So the honest statement: the **data structures** (`CPad`, `CControllerState`, `Pads[2]`, the
config manager) are the retail game's, verified against retail sizes; the **device read** is DirectInput 8 in
retail, and only SDL in the reimplementation.

## The full path, and what it ties

Putting the input chapter together, one keypress travels:

1. **DirectInput 8** reads the keyboard/mouse/joystick device state.
2. The reads land in `CPad`'s three **PC temp** buffers ([C46.1](01-state-and-pad.md)) and reconcile into
   `NewState`.
3. **Edge detection** ([C46.2](02-edge-detection.md)) compares `NewState` to `OldState` for press/hold/release.
4. The **binding layer** maps the raw control to an abstract `eControllerAction`.
5. Consumers ask "is this action active?" — the menu ([C43](../C43-Front-End-Menu/C43-Front-End-Menu.md))
   steps, the ped fires ([C45](../C45-Damage/C45-Damage.md)), the car accelerates
   ([C42](../C42-Vehicle-Physics/C42-Vehicle-Physics.md)).

## Open items

- ⏳ The full `CControllerConfigManager` layout (the per-type binding tables and the default-mapping tables).
- ⏳ The mouse read specifically (sensitivity, invert, the `m_bVehicleMouseLook` path).
- ⏳ Controller vibration (the shake timers in the `CPad` tail).

## Key takeaways

- `CControllerConfigManager` (**`0x12E4`**) maps ~53 abstract `eControllerAction`s onto physical keys per
  input type via `GetControllerKeyAssociatedWithAction`, so game logic asks about *actions*, never keys — and
  rebinding just rewrites the record.
- The shipped backend is **DirectInput 8** (`DINPUT8.dll`/`DirectInput8Create`); `gta-reversed`'s SDL is a
  reimplementation, a reminder that it corroborates *structure*, not *behaviour*.
- The full path is device → `CControllerState` → edge detection → binding → "is action X active?", tying
  input to the menu, weapons and vehicles.

**Continue:** [back to the C46 hub →](C46-Input-Devices.md) · or [C43 — Front-End Menu](../C43-Front-End-Menu/C43-Front-End-Menu.md) (what the pad drives).
