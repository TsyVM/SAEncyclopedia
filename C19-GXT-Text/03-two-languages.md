# C19.3 — Two Languages

> **The one-sentence version:** `american.gxt` and `spanish.gxt` are the same 127 tables under the same
> names, keyed by the same hash of the same key strings — and where they differ (21 tables, three extra
> Spanish lines, 68 % of shared text translated, 12,771 accented bytes against 7) they differ exactly where
> translation forces it and nowhere else.

[← C19.2 — The key hash](02-the-key-hash.md) · [Chapter 19 hub](C19-GXT-Text.md)

**Confidence:** ✅ Verified (structural agreement, divergence census)

---

## 1. A second file that had to agree

The strongest test of a decode is a second instance of the format the decoder never saw while it was being
built. `spanish.gxt` is that instance, and it agrees on everything structural:

| Check | American | Spanish | |
|---|---:|---:|---|
| Header | `4, 8` | `4, 8` | ✅ |
| Tables | 127 | 127 | ✅ |
| Table names identical | — | **127 / 127** | ✅ |
| Keys sorted per table | ✅ | ✅ | ✅ |
| Key hash function | `crc32^~` | `crc32^~` | ✅ |

✅ **Same header, same 127 names, same hash.** A wrong record width or a misread header would not survive
being applied to a second, independently authored file — the agreement is the confirmation that
[C19.1](01-the-container.md) and [C19.2](02-the-key-hash.md) describe the format and not one file's
accident. It is the same class of evidence as the two-file IDE/placement agreement in
[C11](../C11-IDE-And-IPL/C11-IDE-And-IPL.md).

## 2. Where the key sets diverge — and by how little

Identical structure does not mean identical content, and the interesting part is the size of the gap:

| Check | Result |
|---|---:|
| Tables with a **byte-identical key-hash set** | **106 / 127** ✅ |
| Tables that differ | 21 |
| `MAIN` keys — American / Spanish | 5,428 / **5,431** |
| Spanish `MAIN` keys absent from American | **3** |

The 21 differing tables differ by a **handful of keys in both directions** — American has one more in `CAT`
(459 vs 458), Spanish has one more in `CASINO1`, `HEIST3`, and so on. This is not one file being a superset
of the other; it is **localisation drift**, each build gaining or losing the odd line.

🟡 *Reasoned:* the three Spanish-only `MAIN` lines are content the Spanish script needed and the American
one did not — all three are subtitle strings (`~z~¡Sube al jodido coche!` and two like it). The mechanism is
the same one C18 documents: a line exists in the text file because some script names its key, and the two
regional scripts are not byte-identical.

## 3. The text diverges exactly where meaning does

Among the 5,428 keys the two `MAIN` tables share:

| Measurement | Value |
|---|---:|
| Shared `MAIN` keys | 5,428 |
| Strings that **differ** (translated) | **3,715 (68 %)** |
| Strings **identical** across languages | 1,713 (32 %) |

✅ **Two-thirds are translated and one-third are left alone**, and the split is meaningful: the identical
strings are proper nouns and codes — `Ivor Williams`, `BHILL5a` — while the translated ones are the prose.
A tool that assumed a language file is a full translation would be wrong about a third of the table, and a
tool that assumed shared keys imply shared text would be wrong about the other two-thirds.

## 4. The 8-bit codepage, made visible

[C19.1 §1](01-the-container.md) proved the characters are 8-bit. The two files show *why that was a
decision with a cost*:

| Measurement | American | Spanish |
|---|---:|---:|
| Bytes ≥ `0x80` (non-ASCII) | **7** | **12,771** |

American text is essentially pure ASCII; Spanish spends 12,771 bytes on the high half of the codepage —
`á`, `í`, `ñ`, `¿`, `¡`. **One byte per character only works because a codepage supplies the accents**, and
the format commits every localisation to that single-byte page rather than the 16-bit encoding GTA III and
Vice City carried.

