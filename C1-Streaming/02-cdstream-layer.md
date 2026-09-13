# C1.2 — The CdStream Layer

> **The one-sentence version:** `CdStream` is a fixed array of 48-byte request slots served by a
> worker thread over overlapped I/O, addressed by a single 32-bit word that packs an image index into
> the top byte and a sector offset into the low 24 — and its two most important functions live in
> `.HOODLUM` on this build.

[← C1.1 — The IMG VER2 archive model](01-img-ver2-archive-model.md) · [Chapter 1 hub](C1-Streaming.md)

**Confidence:** ✅ Verified by disassembly
**RE status:** Verified

---

## 1. The request struct

Recovered field-by-field from the executable. Every offset below was read off an instruction that
touches it; nothing is inferred from a template.

```c
struct CdStream {           // 0x30 bytes
    uint32_t sectorOffset;  // +0x00  24-bit LBA within the image
    uint32_t sectorCount;   // +0x04  sectors to read; 0 = slot idle
    void*    buffer;        // +0x08  destination
    uint8_t  _pad0C;        // +0x0C  (not observed being read)
    uint8_t  bLocked;       // +0x0D  set before waiting on the semaphore
    uint8_t  bInUse;        // +0x0E  set while the request is live
    uint8_t  _pad0F;        // +0x0F
    int32_t  status;        // +0x10  result; cleared on read
    HANDLE   semaphore;     // +0x14  CreateSemaphoreA(NULL, 0, 2, NULL)
    HANDLE   hFile;         // +0x18  copied from the image handle table
    OVERLAPPED overlapped;  // +0x1C  20 bytes
};                          // = 0x30
```

### Why `0x30` is exactly right

The stride is independently established three times, which is what makes it verified rather than
assumed:

| Evidence | Where |
|---|---|
| `lea eax,[eax+eax*2]; shl eax,4` — index × 3 × 16 = × 48 | `CdStreamSync` body `0x0156CD80` |
| `lea ecx,[eax+eax*2]; shl ecx,4` — same idiom | `CdStreamGetStatus` body `0x01563850` |
| `add esi, 0x30` per iteration when walking the array | `CdStreamInitThread` `0x004068F0`, `CdStreamShutdown` `0x00406370` |

And structurally: `OVERLAPPED` is 20 bytes, so `0x1C + 0x14 = 0x30`. The struct ends exactly where the
last field does — the size is not padded, it is *explained*.

✅ *Verified.* `+0x1C` is confirmed as `OVERLAPPED` by its use, not its size: `CdStreamSync` calls
`GetOverlappedResult(hFile = [esi+0x18], lpOverlapped = [esi+0x1C], &bytesTransferred, …)`.

---

## 2. The globals

| Address | Type | Role | Evidence |
|---|---|---|---|
| `0x008E3FE0` | `u32` | `CreateFileA` flag word, computed at init | ✅ |
| `0x008E3FE4` | `u32` | worker-thread mode enabled | ✅ |
| `0x008E3FE8` | `u32` | overlapped mode enabled (set to 1 at init) | ✅ |
| `0x008E3FEC` | `void*` | request ring buffer | ✅ |
| `0x008E3FF4` | `u32` | ring write index | ✅ |
| `0x008E3FF8` | `u32` | ring modulus | ✅ |
| `0x008E3FFC` | `CdStream*` | channel array base | ✅ |
| `0x008E4004` | `HANDLE` | semaphore released once per queued request | ✅ |
| `0x008E4008` | `HANDLE` | second sync object, closed at shutdown | 🟡 |
| `0x008E4010` | `HANDLE[32]` | open image file handles | ✅ |
| `0x008E4090` | `i32` | channel count | ✅ |
| `0x008E4094` | `i32` | open image count | ✅ |
| `0x008E4098` | `char[32][64]` | image path table | ✅ |
| `0x008E4898` | `u32` | last read end (`buffer + offset`) — diagnostic | 🟡 |

⏳ **Open:** `0x008E3FF0`, `0x008E4000` are referenced by the family but their roles were not pinned
down in this pass.

### The two array bounds are proved by their neighbours

This is the neatest result in the chapter. Both tables' sizes are confirmed by the address of the next
known global — the arrays fit their slots exactly:

- **Handles:** `CdStreamInit` does `mov ecx, 0x20; mov edi, 0x8E4010; rep stosd` — **32 slots** × 4 =
  128 bytes → `0x8E4010 + 0x80 = 0x8E4090`, which is the channel count. ✅
