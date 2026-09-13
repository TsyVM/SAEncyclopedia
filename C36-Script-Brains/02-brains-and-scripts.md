# C36.2 — Object Brains vs Streamed Brains

> **The one-sentence version:** the class manages brains along two axes — attached to an object by *name*
> (looked up with a 6-char key) versus loaded on demand as a *streamed script* — and its methods sit exactly
> between [C31.3](../C31-Streaming-Slot-Tables/03-cstreamedscripts.md)'s streamed-script table (a brain's
> code) and [C32](../C32-CRunningScript-Object/C32-CRunningScript-Object.md)'s running-script object (what a
> brain becomes when it fires).

**Subsystem category:** Scripting — brain types
**Depends on:** [C36.1](01-the-70-slot-table.md), [C31.3](../C31-Streaming-Slot-Tables/03-cstreamedscripts.md),
[C32](../C32-CRunningScript-Object/C32-CRunningScript-Object.md)
**RE status:** Documented
**Confidence:** ✅ for the name-key lookup and the streamed-script tie · 🔷 for methods not disassembled

---

## 1. Lookup by name

`CScriptsForBrains::GetIndexOfScriptBrainWithThisName` (`entry_va 0x0046AA30`) is the table's key accessor,
and the methods that call it pass a **length of 5** for the name compare:

```
; caller (HasAttractorScriptBrainWithThisNameLoaded, 0x0046AB20)
push 5                                   ; compare 5 chars
push <name>
mov  esi, ecx
call 0x46AA30                            ; GetIndexOfScriptBrainWithThisName
```

So a brain is identified by a short **fixed 5-char name** — the same kind of short text key
[C35](../C35-Conversations/02-the-node-and-its-text.md)'s conversation nodes and the streamed-script table
use, and consistent with the 8-byte name fields elsewhere in the script system. The returned index addresses
the 20-byte record from [C36.1](01-the-70-slot-table.md).

## 2. The two brain flavours

The method names split the system cleanly:

- **Attractor brains** — `HasAttractorScriptBrainWithThisNameLoaded` (`0x0046AB20`) and
  `StartAttractorScriptBrainWithThisName` (`0x0046B390`): a brain attached to a *place or object* that
  activates on player proximity. `SwitchAllObjectBrainsWithThisID` (`0x0046A900`) flips every slot matching a
  given id on or off — the bulk enable/disable a mission uses to turn a set of attractors on for a stretch of
  the story.
- **Streamed-script brains** — `AddNewStreamedScriptBrainForCodeUse` (`0x0046A9C0`) and
  `StartOrRequestNewStreamedScriptBrainWithThisName` (`0x0046CED0`): a brain whose code must be *loaded* first,
  so the method requests the streamed script (via the streamer) and registers a slot for it.

## 3. The tie to the streamed-script table and CRunningScript

`StartOrRequestNewStreamedScriptBrainWithThisName` and the `…ForCodeUse` path cross-reference two tables this
encyclopedia has already sized: a per-id state byte array at `0x964E08` (indexed by id × 20) and a
`0xA47B64` table (indexed by id × 32) — the latter is the streamed-script bookkeeping adjacent to
[C31.3](../C31-Streaming-Slot-Tables/03-cstreamedscripts.md)'s 82-slot × 32-byte streamed-script table (same
`×32` stride). In other words, a script brain's *code* is a streamed script (C31.3), and when the brain fires
it starts a [C32](../C32-CRunningScript-Object/C32-CRunningScript-Object.md) `CRunningScript` thread — so
`CScriptsForBrains` is the trigger layer bridging the two: it holds *which* script attaches to *which* model,
asks C31.3's streamer to load the code, and hands off to C32's VM to run it.

> **A note for [C33](../C33-Attributing-The-Unnamed/C33-Attributing-The-Unnamed.md).** The unnamed function
> `0x0046AA80` — which C33 attributed to `CStreaming` *data-only* because it references `0x964E08` — sits
> squarely in this class's `.text` cluster (`0x46A8C0…0x46CED0`) and calls
> `GetIndexOfScriptBrainWithThisName` (`0x46AA30`). It is far more likely a `CScriptsForBrains` method than a
> `CStreaming` one; `0x964E08` is this system's per-id state array, not streaming data. C36 thus *corrects*
> one of C33's four data-only leads — an example of a later structural chapter refining an earlier
> attribution, exactly the growth path [C33.2 §4](../C33-Attributing-The-Unnamed/02-the-eleven-and-what-remains.md#4-why-109-dont-resolve--and-thats-a-result)
> described.

---

### Key takeaways

- Brains are identified by a short **5-char name** (`GetIndexOfScriptBrainWithThisName`), the same short-key
  style as the rest of the script/dialogue system.
- Two flavours: **attractor brains** (proximity-triggered, bulk-toggled by id) and **streamed-script brains**
  (code loaded on demand) — the method names split the class in half.
- `CScriptsForBrains` is the **trigger layer**: it maps model→script, pulls the code from
  [C31.3](../C31-Streaming-Slot-Tables/03-cstreamedscripts.md)'s streamed-script table, and starts a
  [C32](../C32-CRunningScript-Object/C32-CRunningScript-Object.md) `CRunningScript` thread when a brain fires.
- C36 **corrects a C33 data-only lead**: `0x0046AA80` is CScriptsForBrains-adjacent (calls `0x46AA30`), not
  `CStreaming` — a later chapter refining an earlier attribution.

**Next:** [C36.3 — The attractor-brain lifecycle](03-the-brain-lifecycle.md).
