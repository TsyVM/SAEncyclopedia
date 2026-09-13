# C46.4 — The mouse

The mouse is the one device that does not fit the button model cleanly — it reports *motion*, not levels —
and San Andreas handles it through its own path before merging into the shared `CControllerState`. This page
covers that path and the settings that shape it.

## The mouse is its own temp buffer

Recall the five `CControllerState` buffers in `CPad` ([C46.1](01-state-and-pad.md)): one of them,
`PCTempMouseState` (`+0xC0`), exists purely for the mouse. Each frame the mouse's relative motion (`dx`, `dy`)
and its buttons are read from DirectInput into that buffer, then reconciled into `NewState` alongside the
keyboard and joystick. So the mouse does not have a separate code path through the whole game — it has a
separate *read*, after which it becomes ordinary controller state (its motion feeds the right-stick / look
axes, its buttons the fire/aim controls).

This is why aiming with a mouse and aiming with a right stick feel like the same system to the game: by the
time anything downstream looks, both are `RightStickX/Y` in `NewState`. The device difference lives only in
the `PCTempMouseState` read.

## Invert — proven cold

The two mouse settings the executable exposes as globals are the **invert** flags:

| Global | VA | Effect |
|---|---|---|
| `bInvertMouseX` | `0xBA6744` | flip horizontal look |
| `bInvertMouseY` | `0xBA6745` | flip vertical look |

Both are real, code-referenced globals — `derive_input.py` finds `bInvertMouseX` referenced 3× and
`bInvertMouseY` 8× in `.text` (`mouse_invert_flags`). The mouse read consults them to negate `dx`/`dy` before
the value enters `NewState`:

```
invertX = bInvertMouseX ? -1 : 1
invertY = bInvertMouseY ? -1 : 1
```

Vertical invert getting more references than horizontal (8 vs 3) fits the game — Y-invert is offered in more
contexts (on-foot look, vehicle look, flight, each checking it), while X-invert is rarer. The flags are
per-axis booleans, set from the options menu ([C43](../C43-Front-End-Menu/C43-Front-End-Menu.md)) and read at
the point the raw mouse delta is turned into a look value.

## Sensitivity — resolved

The mouse **sensitivity** scalar is not a `CMenuManager` field at all — it lives on the **camera**, as a
pair of acceleration scalars applied to the mouse delta in the look path
([C39](../C39-Camera/C39-Camera.md)):

| Global | VA | Effect |
|---|---|---|
| `m_fMouseAccelHorzntl` | `0xB6EC1C` | horizontal look sensitivity (yaw per `dx`) |
| `m_fMouseAccelVertical` | `0xB6EC18` | vertical look sensitivity (pitch per `dy`) |

Both are real, heavily-used code globals — `derive_openitems.py` finds `m_fMouseAccelHorzntl` referenced
**19** times in `.text` and `m_fMouseAccelVertical` **6** times (`mouse_accel_globals`), which fits: the
horizontal scalar is consulted in more look contexts (on-foot, vehicle, aim, flight) than the vertical. The
mouse read ([C46.1](01-state-and-pad.md)) produces a raw `dx`/`dy`; the camera multiplies it by these
acceleration scalars (after the [invert](#invert--proven-cold) sign flip) to get the yaw/pitch it applies.
So the sensitivity you set in the options menu ([C43](../C43-Front-End-Menu/C43-Front-End-Menu.md)) writes
`m_fMouseAccelHorzntl`/`Vertical`, and *these two globals are the "mouse sensitivity"* — the earlier
"⏳ open" is now closed: it was a camera scalar, not an input-layer one, which is why the input-side search
missed it. (A near neighbour, `m_f3rdPersonCHairMultX` @`0xB6EC14`, scales the third-person crosshair.)

## The mouse in the whole input path

Slotting the mouse into [C46.3](03-bindings-and-backend.md)'s pipeline:

1. **DirectInput 8** reads the mouse device (relative motion + buttons).
2. The read lands in `PCTempMouseState` (`+0xC0`), with `bInvertMouseX/Y` negating the axes.
3. It reconciles into `NewState`, where the motion becomes look-axis values and the buttons become
   fire/aim controls.
4. From there it is ordinary controller state — edge-detected ([C46.2](02-edge-detection.md)), bound
   ([C46.3](03-bindings-and-backend.md)), consumed by the camera ([C39](../C39-Camera/C39-Camera.md)) and
   weapons ([C45](../C45-Damage/C45-Damage.md)).

## Key takeaways

- The mouse has its own read into the `PCTempMouseState` buffer (`+0xC0`), then becomes ordinary
  `CControllerState` — which is why mouse-look and stick-look are one system downstream.
- The **invert** settings `bInvertMouseX` (`0xBA6744`) and `bInvertMouseY` (`0xBA6745`) are code-referenced
  globals (proven cold) that negate the axes before they enter `NewState`.
- The **sensitivity** scalars are `m_fMouseAccelHorzntl` (`0xB6EC1C`, 19 refs) and `m_fMouseAccelVertical`
  (`0xB6EC18`, 6 refs) — **camera** look-acceleration globals, not input-layer ones (now closed).

**Continue:** [back to the C46 hub →](C46-Input-Devices.md) · or [C39 — Camera](../C39-Camera/C39-Camera.md) (what mouse-look drives).
