# C20.3 — The Stream Pack

> **The one-sentence version:** the music packs are XOR-encrypted with a sixteen-byte repeating key, which
> falls out of a residue analysis without a single guess and is then *proven* — decrypting at
> `offset + 8068` yields the Ogg capture pattern `OggS` on **1,922 of 1,922** tracks, and the length field
> inside each header tiles all sixteen packs to the byte.

[← Chapter 20 hub](C20-Audio.md) · [Prev: C20.2 — The SFX bank](02-the-sfx-bank.md) ·
[Next: C20.4 — The loaders in the executable →](04-the-loaders-in-the-executable.md)

**Confidence:** ✅ Verified (cipher, header, exact tiling, codec) / ⏳ (the trailer's rate-like field; an
eight-byte displacement on six tracks)

---

## 1. The first look says "encrypted"

`TrakLkup.dat` gives 1,922 records of `{pack, offset, size}`. Seek to the first one and the bytes are not
audio, not a header, and not anything:

```
ADVERTS @ 0:  15 c5 3b 5e 9a a8 14 f3 b7 4f 28 dc 9d e8 ff f1
              15 c5 3b 5e 9a a8 14 f3 b7 4f 28 dc 9d e8 ff f1
              15 c5 3b 5e 9a a8 14 f3 b7 4f 28 dc 9d e8 ff f1
```

A sixteen-byte pattern repeating verbatim. That is the signature of a **repeating-key XOR over constant
plaintext** — and it also tells us the plaintext there is uniform, which for a header full of unused
slots is exactly what one would expect.

Two further observations pin the cipher down before any key is guessed. First, the same sixteen bytes
appear at offset 0 of *every* stream pack, so the key is shared. Second, at `ADVERTS` offset 1,016,168 the
same pattern appears **rotated by eight** — and `1,016,168 mod 16 = 8`. The key is therefore indexed by
**absolute file position**, not by position within a track.

## 2. Recovering the key without guessing

The key is not searched for; it is *measured*. Take a few megabytes from three different packs, bucket
every byte by its file offset modulo *L*, and take the modal byte in each bucket. If the plaintext's most
common byte is a constant (it is — headers and silence are full of `0x00` and `0xFF`), the modal
ciphertext byte in each bucket is that constant XOR the key byte.

Run it for *L* = 8, 16 and 32:

| *L* | Modal key |
|---:|---|
| 8 | `483ac4239da814f1` |
| 16 | `ea3ac4a19aa814f348b0d7239de8fff1` |
| 32 | `ea3ac4a19aa814f348b0d7239de8fff1` **`ea3ac4a19aa814f348b0d7239de8fff1`** |

The 32-byte result is the 16-byte result **repeated**, which proves the period is 16 and not 32; the
8-byte result is incoherent, which rules out 8. No guessing was involved at any step.

```
key = ea 3a c4 a1 9a a8 14 f3 48 b0 d7 23 9d e8 ff f1      plaintext = cipher ^ key[filePos % 16]
```

## 3. The key is then *proven*, not assumed

A recovered key is a hypothesis. The proof is that decryption produces something it had no reason to
produce unless it is right. Decrypting sixteen bytes at `ADVERTS` offset 1,024,236:

```
cipher    d2 8f 98 a2 …
key[12…]  9d e8 ff f1 …
plain     4f 67 67 53 …   =   'O'  'g'  'g'  'S'
```

The Ogg capture pattern. Applied to every track in the game — decrypt four bytes at
`record.offset + 8068` — it yields `OggS` on **1,922 of 1,922** tracks, with zero exceptions. A wrong key
gives four arbitrary bytes; a wrong offset gives four arbitrary bytes. Getting the correct magic number
1,922 consecutive times is not something an incorrect hypothesis does.

✅ *Verified:* the cipher is a 16-byte repeating XOR keyed on absolute file offset, and the audio payload
begins 8,068 bytes into every track.

