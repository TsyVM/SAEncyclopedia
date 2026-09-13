# C36.4 — Modding World Reactivity with Script Brains

> **The one-sentence version:** making a new world object interactive requires three registrations
> that must be internally consistent — in the IMG streaming archive (C1), in C31.3's 82-slot
> streamed-script table, and in the 70-slot brain table — and the most common failure modes are
> slot-number mismatches between the second and third, late registration after world cells are
> evaluated, and attempting more than 70 concurrent brains in an inline-embedded table that
> cannot be extended without patching the class size.

**Subsystem category:** Scripting — modding
**Depends on:** [C36.1](01-the-70-slot-table.md)–[C36.3](03-the-brain-lifecycle.md),
[C31.3](../C31-Streaming-Slot-Tables/03-cstreamedscripts.md),
[C18](../C18-SCM-Script/C18-SCM-Script.md),
[C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)
**RE status:** Documented — modding approaches derived from the phase analysis of C36.3
**Confidence:** ✅ for the three-registration requirement and the slot-number coupling · 🟡 for
the exact safe-registration window (before `CGame::Initialise` is confirmed; the exact call
that evaluates nearby-cell attractors is ⏳)

---

## 1. The three-registration requirement

Every new interactive object needs three entries to be live simultaneously, and they must be
**consistent with each other** (same slot numbers / name strings):

```
Registration 1: Streaming archive (C1/C2)
  - What: the brain's SCM subscript binary, stored in gta3.img (or a custom .img)
  - Identifies it by: stream model-id (the same integer that CStreaming uses for any asset)
  - Required because: the script must be loadable on demand by the streaming system

Registration 2: C31.3 streamed-script table (82 slots × 32 bytes)
  - What: an entry associating stream model-id → slot index in the 82-slot table
  - Identifies it by: slot index [0..81]
  - Required because: AddNewStreamedScriptBrainForCodeUse references this slot when it
    issues the CStreaming::RequestModel call

Registration 3: C36 brain table (70 slots × 20 bytes)
  - What: an entry associating model-ID (or attractor name) → C31.3 slot index
  - Identifies it by: model ID (uint16) or 6-char attractor name
  - Required because: this is what the proximity trigger checks
```

If Registration 2's slot index and Registration 3's slot-index reference are mismatched, the
proximity trigger fires but `StartOrRequestNewStreamedScriptBrainWithThisName` requests the wrong
C31.3 slot — either loading the wrong script or issuing a request for an empty slot. The script
thread either does not start or starts at an incorrect entry point. **This is the most common
failure mode in brain mods.**

## 2. The registration workflow via ASI

An ASI mod that registers a new interactive object at startup:

```cpp
// Called from: a hook on CGame::Initialise, before the game enters the main loop
// (after streaming system is initialised, C1/C2 — so CStreaming::RequestModel works)

void RegisterMyBrain() {
    // Step 1: Verify the subscript is in the IMG archive (done by the modder beforehand)
    // The subscript has stream model-id = MYBRAIN_MODEL_ID (a value in the streaming ID space,
    // above the vehicle/ped range — choose carefully to avoid collisions, C2/C3)

    // Step 2: Register in C31.3's streamed-script table
    // (patching the 82-slot table directly or via the CStreamedScripts::Init path)
    // Slot index = MYBRAIN_SCRIPT_SLOT (pick a free slot in 0..81 — SA base uses < 30)
    CStreamedScripts_Register(MYBRAIN_SCRIPT_SLOT, MYBRAIN_MODEL_ID);

    // Step 3: Register in the brain table
    // AddNewStreamedScriptBrainForCodeUse(this, model_id_or_name, script_slot_index)
    // The function signature is (this_ptr, ?, model_id_word, script_slot, ...) — exact
    // argument layout from 0x0046A9C0 is ⏳ for the non-id fields
    typedef void(__thiscall* AddBrain_t)(void*, WORD, int, ...);
    AddBrain_t AddBrain = (AddBrain_t)0x0046A9C0;
    AddBrain(pCScriptsForBrains, MYBRAIN_MODEL_ID, MYBRAIN_SCRIPT_SLOT);
}
```

The critical ordering: `CStreaming` must be initialised before Step 2 (otherwise `RequestModel`
calls fail), and the brain table must exist before Step 3 (it is created by `CScriptsForBrains::Init`
at `0x0046A8C0`, which runs during `CGame::Initialise`). Both conditions are satisfied by hooking
at the end of `CGame::Initialise`.

