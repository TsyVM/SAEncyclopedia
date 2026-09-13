# Streaming Store Layouts
*Source: `streaming_stores.json`*
- **$schema:** streaming_stores.v1
- **Generated:** tools/derive_streaming_stores.py

## Summary
| Field | Value |
| --- | --- |
| Colstore slot | 44 |
| Iplstore slot | 52 |
| Streamedscripts slots | 82 |
| Facts checked | 9 |
| Facts passed | 9 |

## Stores
| Key | Slot stride | Pool global | Status byte off | Count | Slots | Records base off | Count word off | Flag byte off |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| CColStore | 44 | 9852256 | 40 | dynamic (CPool) |  |  |  |  |
| CIplStore | 52 | 9322416 |  | dynamic (CPool) |  |  |  |  |
| CStreamedScripts | 32 |  |  |  | 82 | 8 | 2628 | 4 |

## Structural facts
| Entry va | Method | Fact | Verified |
| --- | --- | --- | --- |
| 0x00410730 | CColStore::RemoveCol | CColStore slot = 0x2C (44 B); CPool object @ 0x965560, array@+0 / flag-bytes@+4 | ✅ true |
| 0x00411330 | CColStore::RemoveColSlot | RemoveColSlot: same 0x2C stride; per-slot status byte at slot+0x28 | ✅ true |
| 0x00410820 | CColStore::IncludeModelIndex | IncludeModelIndex indexes the same 0x2C slot | ✅ true |
| 0x00405850 | CIplStore::RequestIplAndIgnore | CIplStore slot = 0x34 (52 B); CPool object @ 0x8E3FB0, array@+0 / flag-bytes@+4 | ✅ true |
| 0x00405B60 | CIplStore::RemoveIplSlot | RemoveIplSlot: same 0x34 stride | ✅ true |
| 0x00404C90 | CIplStore::IncludeEntity | IncludeEntity indexes the same 0x34 slot | ✅ true |
| 0x004706A0 | CStreamedScripts::ReInitialise | streamed-script table = 0x52 (82) slots x 0x20 (32 B); flag byte at slot+4 cleared | ✅ true |
| 0x004706C0 | CStreamedScripts::RegisterScript | RegisterScript: slot = index*32 at this+8; live count word at this+0xA44 | ✅ true |
| 0x004708E0 | CStreamedScripts::RemoveStreamedScriptFromMemory | RemoveStreamedScriptFromMemory: same 32-byte slot (shl 5) | ✅ true |
