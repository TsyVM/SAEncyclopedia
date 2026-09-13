# C20.1 — The Six Config Tables

> **The one-sentence version:** six files in `audio/CONFIG/`, four of which give up their record width to
> a single division with no remainder, one of which announces its own count in a two-byte header, and one
> of which refuses to divide at all — because, as [C20.5](05-eventvol.md) shows, it has no records.

[← Chapter 20 hub](C20-Audio.md) · [Next: C20.2 — The SFX bank →](02-the-sfx-bank.md)

**Confidence:** ✅ Verified (all six tables' structure and counts) / ⏳ (`BankSlot` field meanings)

---

## 1. The two name tables

`PakFiles.dat` is 468 bytes and the `SFX` directory holds nine files. `468 / 9 = 52` exactly, and reading
the file at a 52-byte stride recovers the nine names in directory order:

```
offset    0:  46 45 45 54 00 cd cd cd  cd cd cd cd  00 00 …  "FEET"
offset   52:  47 45 4e 52 4c 00 cd cd  cd cd cd cd  00 00 …  "GENRL"
offset  104:  50 41 49 4e 5f 41 00 cd  cd cd cd cd  00 00 …  "PAIN_A"
   ⋮
offset  416:  53 50 43 5f 50 41 00 cd  cd cd cd cd  00 00 …  "SPC_PA"
```

The names are not merely plausible — every one of the nine **is** a file in `audio/SFX/`, so the table is
self-confirming. The structure of the record is legible from the padding: the name is NUL-terminated and
then padded with `0xCD` to **exactly twelve bytes** in every record, regardless of name length (`FEET` +
NUL + 7 × `CD`; `PAIN_A` + NUL + 5 × `CD`). The remaining 40 bytes are zero.

🟡 The `0xCD` run is Microsoft's debug-heap fill for uninitialised memory. The reasonable reading is that
the writing tool copied a twelve-byte name field out of a buffer it had not fully cleared, and that the
40 zero bytes are runtime fields the exporter left blank. ⏳ Their meaning is not derived; only that they
are zero in all nine records.

`StrmPaks.dat` is 272 bytes against sixteen files in `audio/streams/`. Sixteen does not divide 272 — but
**seventeen does**, at 16 bytes each, and reading at that stride recovers all sixteen names in the right
places with **slot 2 blank**:

| Slot | 0 | 1 | **2** | 3 | 4 | 5 | … | 16 |
|---|---|---|---|---|---|---|---|---|
| Name | `AA` | `ADVERTS` | *(empty)* | `AMBIENCE` | `BEATS` | `CH` | … | `TK` |

This is the difference between a fudge and a result. Rounding sixteen files up to "about 272 bytes" would
have hidden a real structural fact: the game's stream-pack index space has **seventeen** slots, one of
which is unused. And that fact is confirmed from an entirely different file — the 1,922 records of
`TrakLkup.dat` reference pack indices 0, 1, 3, 4 … 16 and **never 2**. Two files agree on the gap.

✅ *Verified:* `PakFiles.dat` = 9 × 52; `StrmPaks.dat` = 17 × 16 with index 2 unused.

## 2. The two directories share one record

`BankLkup.dat` is 8,520 bytes and `TrakLkup.dat` is 23,064. Both divide by twelve with no remainder —
**710** and **1,922** records — and both decode with the same layout:

| Offset | Type | Field |
|---:|---|---|
| 0 | `uint8` | pack index (into `PakFiles` / `StrmPaks`) |
| 1–3 | — | padding; `0xCC` in `BankLkup`, `0xCD` in `TrakLkup` |
| 4 | `uint32` | offset within the pack |
| 8 | `uint32` | size |

That the two files use **different** fill bytes for the same padding is a small forensic detail worth
recording: `0xCC` is the MSVC fill for uninitialised *stack*, `0xCD` for uninitialised *heap*. The two
tables were written by code that allocated its record differently. It has no effect on parsing — the game
reads a byte and three bytes of nothing — but it is evidence that these are two tools, not one.

The proof that offset and size mean what they appear to mean is in [C20.2](02-the-sfx-bank.md) and
[C20.3](03-the-stream-pack.md): each directory tiles its packs to the byte.

✅ *Verified:* both directories are `{ uint8 pack; uint8 pad[3]; uint32 offset; uint32 size }`;
710 banks across 9 packs, 1,922 tracks across 16.

## 3. `BankSlot.dat` announces its own count

216,902 is an awkward number. It divides by 2, 7 and 14 and nothing else useful — which is the signature
of a **header plus records** rather than records alone. Subtracting a two-byte header leaves 216,900,
and the first two bytes of the file read `2d 00` = **45**:

```
216,902 − 2 = 216,900 = 45 × 4,820
```

So the file is `uint16 numSlots = 45` followed by 45 records of 4,820 bytes. And 4,820 is not an arbitrary
number either — it is `16 + 400 × 12 + 4`, which means each slot embeds **the same 400-entry sound
descriptor array a bank header carries** ([C20.2](02-the-sfx-bank.md)), wrapped in a sixteen-byte prefix
and a four-byte tail. Reading at that stride confirms it: the twelve-byte pattern `{0, −1, 0, 0}` — an
unused sound descriptor — repeats at stride 12 from slot offset 28 to 4,804, exactly where entries 1
through 399 would sit.

| Offset in slot | Type | Observed |
|---:|---|---|
| 0 | `uint32` | a buffer offset (slot 0: 20,496) |
| 4 | `uint32` | a buffer size (slot 0: 724,416) |
| 8 | `int32` | −1 in all 45 slots |
| 12 | `int32` | −1 in all 45 slots |
| 16 | `SoundEntry[400]` | the bank descriptor array, mostly unused |
| 4,816 | `uint32` | 0 |

✅ *Verified:* the file is `2 + 45 × 4820`; the record embeds a 400-entry descriptor array.
⏳ The first two words are consistent with an offset and a length into a shared playback buffer, but the
45 pairs do **not** partition anything: their sizes sum to 8,578,144 bytes while their extent — the
highest `offset + size` of any slot — spans only 7,930,032, so they overlap. This chapter therefore does not call them a buffer allocation. `BankSlot.dat`
is best understood as a **snapshot of runtime state** rather than a designed table, and its field meanings
are left open.

⚠️ A tempting wrong turn: the slot sizes look like bank sizes, so one is tempted to match them against
`BankLkup`. **Zero of the 45 match** any bank size, and the largest slot (1,418,752) is a twelfth of the
largest bank (16,837,526). Whatever a slot holds, it is not a whole bank.

## 4. `EventVol.dat` does not divide

45,402 bytes. Its factorisation — `2 × 3 × 7 × 23 × 47` — offers candidate widths of 6, 14, 42, 46, 47,
69, 94, 141 and so on, none of which is more convincing than the others. Its contents are extraordinarily
sparse: **44,805 of 45,402 bytes are `0x80`**, and the 597 that are not fall into 69 contiguous runs whose
gaps share no common divisor. Nothing in the bytes proposes a record.

The executable settles the one thing that *can* be settled. The loader at `0x5b9d68`
([C20.4](04-the-loaders-in-the-executable.md)) pushes a literal length and refuses the file unless the
read returns exactly that many bytes:

```
005b9d86  push  0xB159            ; 45,401
005b9d8d  call  <read>
005b9d95  cmp   eax, 0xB159       ; must match exactly or the load fails
```

✅ *Verified:* the game reads **45,401** bytes — one fewer than the file contains. The last byte on disk
is never read.
⏳ The record structure is **not derived** *from the file*. Per house rule the chapter stops here rather
than adopting a layout it cannot prove — and the resolution, in [C20.5](05-eventvol.md), is that there was
never a record structure to find: the file is a flat `int8[45,401]` indexed by audio event ID. Tracing the
indexing arithmetic at the **use** sites, rather than factorising the file, is what settled it.

## 5. The table of tables

| File | Bytes | Structure | Tier |
|---|---:|---|---|
| `PakFiles.dat` | 468 | `9 × 52` — name[12] (`0xCD`-padded) + 40 zero bytes | ✅ |
| `StrmPaks.dat` | 272 | `17 × 16` — name[16], slot 2 blank | ✅ |
| `BankLkup.dat` | 8,520 | `710 × 12` — `{pack, pad×3, offset, size}` | ✅ |
| `TrakLkup.dat` | 23,064 | `1922 × 12` — same record, `0xCD` padding | ✅ |
| `BankSlot.dat` | 216,902 | `2 + 45 × 4820` | ✅ width / ⏳ fields |
| `EventVol.dat` | 45,402 | `int8[45,401]` indexed by event ID; `0x80` = unset ([C20.5](05-eventvol.md)) | ✅ |

---

### Key takeaways

- ✅ `468 = 9 × 52` and `272 = 17 × 16` fix the two name tables; the SFX names **are** the SFX filenames,
  so the table confirms itself.
- ✅ `StrmPaks` has **seventeen** slots, not sixteen — **slot 2 is blank**, and `TrakLkup` independently
  references every pack index except 2. Two files agree on the gap; rounding it away would have lost it.
- ✅ Both directories use one twelve-byte record, `{uint8 pack; pad[3]; uint32 offset; uint32 size}` —
  710 banks and 1,922 tracks.
- ✅ `BankSlot.dat` is `2 + 45 × 4820`, and 4,820 = `16 + 400 × 12 + 4` embeds the bank descriptor array;
  ⏳ its two leading words overlap and are **not** called an allocation.
- ✅ `EventVol.dat` reads **45,401 of 45,402** bytes — and has no record width because it has **no
  records**. See [C20.5](05-eventvol.md).
- 🧭 The padding fill differs between the two directories (`0xCC` vs `0xCD`) — evidence of two writing
  tools, not one.

**Continue:** [C20.2 — The SFX bank](02-the-sfx-bank.md)
