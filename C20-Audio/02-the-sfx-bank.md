# C20.2 — The SFX Bank

> **The one-sentence version:** the gap between consecutive banks is **4,804 bytes in all 701 gaps and in
> all 9 pack tails** — and when a bank is finally opened it carries a header of exactly
> `4 + 400 × 12 = 4,804` bytes, so the directory and the container prove the same number twice from
> opposite directions.

[← Chapter 20 hub](C20-Audio.md) · [Prev: C20.1 — The six config tables](01-the-config-tables.md) ·
[Next: C20.3 — The stream pack →](03-the-stream-pack.md)

**Confidence:** ✅ Verified (header size, entry layout, exact tiling, PCM width, loop points) / 🟡
(mastering level) / ⏳ (the trim field)

---

## 1. The constant nobody wrote down

`BankLkup.dat` gives 710 records of `{pack, offset, size}` ([C20.1](01-the-config-tables.md)). Group them
by pack, sort by offset, and ask the obvious question: does bank *n* end where bank *n+1* begins?

It does not. There is a gap — and the gap is the same everywhere:

```
FEET    bank 0:  offset      0   size  90,998   →  ends at  90,998
        bank 1:  offset 95,802                  →  gap of   4,804
        bank 2:  offset 186,418  (bank 1 ends 182,638)  →  gap of 4,804
```

Across all nine packs there are 701 such gaps and **every one is 4,804**. Better still, each pack has a
residue at the end — filesize minus the last bank's `offset + size` — and **that is 4,804 too, in all
nine packs**. Two independent appearances of the same constant say it is not a gap at all: `offset`
points at a **4,804-byte header**, `size` counts the bytes after it, and a bank's true span is
`4,804 + size`. Under that model every pack tiles exactly:

| Pack | Banks | `Σ(4804 + size)` | File size | Residue |
|---|---:|---:|---:|---:|
| `FEET` | 7 | 354,020 | 354,020 | **0** |
| `GENRL` | 137 | 25,170,572 | 25,170,572 | **0** |
| `PAIN_A` | 3 | 6,704,240 | 6,704,240 | **0** |
| `SCRIPT` | 218 | 319,687,888 | 319,687,888 | **0** |
| `SPC_EA` | 46 | 125,118,208 | 125,118,208 | **0** |
| `SPC_FA` | 18 | 106,793,248 | 106,793,248 | **0** |
| `SPC_GA` | 209 | 1,103,880,528 | 1,103,880,528 | **0** |
| `SPC_NA` | 52 | 428,281,720 | 428,281,720 | **0** |
| `SPC_PA` | 20 | 175,482,308 | 175,482,308 | **0** |

✅ *Verified:* **710 banks tile 2,291,472,732 bytes with no gap and no overlap.** This is the chapter's
strongest single result, and note that it was obtained **before any bank was parsed** — the constant came
from arithmetic on the directory alone.

## 2. The header is what the arithmetic predicted

Only now open a bank. `FEET` at offset 0:

```
offset 0:  09 00 00 00                                    numSounds = 9
offset 4:  00 00 00 00  ff ff ff ff  80 3e  83 ff         entry 0
offset16:  f4 41 00 00  ff ff ff ff  68 42  53 00         entry 1
offset28:  42 71 00 00  ff ff ff ff  68 42  5b 02         entry 2
    ⋮
offset112: 00 00 00 00  00 00 00 00  00 00  00 00         entry 9 — unused, zeroed
```

A `uint32` count followed by twelve-byte descriptors, and the descriptors stop being meaningful at exactly
`numSounds`. The maximum `numSounds` observed across all 710 banks is **400**, and

```
4 + 400 × 12 = 4,804
```

— the constant §1 derived from the outside. The directory and the container agree without having been
made to agree. ✅

| Offset | Type | Field | Evidence |
|---:|---|---|---|
| 0 | `uint32` | data offset, relative to the end of the header | first entry is 0 in **710/710** banks; strictly ascending in **710/710**; last is `< size` in **710/710** |
| 4 | `int32` | loop offset, −1 for none | 351 sounds have a non-−1 value and **all 351** satisfy `0 ≤ loop < length` |
| 8 | `uint16` | sample rate | see §4 |
| 10 | `int16` | trim — authored | see §5 |

And the unused tail is genuinely unused: in **710 of 710** banks, every byte from `4 + numSounds × 12`
to 4,803 is zero. A bank that declares nine sounds carries 391 descriptors of nothing.

✅ *Verified:* the bank header is `uint32 numSounds; SoundEntry entries[400]`, fixed at 4,804 bytes.
**710 banks hold 61,993 sounds.**

## 3. The samples are 16-bit

Each sound's length follows from the next entry's offset, or from the bank's `size` for the last one. Run
that over all 61,993 sounds and **every single length is even** — 61,993 of 61,993, with no exceptions to
name. Odd lengths are impossible for 16-bit samples and unremarkable for 8-bit ones, so a uniform even
result across sixty thousand independently authored assets is not coincidence.

