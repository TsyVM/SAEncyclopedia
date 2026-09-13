# C2.1 — The Flat ID Space

> **The one-sentence version:** San Andreas addresses every streamable asset — model, texture,
> collision, map section, animation, script — through a single flat integer space of exactly 26,316
> entries, and that number is proved by where the array ends, not by counting anything.

[← Chapter 2 hub](C2-CStreaming.md) · [Next: C2.2 — The streaming-info record →](02-streaming-info-record.md)

**Confidence:** ✅ Verified

---

## 1. Why a flat space at all

The alternative design — one table per asset type, each with its own index — is what most engines do
and what the *stores* underneath actually look like ([C3](../C3-Model-Stores/C3-Model-Stores.md)). San
Andreas puts a flat ID space on top of them anyway, and the reason is visible in the record layout:
every streamable thing needs the same five pieces of bookkeeping (which archive, what offset, how big,
what state, where in the eviction list), and a flat space lets one array carry all of it.

The cost is that an ID no longer tells you what it refers to. That cost is paid once, in the dispatcher
([C3.1](../C3-Model-Stores/01-the-partition.md)), and it buys a streaming core that never branches on
asset type.

## 2. The array

```
base   0x008E4CC0
stride 0x14 (20 bytes)
count  26,316
end    0x009654B0
```

The stride comes from the index arithmetic, which appears at the head of every function that touches a
record:

```
004087ec: lea  edi, [ebp + ebp*4]     ; id × 5
004087f0: shl  edi, 2                 ; × 4  ->  id × 20
```

✅ *Verified.* The idiom is `lea`+`shl` rather than `imul` because multiplying by 20 decomposes into
`×5` (a single `lea` with scale 4) and `<<2`. Recognising it is how you spot streaming-record access
anywhere in the binary.

## 3. Why 26,316 is verified and not counted

There is no constant `26316` in the code to read. The count is established structurally:

```
0x008E4CC0 + 26316 × 20 = 0x009654B0
```

and `0x009654B0` is a **separately referenced global** — 19 cross-references, entirely distinct from
the array. The array ends exactly where its neighbour begins.

The neighbouring values do not work:

| Count | Computed end | Is that a real global? |
|---|---|---|
| 26,315 | `0x0096549C` | no |
| **26,316** | **`0x009654B0`** | **yes — 19 xrefs** |
| 26,317 | `0x009654C4` | no |

✅ This is the same technique that fixed both `CdStream` tables in
[C1.2 §2](../C1-Streaming/02-cdstream-layer.md), and it is worth naming because it recurs: **an array's
capacity is proved when its computed end lands on the next known object.** A capacity that merely
"looks like a round number" is 🟡; a capacity that closes a gap exactly is ✅.

The independent confirmation arrives in [C3.1](../C3-Model-Stores/01-the-partition.md): the eight
partition ranges, each read from a separate `cmp` immediate, sum to 26,316 on the nose. Two unrelated
derivations agreeing is what makes this the firmest number in the chapter.

## 4. The neighbourhood

The streaming globals cluster immediately below the array, and their reference counts are a rough map
of how central each is:

| Address | Refs | Role |
|---|---:|---|
| `0x008E4BA0` | 13 | priority request count |
| `0x008E4BB0` | 33 | count for the 8-slot side table |
| `0x008E4CB4` | 17 | memory used, bytes |
| `0x008E4CB8` | 13 | models requested |
| `0x008E4CC0` | 46 | **array base** |
| `0x008E4CC6` | 80 | array base + 6 → the **flags** byte |
| `0x008E4CD0` | 116 | array base + 0x10 → the **loadState** byte |
| `0x009654B4` | 41 | link-list array pointer |

Note that `0x008E4CC6` and `0x008E4CD0` are not separate globals — they are the array base plus a field
offset, which the compiler folded into the addressing mode. **116 references to `base+0x10`** is the
single loudest signal in the streaming system: `loadState` is what everything checks first.

This is also a small lesson in reading a stripped binary: a "global" with a suspiciously large
reference count that sits a few bytes above a known array base is usually a *field*, not an object.

---

### Key takeaways

- One flat space of **26,316 IDs** covers every streamable asset type; the type is implied by range, not
  stored.
- The record array is at **`0x008E4CC0`, stride 20**, recognisable by the `lea reg,[reg+reg*4]; shl reg,2`
  idiom.
- The count is **verified structurally** — `base + 26316 × 20` lands exactly on the next referenced
  global, and neighbouring counts do not.
- Independently corroborated by [C3.1](../C3-Model-Stores/01-the-partition.md), where eight separately
  read range boundaries sum to the same 26,316.
- `base+0x06` (flags) and `base+0x10` (loadState) appear as pseudo-globals with 80 and 116 references —
  folded addressing modes, not separate objects.

**Continue:** [C2.2 — The streaming-info record](02-streaming-info-record.md)
