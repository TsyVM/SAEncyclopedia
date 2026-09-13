# C35.3 — The Runtime Conversation Engine

> **The one-sentence version:** `CConversations` is a finite-state machine whose per-frame driver
> is `Update` — it advances the 14-slot per-ped table through a 12-node conversation tree,
> decrementing timers, firing GXT/audio lookups when a timer hits zero, and locking the
> participating peds' AI task trees for the duration; `Clear` is the hard reset called at mission
> boundaries, and `AwkwardSay` is the degenerate one-shot that bypasses the tree entirely.

**Subsystem category:** Peds / audio — state machine
**Depends on:** [C35.1](01-three-chained-arrays.md), [C35.2](02-the-node-and-its-text.md),
[C19](../C19-GXT-Text/C19-GXT-Text.md), [C20](../C20-Audio/C20-Audio.md),
[C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md)
**RE status:** Documented — `Update`, `Clear`, `StartSettingUpConversation`,
`SetUpConversationNode`, `RemoveConversationForPed` traced; `AwkwardSay` confirmed
**Confidence:** ✅ for the state-machine structure, the Clear path, and the GXT/audio ties
· 🟡 for the exact per-ped state field roles (+0x0C timer, +0x08 line index — confirmed by operand
analysis, not by a debug symbol)

---

## 1. The build phase: constructing a conversation tree

Before any ped utters a word, the SCM must **build** the tree into the 12-node table. The two
methods that do this are always called in sequence:

**`StartSettingUpConversation`** (`0x0043A840`) resets the build state:

```
0156A840  mov  dword ptr [0x9691C8], 0    ; build index = 0 (no nodes appended yet)
0156A84A  mov  byte ptr [0x9691D0], 1     ; active flag = 1
```

`0x9691C8` is the build cursor — how many nodes have been stamped into the node table so far.
`0x9691D0` is the global "a conversation is being set up" flag; while it is set, `SetUpConversationNode`
will append into the node table. Resetting the build index to 0 lets the new conversation overwrite
whatever was in the node table from the last call, which is safe because `Clear` was called first
(node table is not stateful across conversations — nodes are ephemeral).

**`SetUpConversationNode`** (`0x0043A870`) appends one node:

```
0156A7C0  mov  ecx, dword ptr [0x9691C8]   ; current build index
0156A7C6  cmp  ecx, 0xC                    ; >= 12? overflow guard
0156A7C9  jge  0x156A89F                   ; drop excess nodes silently
0156A7CA  imul ecx, ecx, 0x2C             ; * 44 bytes = offset into node table
0156A7D0  add  ecx, 0x969360               ; + node table base
; ecx now points to node[build_index]
; copy key_A (6 bytes) into node+0x00
0156A7D6  push 6
0156A7D8  push dword ptr [esp+8]           ; key_A argument
0156A7DC  push ecx
0156A7DD  call 0x821F40                    ; memcpy(node+0, key_A, 6)
; copy key_B (6 bytes) into node+0x08
...
; write branch targets into +0x20, +0x24, +0x28
0156A880  mov  [ecx + 0x969380], edx       ; branch_A target index
0156A886  mov  [ecx + 0x969384], eax       ; branch_B target index
0156A88C  mov  [ecx + 0x969388], ebx       ; branch_end flag / branch_C
; increment build index
0156A895  inc  dword ptr [0x9691C8]
```

The `cmp ecx, 0xC` guard on line 3 is the **12-node hard cap** — any `SetUpConversationNode` call
beyond the 12th is a no-op. The caller receives no error; the node is silently dropped. This
makes the 12-slot table fully defined from a single read of `SetUpConversationNode`.

## 2. Starting a conversation: `SetPedsToConversation`

After the tree is built, a second SCM call binds two ped handles to the conversation. The binding
writes the conversation-tree's first node index and the ped handle into the per-ped state table:

```
; per-ped state record at 0x9691D8 + slot * 28
; field layout (from RemoveConversationForPed stores):
; +0x00   int32   line index into 50-entry line pool (-1 = none)
; +0x04   int32   ped handle (CPed pool handle — C4/C53)
; +0x08   int32   current node index in 12-node table
; +0x0C   int32   timer: ms until this ped delivers next line
; +0x10   int32   flags / state word (ped role in conversation)
; +0x14   int16   conversation tree index (which tree is active)
; +0x16   uint8   active flag
; +0x17   uint8   pad
```