Parsing on gives the codec outright. The first Ogg page's packet is the Vorbis identification header, and
across all 1,922 tracks it reports **2 channels** without exception:

| Vorbis sample rate | Tracks |
|---:|---:|
| 32,000 Hz | 1,882 |
| 24,000 Hz | 40 — all of `AMBIENCE` |

✅ *Verified:* streamed audio is **Ogg Vorbis, stereo**, at 32 kHz except for `AMBIENCE` at 24 kHz.

## 4. The 8,068-byte header

Decrypt the header itself and it is almost entirely two alternating words:

```
ff ff ff ff  00 00 00 00   ff ff ff ff  00 00 00 00   …
```

That is a `{ int32 = −1, int32 = 0 }` pair repeated — an eight-byte record with an "unused" sentinel, in
the same idiom the bank header uses. It runs for exactly 8,000 bytes = **1,000 entries**, leaving a
68-byte trailer:

```
8,068 = 1,000 × 8  +  68
```

Six tracks in the game populate it, and they are all in `BEATS`:

| Track (global) | Pack | Populated entries | First entries `{ms, type}` |
|---:|---|---:|---|
| 175 | `BEATS` | 174 | `(18328,4) (18891,4) (19438,4) (20563,3)` |
| 177 | `BEATS` | 152 | `(9969,3) (11172,3) (12313,4) (14657,3)` |
| 178 | `BEATS` | 144 | `(10156,4) (11453,3) (13359,4) (13953,3)` |
| 179 | `BEATS` | 174 | `(18328,4) (18891,4) (19438,4) (20563,3)` |
| 180 | `BEATS` | 76 | `(4375,10) (8734,9) (13047,10) (15250,9)` |
| 181 | `BEATS` | 136 | `(9916,14) (11004,14) (11987,14) (14158,13)` |

🟡 *Reasoned:* the first `int32` is a time in milliseconds and the second a small category tag. The times
are strictly ascending, start a few seconds in, and are spaced in the hundreds of milliseconds — the
spacing of musical beats — while the second word takes small values in a narrow range. The pack is named
`BEATS`. The chapter reads them as `{timeMs, type}` at 🟡 and does not claim to know the type codes.

## 5. The length field tiles every pack

The trailer's first `uint32` is the length of the Ogg payload. It proves itself the same way §1 of
[C20.2](02-the-sfx-bank.md) did — by tiling:

```
Σ over the pack's tracks of ( 8,068 + oggLength )  =  filesize      for all 16 of 16 packs
```

`AA`, `ADVERTS`, `AMBIENCE`, `BEATS`, `CH`, `CO`, `CR`, `CUTSCENE`, `DS`, `HC`, `MH`, `MR`, `NJ`, `RE`,
`RG`, `TK` — every one, to the byte, over 1,187,289,114 bytes. ✅

### 5.1 …and it exposes a bug in `TrakLkup`

This matters, because the `size` field in `TrakLkup.dat` **does not agree with itself**:

| Convention | Tracks |
|---|---:|
| `size == oggLength` | 804 |
| `size == oggLength + 8068` | 1,118 |

Two different meanings for one field, in one file. Using `TrakLkup.size` alone, twelve of the sixteen
packs fail to tile — and they fail by an *exact multiple of 8,068* (e.g. `TK` by 93 × 8,068), which is the
tell that the discrepancy is structural rather than random. The header's own length field has no such
problem.

⚠️ **The practical rule:** `TrakLkup.offset` is reliable; `TrakLkup.size` is not. Read the length from the
track header at `offset + 8000`. A tool that trusts `size` will mis-slice 1,118 of 1,922 tracks by 8,068
bytes.

### 5.2 The trailer's second field is not the sample rate

The `uint32` immediately after the length is the chapter's most tempting trap. It looks exactly like a
sample rate:

| Value | Tracks |
|---:|---:|
| 48,000 | 1,744 |
| 0 | 137 |
| 24,000 | 40 |
| 25,137 | 1 |

