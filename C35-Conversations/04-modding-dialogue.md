# C35.4 — Modding the Ambient Dialogue System

> **The one-sentence version:** the three-layer model (GXT text ↔ `CConversations` state machine ↔
> audio speech) means modders can change dialogue at three independent depths — subtitle text with no
> exe patch, conversation trees with SCM edits, or audio content with archive replacement — but the
> hard limits (14 concurrent peds, 12 nodes per tree, 50 simultaneous lines, 7-char GXT keys) are
> baked into static array geometry and cannot be raised without patching the BSS addresses that
> [C35.1](01-three-chained-arrays.md) proved.

**Subsystem category:** Peds / audio — modding
**Depends on:** [C35.1](01-three-chained-arrays.md) through [C35.3](03-the-runtime-conversation-engine.md),
[C19](../C19-GXT-Text/C19-GXT-Text.md), [C20](../C20-Audio/C20-Audio.md),
[C18](../C18-SCM-Script/C18-SCM-Script.md)
**RE status:** Documented
**Confidence:** ✅ for the limit values and their structural sources · 🟡 for the audio-key mapping
(exact relationship between GXT key strings and audio script identifiers not fully traced)

---

## 1. The three-layer model

`CConversations` owns no text and no audio. It is a **sequencing engine** that holds token references
and hands them off to the two systems that do own content:

```
Layer 1: GXT (C19)
  - Stores the subtitle string for every conversation line
  - Key: a ≤7-char ASCII string stored in the 44-byte node's +0x00 / +0x08 / +0x10 fields
  - Accessed via: CText::Get(key) → char* subtitle string

Layer 2: CConversations (C35)
  - Stores the conversation tree (which keys/lines follow which)
  - Timer: fires the GXT lookup and audio request at the right moment
  - Owns: 12-node tree, 14-ped state table, 50-entry line pool

Layer 3: Audio (C20)
  - Stores the actual speech samples
  - Indexed by: ped speech-set + script audio key (mapping ⏳)
  - Output: played asynchronously; duration fed back to CConversations timer
```

A **subtitle-only mod** needs only Layer 1. A **conversation-structure mod** needs Layer 2 (SCM
changes). A **voice-replacement mod** needs Layer 3 (audio archive replacement). All three are
independent and additive — you can retranslate subtitles without touching audio, or replace audio
without changing the tree structure.

## 2. Changing subtitle text (Layer 1 mod)

Each conversation node holds up to three **6-byte GXT key fields** at offsets `+0x00`, `+0x08`,
`+0x10` within the 44-byte record (C35.2 §1). These keys are looked up in `american.gxt` (or
the active locale GXT) at runtime via `CText::Get`. To change what a ped *says* on screen:

1. Extract `american.gxt` using a GXT tool (OpenIV, GXT editor).
2. Identify the key the conversation uses. The key string is a ≤7-char identifier visible in
   either the SCM that builds the conversation tree (`SetUpConversationNode` arguments) or the
   GXT table itself (search for the dialogue text).
3. Edit the string value in the GXT.
4. Re-pack and replace `american.gxt`.

**The 7-char constraint.** [C19](../C19-GXT-Text/C19-GXT-Text.md)'s GXT hash accepts keys up to
7 characters (8 bytes including the null terminator). The 6-byte copy in `SetUpConversationNode`
(`push 6; call memcpy`) means the node stores exactly 6 bytes of the key, which is consistent:
a 7-char key including its null terminator is 8 bytes, but only the 6 non-null bytes are copied
into the node — the node does not need to null-terminate because it passes the key to `CText::Get`
with a length argument. Any key longer than 7 characters will be silently truncated at the GXT
lookup. Keep mod dialogue keys to 6 or fewer characters for perfect fidelity.

## 3. Adding new ambient conversations (Layer 2 mod)

A mod SCM script can create a conversation between any two peds using the opcode sequence:

