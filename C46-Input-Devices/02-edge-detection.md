# C46.2 — Edge Detection

> **The one-sentence version:** San Andreas derives all three input events (just-pressed, held,
> just-released) from a single comparison between this frame's `NewState` and last frame's
> `OldState`, both stored in `CPad`; the once-per-frame `CPad::Update` copy that maintains
> this double buffer is the pulse every menu, weapon, and camera system runs on.

**Subsystem category:** Input — edge detection and event generation
**Depends on:** [C46.1](01-state-and-pad.md) (the `CPad` layout and the two state buffers)
**RE status:** Documented — double-buffer mechanism confirmed; the three macros proven from call
sites
**Confidence:** ✅ for the `NewState`/`OldState` double-buffer mechanism and the three-test
comparison · ✅ for the menu-step, weapon-fire, and camera-toggle examples

---

## 1. The double buffer: `NewState` and `OldState`

`CPad`'s control-state representation uses two parallel `CControllerState` buffers
([C46.1](01-state-and-pad.md)):

| Offset | Name | Updated when |
|--------|------|---|
| `+0x00` | `NewState` | Every frame from the device read |
| `+0x30` | `OldState` | Copied from `NewState` before the new device read |

`CPad::Update` runs exactly **once per pad per game-logic frame** (`CGame::Process` → `CPad::UpdatePads`):

```
1. Copy NewState → OldState      (preserve the previous frame's level)
2. Read device → NewState        (overwrite with the current hardware state)
```

The order is critical: OldState must capture the state *before* the new read, not after. If the
copy happened in the other order (read first, then copy), `OldState` would always equal `NewState`
and edge detection would be impossible.

---

## 2. The three edge tests

With `NewState` (this frame) and `OldState` (last frame) in hand, the three events derive directly:

| Event | Formula | Meaning |
|---|---|---|
| **Just pressed** | `NewState.b && !OldState.b` | Rising edge: button was up last frame, down this frame |
| **Held (down)** | `NewState.b` | Level: button is currently down (may have been down last frame too) |
| **Just released** | `!NewState.b && OldState.b` | Falling edge: button was down last frame, up this frame |

```
Frame timeline for a tap-and-release:

Frame N-1: OldState.b = 0, NewState.b = 0   → held=F, pressed=F, released=F
Frame N:   OldState.b = 0, NewState.b = 1   → held=T, pressed=T, released=F  ← press detected
Frame N+1: OldState.b = 1, NewState.b = 1   → held=T, pressed=F, released=F  ← held
Frame N+2: OldState.b = 1, NewState.b = 0   → held=F, pressed=F, released=T  ← release detected
Frame N+3: OldState.b = 0, NewState.b = 0   → held=F, pressed=F, released=F
```

"Just pressed" fires on exactly **one frame** — frame N. "Held" fires on frames N and N+1 (the full
duration). "Just released" fires on exactly one frame — N+2. The rest are silent.

---

## 3. Why every consumer needs the correct test

The choice of test for each game system is not cosmetic — the wrong test produces the wrong behaviour:

### 3.1 Menus: must use just-pressed

The pause menu ([C43](../C43-Front-End-Menu/C43-Front-End-Menu.md)) moves the selection cursor by
one item per directional input. If it used `BUTTON_IS_DOWN` (held), holding the D-pad down would
move the cursor through every item in one frame — the menu would be unnavigable. Using
`BUTTON_IS_PRESSED` (just-pressed) means one physical press = one cursor movement, regardless of
how long the button is held.

This is the canonical reason the double buffer exists: the device hardware reports only a level (up
or down); the double buffer transforms that level into an event (this specific frame, the button
transitioned), which is the signal menus and toggles need.

### 3.2 Weapons: semi-auto vs. full-auto

A pistol fires once per press; a submachine gun fires every frame while held. The two-fire-mode
implementation at the weapon-system level:

```
if weapon.is_semi_auto:
    if BUTTON_IS_PRESSED(FIRE):    // just-pressed — one shot per click
        weapon.Fire()
else:  // full-auto
    if BUTTON_IS_DOWN(FIRE):       // held — fires every frame
        weapon.Fire()
```

The weapon type chooses the edge test; the hardware is the same in both cases. This is why a pistol
fires once per click and a minigun fires continuously — not two different input-reading systems, but
one system using two different tests on the same `CPad` double buffer.

### 3.3 Cameras: hold vs. toggle

The "look behind" camera mode uses `BUTTON_IS_DOWN` — it is active for exactly as long as the
button is held, restoring the normal camera on release. The "change view" button uses
`BUTTON_IS_PRESSED` — it toggles the camera mode on a single-frame pulse. If "look behind" used
`BUTTON_IS_PRESSED`, it would activate on one frame and immediately be overridden; if "change view"
used `BUTTON_IS_DOWN`, the camera would cycle through all modes every frame the button was held.
Both use the same `CPad` buffer; the difference is which of the three macros is applied.

