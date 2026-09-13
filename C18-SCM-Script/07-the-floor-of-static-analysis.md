# C18.7 — The Floor of Static Analysis

> **The one-sentence version:** a search that lets **every** unknown opcode take **any** arity — 0 to 16,
> variadic, or an untagged inline string — still cannot walk 24 of the 26 open scripts to their declared
> end, which proves the remaining blocker is **not a missing arity** and redirects the whole line of
> attack.

[← Chapter 18 hub](C18-SCM-Script.md) · [Prev: C18.6 — The dispatch table](06-the-dispatch-table.md)

**Confidence:** ✅ Verified (the negative result, the forced arity, the resident buffer size) / ⏳ (why
the 24 scripts cannot be walked; three header constants)

---

## 1. What was believed, and what is now measured

[C18.4](04-deriving-the-opcode-table.md)–[C18.6](06-the-dispatch-table.md) took the opcode table to
**1,448 validated arities and 188 of 214 scripts closing exactly**. The 26 that did not close were
recorded as blocked by 26 opcodes whose arity the executable scan could not confirm, with the working
explanation that those handlers *compute their argument count at runtime*.

That explanation was a reasonable inference. It is now testable, and it does not survive the test.

The instrument is a **sound exhaustive forcer**: a depth-first walk in which any opcode absent from the
validated table may take any arity the encoding permits, with the [C18.5](05-the-opcode-ceiling.md)
opcode-ceiling prune (`op ≤ 0x0A8C`) to kill impossible successors. The candidate set is deliberately
wider than anything tried before:

| Candidate | Meaning |
|---|---|
| `0` … `16` | that many typed arguments |
| `V` | variadic — typed arguments until a `0x00` terminator |
| `('S', k)` | an **untagged NUL-terminated inline string**, then `k` typed arguments |

The untagged-string candidate is there because several open scripts contain long printable runs —
`WHEELO` carries `FIXED_CAMERA_POSITION - OFFSET FROM TABLE = ` — that no tagged argument type produces.

## 2. The result

Of the 26 open scripts, **two** admit at least one complete walk and **24 admit none at all**:

| Script | Walks reaching the end | Furthest position reachable |
|---|---:|---:|
| `MISSION37` | **4** | (closes) |
| `GYMDUMB` | **58** | (closes) |
| `BASKETB` | 0 | 98.3 % |
| `WHEELO` | 0 | 97.7 % |
| `MISSION26` | 0 | 97.5 % |
| `MISSION51` | 0 | 95.2 % |
| `MISSION50` | 0 | 94.4 % |
| `MISSION125` | 0 | 90.9 % |
| `BCESAR3` | 0 | 84.9 % |
| `MISSION19` | 0 | 65.7 % |
| `MISSION59` | 0 | 66.7 % |
| `BLACKJ` | 0 | 63.9 % |
| `MISSION9` | 0 | 61.5 % |
| `BARBER` | 0 | 54.9 % |
| `MAIN` | 0 | 51.9 % |
| `VIDPOK` | 0 | 46.9 % |
| `CLOTHES` | 0 | 45.2 % |
| `WARDROBE` | 0 | 32.4 % |
| `TATTOO` | 0 | 28.7 % |
| `MISSION69` | 0 | 27.0 % |
| `MISSION41` | 0 | 16.0 % |
| `MISSION24` | 0 | 12.8 % |
| `MISSION35` | 0 | 5.7 % |
| `MISSION8` | 0 | 2.7 % |
| `MISSION101` | 0 | 0.6 % |
| `MISSION97` | 0 | 0.5 % |

✅ *Verified:* **no assignment of arities whatsoever** — including variadic and untagged-string
encodings — permits a type-consistent linear walk of these 24 scripts to land on their declared end.

That is a strictly stronger statement than "the forcer forces nothing more". The earlier result said the
search could not *choose* between candidates. This one says there is nothing to choose **from**: the
candidate set is empty, and enlarging the arity vocabulary cannot help.

## 3. What it rules out, and what it leaves

The negative eliminates the runtime-argument-count hypothesis as the *cause of the blockage*. A handler
that computes its count at runtime would still consume some definite number of typed arguments on any
given occurrence, and the forcer tries every one of them. Something else is wrong.

⏳ Three explanations remain live, and this page does not choose between them:

**The code stream is not contiguous.** A linear walk assumes every byte between a script's start and its
declared end is an instruction or an instruction's argument. If a script embeds a data block that control
flow jumps over — a jump table, an alignment pad, an author's scratch region — no arity assignment can
consume it, and the walk must die exactly where the forcer says it does.