```scm
; build a two-line exchange
START_SETTING_UP_CONVERSATION
SET_UP_CONVERSATION_NODE "GREET1" "GREET2" 1 -1    ; node 0: ped A says GREET1, ped B says GREET2
                                                     ; branch to node 1 if "yes", end if "no"
SET_UP_CONVERSATION_NODE "CHAT1"  "CHAT2"  -1 -1   ; node 1: end both branches
; bind peds to the conversation
SET_PEDS_TO_CONVERSATION $ped_a $ped_b
```

`START_SETTING_UP_CONVERSATION` resets the build index (C35.3 §1). Each `SET_UP_CONVERSATION_NODE`
call appends one node with two 6-char text keys and two branch targets (positive integer = jump to
that node index, −1 = end conversation). `SET_PEDS_TO_CONVERSATION` writes both ped handles into
the per-ped state table and starts the `Update` timer loop.

**Branch semantics.** The two branch arguments represent the player-response choices in a *player-
interactive* conversation. For purely ambient two-ped exchanges, both branches are `−1` (or both
point to the next node in sequence) — the system auto-advances. For player-triggered dialogue
(e.g., a shopkeeper asking a yes/no question), one branch handles "yes" and the other "no."

**The 12-node cap in practice.** 12 nodes = up to 12 *lines spoken* by each ped — a 12-exchange
dialogue with 24 total lines. Ambient SA conversations are typically 2–4 nodes. A mission with a
longer NPC briefing (say, 8 nodes) should plan for the 12-node ceiling early.

**Simultaneous conversations.** At most 7 two-ped conversations can run concurrently (14 slots ÷ 2
peds per conversation). In a dense scripted scene with many NPCs speaking, the 8th conversation
attempt silently fails — `SetPedsToConversation` returns without writing to a slot. Mod scripts
in crowd scenes should serialize dialogue or check `IS_CONVERSATION_AT_NODE` to verify a
conversation started.

## 4. The timing loop and audio synchronization

As established in [C35.3 §3](03-the-runtime-conversation-engine.md), the timer that paces line
delivery is driven by audio clip duration — not by a hard-coded delay in the SCM. This means:

- **Replacing an audio sample with a longer version** → the subtitle remains on screen longer
  (the conversation engine waits for the audio to end before advancing the tree). This is the
  correct behavior.
- **Replacing an audio sample with a shorter version** → the subtitle disappears sooner.
- **Using a GXT key that has no corresponding audio** → the audio system plays nothing; the
  conversation engine likely falls back to a minimum display time (⏳ — the fallback duration
  is not confirmed). The subtitle will flash briefly and the conversation may advance faster
  than expected.

For best results, keep the audio clip and subtitle text synchronized: write subtitles whose
reading time matches the audio duration.

## 5. Adding new speech samples (Layer 3 mod)

New speech samples must be placed in the correct location in the audio archive and assigned the
correct audio script identifier. The details belong to [C20](../C20-Audio/C20-Audio.md)'s scope,
but the interface that `CConversations` uses is:

- The speech system is invoked by the 6-byte GXT key and the ped's speech set identifier.
- SA assigns each ped model a speech set (a number in `ped.dat`) that selects which set of voice
  samples that ped type uses (male gang, female civilian, etc.).
- The exact mapping from GXT key → audio script identifier is ⏳. Community tools (MTA:SA audio
  tools, OpenIV speech extractor) provide practical access to the sample archive, but the
  key→sample lookup is inside C20's speech routing, not in `CConversations`.

A practical workflow for replacing voice lines:
1. Extract the existing speech sample from `audio/SFX/SCRIPT` (the script speech bank).
2. Record or synthesize the replacement at the same sample rate (22050 Hz mono is standard for SA speech).
3. Re-encode and inject back into the speech bank with the same audio script identifier.
4. The GXT key can remain unchanged — only the audio content changes.

## 6. The limits table and what breaks them