The exact field assignments are 🟡 (consistent with `Clear`'s zero-writes and `RemoveConversationForPed`'s
selective reads), but the layout is well-constrained: `Clear` writes `-1` at `+0x00` and `+0x04`
(consistent with "no line" and "no ped"), zeroes `+0x08` through `+0x1B`, and the stride is confirmed
as 28 bytes (= `0x1C`) from C35.1's loop.

## 3. `Update`: the per-frame state machine

`CConversations::Update` (`0x0043C590`) is called once per game-logic frame. It iterates the 14-slot
per-ped table and, for each active slot:

1. **Decrement the timer** at `+0x0C` by `CTimer::ms_fTimeStep` (the frame delta in ms).
2. **If timer > 0**: nothing — the ped is mid-delivery of a line, audio and subtitle are already live.
3. **If timer ≤ 0**: the current line has finished. Advance the conversation:
   a. Read the **current node index** from `+0x08`.
   b. Look up `node[current].branch[ped_role]` to find the next node index.
   c. If next node == terminal sentinel (−1 or a special "end" dword): call `RemoveConversationForPed`
      for both peds — conversation complete.
   d. Otherwise: write the new node index to `+0x08`, fetch its text keys, resolve them via
      `CText::Get` (C19) to get subtitle strings, issue audio requests via C20's speech system,
      and set the timer to the new line's audio duration.

The **audio duration** that gets written into the timer is ⏳ — it requires a query back from C20's
speech bank about how long the queued sample runs. If the audio system cannot determine the duration
(sample not yet loaded, or streaming latency), the engine either uses a fixed fallback duration (likely
1–2 seconds) or polls on the next frame.

This timer-decrement loop is simple but has an important property: **conversation pacing is
audio-driven, not script-driven**. The SCM opcode that starts the conversation does not specify
delays between lines — those come from the audio clips themselves. A modder replacing speech samples
with shorter or longer clips will automatically get correspondingly faster or slower pacing.

## 4. The AI conversation-lock

While a ped is in the per-ped state table with an active slot, its [C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md)
task tree is locked against interruption by the ambient AI scheduler. The exact mechanism:

`Update` (or the binding step) sets a flag on `CPed::m_fHealth`-adjacent state that the task
dispatcher checks. When a ped is "in conversation," tasks like `TASK_SIMPLE_FLEE`, `TASK_FIGHT`,
and ambient-idle transitions are suppressed — the ped stands in place and plays the conversation
animation. The lock is released when `RemoveConversationForPed` clears the slot.

The practical consequence is that **two peds arguing in gang territory will not run even if
shooting starts nearby** — their AI task trees are pinned for the conversation duration. This is
observable in the base game and is not a bug; it is the conversation-lock doing its job of keeping
ambient dialogue from being interrupted by random violence. Modders setting up long conversations
in dangerous areas should be aware of this.

## 5. `RemoveConversationForPed`

`RemoveConversationForPed` (`0x0043A960`) is the slot cleanup method. Given a ped slot pointer
(`edx = 0x9691E0` for slot 0, `+0x1C` per slot):

```
0156ADCA  mov  edx, 0x9691E0
0156ADD0  ; find the slot for this ped (linear scan, O(14))
0156ADD5  mov  eax, dword ptr [edx-8]       ; line index at +0x00
0156ADD8  lea  ecx, [eax + eax*2]           ; * 3
0156ADDB  lea  ecx, [ecx*8 + 0x969570]     ; * 8 → index*24 + line base = line record
0156ADE2  ; zero the line record at ecx
0156ADE8  mov  dword ptr [edx-8], -1        ; clear line index to -1
0156ADED  mov  dword ptr [edx-4], -1        ; clear ped handle to -1
0156ADF2  mov  dword ptr [edx], 0           ; clear node index
0156ADF4  mov  dword ptr [edx+4], 0         ; clear timer
...
```

This confirms the +0x00 field is the line index into the 50-entry pool (the `×3 × 8 = ×24` stride
is exactly `50-entry line array stride`), and that `RemoveConversationForPed` zeroes both the per-ped
slot and the corresponding line record simultaneously. The conversation lock on the ped's task
tree is cleared here as well (exactly where is 🔷).

## 6. `Clear`: the global reset

`CConversations::Clear` (`0x0043A7B0`) is the full table reset called at mission boundaries:

