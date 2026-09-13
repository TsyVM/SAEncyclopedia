# C3.2 — The Unhandled Band

> **The one-sentence version:** 475 streaming IDs fall through the dispatcher with no store call at
> all — a hole big enough to matter, deliberate enough to be interesting, and not yet explained.

[← C3.1 — The partition](01-the-partition.md) · [Chapter 3 hub](C3-Model-Stores.md) ·
[Next: C3.3 — Model-info polymorphism →](03-model-info-polymorphism.md)

**Confidence:** ✅ Verified (the gap exists) / 🟡 (what owns it) / ⏳ (why)

---

## 1. The instructions

```
00408a91: cmp  esi, 0x649b        ; 25755
00408a97: jge  0x408aaa           ; -> jump to the NEXT test, no handler
00408a99: lea  edx, [esi - 0x63e7] ; (this arm is range 6, animations)
00408a9f: push edx
00408aa0: call 0x4d3f40
00408aa5: add  esp, 4
00408aa8: jmp  0x408ac3

00408aaa: cmp  esi, 0x6676        ; 26230
00408ab0: jl   0x408ac3           ; -> fall through to the shared tail, NO handler
00408ab2: lea  eax, [esi - 0x6676]
00408ab8: push eax
00408ab9: mov  ecx, 0xa47b60
00408abe: call 0x4708e0
00408ac3: ...                     ; shared tail: memory accounting
```

✅ *Verified.* An ID in `25755 … 26229` satisfies `jge 0x408aaa` at the first test and `jl 0x408ac3` at
the second. It reaches the shared tail without any store being called.

**475 IDs** — 1.8 % of the space, and the third-largest range in the partition.

## 2. What still happens to it

Falling through is not the same as being ignored. The shared tail at `0x408AC3` runs for every ID:

```
00408ac3: mov  ecx, dword ptr [edi + 0x8e4ccc]   ; cdSize
00408ac9: mov  eax, dword ptr [0x8e4cb4]         ; memory used
00408ace: neg  ecx
00408ad0: shl  ecx, 0xb
00408ad3: add  eax, ecx
00408ad5: mov  dword ptr [0x8e4cb4], eax
```

So an asset in this band **is** accounted for in the streaming budget, **is** removed from the
eviction list, and **is** state-tracked like everything else. The only thing that does not happen is a
call telling some store to release its payload.

That asymmetry is the finding: the band is fully integrated into the streaming bookkeeping and fully
absent from the store dispatch.

## 3. The reading

🟡 *Reasoned:* the band is **vehicle recordings** — the `.rrr` path-playback data used for scripted
vehicle movement. Two things support it:

- **Position.** It sits between animations (range 6) and streamed scripts (range 8), which is where
  recordings fall in every community ID map.
- **Size.** 475 is a plausible count for recording slots and matches no other candidate asset class in
  the ID space.

And one thing explains the missing call: recordings are consumed by a playback subsystem that owns its
own buffers. There may be no store-style "release slot N" entry point to call, because the data is not
held in a slot-indexed store at all.

## 4. Why this is left open rather than closed

⏳ The honest position: this page verifies that **475 IDs have no dispatch arm** and that they are
otherwise fully tracked. It does not verify what they are.

Closing it needs one of:

- the **load** path for the same band — whatever reads the record and issues the `CdStreamRead` will
  name the consumer;
- a **cross-reference sweep** for other functions that index `0x008E4CC0` with an ID in this range;
- the **`.img` directory**, since recordings ship as `carrec.img` content and their count is countable
  from the archive rather than the binary.

The third is cheapest and was not attempted here.

## 5. Why it matters beyond curiosity

A 475-ID hole in the unload path has a practical consequence: **anything in this band that is streamed
in is never told to release**. Its memory accounting is decremented, so the budget believes it has room,
but no store dropped anything.

🟡 If the reading in §3 is right this is harmless — the playback subsystem frees its own buffers and the
streaming record is pure bookkeeping. If the reading is wrong, this is a leak with a 475-slot surface.

That is precisely why it is tagged rather than smoothed over. A chapter that silently labelled row 7
"vehicle recordings" and moved on would have hidden a question worth asking.

---

### Key takeaways

- **475 IDs (`25755 – 26229`) reach no store handler** — verified from the two comparisons that skip
  them.
- They are still **fully tracked**: budget accounting, eviction-list removal and state all run.
- 🟡 The band is very likely **vehicle recordings**, on position and size, with a plausible reason for
  having no store call.
- ⏳ Not confirmed. The cheapest confirmation is counting the recordings archive, which was not done
  here.
- The stakes: if the reading is wrong, this is a **475-slot leak** in which the budget is decremented
  but nothing is freed.

**Continue:** [C3.3 — Model-info polymorphism](03-model-info-polymorphism.md)