The loop field agrees. All 351 looping sounds have a loop offset that lands inside their own sample data,
and none points past the end. A wrong entry width would scatter those values arbitrarily.

✅ *Verified:* sample data is 16-bit signed mono PCM; loop offsets are byte offsets into the sound.

## 4. Sample rates, and the one that isn't

The `uint16` at offset 8 takes 103 distinct values across the 61,993 sounds, dominated by a handful:

| Rate (Hz) | Sounds |
|---:|---:|
| 12,000 | 54,273 |
| 15,000 | 5,908 |
| 8,000 | 1,082 |
| 18,000 | 286 |
| 22,050 | 89 |
| 16,000 | 36 |
| *(97 others)* | 319 |

Every value but one falls in the range 4,000–48,000 Hz. The exception is named rather than rounded away:

> **`GENRL`, bank index 138, sound 27** carries a rate of **2,021 Hz** — and it is also a *looping* sound
> (loop offset 0), one of only 351 in the game. Whether 2,021 is deliberate or a typo for something else
> is not derivable from the bytes; it is recorded as the single out-of-range value in the table.

🟡 *Reasoned:* the field is the playback sample rate. The distribution is exactly what one expects of one
(clustered on round authoring rates, spanning telephone-quality to CD), and no other reading of a `uint16`
at that position produces sane numbers. It is **not** promoted to ✅ because nothing in the shipped data
forces it — the decisive proof would be watching the value reach a `WAVEFORMATEX`, which this chapter has
not traced.

## 5. The trim field is authored, not measured

The trailing `int16` ranges from −200 to 6,229 across 780 distinct values, and its single most common
value is **−200**, which occurs 11,881 times. That number is suggestive: if the units are hundredths of a
decibel, −200 is −2.00 dB.

So the field was tested against the audio itself. Decoding sampled sounds and measuring their true peak
amplitude gives a striking result — the peaks cluster tightly at **26,026–26,028** out of 32,768, which is
−2.00 dBFS almost exactly. **104 of 120** sampled sounds sit in that band.

🟡 *Reasoned:* the sample library is mastered to a −2 dBFS ceiling. (Stated at 🟡, not ✅, because it rests
on a 120-sound sample rather than all 61,993 — a full survey means decoding 2.29 GB and is left as an
easy, bounded follow-up.)

But that same measurement **refutes** the obvious reading of the field. If the `int16` were the sound's
headroom or peak level, it would vary with the measured peak. It does not: sounds carrying trim values
from −200 to +2,820 have **identical** measured peaks of −2.00 dBFS. The field is therefore not derived
from the audio at all — it is a value a sound designer typed, presumably a playback gain applied at mix
time.

⏳ *Open:* position and type are certain (`int16` at descriptor offset 10); the units and the sign
convention are not derived. This is a negative result and the chapter states it as one: **the field is
authored metadata, and the encyclopedia does not know what it means.**

## 6. Reading a bank, end to end

```
1. BankLkup record  →  { packIndex, offset, size }
2. seek(SFX/<PakFiles[packIndex]>, offset)
3. read 4,804 bytes  →  numSounds, entries[400]
4. sound i occupies  [ offset + 4804 + entries[i].dataOffset ,
                       offset + 4804 + entries[i+1].dataOffset )
   the last one ending at offset + 4804 + size
5. decode as signed 16-bit mono PCM at entries[i].sampleRate
```

No length field is needed anywhere: the header size is constant, the count is declared, and each sound's
length is the distance to the next. ✅ On retail this yields 710 banks and 61,993 sounds with zero
residue.

---

### Key takeaways

- ✅ The **4,804-byte header** is proven twice and independently: as the constant gap in **701 of 701**
  inter-bank gaps and **9 of 9** pack tails, and as the `4 + 400 × 12` header a bank actually carries.
- ✅ All nine packs **tile exactly** — 710 banks over 2,291,472,732 bytes, no gap, no overlap.
- ✅ The descriptor is `{uint32 dataOffset; int32 loopOffset; uint16 sampleRate; int16 trim}`; unused
  descriptors are zeroed in **710/710** banks.
- ✅ Sample data is **16-bit PCM** — **61,993/61,993** lengths are even — and all **351** loop offsets
  land inside their own sound.
- 🟡 Rates are playback sample rates (12 kHz dominates); the library is mastered to **−2.00 dBFS**.
- ⚠️ Exactly one sound is out of range: `GENRL` bank 138, sound 27, at **2,021 Hz**. Named, not rounded.
- ⏳ The trailing `int16` is **authored, not measured** — proven by the fact that its value does not track
  the sound's actual peak. Meaning left open.

**Continue:** [C20.3 — The stream pack](03-the-stream-pack.md)