- **Paths:** `CdStreamInit` walks `edx = 0x8E4098`, `add edx, 0x40`, `cmp edx, 0x8E4898` — **32
  entries × 64 bytes** = 2,048 bytes → ends precisely at `0x8E4898`. ✅

Neither bound was guessed from a plausible power of two; both were read out of the loop that clears
them and then corroborated by the layout closing exactly.

---

## 3. The function family

| Entry VA | Name (behavioural) | Location on this build | What it does |
|---|---|---|---|
| `0x00406360` | `CdStreamShutdown` | **stub → `0x015700D0`**, real code at `0x00406370` | `LocalFree` the ring; close both sync objects; close every channel semaphore |
| `0x004063E0` | `CdStreamGetStatus` | **relocated → `0x01563850`** | poll: `0xFF` if `bInUse`, `0xFA` if `sectorCount != 0`, else return and clear `status` |
| `0x00406460` | `CdStreamSync` | **relocated → `0x0156CD80`** | block on the channel semaphore (async) or `GetOverlappedResult` (sync); clear `bInUse`; return `status` |
| `0x00406690` | `CdStreamRemoveImages` | in place | sync every channel, then `CloseHandle` all 32 images and clear their paths |
| `0x004067B0` | `CdStreamOpen` | **relocated → `0x01564A90`** | find a free handle slot, `CreateFileA`, store the path, **return `index << 24`** |
| `0x004068F0` | `CdStreamInitThread` | in place | one `CreateSemaphoreA(NULL,0,2,NULL)` per channel → `+0x14`; `LocalAlloc` the ring; start the worker |
| `0x00406A20` | `CdStreamRead` | **relocated → `0x0156C2C0`** | split the packed handle, fill a slot, push the ring, release the semaphore |
| `0x00406B70` | `CdStreamInit` | in place | zero both tables; `GetDiskFreeSpaceA`; compute the flag word |

🟡 *Reasoned:* the names are behavioural descriptions chosen to match long-standing community usage.
The *behaviour* of each is ✅ verified by disassembly; the *names* are a convention, and the SDK will
bind to symbol IDs rather than to these strings.

**Four of eight are relocated.** Two of those four — `CdStreamRead` and `CdStreamSync` — are the
functions any streaming mod hooks first.

---

## 4. The packed sector handle, both ends

`CdStreamOpen` ends:

```
01564b9a: mov  eax, esi          ; esi = free slot index
01564b9d: shl  eax, 0x18         ; index << 24
01564ba1: ret
```

`CdStreamRead` begins:

```
0156c2d8: mov  edx, esi          ; esi = the packed word
0156c2e2: shr  edx, 0x18         ; image index
0156c2e5: mov  eax, [edx*4 + 0x8e4010]   ; -> HANDLE
0156c2fb: and  esi, 0xffffff     ; sector offset
0156c301: mov  [edi + 0x18], eax ; stash the handle in the request
```

✅ *Verified* from both sides. Callers OR a sector offset into the value `CdStreamOpen` handed them,
and never need to know which archive it refers to.

**The limits this creates:**

| Limit | Value | Imposed by |
|---|---|---|
| Concurrent archives | **32** | the `rep stosd` bound and the path table, *not* the 8-bit field |
| Offset per archive | 2²⁴ sectors = **32 GiB** | `and esi, 0xFFFFFF` |

The 8-bit index field would allow 256 archives; the tables hold 32. **The binding constraint is the
table, not the encoding** — which is why raising the archive limit is a matter of relocating two
arrays rather than changing the handle format.

---

## 5. Open, read, complete

**`CdStreamInit`** (`0x00406B70`) computes the flag word once:

```
00406bb4: call [0x8580c0]                ; GetDiskFreeSpaceA
00406bc0: cmp  ecx, 0x800                ; bytes per sector > 2048?
00406bc6: ja   0x406bcd
00406bc8: mov  eax, 0x20000000           ; FILE_FLAG_NO_BUFFERING
00406bcd: ...
00406bd1: or   eax, 0x40000000           ; FILE_FLAG_OVERLAPPED (always)
00406be5: mov  [0x8e3fe0], eax
```

✅ *Verified.* Unbuffered I/O is enabled **only if the volume's sector size is ≤ 2048** — because the
IMG sector is 2048 and `NO_BUFFERING` requires reads aligned to the physical sector. On a 4K-native
drive the check fails and the game falls back to buffered reads. That is a real, still-relevant
compatibility behaviour recovered from four instructions.