## 3. The 6-char attractor name as the coupling point

For name-keyed attractors (shop clerks, barbershops, tattoo parlors, gyms), the name string is the
coupling between the world-placement data and the brain table:

- The **world object** (in the IPL — [C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md)) has an
  "attractor" attribute whose value is the 6-char name string.
- The **brain table slot** stores that same 6-char string at `+0x00` (the model-id field is repurposed
  as a char[6] for attractor brains).
- **`GetIndexOfScriptBrainWithThisName`** (`0x0046AA30`) compares the first **5 bytes** against
  the query (C36.3 §2) — so the 6th byte is decoration only.

Implications:

| Scenario | Result |
|----------|--------|
| Two objects with the same attractor name | Same brain triggers for both — the script fires for whichever object the player is near; no per-instance disambiguation |
| Brain name changed in table but not in IPL | Proximity trigger never fires (name mismatch) |
| Brain name changed in IPL but not in table | Same — the table scan finds nothing |
| Attractor name > 5 unique chars | Matching still works, but the 6th char is invisible to `GetIndexOfScriptBrainWithThisName` |

SA's built-in attractor names are documented in the community (names like `BARBER`, `TATTOO`,
`GYM1`, `CASHIER`). A mod must avoid collisions with these. A prefix convention
(e.g., `MMOD_`) guarantees uniqueness up to the 5-char compare limit.

## 4. Per-instance state: what is impossible

`CScriptsForBrains` does not pass any per-object context to the script thread it creates. Two
identical shop objects at different map coordinates trigger the same script with no information
distinguishing them. The brain script itself must probe the world to determine which instance
it is serving:

```scm
; at script start, find which instance triggered this brain
GET_PLAYER_CHAR 0 → $player
GET_CHAR_COORDINATES $player → $px $py $pz
; check proximity to known object positions
IS_POINT_NEAR_POINT $px $py 100.0 200.0 5.0 → $near_shop_1
IS_POINT_NEAR_POINT $px $py 300.0 400.0 5.0 → $near_shop_2
IF $near_shop_1
  ; ... shop 1 logic
ELSE IF $near_shop_2
  ; ... shop 2 logic
END
```

This is the **per-instance state impossibility** — a fundamental architectural gap. The brain
table is indexed by model ID, not by world-object instance, so all instances of a model share
one brain slot and one script. Workarounds:

- **Distinct model IDs per interactive instance** — assign unique model IDs to objects that need
  per-instance behavior; each registers its own brain slot. This consumes one brain slot per
  instance (max 70 total).
- **Proximity-based branching in the script** — as above; the script detects which instance is
  active by player position.
- **Pre-defined SCM labels** — place each instance at a pre-known coordinate and hard-code those
  coordinates in the brain script.

## 5. The gang-territory brain mechanism

C30.4's [CGangWars](../C30-Gameplay-Managers/04-cgangwars.md) and the gang-territory system
interact with script brains in a specific way: territory "ownership" is not managed by brains
directly — it is managed by the `0xC8B2C0` zone-ownership byte array (C30.4, C33.3 §2). But the
*event* that fires when the player enters rival territory — the gang members responding, the
aggression-level increase — can be driven by model-ID-keyed brains attached to the territorial
markers (spray paint tags, gang signs) placed around the map.

The relationship:

1. **Gang markers** (spray-tag model IDs) are placed in the IPL throughout SA's gang neighborhoods.
2. **Model-ID brains** are registered for each gang marker model ID.
3. When the player approaches a gang marker, the brain fires a script that checks the
   zone-ownership byte (`0xC8B2C0[zone_id]`), reads the attacking/defending gang state, and
   escalates accordingly.
4. The `CGangWars` state machine (C30.4) handles the actual war simulation; the brain script
   is the *trigger*, not the simulation.

A mod can add new gang factions with new territorial model IDs by following this same pattern:
assign a new model ID to the faction's markers, register a brain for that model ID, write a
brain script that updates the `0xC8B2C0` ownership bytes for the relevant zones, and let
`CGangWars` handle the rest.

## 6. The C31.3 triple-registration consistency requirement

C31.3's 82-slot × 32-byte streamed-script table stores three fields per entry that brain
registration must match:

