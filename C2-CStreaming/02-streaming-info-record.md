# C2.2 — The Streaming-Info Record

> **The one-sentence version:** twenty bytes per asset — two list links, a load-order link, flags, an
> archive id, a packed sector handle, a size and a state byte — of which four fields are proved by
> instruction and three are honestly marked as inference.

[← C2.1 — The flat ID space](01-the-id-space.md) · [Chapter 2 hub](C2-CStreaming.md) ·
[Next: C2.3 — The memory budget →](03-memory-budget-and-stream-ini.md)

**Confidence:** ✅ Verified (4 fields) / 🟡 Reasoned (3 fields)

---

## 1. The layout

```c
struct CStreamingInfo {      // 0x14 = 20 bytes
    uint16_t nextIndex;      // +0x00  ✅ eviction-list forward link
    uint16_t prevIndex;      // +0x02  ✅ eviction-list back link
    uint16_t nextIndexOnCd;  // +0x04  🟡 load-order chain
    uint8_t  flags;          // +0x06  ✅ request flags
    uint8_t  imgId;          // +0x07  🟡 archive index
    uint32_t cdPosn;         // +0x08  🟡 sector offset
    uint32_t cdSize;         // +0x0C  ✅ size in sectors
    uint8_t  loadState;      // +0x10  ✅
    // +0x11..0x13 padding
};
```

## 2. Field by field

### `+0x00` / `+0x02` — the list links ✅

Proved by the unlink sequence in `RequestModel`, which is a textbook doubly-linked-list splice
performed with 16-bit indices instead of pointers:

```
0040885e: movsx eax, word ptr [esi]          ; next
00408861: mov   cx,  word ptr [esi + 2]      ; prev
00408865: mov   edx, dword ptr [0x9654b4]    ; the link array
0040886b: lea   eax, [eax + eax*4]           ; next × 20
0040886e: mov   word ptr [edx + eax*4 + 2], cx    ; link[next].prev = prev
00408873: movsx eax, word ptr [esi + 2]      ; prev
00408877: mov   cx,  word ptr [esi]          ; next
0040887a: mov   edx, dword ptr [0x9654b4]
00408880: lea   eax, [eax + eax*4]
00408883: mov   word ptr [edx + eax*4], cx        ; link[prev].next = next
00408887: mov   word ptr [esi],     0xffff
0040888c: mov   word ptr [esi + 2], 0xffff
```

Both neighbours are patched, then both links are set to `0xFFFF`. **`0xFFFF` is the null sentinel**, and
the list lives in a *second* array reached through `0x009654B4` — also stride 20, also indexed by
streaming ID.

That the links are `movsx`-loaded (sign-extended) while the sentinel is `0xFFFF` is consistent: as a
signed 16-bit value that is `-1`.

### `+0x06` — flags ✅

```
0040882f: mov  dl, byte ptr [edi + 0x8e4cc6]
0040883b: or   dl, bl                        ; incoming request flags
0040883d: mov  byte ptr [esi + 6], dl
```

Flags are **accumulated, never replaced** — two requests for the same model with different flags
produce the union. The only bit whose meaning is established here is **`0x10` = priority**
([C2.4](04-the-request-path.md)).

### `+0x0C` — size in sectors ✅

```
00408ac3: mov  ecx, dword ptr [edi + 0x8e4ccc]   ; base + 0x0C
00408ac9: mov  eax, dword ptr [0x8e4cb4]         ; memory used
00408ace: neg  ecx
00408ad0: shl  ecx, 0xb                          ; × 2048
00408ad3: add  eax, ecx
00408ad5: mov  dword ptr [0x8e4cb4], eax
```

The field is a **sector count**, converted to bytes only at the accounting boundary. `neg` then `add`
rather than `sub` is just the compiler's choice.

This is the seam described in the hub: `shl 0xB` is the exact instruction where the sector world of
[C1](../C1-Streaming/C1-Streaming.md) becomes the byte world of the budget.

### `+0x10` — load state ✅

```
00408840: mov  al, byte ptr [edi + 0x8e4cd0]
00408846: cmp  al, 1                      ; loaded?
004089b2: test al, al                     ; not loaded at all?
004089be: cmp  al, 1
00408aed: cmp  al, 2                      ; requested?
```

| Value | Meaning |
|---|---|
| `0` | not loaded |
| `1` | loaded |
| `2` | requested / in flight |

With **116 references** to this address it is the most-read field in the streaming system, which is
what you would expect: nearly every entry point begins by asking what state the asset is in.

✅🔷 **Closed** in [X1 §3.2](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md): five states exist —
`3 = CHANNELED`, `4 = FINISHING`. Only 0/1/2 appear in the request and unload paths read here, which is
consistent: 3 and 4 live in the loading code.

## 3. The three inferred fields

`+0x04`, `+0x07` and `+0x08` are marked 🟡 deliberately. They are the conventional reading, they are
consistent with the 20-byte total and with the gaps left by the four verified fields, and `+0x08`
holding a packed sector handle would match `CdStreamRead`'s parameter exactly
([C1.2 §4](../C1-Streaming/02-cdstream-layer.md)).

But this pass did not read an instruction that proves any of them, and the encyclopedia's rule is that
consistency is not proof. They are recorded so the layout is complete and flagged so nobody builds on
them.

✅🔷 **Closed** in [X1 §3.1](../X1-SDK-Cross-Reference/X1-SDK-Cross-Reference.md). All three readings
were correct: `+0x04 m_nNextIndexOnCd`, `+0x07 m_nImgId`, `+0x08 m_nCdPosn` — the last confirming that
it holds the packed sector handle of [C1.2 §4](../C1-Streaming/02-cdstream-layer.md).

## 4. Twenty bytes, and why it matters

At 26,316 entries × 20 bytes the array is **526,320 bytes** — half a megabyte of always-resident
bookkeeping, against a streaming budget of 13.8 MB
([C2.3](03-memory-budget-and-stream-ini.md)). The metadata is 3.8 % of the payload budget.

That ratio explains the design pressure visible in the record: 16-bit list indices instead of 32-bit
pointers, a byte for state, a byte for archive id, sizes in sectors instead of bytes. Every field is as
narrow as it can be. The 3-byte tail padding after `loadState` is the only slack, and it exists only
because the struct is aligned to 4.

---

### Key takeaways

- **20 bytes per asset**; four fields verified by instruction, three marked as inference rather than
  quietly asserted.
- The eviction list is **indices, not pointers** — 16-bit, with `0xFFFF` as null, in a second array
  reached via `0x009654B4`.
- **Flags accumulate** (`or`), so repeated requests union their flags.
- `cdSize` is in **sectors**; `shl 0xB` at the accounting site is the sector→byte seam.
- `loadState` has **116 references** — the most-read field in the subsystem; values 0/1/2 observed.
- The array costs **526,320 bytes** of always-resident metadata, which is why every field is minimally
  sized.

**Continue:** [C2.3 — The memory budget & `stream.ini`](03-memory-budget-and-stream-ini.md)
