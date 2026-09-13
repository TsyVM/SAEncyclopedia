# C2.4 — The Request Path

> **The one-sentence version:** `RequestModel` is mostly bookkeeping about *state you are already in* —
> it accumulates flags, guards a priority bit that is meaningless unless the model is already queued,
> and unlinks loaded models from the eviction list so that asking for something protects it.

[← C2.3 — The memory budget](03-memory-budget-and-stream-ini.md) · [Chapter 2 hub](C2-CStreaming.md)

**Confidence:** ✅ Verified
**Address:** `0x004087E0` — **in place** on this build ([C0.1](../C0-Binary-Identity/01-the-hoodlum-layer.md))

---

## 1. Signature and prologue

```
004087e0: push ebx
004087e1: mov  ebx, dword ptr [esp + 0xc]    ; arg2 = flags
004087e5: push ebp
004087e6: mov  ebp, dword ptr [esp + 0xc]    ; arg1 = model id
004087ec: lea  edi, [ebp + ebp*4]
004087f0: shl  edi, 2                        ; edi = id × 20
```

`__cdecl(int modelId, int flags)`. The record offset is computed once into `edi` and every subsequent
field access is `[edi + <base+offset>]` — the compiler folding the array base into the displacement,
which is why `0x008E4CC6` and `0x008E4CD0` look like standalone globals
([C2.1 §4](01-the-id-space.md)).

## 2. The priority guard

```
004087f9: cmp  al, 2                          ; loadState == requested?
004087fb: jne  0x408828
004087fd: test bl, 0x10                       ; caller asked for priority?
00408800: je   0x40882f
00408802: test byte ptr [edi + 0x8e4cc6], 0x10 ; already priority?
00408809: jne  0x40882f
0040880b: mov  ecx, dword ptr [0x8e4ba0]
00408811: mov  al,  byte ptr [edi + 0x8e4cc6]
00408817: inc  ecx                            ; ++priorityCount
00408818: or   al, 0x10
0040881a: mov  dword ptr [0x8e4ba0], ecx
00408820: mov  byte ptr [edi + 0x8e4cc6], al
00408826: jmp  0x40882f

00408828: test al, al                         ; loadState == 0 (not loaded)?
0040882a: je   0x40882f
0040882c: and  ebx, 0xffffffef                ; strip 0x10 from the incoming flags
```

✅ *Verified.* Three things fall out, and all three are the kind of behaviour that only shows up in the
instructions:

**Priority is only meaningful for something already in flight.** If `loadState == 2`, the bit is
accepted and the counter incremented. Otherwise — specifically when `loadState == 1`, *loaded* — the
`and ebx, 0xFFFFFFEF` strips it. Requesting an already-loaded model at high priority silently
downgrades to a normal request, which is correct: there is nothing to prioritise.

**The counter is guarded against double-counting.** `0x008E4BA0` is only incremented if the record did
not already have the bit. Ten priority requests for the same model increment it once.

**`loadState == 0` falls through both branches** and keeps its priority flag. A genuinely new request
may be priority; only the loaded case is stripped.

## 3. Flag accumulation

```
0040882f: mov  dl, byte ptr [edi + 0x8e4cc6]
00408835: lea  esi, [edi + 0x8e4cc0]          ; esi = &record
0040883b: or   dl, bl
0040883d: mov  byte ptr [esi + 6], dl
```

Flags are OR-ed, never assigned. From here `esi` holds the record address and the code switches from
`[edi + base+off]` to `[esi + off]` — the same struct, addressed two ways within one function.

## 4. The touch-on-request unlink

```
00408840: mov  al, byte ptr [edi + 0x8e4cd0]
00408846: cmp  al, 1                          ; loaded?
00408848: jne  0x4088db
0040884e: cmp  word ptr [esi], -1             ; already unlinked?
00408852: je   0x408990
00408858: cmp  ebp, 0x4e20                    ; model id < 20000?
```

If the model is **loaded** and **still linked**, it is spliced out of the list
([C2.2 §2](02-streaming-info-record.md)).

This is the heart of the eviction policy visible from here: **the list holds loaded-but-unreferenced
assets, and requesting one removes it from eviction candidacy.** The list is not an LRU of everything;
it is a free-list of things safe to throw away.

The `cmp ebp, 0x4e20` immediately after is the model/texture split
([C3.1](../C3-Model-Stores/01-the-partition.md)) — the two sides are unlinked from different lists or
with different accounting. That branch was not followed in this pass.

⏳ **Open:** what happens on the `jne 0x4088db` path (`loadState != 1`) — presumably the actual enqueue
for a not-yet-requested model. `RequestModel`'s *request* half is the part this page does not cover;
everything above is its *bookkeeping* half.

## 5. The counters this path maintains

| Global | Changed by | Semantics |
|---|---|---|
| `0x008E4BA0` | `RequestModel` ++, dispatcher −− | models with the priority bit set |
| `0x008E4CB8` | dispatcher −− | models currently requested |
| `0x008E4CB4` | dispatcher (`± cdSize × 2048`) | bytes resident |

The decrements all live in the unload dispatcher at `0x004089A0`
([C3](../C3-Model-Stores/C3-Model-Stores.md)), which mirrors this function: where `RequestModel` sets
the priority bit and increments, the dispatcher clears it and decrements, guarded the same way.

```
00408af1: mov  ecx, dword ptr [0x8e4cb8]
00408af7: mov  al,  byte ptr [edi + 0x8e4cc6]
00408afd: dec  ecx                            ; --numRequested
00408afe: test al, 0x10
00408b06: je   0x408b22
00408b08: mov  cl,  byte ptr [edi + 0x8e4cc6]
00408b13: and  cl, 0xef                       ; clear priority
00408b16: dec  eax                            ; --priorityCount
```

✅ Verified. The symmetry is exact, which is a good sign that both counters are correctly understood.

## 6. Hooking this function

`RequestModel` is the natural site for an `OnStreamingRequest` event: every request converges here
regardless of asset type, and it is **not relocated** on this build, so its prologue is real code and a
signature anchored at `0x004087E0` will match directly.

Two cautions:

- **It is called extremely often** — the streamer requests around the camera every frame. A logging
  hook here is a performance hazard, not a curiosity.
- **Do not infer "a load is starting."** As §2 and §4 show, most calls are re-requests for things
  already loaded or already queued. An event fired here means *asked for*, not *loading*.

---

### Key takeaways

- `__cdecl(modelId, flags)` at `0x004087E0`, **in place** on this build; record offset computed once as
  `id × 20`.
- **Priority (`0x10`) is only honoured when `loadState == 2`** and is stripped for already-loaded
  models.
- The priority counter is **guarded against double-counting** — repeated requests increment once.
- **Flags accumulate** by `or`, never assignment.
- Requesting a loaded model **unlinks it from the eviction list** — the list is a free-list of
  discardable assets, not an LRU of everything.
- The unload dispatcher mirrors this function exactly, decrementing what it increments — the symmetry
  validates both readings.
- Good hook site for a request event; **bad** place to assume a load is beginning.

**Continue:** [Chapter 2 hub](C2-CStreaming.md) · [Chapter 3 — The Model Stores](../C3-Model-Stores/C3-Model-Stores.md)