| C31.3 field | What it must match in the brain table |
|-------------|--------------------------------------|
| Stream model-id | The same model-id used in Registration 1 (IMG archive entry) |
| Script slot index | The same slot index stored in the brain table's record (at +0x10 / +0x14 — exact offset 🟡) |
| Load status byte | Set to "requested" by `StartOrRequestNewStreamedScriptBrainWithThisName` when the script is demanded |

The consistency requirement is strict because the two tables are joined by slot index, and the
slot index is a raw integer with no bounds-checked cross-reference. A mismatched slot index is
not detected at registration — it is only detected (as a crash or wrong-script execution) when
the brain fires. Test every new brain registration in a clean game session before releasing a mod.

## 7. Raising the 70-slot limit

The 70-slot table is **embedded in the `CScriptsForBrains` object** — it is not a separate heap
allocation (C36.0, C36.1 §2). The object's size is compiled into every `new CScriptsForBrains`
and every stack allocation of the class, as well as into the `Init` and `ShutDown` method bodies.
Raising the limit to, say, 100 requires:

1. Patching the allocation size everywhere the class is allocated (`new`, `operator new` calls).
2. Patching `Init`'s initialization loop bound (`0x46` → new count).
3. Patching `SwitchAllObjectBrainsWithThisID` and `AddNewStreamedScriptBrainForCodeUse`'s loop
   bounds.
4. Patching the per-id state array at `0x964E08` — its capacity is determined by the model-ID
   range, not the brain-slot count, but it must remain large enough.

Unlike the entity pools ([C53](../C53-Memory-And-Pool-Architecture/C53-Memory-And-Pool-Architecture.md)),
where one allocator call controls the pool size, the brain table's inline embedding means the
change percolates to several methods. **Keep custom brain counts well below 60** to leave a safe
margin for SA's own brains and any other mods.

## 8. The corrected C33 attribution: `0x0046AA80` is CScriptsForBrains

[C33.2](../C33-Attributing-The-Unnamed/02-the-eleven-and-what-remains.md)'s data-only leads
include the unnamed function at `0x0046AA80`, which was attributed to `CStreaming` because it
references `0x964E08` — the per-id state array. C36.3 §1 shows that `0x964E08` is this class's
own bookkeeping array, not a streaming-system array. Furthermore:

- `0x0046AA80` sits at offset `+0x50` from `CScriptsForBrains::Init` at `0x0046A8C0`.
- That puts it squarely inside the method range `0x0046A8C0`–`0x0046CED0` established by the
  named methods in C36.2.
- `0x0046AA80` calls `GetIndexOfScriptBrainWithThisName` (`0x0046AA30`) — a method that only
  `CScriptsForBrains` would call.

The data-only lead is now a **STRONG attribution**: `0x0046AA80` is a `CScriptsForBrains` method,
not a `CStreaming` method. This corrects C33.2's Table 2 entry and is an example of the growth
path [C33.2 §4](../C33-Attributing-The-Unnamed/02-the-eleven-and-what-remains.md#4) described:
a later structural chapter refining an earlier attribution by supplying the call-graph evidence
(it calls an unambiguous `CScriptsForBrains` method) and the address-cluster membership signal.

---

### Key takeaways

- **Three registrations must be consistent**: IMG archive (stream model-id), C31.3 slot table
  (slot index), and brain table (model-id → slot index coupling). A slot-number mismatch is
  silent at registration and fatal at trigger time.
- **Register before `CGame::Initialise` completes** — the safe window is a hook at the end of
  `CGame::Initialise` after `CScriptsForBrains::Init` has run.
- **Name-keyed brains compare only 5 chars** — keep mod attractor names to 5 unique chars;
  use a prefix to avoid collisions with SA's built-in names.
- **Per-instance state is impossible by design** — all instances of a model share one brain
  slot; use distinct model IDs or player-position probing in the script for per-instance behavior.
- **The 70-slot limit is inline-embedded** — raising it requires patching the class size, `Init`,
  and all loop bounds; keep well below 60 in practice.
- **`0x0046AA80` is now STRONG-attributed to `CScriptsForBrains`** — C36 corrects a C33
  data-only lead by supplying both call-graph and address-cluster evidence.

**Previous:** [C36.3 — The attractor-brain lifecycle](03-the-brain-lifecycle.md)
**Up:** [C36 — CScriptsForBrains hub](C36-Script-Brains.md)