| Limit | Value | Source | Effect at limit |
|-------|-------|--------|----------------|
| Concurrent conversing peds | 14 | Per-ped table: `14 × 28 B @ 0x9691D8`; end = `0x969360` (C35.1) | `SetPedsToConversation` silently fails; conversation does not start |
| Nodes per conversation tree | 12 | Node table: `12 × 44 B @ 0x969360`; build-cursor guard `cmp, 0xC` (C35.3 §1) | Excess `SetUpConversationNode` calls are no-ops |
| Simultaneous audio lines | 50 | Line pool: `50 × 24 B @ 0x969570`; end = `0x969A20` (C35.1) | Old lines may be evicted; audio may not play |
| GXT key length | 7 chars | C19's hash function; 6-byte copy in `SetUpConversationNode` (C35.2 §1) | Keys > 7 chars silently truncated |
| Concurrent conversations | 7 pairs | Derived from 14 peds ÷ 2 per conversation | 8th pair silently dropped |

**Raising the limits without BSS patching** is not possible — the three arrays are consecutive
static storage (C35.1 §2), and the capacity of each is the address of the next. Expanding any
array requires shifting all subsequent arrays and patching every absolute address reference in
the binary. The two boundary equalities (`0x9691D8 + 14×28 = 0x969360`, `0x969360 + 12×44 = 0x969570`)
become patch targets, not proofs, for anyone pursuing an expanded limit.

## 7. The conversation-lock interaction with missions

When `CConversations::Update` drives two peds through a conversation, their AI task trees are
locked (C35.3 §4). This interacts with missions in non-obvious ways:

- A mission script that uses `CLEAR_AREA_OF_PEDS` will **remove the conversing peds' bodies** but
  not necessarily their conversation slots. The per-ped table entry may hold a stale ped handle
  pointing to a freed object, and `Update`'s next decrement will read from the dead slot. `Clear`
  (called at mission start) prevents this by flushing the table before the mission's peds are
  established.
- An ASI mod that installs a persistent ambient-conversation script for world-feel should call
  `WAIT 0` in a polling loop and check `IS_CONVERSATION_AT_NODE 0` (or equivalent) before starting
  a new conversation to ensure the previous one has cleared out. Stacking conversation-starts
  without waiting for completion is the primary cause of the "peds standing still forever" bug
  in conversation mods.

## 8. `AwkwardSay` for quick one-liners

`AwkwardSay` (`0x0043A810`) bypasses the entire tree system (C35.3 §7). From a modding perspective,
it is the appropriate opcode for ambient one-liners that require no subtitle, no branching, and no
interaction — a gang member muttering as the player walks by, a passing comment that requires no
response. Because `AwkwardSay` sets the global active flag at `0x9691D0`, calling it while a real
conversation is in progress will set a flag that `IsConversationGoingOn` sees as true — no practical
harm since the flag stays set anyway, but worth knowing when logging conversation state.

---

### Key takeaways

- The three layers (GXT text / conversation state machine / audio speech) are **independently
  moddable**: subtitle replacement requires only a GXT edit; conversation structure requires an
  SCM change; voice replacement requires an audio archive patch.
- Keys must be **≤ 6 bytes stored** (≤ 7-char ASCII) — the node only copies 6 bytes; longer keys
  are silently truncated at the GXT lookup.
- The **12-node cap** and **14-ped cap** are enforced by bounds guards in disassembled methods;
  raising either requires BSS layout patching (shift all three chained arrays + patch every
  absolute address reference).
- **Audio pacing is automatic**: replace a speech sample with a longer version and the subtitle
  remains on screen longer — no SCM changes needed.
- `AwkwardSay` is for **subtitle-free one-liners** that bypass the tree; using it during an active
  tree conversation is harmless but redundant.

**Previous:** [C35.3 — The runtime conversation engine](03-the-runtime-conversation-engine.md)
**Up:** [C35 — CConversations hub](C35-Conversations.md)
