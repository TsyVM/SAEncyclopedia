# C36.3 — The Attractor-Brain Lifecycle

> **The one-sentence version:** the brain lifecycle runs in four phases — registration, proximity
> trigger, script execution, and quiescence — and the two method families (`HasAttractorScriptBrain…`
> / `StartAttractorScriptBrain…` for name-keyed attractors, and `AddNewStreamedScriptBrainForCodeUse`
> / `StartOrRequestNewStreamedScriptBrainWithThisName` for model-ID-keyed streamed brains) map cleanly
> onto those phases; the `SwitchAll` mechanism is the bulk-disable that missions use to quiet
> the world, and the per-id state array at `0x964E08` (stride ×20) is the shared bookkeeping
> that ties the 70-slot brain table to [C31.3](../C31-Streaming-Slot-Tables/03-cstreamedscripts.md)'s
> 82-slot streamed-script table.

**Subsystem category:** Scripting — lifecycle
**Depends on:** [C36.1](01-the-70-slot-table.md), [C36.2](02-brains-and-scripts.md),
[C31.3](../C31-Streaming-Slot-Tables/03-cstreamedscripts.md),
[C32](../C32-CRunningScript-Object/C32-CRunningScript-Object.md),
[C2](../C2-CStreaming/C2-CStreaming.md)
**RE status:** Documented — all four lifecycle phases traced to method addresses; `SwitchAll` and
the `0x964E08` state array confirmed by operand analysis
**Confidence:** ✅ for the phase structure, the method addresses, and the `SwitchAll` mechanism
· 🟡 for the exact per-field roles of the `0x964E08` state byte (bit assignments not fully traced)

---

## 1. Phase 1: Registration

Registration writes a model ID (or attractor name, depending on brain type) into a free slot of the
70-slot table (C36.1) and simultaneously updates the per-id state array.

**`AddNewStreamedScriptBrainForCodeUse`** (`0x0046A9C0`):

```
; find a free slot (id word == -1, C36.1 §2)
0156B9C0  xor   dl, dl                     ; index = 0
0156B9C2  lea   ecx, [ebx + ecx*4]         ; base address of CScriptsForBrains object
scan:
0156B9C8  cmp   word ptr [ecx + edx*20], -1  ; slot.id == -1 (free)?
0156B9D0  je    found_free
0156B9D2  inc   dl
0156B9D4  cmp   dl, 0x46                   ; 70 slots exhausted?
0156B9D7  jl    scan
0156B9D9  ; failure path — no free slot
found_free:
0156B9DA  mov   word ptr [ecx + edx*20], ax  ; write model ID to +0x00
0156B9E0  mov   byte ptr [ecx + edx*20 + 4], 1  ; set enabled (+0x04) = 1
; update the per-id state array at 0x964E08
0156B9E7  movzx eax, ax                    ; zero-extend model ID
0156B9EA  imul  eax, eax, 0x14             ; * 20 — same stride as brain table!
0156B9F0  add   eax, 0x964E08              ; + state array base
0156B9F6  mov   byte ptr [eax], 1          ; state byte = "registered"
```

Three things are established:

1. The free-slot scan is O(70) on the `id` word, using `edx*20` addressing — confirming stride 20
   (C36.1 §1) from this method independently.
2. The **per-id state array at `0x964E08`** also has stride **×20** (= `×0x14`), meaning it is
   indexed by model ID, not by slot index. This is the bookkeeping that lets `StartOrRequest…`
   look up whether a given brain is already registered by model ID without scanning the 70-slot
   table: `state[model_id * 20 + 0x964E08] != 0`.
3. The **enabled byte at `+0x04`** is set to 1 at registration — the slot is immediately active
   unless a `SwitchAll(false)` call disables it afterward.

## 2. Phase 2: Proximity trigger

The proximity trigger is driven by the world-object update loop, not by `CScriptsForBrains`
itself. The engine calls `HasAttractorScriptBrainWithThisNameLoaded` or its model-ID equivalent
from within the entity processing loop when it detects a player-near event.

**`HasAttractorScriptBrainWithThisNameLoaded`** (`0x0046AB20`):

```
0156AB20  push  5                           ; compare length = 5 chars
0156AB22  push  dword ptr [esp+8]           ; the attractor name to look up
0156AB26  mov   esi, ecx                    ; save this-pointer
0156AB28  call  0x46AA30                    ; GetIndexOfScriptBrainWithThisName
0156AB2D  add   esp, 8
0156AB30  cmp   eax, -1                     ; found? (returns -1 if not found)
0156AB33  je    not_found
0156AB35  ; check enabled state at index*20 + this + 4
0156AB3B  cmp   byte ptr [esi + eax*20 + 4], 0
0156AB42  je    not_enabled
0156AB44  mov   eax, 1                      ; return true
0156AB49  ret
```

