# C20.6 — The namespace, recovered: `AudioEvents.txt`

> **This page corrects [C20.5 §7](05-eventvol.md).** That section concluded the audio-event ID namespace was
> "**not recoverable from the shipped files alone**" and "could only be supplied from outside — community
> data." That is **wrong**, and the correction is written up here per the house rule rather than silently
> patched. A shipped file — **`data/AudioEvents.txt`** — names the audio events, and it names **447 of the
> 597** configured `EventVol.dat` entries. C20.5 checked the *executable* (correctly: it has bare integers)
> but never checked `data/`.

## What went wrong

C20.5 conflated two different statements:

- **"not recoverable from the binary"** — *true*. `gta_sa.exe` reaches each event as a bare integer immediate
  with no id→name table and no strings at the access sites ([C20.5 §7](05-eventvol.md)). That finding stands.
- **"not recoverable from the shipped files alone"** — *false*. The shipped tree contains
  `data/AudioEvents.txt`, a plain-text `name → id` table, which C20.5 did not examine. The names were never
  "outside" the tree; they were one directory over from the audio config C20 was decoding.

This is exactly the failure mode the house rules name as the enemy: a ✅-marked closure that was actually
wrong because a shipped file was overlooked. It is corrected now.

## `AudioEvents.txt` — the format

`data/AudioEvents.txt` is plain text, one event per line (blank-line separated):

```
SOUND_DISABLE_HELI_AUDIO 1000
SOUND_ENABLE_HELI_AUDIO 1001
SOUND_CEILING_VENT_LAND 1002
...
SOUND_ZER2_BA 44804
```

It defines **7,071** events, IDs **1000 … 45400**, each a `SOUND_*` name paired with the integer ID the
scripts and engine use. This is the script-facing audio-event namespace — the identifiers behind the SCM
`AUDIO_EVENT` opcodes ([C18](../C18-SCM-Script/C18-SCM-Script.md)) and the `EventVol.dat` trim index
([C20.5](05-eventvol.md)). `derive_audioevents.py` parses it and asserts the count and range
(`audioevents_is_name_map`).

## The cross-check: 447 of 597 named

[C20.5](05-eventvol.md) proved `EventVol.dat` is a flat `int8[45401]` indexed by event ID, with **597**
entries configured (not the `0x80` sentinel). Resolving those 597 indices through `AudioEvents.txt`:

| Result | Count | |
|---|---:|---|
| Configured `EventVol` entries | **597** | (C20.5) |
| **Named by `AudioEvents.txt`** | **447** | **74.9 %** |
| Unnamed — low block (ID < 1000) | 149 | engine-internal events |
| Unnamed — high (≥ 1000, gap) | 1 | |

So three quarters of the trim table that C20.5 could only describe as "event 76, trimmed −26" can now be
named. Examples, straight from the two shipped files:

| ID | Name | Trim (dB) |
|---:|---|---:|
| 1002 | `SOUND_CEILING_VENT_LAND` | −8 |
| 1009 | `SOUND_BONNET_DENT` | −6 |
| 1010 | `SOUND_BASKETBALL_BOUNCE` | −6 |
| 1011 | `SOUND_BASKETBALL_HIT_HOOP` | −2 |
| 1012 | `SOUND_BASKETBALL_SCORE` | −5 |
| 1013 | `SOUND_POOL_BREAK` | −2 |
| 1014 | `SOUND_POOL_HIT_WHITE` | −6 |

`derive_audioevents.py` asserts the **447/597** cross-check (`configured_entries_named`) and spot-checks the
names (`names_spot_check`).

## What remains genuinely unnamed

The correction is bounded, and honesty cuts both ways. **149** of the configured entries are in the **low
block (ID < 1000)** — engine-internal audio events that `AudioEvents.txt` does not cover (it starts at 1000).
For *those* specific events (the ones C20.5's §7 disassembly happened to sample, like event 145), the "not in
the shipped files" statement still holds — they are hard-coded in the engine and named nowhere on disc. So
the accurate, corrected claim is:

> The **script-facing** audio-event namespace (IDs 1000–45400, 7,071 events) **is** recoverable from the
> shipped `data/AudioEvents.txt`, naming **447 of the 597** configured `EventVol` trims. The **low
> engine-internal block** (IDs < 1000, 149 configured) remains unnamed by any shipped file — for it, C20.5's
> negative result stands.

Not "not recoverable"; rather "recoverable for the script namespace, not for the low engine block." This is
not community data — it is a first-party file in `data/`, so it is fully in bounds.

## Key takeaways

- **Correction to [C20.5 §7](05-eventvol.md):** the audio-event namespace *is* recoverable from a shipped
  file — `data/AudioEvents.txt` names **7,071** events and **447 of the 597** configured `EventVol` trims
  (74.9 %). The "not recoverable from the shipped files" claim was wrong because `data/` was never checked.
- The names are real and specific (`SOUND_BASKETBALL_BOUNCE`, `SOUND_POOL_BREAK`, `SOUND_CEILING_VENT_LAND`),
  so C20.5's anonymous trim table can now be read.
- The bound: only the **low engine-internal block** (ID < 1000, 149 configured) stays unnamed — for it the
  original negative result holds. C20.5's "not in the *binary*" was always correct; "not in the shipped
  *files*" was the error.

**Continue:** [back to C20.5 — EventVol](05-eventvol.md) · or [the C20 hub](C20-Audio.md).
