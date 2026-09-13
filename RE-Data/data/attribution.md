# Class Attribution for Unidentified Stubs
*Source: `attribution.json`*
- **$schema:** attribution.v1
- **Generated:** tools/derive_attribution.py
- **Note:** Class attributions (NOT method names) for C27's unnamed functions, by data-flow reference to C28-C32's recovered class statics.

## Summary
| Field | Value |
| --- | --- |
| Unnamed total | 124 |
| With signature ref | 15 |
| Strong attributions | 11 |
| Data only leads | 4 |

## Attributions
| Entry va | Body extent | Direct call sites | Attributed class | Hits | In cluster | Tier |
| --- | --- | --- | --- | --- | --- | --- |
| 0x00407800 | 32 | 1 | CStreaming | 1 | ✅ true | STRONG |
| 0x00410F80 | 96 | 1 | CColStore | 1 | ✅ true | STRONG |
| 0x00411030 | 112 | 0 | CColStore | 1 | ✅ true | STRONG |
| 0x0043E400 | 16 | 1 | CEntryExitManager | 1 | ✅ true | STRONG |
| 0x0043EF00 | 32 | 3 | CEntryExitManager | 1 | ✅ true | STRONG |
| 0x0043EF20 | 112 | 1 | CEntryExitManager | 2 | ✅ true | STRONG |
| 0x0043EF90 | 64 | 1 | CEntryExitManager | 2 | ✅ true | STRONG |
| 0x0043F720 | 96 | 0 | CEntryExitManager | 1 | ✅ true | STRONG |
| 0x0043F7D0 | 112 | 1 | CEntryExitManager | 1 | ✅ true | STRONG |
| 0x00448990 | 96 | 1 | CGarages | 2 | ✅ true | STRONG |
| 0x0044D520 | 608 | 1 | CPathFind | 17 | ✅ true | STRONG |
| 0x00422590 | 144 | 4 | CPathFind | 4 | ❌ false | data-only |
| 0x0042F870 | 352 | 16 | CPathFind | 1 | ❌ false | data-only |
| 0x0044CC20 | 304 | 1 | CTheScripts | 3 | ❌ false | data-only |
| 0x0046AA80 | 112 | 1 | CStreaming | 1 | ❌ false | data-only |