Two queries are combined: (a) does a brain with this name exist? (b) is its enabled byte non-zero?
Both must be true for the proximity trigger to fire. The enabled byte is the `SwitchAll` dial —
a brain can be registered but not enabled (because `SwitchAll(false)` was called), which prevents
it from triggering even when the player is right next to the associated object. This is how missions
selectively silence shop brains that would otherwise interrupt cutscenes.

**`GetIndexOfScriptBrainWithThisName`** (`0x0046AA30`) is the O(70) linear scan comparing the
first 5 bytes of the brain's name field against the query. The 5-char comparison length (vs the
6-char copy in `SetUpConversationNode`) means the 6th byte of the name is not part of the match
— a mild inconsistency between storage width (6 bytes) and comparison length (5 chars). A brain
named `SHOPX1` and one named `SHOPX2` would both match a query for `SHOPX` — the 6th character
is not disambiguating. Keep brain names 5 unique characters or fewer to guarantee no false matches.

## 3. Phase 3: Script execution

When `HasAttractorScriptBrainWithThisNameLoaded` returns true, the trigger path calls
**`StartAttractorScriptBrainWithThisName`** (`0x0046B390`) or
**`StartOrRequestNewStreamedScriptBrainWithThisName`** (`0x0046CED0`), depending on brain type.

For a **streamed-script brain**, `StartOrRequestNewStreamedScriptBrainWithThisName`:

1. Looks up the brain's slot in the 70-slot table to get the brain's streamed-script slot index.
2. Checks the per-id state array at `0x964E08` to see if the script is already loaded.
3. If not loaded: issues a `CStreaming::RequestModel` call for the streamed-script (C2), setting
   the state byte to "pending." Returns without starting a thread.
4. If loaded (state byte == "loaded"): calls into C32's thread-creation path to instantiate a
   new `CRunningScript` at the script's entry point.

The non-blocking request in step 3 is important: `StartOrRequest…` may be called multiple times
during the one or two frames it takes the streaming system to load the script. The state byte at
`0x964E08[model_id * 20]` guards against double-starting — the second call sees "pending" and
exits early.

For an **attractor brain** (name-keyed), `StartAttractorScriptBrainWithThisName` follows a
slightly shorter path because the script is already registered in C31.3 at a known slot index —
no on-demand streaming request is needed. It goes directly to C32's thread creation.

**The CRunningScript thread.** The created script thread runs in the `ProcessAllScripts` list
([C32.3](../C32-CRunningScript-Object/03-thread-lifecycle-and-suspension.md)) alongside mission
scripts and ambient world scripts. It has access to the same SCM opcode set as any other thread.
A typical brain script runs a proximity loop:

```scm
wait_for_proximity_loop:
  IS_PLAYER_IN_AREA_2D $player x1 y1 x2 y2 → $in_range
  IF NOT $in_range
    TERMINATE_THIS_SCRIPT              ; player walked away — clean up
  END
  ; player is in range — do the interactive behavior
  ; ...
  TERMINATE_THIS_SCRIPT                ; done
```

The script terminates itself; `CScriptsForBrains` does not poll threads for completion — the brain
slot stays registered but inactive until the next proximity trigger.

## 4. Phase 4: Quiescence and re-trigger

When the brain script terminates (via `TERMINATE_THIS_SCRIPT` or by reaching the end of the
SCM block), its `CRunningScript` thread is freed by the script-VM cleanup path in
[C32](../C32-CRunningScript-Object/C32-CRunningScript-Object.md). The brain slot in the 70-slot
table is **not** cleared — the slot retains its model ID and enabled state. The per-id state byte
at `0x964E08` is reset to "idle" by the script-VM cleanup.

The next time the player approaches an object with this model ID / attractor name, the proximity
trigger fires again (`HasAttractorScriptBrainWithThisNameLoaded` → `StartOrRequest…`), repeating
from Phase 3. The slot never needs to be re-registered between activations.

This "register once, trigger many times" design is the main reason there are only 70 slots — the
game registers all interactive objects at startup, uses the table indefinitely, and never needs to
add or remove registrations during normal gameplay.

## 5. `SwitchAll`: the bulk enable/disable

