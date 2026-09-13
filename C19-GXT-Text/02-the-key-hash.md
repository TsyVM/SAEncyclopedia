# C19.2 — The Key Hash

> **The one-sentence version:** each line is keyed by a 4-byte value, the values are sorted for binary
> search, and the function is **CRC-32 with its final complement removed** — not guessed from a table but
> proven by taking the 1,186 text keys the mission scripts name in their own bytecode and watching **1,180
> of them land exactly on a stored key.**

[← C19.1 — The container](01-the-container.md) · [Chapter 19 hub](C19-GXT-Text.md) ·
[Next: C19.3 — Two languages →](03-two-languages.md)

**Confidence:** ✅ Verified (sorted invariant, hash identity by 99.5 % cross-reference)

---

## 1. The keys are sorted, so they are searched

A `TKEY` entry is eight bytes: a `uint32` offset into `TDAT` and a `uint32` key. Reading the key column of
every table:

| Check | Result |
|---|---:|
| Keys strictly ascending within a table | **127 / 127 tables** ✅ |
| Keys unique within a table | ✅ |

✅ **Every table's keys are sorted ascending.** Sorted keys mean the loader does a **binary search** —
`O(log n)` over 5,428 `MAIN` entries is 13 comparisons instead of thousands — which is only possible if the
key is a number, not the original string. So the key is a **hash**, and the sort order is the proof that it
is used as one.

That immediately explains why the format stores a hash at all. The mission scripts of
[Chapter 18](../C18-SCM-Script/C18-SCM-Script.md) fetch text by naming a key — a subtitle every few frames
during a cutscene — and a string compare against a 5,428-entry table each time is a cost the hash removes.

## 2. The keys' source is the script

The right-hand column of a `TKEY` block is a hash of *something*, and the something is not in the GXT file —
the original key names are discarded, exactly as the opcode-table names were absent from `main.scm`
([C18.3 §1](../C18-SCM-Script/03-the-code-stream.md)). But unlike the opcode table, **the key names survive
in another file we have already decoded.**

The SCM stores them as literal string arguments to its text opcodes. Grouping the string arguments of the
124 exactly-decoding scripts ([C18.4](../C18-SCM-Script/04-deriving-the-opcode-table.md)) by opcode, three
opcodes carry short uppercase codes that are plainly GXT keys rather than model or animation names:

| Opcode | Sample string arguments |
|---:|---|
| `0x00BC` | `INTROB` `INT1_AP` `INT2_F4` `INTRO2E` |
| `0x00BA` | `M_FAIL` `FUNERAL` `CAT_1` `ASS_ACQ` |
| `0x03D5` | `HELP21` `HELP26` `AMUHLP` `INTRO2G` |

🟡 *Reasoned:* these are the text-print and help-message commands, and their argument is the GXT key to
display. That reading is about to be confirmed by the hash itself.

## 3. The function is CRC-32 without the final complement

Hashing those keys and testing membership in the GXT key set ruled the obvious candidates out immediately —
Jenkins one-at-a-time (the function GTA uses elsewhere for name lookups), plain CRC-32, FNV, djb2, and every
case variant scored **zero**. The function that scored is a one-line variant of CRC-32:

```python
def gxt_key(name):
    return (binascii.crc32(name.upper()) ^ 0xFFFFFFFF) & 0xFFFFFFFF
```

That is the standard reflected CRC-32 (init `0xFFFFFFFF`, polynomial `0xEDB88320`) **with the final
`^ 0xFFFFFFFF` omitted** — the raw register value before CRC's usual finalisation. On the uppercased key.

| Check | Result |
|---|---:|
| Distinct GXT-key strings referenced by the scripts | **1,186** |
| Whose `gxt_key()` hash is a stored GXT key | **1,180 (99.5 %)** ✅ |

✅ **1,180 of 1,186 land exactly.** A wrong hash function scores near zero on a 16,588-of-2³² target; a
correct one scores ~100 %. This both **identifies the function** and **confirms §2's reading** that those
opcodes carry GXT keys — two results from one measurement, in the manner of the `SET_VAR_INT`/`_FLOAT`
derivation in [C18.3 §4](../C18-SCM-Script/03-the-code-stream.md).

⚠️ **Do not read the low miss rate as noise to round away.** The six misses are named and accounted for in
§4, not swept under the 99.5 %. A number reported without its exceptions is the error corrected twice in
[C18.3 §4](../C18-SCM-Script/03-the-code-stream.md) and [C10.1](../C10-2dEffect/01-the-record-and-corrections.md).

## 4. ⏳ The six that miss

The keys whose hash is **not** in `american.gxt` are `CM2_2`, `CM2_7`, `WZI2_B4`, `WZI2_9`, `BCOU_2`,
`BCOU_1`.

🟡 *Reasoned:* these are keys the script names but the American text file does not define — cut lines, or
strings assembled at runtime, or keys present only in another language's file. They are **not** hash
failures: the function is exact on the 1,180, and there is no reason a correct hash would miss six by
accident and hit 1,180 by design. The honest statement is that **six script keys have no American string**,
which is a fact about the content, not the hash.

This is the same discipline as the `AAA` outlier in
[C18.2 §3](../C18-SCM-Script/02-mission-and-external-tables.md): the exception is excluded *and named*, and
the headline figure is stated as 1,180 of 1,186 rather than rounded to "all."

## 5. Why this matters for a tool

A decompiler or a translation tool has to go **name → hash → line**, never the reverse, because the hash is
not invertible. The consequences:

- A tool that adds a new line must **hash the new key with `gxt_key()`** and insert it in **sorted order**,
  or the game's binary search will silently fail to find it.
- Two keys that collide under the hash cannot coexist; the format has no tie-break, so a mod author is
  relying on CRC-32's spread over a 16,588-entry table (⏳ no collision was observed in either shipped file,
  but the format does not prevent one).
- The key name is **gone** from the shipped file. Recovering it means a dictionary attack or, as here,
  reading it out of the **script** that references it — which is why C18 and C19 are worth decoding
  together.

---

### Key takeaways

- ✅ Keys are **strictly ascending in all 127 tables** — the signature of a **binary search**, which is only
  possible because the key is a **hash**, not the original string.
- The hash is **CRC-32 with the final complement removed**, on the uppercased key:
  `crc32(name.upper()) ^ 0xFFFFFFFF`.
- ✅ It is proven, not assumed: **1,180 / 1,186** GXT keys named in the [C18](../C18-SCM-Script/C18-SCM-Script.md)
  scripts hash exactly onto stored keys — identifying the function and confirming which script opcodes carry
  text keys in one stroke.
- ⏳ The **six misses are named** — keys the script references but `american.gxt` does not define — and are a
  content fact, not a hash failure.
- The mapping is **one-way**; tools must hash-then-insert-sorted, and a recovered key name has to come from
  the script, because the GXT file throws it away.

**Continue:** [C19.3 — Two languages](03-two-languages.md)