**An argument encoding is missing from the model.** The walker knows sixteen tagged types plus the
variadic terminator, all derived in [C18.3](03-the-code-stream.md). A seventeenth encoding used by a
handful of opcodes would produce precisely this signature.

**A declared end is wrong.** `MISSION97` and `MISSION101` die within 0.6 % of their start, which is more
consistent with a wrong *start* or a wrong boundary than with a subtle arity error deep inside.

The distribution itself is a clue worth recording: the failures are **bimodal**. Six scripts get past 90 %
and then die in the last few hundred bytes; six die in the first 20 %. Those are unlikely to be the same
defect.

## 4. One arity did fall out

`MISSION37` admits exactly four complete walks, and **all four assign the same arity to `0x088A`**:

```
0x088A  →  11        forced: identical in every walk that reaches the declared end
```

By the soundness argument of [C18.5](05-the-opcode-ceiling.md) — whichever walk is the real one, the
opcode's arity is the same — this is adoptable without knowing which walk is real. It also happens to
agree with the value the [C18.6](06-the-dispatch-table.md) executable scan proposed and could not confirm,
which is a second, independent source landing on the same number.

✅ **`0x088A = 11`** moves from ⏳ *unvalidated* to ✅ *verified*. The table is now **1,449 validated,
25 open**.

⚠️ Note what this does *not* do: `MISSION37` still fails to close deterministically, because two further
opcodes in it (`0x0861`, `0x0802`) remain ambiguous across the four walks. A forced arity and a closing
script are different things, and only the first is claimed.

`GYMDUMB` produces 58 complete walks and forces **nothing** — every one of its four unknown opcodes takes
a different value in different walks. Reported, not guessed.

## 5. A byproduct: the resident buffer

Chasing the SCM loader for the header constants turned up a number worth keeping. The loader at
`0x468EC4` reads `main.scm` into a fixed buffer at `0xA49960`:

```
00468ec4  push   0x859d60           ; "main.scm"
00468ec9  call   <open>
00468ece  push   0x30d40            ; 200,000 bytes
00468ed5  push   0xa49960           ; destination buffer
00468eda  push   esi
00468edb  call   <read>
```

✅ *Verified:* the game holds **200,000 bytes** of `main.scm` resident. The file is 3,079,744 bytes, so
the buffer covers the six segments and the main script only — and `missionBase` is **194,125**, which
fits with **5,875 bytes to spare**. Missions are streamed from disk individually, which is exactly why
segment 3 stores their absolute offsets.

That is an independent confirmation of the container arithmetic from
[C18.2](02-mission-and-external-tables.md): a value derived from the file (194,125) sits just under a
limit compiled into the executable (200,000), and the relationship is not a coincidence.

## 6. The three header constants remain open

⏳ Segment 3's `964`, segment 6's `569` and segment 1's leading byte `0x73` are **still not derived**.
Their positions are certain and the arithmetic around them closes ([C18.1](01-the-segment-chain.md)); the
loader above stores the buffer wholesale without decomposing the header at the load site, so the meanings
are at the *use* sites. Searching for the values as immediates does not discriminate — `964` occurs 42
times as a `dword` in the image and `115` occurs 445 times.

The productive next step is the one that worked for
[C20.5](../C20-Audio/05-eventvol.md): find the globals the parsed fields are written into and read their
consumers, rather than searching for the constants themselves.

---

### Key takeaways

- ✅ A forcer allowing **any** arity in `0…16`, plus **variadic**, plus **untagged inline strings**, with
  the sound opcode-ceiling prune, finds **no complete walk** for **24 of the 26** open scripts.
- ✅ That **refutes** the standing explanation: the blocker is *not* a missing or runtime-computed arity,
  because every possible arity is tried and none works.
- ✅ **`0x088A = 11`** is forced across all four complete walks of `MISSION37` — adopted, agreeing with the
  C18.6 executable scan. The table is **1,449 validated / 25 open**; scripts remain **188 / 214**.
- ⚠️ `GYMDUMB` admits 58 walks and forces **nothing**; `MISSION37` closes only under an assignment whose
  other two opcodes stay ambiguous. A forced arity is not the same as a closing script.
- 🧭 Failures are **bimodal** — six scripts die past 90 %, six inside the first 20 % — so probably more
  than one defect.
- ✅ The loader holds **200,000 bytes** of `main.scm` resident; `missionBase` = 194,125 fits with 5,875 to
  spare, independently confirming the container arithmetic.
- ⏳ Live explanations: non-contiguous code (embedded data), a seventeenth argument encoding, or a wrong
  declared boundary. The three header constants are still open, and should be chased at their **use**
  sites.

**Continue:** [← Chapter 18 hub](C18-SCM-Script.md)
