# CConversations Layout
*Source: `conversations.json`*
- **$schema:** conversations.v1
- **Generated:** tools/derive_conversations.py

## Summary
| Field | Value |
| --- | --- |
| Per ped slots | 14 |
| Per ped stride | 28 |
| Node slots | 12 |
| Node stride | 44 |
| Line slots | 50 |
| Line stride | 24 |
| Build index global | 0x9691c8 |
| Facts checked | 4 |
| Facts passed | 4 |

## Arrays
| Name | Base | Slots | Stride | Ends at |
| --- | --- | --- | --- | --- |
| per-ped conversation state | 0x9691d8 | 14 | 28 | 0x969360 |
| conversation-node table | 0x969360 | 12 | 44 | 0x969570 |
| dialogue-line array | 0x969570 | 50 | 24 |  |

## Structural facts
| Entry va | Method | Fact | Verified |
| --- | --- | --- | --- |
| 0x0043A7B0 | CConversations::Clear | Clear walks the per-ped array (stride 0x1C) and the line array (stride 0x18) | ✅ true |
| 0x0043A870 | CConversations::SetUpConversationNode | SetUpConversationNode: node = index*0x2C at 0x969360; build index @0x9691C8; 6-char text-key copies | ✅ true |
| 0x0043A960 | CConversations::RemoveConversationForPed | RemoveConversationForPed: per-ped array @0x9691E0; line record = index*24 @0x969570 | ✅ true |
| 0x0043A840 | CConversations::StartSettingUpConversation | StartSettingUpConversation resets build index @0x9691C8 and sets active flag @0x9691D0 | ✅ true |
