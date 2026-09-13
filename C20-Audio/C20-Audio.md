# Chapter 20 — Audio: Banks, Streams and the Tables That Index Them

> **Goal of this chapter:** decode the largest untouched subsystem in the game — **3.5 GB** of sound data
> across 25 files — with **every one of those bytes accounted for**: 9 sample packs that tile exactly into
> 710 banks, 16 music packs that tile exactly into 1,922 Ogg Vorbis tracks, the **XOR key that encrypts
> the music recovered and proven**, and **every record width confirmed twice** — once by residue-free
> division of the file, once by the divide instruction in `gta_sa.exe` that computes it.

**Subsystem category:** Audio
**Depends on:** [C0.1 — The HOODLUM layer](../C0-Binary-Identity/01-the-hoodlum-layer.md) (address resolution
into the packed executable)
**Ties:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C18](../C18-SCM-Script/C18-SCM-Script.md), [C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md), [C45](../C45-Damage/C45-Damage.md)
**RE status:** Verified
**Confidence:** ✅ Verified (all six config tables, both container formats, the stream cipher, the codec)
/ 🟡 (sample normalisation, the event-volume unit) / ⏳ (two header fields, the audio-event ID namespace)

---

## Deep-dive pages

- [C20.1 — The six config tables](01-the-config-tables.md): `PakFiles`, `StrmPaks`, `BankLkup`,
  `TrakLkup`, `BankSlot`, `EventVol` — six files, five of them solved by division alone, and one blank
  slot that turns out to be load-bearing.
- [C20.2 — The SFX bank](02-the-sfx-bank.md): a fixed **4,804-byte** header of 400 sound descriptors in
  front of raw 16-bit PCM; 710 banks holding **61,993 sounds**, and every sample pack tiling to the byte.
- [C20.3 — The stream pack](03-the-stream-pack.md): music is **XOR-encrypted**; the key is recovered
  without guessing and then *proven* — decryption yields `OggS` on all 1,922 tracks. Inside: a 1,000-entry
  beat table and a length field that tiles all 16 packs exactly.
- [C20.4 — The loaders in the executable](04-the-loaders-in-the-executable.md): six `push` instructions,
  six divide-by-constant idioms, and an independent confirmation of every record width in the chapter.
- [C20.5 — `EventVol.dat`](05-eventvol.md): the one file that would not divide — because it has no
  records. A flat signed-byte array indexed by event ID, proved by a 29-for-29 coincidence that isn't one.
- [C20.6 — The namespace, recovered](06-audioevents-namespace-recovered.md): **corrects C20.5 §7** — a
  shipped file, `data/AudioEvents.txt`, names **447 of the 597** configured `EventVol` trims (the namespace
  was recoverable after all; C20.5 only checked the binary, not `data/`).

---

## 20.1 The result first

| Claim | Evidence |
|---|---|
| **9 sample packs, 710 banks, 61,993 sounds** | every pack tiles as `Σ(4804 + size)` = filesize ✅ |
| **16 music packs, 1,922 tracks** | every pack tiles as `Σ(8068 + oggLength)` = filesize ✅ |
| Bank header is **`4 + 400 × 12` = 4,804** | the constant inter-bank gap derived from `BankLkup` *equals* the header the bank actually carries ✅ |
| Streams are **XOR-encrypted, 16-byte repeating key** | period proven by residue analysis; decryption yields `OggS` on **1,922 / 1,922** tracks ✅ |
| Music is **Ogg Vorbis, stereo** | Vorbis identification header parsed on all 1,922 tracks ✅ |
| Sample data is **16-bit PCM** | **61,993 / 61,993** sound lengths are even ✅ |
| Loop points are real offsets | **351 / 351** looping sounds have `0 ≤ loop < length` ✅ |
| Record widths **12 / 52 / 16 / 4820** | residue-free division **and** the executable's own divide constants ✅ |
| `EventVol.dat` is a **flat `int8[45401]`** | 56/56 accesses are unscaled `movsx byte`; **29/29** exe-literal event IDs are configured ✅ |

**3,478,761,846 bytes of audio data — every byte assigned to a bank or a track, with no gap and no
overlap in any of the 25 container files.**

## 20.2 The shape of the subsystem

San Andreas splits its audio in two, and the split is visible in the file layout before any field is
decoded. Short sounds that must start instantly — footsteps, gunfire, speech lines — live in **banks** of
raw PCM inside nine `SFX` packs. Long sounds that can afford a decoder and a buffer — radio stations,
adverts, cutscene dialogue, ambience — live as **Ogg Vorbis tracks** inside sixteen `streams` packs. Each
half has its own directory file, and both directories use the same twelve-byte record.

```
audio/
├── CONFIG/
│   ├── PakFiles.dat    468 B  =   9 × 52   names of the nine SFX packs
│   ├── StrmPaks.dat    272 B  =  17 × 16   names of the stream packs (one slot blank)
│   ├── BankLkup.dat  8,520 B  = 710 × 12   { pack, offset, size } → one per bank
│   ├── TrakLkup.dat 23,064 B  = 1922 × 12  { pack, offset, size } → one per track
│   ├── BankSlot.dat 216,902 B = 2 + 45 × 4820   45 runtime bank slots
│   └── EventVol.dat  45,402 B  = int8[45,401]     per-event volume trim, 0x80 = unset
├── SFX/     FEET GENRL PAIN_A SCRIPT SPC_EA SPC_FA SPC_GA SPC_NA SPC_PA     2.29 GB
│              └── bank = [ uint32 numSounds ; SoundEntry[400] ] + raw 16-bit PCM
└── streams/ AA ADVERTS AMBIENCE BEATS CH CO CR CUTSCENE DS HC MH MR NJ RE RG TK   1.19 GB
               └── track = [ Beat[1000] ; 68-byte trailer ] + Ogg Vorbis   (whole file XOR-encrypted)
```