---

## 4. Analog inputs: a different path

The three-test edge model applies to **digital buttons** (boolean states). Analog inputs (thumbsticks,
triggers) in `CControllerState` are stored as `int16_t` values rather than bit flags. Analog edge
detection is different: instead of a binary comparison, the code checks a **threshold crossing**:

```
// Analog just-pressed (conceptual):
old_magnitude = fabs(OldState.stick_x)
new_magnitude = fabs(NewState.stick_x)
ANALOG_JUST_CROSSED(threshold) = new_magnitude >= threshold && old_magnitude < threshold
```

The threshold for the stick-as-button interpretation is defined per control function
([C46.3](03-bindings-and-backend.md) — the bindings that map raw inputs to game actions). A small
stick movement within the deadzone counts as "not pressed"; beyond the threshold, it triggers a
game action, and the same edge-detection principle applies (the crossing is a one-frame event, not
a continuous signal).

For continuous game controls (steering, camera rotation), the raw `int16_t` value is used directly
without thresholding — there is no "just-pressed" concept for an axis that the game reads as a
magnitude.

---

## 5. The two-player case

In two-player splitscreen mode, SA maintains **two `CPad` objects** — `Pad[0]` for player 1 and
`Pad[1]` for player 2. Each is updated independently with `CPad::Update(playerIndex)`. Edge
detection for each player reads from that player's own `NewState`/`OldState` pair — the double
buffer is per-player. A two-player game reads pad 0 for CJ's jump and pad 1 for the second player's
jump; there is no shared input state.

The implication for mods: hooking `CPad::GetPad(0)` gives CJ's input, but `CPad::GetPad(1)` gives
the second player's input (even in single-player mode, where pad 1 is always unpressed). A mod that
wants to intercept all player input must hook both pads or hook `UpdatePads` before it propagates.

---

## 6. Frame-rate interaction

The double buffer is frame-count-based, not time-based. A "just-pressed" event fires on the single
frame of the press, regardless of how long that frame takes. At 60fps, two consecutive frames are
~16ms apart; at 30fps they are ~33ms apart. For held inputs with per-frame effects (throttle held
in a vehicle), this frame-rate dependency is one of the non-scaled code paths from
[C52.2](../C52-CTimer-And-Game-Loop/02-frame-rate-and-physics.md) — holding throttle at 60fps
applies the throttle force twice as many times per second.

For **just-pressed** inputs (menus, toggles), the frame-rate sensitivity is less visible: a "just
pressed" event still fires once per physical press, and the human hand cannot press at more than
~10 presses per second — far below any difference between 30fps and 60fps per-frame polling.

---

## 7. Modding the edge detection

### 7.1 Reading input in an ASI mod

An ASI mod reads from the same `CPad` globals. The standard pattern:

```cpp
CPad* pPad = CPad::GetPad(0);  // player 1 pad
if (pPad->NewState.bButton1 && !pPad->OldState.bButton1) {
    // BUTTON_IS_PRESSED equivalent
    DoCustomAction();
}
```

The `OldState`/`NewState` offsets are from [C46.1](01-state-and-pad.md): `+0x00` for `NewState`,
`+0x30` for `OldState`. Reading from `NewState` directly gives the held test; comparing with
`OldState` gives the rising or falling edge.

### 7.2 Injecting input

A mod can simulate a button press by writing directly to `NewState`:

```cpp
// Simulate a jump button press next frame:
CPad::GetPad(0)->NewState.bButtonTriangle = 1;
```

The catch: `CPad::Update` overwrites `NewState` on the next frame from the real device. A synthetic
press survives exactly one game-logic tick — the frame after the write, `Update` will read the real
hardware state (button not pressed) and overwrite with 0. For a sustained synthetic press, the mod
must write `NewState.b = 1` every frame. For a one-shot "tap," a single write before the frame's
`UpdatePads` call produces exactly one `BUTTON_IS_PRESSED` event.

---

### Key takeaways

- Edge detection is a **double buffer**: `NewState` (this frame) vs `OldState` (last frame), updated
  by `CPad::Update` once per frame in the correct order (copy-old, then read-new).
- The three macros — `BUTTON_IS_PRESSED` (`New && !Old`), `BUTTON_IS_DOWN` (`New`),
  `BUTTON_JUST_UP` (`!New && Old`) — cover all input-event types from a single comparison.
- The **choice of macro** per consumer defines the experience: menus use just-pressed for one-step-
  per-press navigation, full-auto weapons use held, toggles use just-pressed.
- Analog inputs use a **threshold-crossing** variant of the same edge principle for game actions;
  continuous magnitudes bypass it.
- In two-player mode, each `CPad[i]` has its own `NewState`/`OldState` pair — edge detection is
  per-player.

**Continue:** [C46.3 — Bindings and the backend →](03-bindings-and-backend.md)
