# Input Devices (CPad / CControllerState)
*Source: `input_structure.json`*
- **$schema:** input_structure.v1
- **Generated:** tools/derive_input.py

## Summary
| Field | Value |
| --- | --- |
| CControllerState size | 0x30 (24 int16) |
| CPad size | 0x134 |
| State buffers | 5 (0xf0) |
| Pads array | 0xb73458 x2 (stride 0x134) |
| Config manager | 0x12e4 (53 actions) |
| Backend | DirectInput 8 (DINPUT8.dll / DirectInput8Create) |
| Checks passed | 8/8 |

## Edge detection
BUTTON_IS_PRESSED = NewState.btn && !OldState.btn (rising edge from double buffer)

## Checks
| Check | Result |
| --- | --- |
| cpad_stride_0x134 | PASS |
| pads_base_0xB73458 | PASS |
| pads_in_data | PASS |
| ccontrollerstate_tiles | PASS |
| cpad_tiles | PASS |
| cpad_tail_after_buffers | PASS |
| directinput_backend | PASS |
| mouse_invert_flags | PASS |
