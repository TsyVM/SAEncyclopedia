# C21.1 — The Grammar

> **The one-sentence version:** the format has exactly three variable-length constructs, each announced by
> its own `NUM_*` field, and a recursive-descent parser that trusts nothing but those three counts walks
> from `FX_PROJECT_DATA:` to `FX_PROJECT_DATA_END:` consuming **all 43,617 lines with zero remainder**.

[← Chapter 21 hub](C21-Particles.md) · [Next: C21.2 — The effect library →](02-the-effect-library.md)

**Confidence:** ✅ Verified (encoding, grammar, exact closure, all 29 schemas) / ⏳ (the version word)

---

## 1. The encoding, first

Before any grammar, establish what the bytes are:

| Property | Measurement |
|---|---:|
| File size | 616,708 bytes |
| CRLF line terminators | 43,617 |
| Bare `LF` (i.e. unpaired) | **0** |
| Bytes outside printable ASCII or CRLF | **0** |
| Blank lines | 1,958 |

✅ *Verified:* the file is **pure text with CRLF terminators**. There is no binary section, no length
prefix and no offset table anywhere in it — which is worth stating explicitly because the file extension
and the 616 KB size both suggest otherwise, and because it means every claim in this chapter is about a
*grammar* rather than a struct layout.

## 2. Only three things vary in length

Read a page of the file and the structure is obvious enough; the question is where each block *ends*,
because nothing is indented and there are no closing tags. There is exactly one terminator in the whole
file — `FX_PROJECT_DATA_END:`, which appears once.

The answer is that blocks do not need terminators, because **every variable-length list declares its own
length first**:

| Construct | Count field | Declared total | Blocks actually found |
|---|---|---:|---:|
| Emitters in a system | `NUM_PRIMS` | **161** | **161** ✅ |
| Info blocks in an emitter | `NUM_INFOS` | **1,470** | **1,470** ✅ |
| Keyframes in a curve | `NUM_KEYS` | **6,563** | **6,563** ✅ |

Every other block is a **fixed sequence of fields**. A `FX_KEYFLOAT_DATA:` is always three lines; a
`FX_PRIM_BASE_DATA:` is always ten. So the parser never has to guess: it either steps over a known number
of fields or over a declared number of children.

## 3. The grammar

```
project    := 'FX_PROJECT_DATA:' system* 'FX_PROJECT_DATA_END:'

system     := 'FX_SYSTEM_DATA:'  <version>            ← a bare number on its own line
              FILENAME NAME LENGTH LOOPINTERVALMIN LENGTH PLAYMODE CULLDIST BOUNDINGSPHERE
              'NUM_PRIMS: n'  prim{n}
              OMITTEXTURES TXDNAME

prim       := 'FX_PRIM_EMITTER_DATA:'
              'FX_PRIM_BASE_DATA:' NAME MATRIX TEXTURE TEXTURE2 TEXTURE3 TEXTURE4
                                   ALPHAON SRCBLENDID DSTBLENDID
              'NUM_INFOS: n'  info{n}
              LODSTART LODEND

info       := 'FX_INFO_<TYPE>_DATA:'  scalar*  ( <curveName> ':' interp )*

interp     := 'FX_INTERP_DATA:' LOOPED 'NUM_KEYS: n'  keyframe{n}

keyframe   := 'FX_KEYFLOAT_DATA:' TIME VAL
```

Two details in that grammar were derived the hard way and are worth flagging, because both are places a
first attempt goes wrong:

**`LENGTH` appears twice in a system**, separated by `LOOPINTERVALMIN`. A parser that treats field names
as a set rather than a sequence silently loses one of them.

**`LODSTART` and `LODEND` belong to the *emitter*, not the system.** They are written *after* the
emitter's info blocks, which makes them look like system-level trailing fields — and a parse that assumes
so runs off the rails at line 814 of the retail file, where a `LODSTART` turns up exactly where the next
`FX_PRIM_EMITTER_DATA:` was expected. The system's own trailing fields are `OMITTEXTURES` and `TXDNAME`,
and they appear only once per system, after the last emitter.