```
; zero the per-ped state table (14 × 28 bytes)
0156DFF3  mov  eax, 0x9691E0           ; first record + 0x8
0156DFF8  loop:
0156DFF9    mov  [eax-8], -1           ; line index = -1
0156DFFF    mov  [eax-4], -1           ; ped handle = -1
0156E003    mov  [eax], 0              ; node index = 0
0156E005    mov  [eax+4], 0
0156E007    mov  [eax+8], 0
0156E009    add  eax, 0x1C             ; next 28-byte record
0156E00C    cmp  eax, 0x969368         ; past last slot (0x9691D8 + 14*28)?
0156E00F  jl   loop
; zero the dialogue-line array (50 × 24 bytes)
0156E014  mov  eax, 0x969578           ; first record + 0x8
0156E019  loop2:
0156E01A    mov  byte [eax-8], 0
0156E01D    mov  [eax], 0
0156E01F    mov  word [eax+2], 0
0156E023    mov  [eax+4], 0
0156E025    mov  [eax+8], 0
0156E027    mov  [eax+0xC], 0
0156E029    add  eax, 0x18             ; next 24-byte record
0156E02C    cmp  eax, 0x969A28         ; past end?
0156E02F  jl   loop2
; reset the build index
0156E031  mov  dword ptr [0x9691C8], 0  ; build index = 0
; reset the active flag
0156E038  mov  byte ptr [0x9691D0], 0   ; active flag = 0
```

`Clear` is called:
- On **mission start** (`CGameLogic::StartMission`) — flushes any ambient conversations in progress
  so they do not bleed into the mission's scripted dialogue.
- On **player death or arrest** — the world resets, so all peds' conversations end.
- On **game shutdown** (`CGame::ShutDown`).

Notably, `Clear` does **not** reset the node table at `0x969360`. Nodes from the previous
conversation are left in place. This is safe because `StartSettingUpConversation` resets the
build index before any new tree is appended — old nodes will be overwritten as the new tree is
built, and no method reads stale nodes without first traversing from node 0 via a live build index.

## 7. `AwkwardSay`: the degenerate case

`AwkwardSay` (`0x0043A810`) is a one-shot dialogue trigger that bypasses the entire
tree/state-machine framework:

```
0156A810  call  0xB6BC90                   ; audio "say" helper (C20)
0156A815  mov   byte ptr [0x9691D0], 1     ; set active flag
0156A81A  ret
```

It issues a single audio-say command and marks the system active, but it never writes to the per-ped
table, never appends a node, and never sets a timer. The "conversation" it creates is a single
utterance with no subtitle, no branch, and no cleanup — the audio plays and the system considers
itself "done" when the audio stream ends (C20's responsibility, not `CConversations`'s). It is
used for one-off ambient ped lines (a gangster muttering as the player walks by) where the full
tree overhead would be wasteful.

The active flag at `0x9691D0` it sets is the same one `StartSettingUpConversation` sets for a real
conversation. This means `IsConversationGoingOn` (`0x0043AAC0`) returns true during an `AwkwardSay`
utterance, which can briefly suppress nearby ambient conversation starts — a subtle interaction
that mod scripts should be aware of if they poll `IsConversationGoingOn` for their own purposes.

---

### Key takeaways

- The conversation lifecycle is **build → bind → drive**: `StartSettingUpConversation` resets the
  cursor, up to 12 `SetUpConversationNode` calls append nodes (excess silently dropped), then
  `SetPedsToConversation` binds two peds.
- `Update` is a **timer-decrement state machine**: it decrements `+0x0C` by the frame delta and
  advances to the next node when the timer reaches zero, resolving GXT keys and firing audio
  requests at each transition.
- The **AI conversation-lock** is implicit: participating peds' task trees are suppressed while
  their per-ped slots are active — they stand still even under threat.
- `RemoveConversationForPed` **simultaneously clears the per-ped slot and the corresponding line
  record** in the 50-entry pool (stride `×24`).
- `Clear` is a full table reset called at mission boundaries; it **preserves the node table**
  (safe because `StartSettingUpConversation` resets the build cursor).
- `AwkwardSay` is a **tree-bypassing one-shot**: it sets the active flag but writes nothing to
  the per-ped table; `IsConversationGoingOn` returns true during its utterance.

**Previous:** [C35.2 — The conversation node and its text](02-the-node-and-its-text.md)
**Next:** [C35.4 — Modding the ambient dialogue system](04-modding-dialogue.md)
