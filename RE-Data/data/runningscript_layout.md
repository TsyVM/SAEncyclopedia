# CRunningScript Object Layout
*Source: `runningscript_layout.json`*
- **$schema:** runningscript_layout.v1
- **Generated:** tools/derive_runningscript.py

## Object size
224

## Summary
| Field | Value |
| --- | --- |
| Facts checked | 7 |
| Facts passed | 7 |
| Local var dwords | 34 |
| Return stack dwords | 8 |
| Mission lvar block | 10783072 |
| Global var space | 10787168 |

## Fields
| Offset | Size | Name |
| --- | --- | --- |
| 0 | 4 | next (intrusive list) |
| 4 | 4 | prev (intrusive list) |
| 8 | 8 | name[8] |
| 16 | 4 | base pointer (script code base) |
| 20 | 4 | program counter (current IP) |
| 24 | 32 | return/gosub stack (8 dwords) |
| 56 | 4 | stack depth + pad |
| 60 | 136 | local variables (34 dwords = 32 locals + 2 timers) |
| 196 | 28 | condition/state flag bytes (+0xC4..+0xC9=0xFF, +0xCC, +0xD0..+0xD4, +0xD8) |
| 220 | 4 | +0xDC mission-LVAR-mode flag (within the flag block above; last dword) |

## Structural facts
| Entry va | Method | Fact | Verified |
| --- | --- | --- | --- |
| 0x004648E0 | CRunningScript::Init | Init lays out the object: return/gosub stack = 8 dwords @+0x18; local-var array = 0x22 (34) dwords @+0x3C; init flags +0xC9=0xFF, +0xD3=1 | ✅ true |
| 0x00464DA0 | CRunningScript::UpdatePC | UpdatePC: base pointer @+0x10, program counter (PC) @+0x14 | ✅ true |
| 0x00463CA0 | CRunningScript::GetPointerToLocalVariable | GetPointerToLocalVariable: +0xDC flag redirects locals to the shared mission-LVAR block at 0xA48960 (C28.2's 4 KB block) | ✅ true |
| 0x00463CF0 | CRunningScript::ReadArrayInformation | ReadArrayInformation: own locals @this+0x3C; global array space @0xA49960 | ✅ true |
| 0x00464700 | CRunningScript::GetIndexOfGlobalVariable | GetIndexOfGlobalVariable: PC @+0x14 walked; global var space @0xA49960 | ✅ true |
| 0x00464C00 | CRunningScript::AddScriptToList | AddScriptToList: intrusive list links next@+0, prev@+4 | ✅ true |
| 0x00464BD0 | CRunningScript::RemoveScriptFromList | RemoveScriptFromList: same next@+0 / prev@+4 links | ✅ true |