But §3 already read the *real* sample rate out of the Vorbis identification header, and the two agree on
only **40 of 1,922** tracks — the `AMBIENCE` ones, where both say 24,000. Everywhere else the field says
48,000 and the audio is 32,000 Hz.

⏳ *Open:* the field is positioned and typed but **not named**. Adopting "sample rate" here would have
been the classic poisoning error: a plausible label that a downstream tool would use to resample 1,882
tracks by a factor of 1.5.

> The single value of **25,137** — on `CUTSCENE` track 642 — is named rather than smoothed away. It is
> the only value in the column that is not one of {0, 24000, 48000}.

### 5.3 The trailer moves by eight bytes on six tracks

On 1,916 tracks the `{length, rate}` pair sits at trailer offset 0 (header offset 8,000). On the other six
it sits eight bytes later, with `0xCD` fill in front of it — and those six are **exactly** the six tracks
that carry beat data:

```
ADVERTS trk0 @7984:  ff ff ff ff 00 00 00 00 | e4 61 0f 00  80 bb 00 00 | cd cd …
BEATS   trk0 @7984:  ff ff ff ff 00 00 00 00 | cd cd cd cd  cd cd cd cd | 1a f2 37 00  80 bb 00 00 …
```

The correlation is perfect — all six beat-annotated tracks are displaced, no other track is. 🟡 The
reasonable reading is a second writer variant: whatever tool emitted the beat annotations also emitted an
extra eight-byte field ahead of the length. ⏳ The eight bytes are uninitialised fill in every case, so
their purpose cannot be read out of them. A parser must detect the case, which is why
`tools/audio_config.py` keys off "does this track have beats" rather than a fixed offset.

The trailer's last four bytes are `01 00 cd cd` on every track — a `uint16` of 1 at header offset 8,064,
constant across all 1,922. ⏳ Meaning not derived.

## 6. Reading a track, end to end

```
1. TrakLkup record  →  { packIndex, offset }            ← ignore `size`, see §5.1
2. seek(streams/<StrmPaks[packIndex]>, offset)
3. read 8,068 bytes, XOR-decrypt with key[(offset + i) % 16]
4. beats = the 1,000 {int32,int32} entries that are not {−1, 0}
5. length = uint32 at header offset 8000, or 8008 if beats is non-empty
6. the Ogg Vorbis stream is the next `length` bytes, also XOR-decrypted
```

✅ On retail this yields 1,922 tracks across 16 packs and consumes every byte of all sixteen files.

---

### Key takeaways

- ✅ Stream packs are **XOR-encrypted** with `ea3ac4a19aa814f348b0d7239de8fff1`, indexed by **absolute
  file position**. The period-16 result was *measured*, not guessed, and confirmed because the 32-byte fit
  is the 16-byte key repeated.
- ✅ The key is **proven**, not assumed: decryption yields `OggS` on **1,922 / 1,922** tracks.
- ✅ Every track is preceded by an **8,068-byte header** = `1,000 × 8` beat slots + a 68-byte trailer, and
  `Σ(8068 + oggLength)` tiles **all 16 packs** to the byte over 1.19 GB.
- ✅ Streamed audio is **Ogg Vorbis, stereo**: 32 kHz on 1,882 tracks, 24 kHz on the 40 `AMBIENCE` tracks.
- ⚠️ **`TrakLkup.size` is internally inconsistent** — 804 tracks exclude the header, 1,118 include it.
  Use the length in the track header instead.
- ⏳ The `uint32` after the length is **not** the sample rate: it disagrees with the Vorbis header on
  1,882 of 1,922 tracks. Left unnamed rather than guessed.
- 🟡 Six `BEATS` tracks carry `{timeMs, type}` beat annotations, and are the only six whose trailer is
  displaced by eight bytes.

**Continue:** [C20.4 — The loaders in the executable](04-the-loaders-in-the-executable.md)