⚠️ A reader that decodes `spanish.gxt` as ASCII or UTF-8 corrupts one line in several; the bytes are a
Western 8-bit codepage and must be decoded as one. This is the text-format counterpart of the addressing-mode
warning in [C18.3 §5](../C18-SCM-Script/03-the-code-stream.md): the encoding is a property of the file that
nothing in the bytes announces, so it has to be stated.

## 5. What the pair says about the game

**The text system is one format, filled per region.** The engine ships identical machinery — 127 tables,
one hash, one codepage — and the language files are pure content. That is why adding a language is a data
job, not a code job, and why the modding scene can retext the whole game without touching the executable.

🟡 *Reasoned:* the small key-set differences mean the regional builds were compiled from **slightly
different scripts**, not a single master with swapped strings. If translation were a pure string swap, the
key sets would be identical (106/127) across the board; the 21 exceptions say the divergence happened
upstream, in the script, exactly where C18's per-region external tables would predict.

## 5. Five languages, of which two shipped

The retail tree contains two `.gxt` files, so the chapter above is a two-language study. The executable
says the format was built for more. At VA `0x870F70` in `.rdata` sits a contiguous run of filename
literals, immediately after the `TDAT` and `TKEY` tag strings:

```
0x870f60:  "TDAT"  "TKEY"
0x870f70:  "SPANISH.GXT"  "ITALIAN.GXT"  "GERMAN.GXT"  "FRENCH.GXT"  "AMERICAN.GXT"
```

and the language-selection code at `0x69FD01` is a five-way branch, one `push` per name, all converging on
the same `sprintf` and the same loader:

```
0069fd01  push  0x870fa0      ; "AMERICAN.GXT"   ┐
0069fd0d  push  0x870f94      ; "FRENCH.GXT"     │
0069fd14  push  0x870f88      ; "GERMAN.GXT"     ├─ all jump to 0x69FD36
0069fd20  push  0x870f7c      ; "ITALIAN.GXT"    │
0069fd2c  push  0x870f70      ; "SPANISH.GXT"    ┘
```

✅ *Verified:* the 1.0 US executable can load **five** GXT files — American, Spanish, Italian, German and
French — and the retail tree ships **two**. The other three are named in code and absent from disk.

This is recorded the same way [C21.3](../C21-Particles/03-the-parser-in-the-executable.md) records the
three particle behaviours the engine parses but no effect uses, and the same way
[C22.1](../C22-Map-Zones/01-the-zone-tables.md) records `NAVIG.ZON`: a **capability present in the
executable with no data behind it**. It says nothing about whether those builds exist elsewhere, and this
chapter does not speculate — only that this build could read them.

⏳ It does *not* resolve the header's first word. If that word were a language or region tag it would have
to vary, and it is `4` in both files here; with three of the five languages missing there is nothing
further to test it against.

---

### Key takeaways

- ✅ `spanish.gxt` **confirms the format**: same header, **127/127 table names**, same sorted-hash keying —
  a second file the decoder never trained on.
- ✅ The executable names **five** language files (`AMERICAN`, `SPANISH`, `ITALIAN`, `GERMAN`, `FRENCH`) in
  a five-way branch at `0x69FD01`; **three of them ship no data**.
- Key sets are **identical in 106/127 tables**; the 21 that differ do so by a few keys **in both
  directions**, and Spanish has **three `MAIN` lines** American lacks — localisation drift, not a superset.
- **68 % of shared `MAIN` strings are translated**, 32 % identical — the identical ones are proper nouns and
  codes, so neither "full translation" nor "shared key ⇒ shared text" is a safe assumption.
- The **8-bit codepage** is made visible: **7** non-ASCII bytes in American against **12,771** in Spanish —
  the single-byte design paying for accents with a codepage.
- 🟡 The small key-set differences imply the regional builds came from **slightly different scripts**, the
  same regionalisation C18's external tables encode.

**Continue:** [Chapter 19 hub](C19-GXT-Text.md)