## 4. Where an info block ends

`info` is the one production without a count, so its extent has to be inferred — and this is the only
inference in the grammar. An info block consumes scalar fields and named curves until it reaches a line
that cannot belong to it: the start of the next info or emitter, the start of the next system, the end of
the project, or one of the enclosing blocks' trailing fields (`NUM_INFOS`, `LODSTART`, `OMITTEXTURES`).

That rule is a hypothesis, and §5 is what turns it into a result. If the boundary rule were wrong for even
one of the 1,470 info blocks, the parse would consume the wrong number of lines somewhere and could not
finish with an empty remainder.

There is a second, independent confirmation. Having parsed all 1,470 blocks, group them by type and ask
whether each type always has the same children:

✅ **All 29 types have exactly one child schema.** `FX_INFO_SIZE_DATA` is `TIMEMODEPRT` + the four curves
`SIZEX SIZEY SIZEXBIAS SIZEYBIAS` in all 161 of its appearances; `FX_INFO_EMRATE_DATA` is the single curve
`RATE` in all 144; and so on. A boundary rule that occasionally swallowed a line too many or too few would
produce a type whose observed schema varied between instances. None does.

## 5. The closure test

The proof is one number. Run the parser from byte 0, driven only by the grammar above and the three
declared counts, and see where it stops:

```
systems parsed  : 82
emitters parsed : 161
info blocks     : 1,470
curves          : 4,120
keyframes       : 6,563

lines consumed  : 43,617 of 43,617        LEFT OVER: 0
```

✅ *Verified.* Every line of the file is claimed by exactly one production. The parser reaches
`FX_PROJECT_DATA_END:` with nothing before it and nothing after it.

This is the text-format analogue of a binary table tiling its span, and it fails loudly under error. A
mis-declared count, a missed field, or a wrong block boundary shifts the parse and it either asserts on an
unexpected token or finishes with a non-zero remainder. Getting `0` after 43,617 lines and five levels of
nesting is not something an incorrect grammar does.

## 6. Two constants, left open

Each `FX_SYSTEM_DATA:` tag is followed by a **bare number on its own line** — no key, no colon:

```
FX_SYSTEM_DATA:
109
```

⏳ It is `109` on **all 82** systems. Position and type are certain; meaning is not derived. It is
*not* a system count (there are 82), and it is not a length. A version or format stamp is the obvious
reading and this chapter declines to assert it.

⏳ Every system ends `OMITTEXTURES: 0` and `TXDNAME: NOTXDSET`, identically, on all 82. Constant fields
carry no information to derive from, so both are recorded as positioned and unexplained. `NOTXDSET` reads
as a sentinel meaning "no texture dictionary override", but with 82 identical values there is nothing to
prove it against.

---

### Key takeaways

- ✅ The file is **pure CRLF text** — 43,617 lines, 1,958 blank, **zero** bytes outside printable ASCII.
- ✅ Only **three** constructs vary in length, and each declares its own count: `NUM_PRIMS` → 161,
  `NUM_INFOS` → 1,470, `NUM_KEYS` → 6,563 — each matching the blocks found.
- ✅ The count-driven parser consumes **43,617 / 43,617** lines with **0 left over** — the text-format
  equivalent of a table tiling its span.
- ✅ All **29** `FX_INFO` types have a **single constant schema**, an independent check on the one
  inferred rule in the grammar (where an info block ends).
- ⚠️ **`LODSTART`/`LODEND` belong to the emitter, not the system**, and are written after the emitter's
  info blocks. **`LENGTH` appears twice** per system. Both trip a naive parser.
- ⏳ The bare `109` after each `FX_SYSTEM_DATA:`, and the constant `OMITTEXTURES: 0` / `TXDNAME: NOTXDSET`,
  are positioned but not interpreted.

**Continue:** [C21.2 — The effect library](02-the-effect-library.md)
