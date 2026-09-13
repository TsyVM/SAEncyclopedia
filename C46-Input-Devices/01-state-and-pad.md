# C46.1 — The controller state and the pad

Input in San Andreas is two nested structures: a `CControllerState` that holds the raw button and axis
values, and a `CPad` that wraps several of them plus the machinery to compare frames. This page sizes both to
the byte and locates the two-player array, all from the executable's own indexing.

## CControllerState — 24 numbers

`CControllerState` is a flat record of **24 `int16`s**, one per control, totalling **`0x30`** bytes:

| Group | Fields |
|---|---|
| Sticks | `LeftStickX`, `LeftStickY`, `RightStickX`, `RightStickY` |
| Shoulders | `LeftShoulder1/2`, `RightShoulder1/2` |
| D-pad | `DPadUp`, `DPadDown`, `DPadLeft`, `DPadRight` |
| System | `Start`, `Select` |
| Face buttons | `ButtonSquare`, `ButtonTriangle`, `ButtonCross`, `ButtonCircle` |
| Sticks-in / misc | `ShockButtonL`, `ShockButtonR`, chat, ped-walk, vehicle-mouse-look, radio-track-skip |

`24 × 2 = 0x30`, no residue (`derive_input.py`: `ccontrollerstate_tiles`). Every field is a **signed 16-bit
value**, not a bit flag — a button is `0` (up) or `255` (down), and an axis runs roughly `-128…+128`. Storing
buttons as full `int16`s rather than bits is what lets keyboard, analog stick and mouse all write the same
field with their natural resolution ([C46](C46-Input-Devices.md)); the PlayStation-flavoured names
(`ButtonCross`, `ButtonSquare`) are a legacy of the console original, kept as the canonical control names on
PC too.

## CPad — five state buffers plus a tail

`CPad` is **`0x134`** bytes, and it opens with **five** `CControllerState` buffers back to back:

```
+0x00  NewState          \
+0x30  OldState           |  live edge-detection pair (C46.2)
+0x60  PCTempKeyState     |
+0x90  PCTempJoyState     |  the three PC device reads, merged into NewState
+0xC0  PCTempMouseState  /
+0xF0  ... tail (Mode @+0x108, shake timers, flags) ...  0x44 bytes
```

`5 × 0x30 = 0xF0`, and `0xF0 + 0x44 = 0x134` (`derive_input.py`: `cpad_tiles`). The tail after the buffers
holds the pad's own state — its `Mode`, disable flags, vibration timers — and the executable confirms it: the
pad code reads a field at `[ecx + 0x108]` (`cpad_tail_after_buffers`), which lands in that `0x44`-byte tail,
past the five `0x30` buffers.

The three "PC temp" buffers are the device edges. On PC, the keyboard is read into `PCTempKeyState`, a
joystick into `PCTempJoyState`, and the mouse into `PCTempMouseState`; those three are then **reconciled**
into the single `NewState` the game reads. That is how a player using keyboard *and* mouse *and* a pad at once
still presents one coherent controller to the game — the merge happens here, in `CPad`, before anything
downstream looks.

## Pads[2] — one per player, located cold

There are two pads, and `CPad::GetPad` at `0x53FB70` computes their addresses:

```
0x53FB70: mov  eax, [esp+4]        ; pad index (0 or 1)
0x53FB74: imul eax, eax, 0x134     ; * sizeof(CPad)
0x53FB7A: add  eax, 0xB73458       ; + &Pads[0]
0x53FB7F: ret
```

So `Pads` is a static array at **`0xB73458`** in `.data`, stride **`0x134`**, and `GetPad(i) = 0xB73458 +
i × 0x134` — `Pads[0]` @`0xB73458`, `Pads[1]` @`0xB7358C`. `derive_input.py` asserts the stride, the base and
that the array is in `.data` (`cpad_stride_0x134`, `pads_base_0xB73458`, `pads_in_data`). This is the same
proof shape as every other fixed array in the project — the index multiply and the base add read straight out
of the accessor — applied to the two-player pad table.

## Key takeaways

- `CControllerState` is **`0x30`** (24 `int16` controls); buttons are `0`/`255` values and axes `-128…+128`,
  not bit flags, so any device can write them at its own resolution.
- `CPad` is **`0x134`** = five `CControllerState` buffers (`0xF0`: live New/Old + three PC temp key/joy/mouse)
  + a `0x44` tail (Mode @`+0x108`); the three temp buffers are where keyboard/joy/mouse merge into `NewState`.
- `Pads[2]` is a static array at **`0xB73458`**, one pad per player, reached by `GetPad(i) = base + i × 0x134`
  — proven cold.

**Continue:** [C46.2 — Edge detection →](02-edge-detection.md)