**`CdStreamOpen`** then issues:

```
CreateFileA(path,
            GENERIC_READ,                       // 0x80000000
            FILE_SHARE_READ,                    // 1
            NULL,
            OPEN_EXISTING,                      // 3
            [0x8E3FE0] | FILE_FLAG_RANDOM_ACCESS | FILE_ATTRIBUTE_READONLY,   // | 0x10000001
            NULL)
```

**`CdStreamRead`** fills the slot and queues it:

```
0156c331: mov  [edi+0x10], 0      ; status = 0
0156c338: mov  [edi],     esi     ; sectorOffset
0156c33a: mov  [edi+0x04], ebx    ; sectorCount
0156c33d: mov  [edi+0x08], eax    ; buffer
0156c344: mov  [edi+0x0d], 0      ; bLocked = 0
...
0156c35e: mov  [edx+ecx*4], eax   ; ring[write] = channel
0156c361: mov  eax, [0x8e3ff4]
0156c366: inc  eax
0156c368: idiv dword ptr [0x8e3ff8]   ; write = (write+1) % size
0156c379: mov  [0x8e3ff4], edx
                                   ; then ReleaseSemaphore([0x8E4004], 1, 0)
```

Note `SetLastError(0)` is called before the queue is touched — the layer uses the thread's last-error
slot as part of its own status reporting.

**`CdStreamSync`** collects, choosing its path from the mode globals:

```
0156cd94: mov  eax, [0x8e3fe4]    ; worker-thread mode?
0156cd9d:   mov eax,[esi+4]       ;   yes: if sectorCount != 0
0156cda4:   mov byte [esi+0xd], 1 ;        bLocked = 1
0156cda8:   WaitForSingleObject([esi+0x14], INFINITE)
0156cdb4: mov  byte [esi+0xe], 0  ; bInUse = 0
0156cdb8: mov  eax, [esi+0x10]    ; return status
```

with the `0x8E3FE8` branch instead calling `GetOverlappedResult`. Both converge on the same two lines:
clear `bInUse`, return `status`.

---

## 6. Status codes

`CdStreamGetStatus` returns:

| Value | Meaning |
|---|---|
| `0xFF` (255) | `bInUse` set — the channel is busy |
| `0xFA` (250) | `sectorCount != 0` — queued, not yet started |
| `status` | the stored result, **and the slot is cleared as a side effect** |
| `0` | nothing pending |

⚠️ The read is destructive: `mov eax,[ecx+0x10]; mov dword [ecx+0x10], 0`. Calling `GetStatus` twice
returns the real result once and `0` thereafter. Any hook that polls this function for logging will
consume results the game needed.

---

## 7. Consequences for SASDK

- **`SA::Streaming::Read` must take a symbol, not `0x406A20`.** On this build that address is a
  `jmp`; `SA::Hook::Install("CdStreamRead", …)` resolves entry-vs-body per
  [C0.2 §2](../C0-Binary-Identity/02-build-fingerprint-and-address-resolver.md).
- **Signatures for the four relocated functions must be authored against their `.HOODLUM` bodies** and
  the scanner must cover that section — the worked `CdStreamRead` pattern in C0.2 §6 is one of these.
- **`OnStreamingLoad` belongs on `CdStreamRead`'s entry**, where all callers converge, not on the
  worker thread.
- **Do not wrap `CdStreamGetStatus` naively** (§6) — the SDK should cache rather than re-poll.

---

### Key takeaways

- `CdStream` is a **48-byte** request slot; the size is *explained* by `OVERLAPPED` at `+0x1C`, and the
  stride is confirmed by three independent instruction idioms.
- **32 image slots and 32 × 64-byte paths**, both bounds proved by the loops that clear them and by the
  next global sitting exactly where each table ends.
- The **packed handle** (`index << 24 | sectorOffset`) is verified at both ends — produced by
  `CdStreamOpen`, split by `CdStreamRead`.
- The real archive limit is **32, imposed by the tables, not by the 8-bit field** that would allow 256.
- `FILE_FLAG_NO_BUFFERING` is applied **only when the volume sector size is ≤ 2048** — a live
  compatibility behaviour on 4K-native drives.
- **`CdStreamGetStatus` clears the status it returns** — a trap for anything that polls it.
- **Four of the eight family members are relocated**, including the two most-hooked. Without C0's
  relocation map this page could not have been written.

**Continue:** [Chapter 1 hub](C1-Streaming.md) · next chapter: `C2 — CStreaming & the Model-ID Space`