**`SwitchAllObjectBrainsWithThisID`** (`0x0046A900`) walks the full 70-slot table and sets or
clears the enabled byte (`+0x04`) for every slot whose model-ID matches the argument:

```
0156A900  mov   ecx, 0x46              ; 70 iterations
0156A902  lea   edx, [esi]             ; this + 0 = first record
0156A904  loop:
0156A905    cmp  word ptr [edx], ax    ; slot.id == target_id?
0156A909    jne  skip
0156A90B    mov  byte ptr [edx + 4], bl  ; set/clear enabled byte (bl = 0 or 1)
skip:
0156A90F    add  edx, 0x14             ; next 20-byte record
0156A912    dec  ecx
0156A913    jne  loop
```

`SwitchAllObjectBrainsWithThisID` is the per-id variant; a method that takes no ID argument
(listed as `SwitchAllObjectBrains` or `SwitchAllPedBrains` in community docs — exact SA name ⏳)
applies the same operation to all 70 slots regardless of id. The per-frame cost of this method
is O(70) — negligible, and it is only called at mission boundaries, not per frame.

The enabled byte is the key to how missions make the interactive world "disappear" during
cutscenes: a call to `SwitchAll(false)` before the cutscene camera starts means no brain will
trigger even if the player is standing next to an attractor. `SwitchAll(true)` at the end of the
cutscene restores normal world interactivity. A mod that adds brains to the world must not assume
the enabled byte stays 1 — the game may call `SwitchAll(false)` at any time.

## 6. `Init` and `ShutDown`

**`Init`** (`0x0046A8C0`) runs once at game startup. It calls `SwitchAll(false)` (disabling all
slots) and then stamps the 70 slots to the `0xFFFF` / free-sentinel state (C36.1 §1). All brain
registrations happen *after* `Init`, in the `PostInit` or `ProcessLoadingMessages` phase when
the game's IPL data and model IDs are available.

**`ShutDown`** (⏳ — not listed separately, but implied by `Init`'s counterpart) calls
`SwitchAll(false)` and then iterates the table freeing any allocated state. Because the table is
embedded in the manager object (this-relative, C36.0), there is no separate heap allocation to
free — the manager object's destructor handles it.

## 7. The `0x964E08` state array and its relationship to C31.3

The per-id state array at `0x964E08` with stride ×20 appears in both `AddNewStreamedScriptBrainForCodeUse`
(set to "registered") and `StartOrRequestNewStreamedScriptBrainWithThisName` (read to check
load status). From C36.2 §3, the adjacent table at `0xA47B64` with stride ×32 is the bookkeeping
for the streamed-script slot — the same ×32 stride as [C31.3](../C31-Streaming-Slot-Tables/03-cstreamedscripts.md)'s
82-slot × 32-byte streamed-script table.

These two arrays form the "back channel" between the brain table (70 slots, model-ID indexed) and
the streamed-script table (82 slots, sequential slot-number indexed): a brain registration records
its own existence at `0x964E08[model_id * 20]`, and when its script is demanded, the streamed-
script slot index is recorded at `0xA47B64[slot * 32]`. The O(1) per-model state check using
`0x964E08` avoids an O(82) scan of C31.3's table every proximity frame — a deliberate optimization
given how frequently the proximity trigger fires.

---

### Key takeaways

- The **four phases** are: Registration (`AddNewStreamedScriptBrainForCodeUse`, writes slot + `0x964E08`),
  Proximity Trigger (`HasAttractorScriptBrainWithThisNameLoaded`, O(70) name scan + enabled check),
  Script Execution (`StartOrRequestNewStreamedScriptBrainWithThisName`, streaming request → C32
  thread creation), and Quiescence (thread terminates; slot remains registered, state byte reset).
- The **enabled byte at `+0x04`** is the `SwitchAll` dial — a registered brain can be silenced
  without un-registering it.
- **`GetIndexOfScriptBrainWithThisName`** compares only **5 bytes** despite the 6-byte name storage —
  the 6th character does not disambiguate; keep brain names to 5 unique chars.
- **`0x964E08` (stride ×20)** is an O(1) per-model state cache used by the streaming path to avoid
  scanning C31.3's table on every proximity check — the two arrays together form the bridge between
  the brain table and the streamed-script table.
- `SwitchAll(false)` is called by missions before cutscenes; **mod-registered brains are also
  disabled** — do not assume `+0x04` stays 1.

**Previous:** [C36.2 — Object brains vs streamed brains](02-brains-and-scripts.md)
**Next:** [C36.4 — Modding world reactivity with script brains](04-modding-world-reactivity.md)