## 20.3 Why the arithmetic closes so cleanly here

The audio subsystem is unusually kind to a reverse engineer, and the reason is architectural. The game
streams sound off disk while the player moves; it cannot afford to parse a variable-length container to
find the fourteenth footstep. So every container is **fixed-stride**: a bank header is always 4,804 bytes
whether it holds one sound or four hundred, and a track header is always 8,068 bytes whether it holds
zero beats or 174. The engine seeks to `offset` and reads a known number of bytes. That design decision
is what makes `Σ(constant + size) = filesize` hold to the byte across all 25 files, and the tiling is in
turn what proves the constant.

This gives the chapter two independent proofs of the same number. The 4,804-byte bank header was first
derived **from outside** the bank — as the constant gap between consecutive `BankLkup` entries, plus the
constant residue at the end of every pack. Only afterwards was a bank actually parsed, and it turned out
to carry a header of exactly `4 + 400 × 12 = 4,804` bytes. The directory and the container agree without
having been made to.

## 20.4 What this chapter does not claim

Two things are recorded as unresolved rather than guessed. (A third, `EventVol.dat`, was open when this
chapter was first written and is now solved in [C20.5](05-eventvol.md) — it has no record width because it
has no records. What remains open there is the **audio-event ID namespace**: the chapter can say event 76
is trimmed by −26 dB but not what event 76 is.)

The **`int16` at the end of each sound descriptor** is authored metadata, not a measurement. It ranges
from −200 to 6,229 and its most common value, −200, matches the level the samples themselves are mastered
at. But sounds carrying wildly different values have *identical* peak levels, so it cannot be derived from
the audio. Named, positioned, not interpreted.

The **`uint32` after the Ogg length in each track header** is 48,000 on 1,744 tracks, 24,000 on 40 and 0
on 137. It looks like a sample rate, and on the 40 `AMBIENCE` tracks it happens to equal one — but the
Vorbis identification header says the other 1,882 tracks are 32,000 Hz, not 48,000. Whatever the field
means, it is not the rate the audio is stored at, and this chapter declines to name it.

---

### Key takeaways

- The subsystem divides into **banks** (raw PCM, instant start) and **streams** (Ogg Vorbis, buffered),
  each with a directory of identical twelve-byte records.
- ✅ **710 banks / 61,993 sounds** and **1,922 tracks** account for **every byte** of 3.48 GB across 25
  container files, with no gap and no overlap.
- ✅ The two fixed header sizes — **4,804** and **8,068** — are each proven *twice*: as the constant that
  makes the pack tile, and as the header the container actually carries.
- ✅ The stream cipher is a **16-byte repeating XOR keyed on absolute file position**, confirmed by the
  Ogg capture pattern appearing on all 1,922 tracks.
- ✅ Every record width is confirmed against the executable's own divide-by-constant idiom
  ([C20.4](04-the-loaders-in-the-executable.md)).
- ✅ `EventVol.dat` is a **flat `int8[45,401]`** indexed by audio event ID ([C20.5](05-eventvol.md)) —
  no records, hence no divisor. Proved **29/29** against a 1.31 % base rate.
- ✅ **The audio-event ID namespace is recovered** ([C20.6](06-audioevents-namespace-recovered.md)): the
  shipped `data/AudioEvents.txt` names 7,071 events and 447 of the 597 configured `EventVol` trims —
  correcting C20.5 §7. Only the low engine-internal block (ID < 1000) stays unnamed.
- ⏳ The per-sound trim field and the track header's rate-like field remain
  open.

**Continue:** [C20.1 — The six config tables](01-the-config-tables.md)

## See also (forward links)

The audio-event namespace is recovered in [C20.6](06-audioevents-namespace-recovered.md) (correcting C20.5). Audio events drive gameplay via [C45 — Damage](../C45-Damage/C45-Damage.md) (drowning) and the [C41 — Ped AI](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md) event system.

## Engine relationships

*Generated by [`tools/apply_relationships.py`](../tools/apply_relationships.py); call-graph counts from a 72,719-site `.text` pass, bugs/modding/performance curated. Data: [`RE-Data/data/relationships.json`](../RE-Data/data/relationships.json).*

- **Loaded / consumed by:** [C0](../C0-Binary-Identity/C0-Binary-Identity.md), [C18](../C18-SCM-Script/C18-SCM-Script.md), [C41](../C41-Ped-AI-Tasks-Wanted/C41-Ped-AI-Tasks-Wanted.md), [C45](../C45-Damage/C45-Damage.md)
- **Known bugs / gotchas:** TrakLkup.size is internally inconsistent (read the header instead); trailer moves on BEATS tracks.
- **Modding:** the XOR key + bank/stream formats are the audio-replacement path.
- **Performance:** 3.48 GB streamed off disk; fixed-stride seeks.
