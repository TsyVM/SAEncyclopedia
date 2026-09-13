# C35.2 — The Conversation Node and Its Text

> **The one-sentence version:** a 44-byte conversation node holds several fixed **6-byte text keys** copied
> in by `SetUpConversationNode`, plus three dword branch fields — it stores *references*, not words, so the
> displayed text comes from [C19](../C19-GXT-Text/C19-GXT-Text.md)'s GXT and the spoken line from
> [C20](../C20-Audio/C20-Audio.md)'s speech banks, with `CConversations` as the state machine that sequences
> both.

**Subsystem category:** Peds / audio — node content
**Depends on:** [C35.1](01-three-chained-arrays.md), [C19](../C19-GXT-Text/C19-GXT-Text.md),
[C20](../C20-Audio/C20-Audio.md)
**RE status:** Documented
**Confidence:** ✅ for the key-copy length and the branch dwords · 🟡 for the exact per-field roles

---

## 1. What SetUpConversationNode writes

`SetUpConversationNode` (`entry_va 0x0043A870`) builds one node. Reading its stores gives the node's shape:

```
imul ecx, ecx, 0x2C ; add ecx, 0x969360        ; node = index*44 + base
push 6 ; push <src> ; push ecx ; call 0x821F40   ; copy 6 bytes into node+0  (text key)
...
mov [eax+0x969380], ecx                          ; node+0x20  branch dword A
mov [eax+0x969384], edx                          ; node+0x24  branch dword B
mov [eax+0x969388], ecx                          ; node+0x28  branch dword C
...
lea edx, [eax + 0x969368] ; push 6 ; call copy    ; copy 6 bytes into node+8  (second text key)
...
add ecx, 0x969370 ; ... copy                       ; node+0x10 (third text key)
```

So a node carries at least **three fixed-length 6-byte string fields** (at node +0, +8, +0x10) and **three
dword fields** (at +0x20/+0x24/+0x28), inside the 44 bytes. The `push 6` before each copy (`call 0x821F40`,
a `memcpy`/`strncpy`-shaped helper) fixes the key width at 6 bytes — short fixed keys, not inline text. The
three dwords are the node's branch/link data (which node or line follows on each answer). The remaining bytes
(44 − 3×6 − 3×4 = 14) hold flags and the small scalars `Clear` zeroes.

## 2. Keys, not words — the C19 tie

A 6-byte key is not dialogue; it is a lookup token. San Andreas resolves displayed text through the GXT
([C19](../C19-GXT-Text/C19-GXT-Text.md)), whose keys hash to table entries, and a conversation node's text
fields are exactly such keys: the state machine holds the *reference*, and the words are fetched from the GXT
at display time. This is the same indirection the whole engine uses — [C22](../C22-Map-Zones/C22-Map-Zones.md)'s
zone labels are GXT keys, [C18](../C18-SCM-Script/C18-SCM-Script.md)'s `print` opcodes take GXT keys — and
it is why a conversation node is only 44 bytes: it stores tokens, and C19 stores the language-specific text
behind them.

The spoken audio follows the same pattern one subsystem over: the line the ped actually *says* is a sample
in [C20](../C20-Audio/C20-Audio.md)'s speech banks, selected by the conversation state. `CConversations`
therefore sits at the junction of three already-decoded subsystems — it owns no text and no audio of its own,
only the tree that decides *which* GXT key and *which* speech line come next.

## 3. The query and update methods

The rest of the class (🔷) reads and advances that tree: `IsConversationGoingOn` (`0x0043AAC0`) tests the
active flag ([C35.1 §4](01-three-chained-arrays.md#4-the-build-index-and-the-reset)),
`IsConversationAtNode` (`0x0043B000`) — which [C27.3 §3](../C27-Function-Catalogue/03-verification-and-the-remaining-124.md#3)
disassembled as "a linear scan of a fixed-stride node table," i.e. exactly the 44-byte node table recovered
here — checks the current node, `IsPlayerInPositionForConversation` (`0x0043B0B0`) gates player dialogue on
proximity, and `Update` (`0x0043C590`) is the per-frame step that walks the tree and fires the next key/line.
`AwkwardSay` (`0x0043A810`) is a direct one-off line (it calls into an audio-say helper at `0xB6BC90` and
sets the active flag) — the degenerate "conversation" of a single utterance. Full list in
[`conversations.json`](../RE-Data/data/conversations.json).

---

### Key takeaways

- A 44-byte node holds **three 6-byte text-key fields** (+0, +8, +0x10) and **three branch dwords**
  (+0x20/+0x24/+0x28); the `push 6` before each copy fixes the key width.
- Nodes store **keys, not words** — displayed text resolves through [C19](../C19-GXT-Text/C19-GXT-Text.md)'s
  GXT, spoken audio through [C20](../C20-Audio/C20-Audio.md)'s speech banks; `CConversations` only sequences
  the references.
- `IsConversationAtNode`, the method [C27.3](../C27-Function-Catalogue/03-verification-and-the-remaining-124.md)
  spot-checked as "a linear scan of a fixed-stride node table," is confirmed here to scan exactly this
  44-byte node table.

**Next:** [C35.3 — The runtime conversation engine](03-the-runtime-conversation-engine.md).
