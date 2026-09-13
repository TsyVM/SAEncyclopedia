# C1.3 — The streaming request lifecycle and channel model

## What happens between "model needed" and "model in memory"

C1.1 and C1.2 documented the two containers: the IMG archive format and the CdStream request layer. This page traces the runtime path — what happens from the moment `CStreaming` decides it needs a model loaded to the moment the raw bytes are available in the destination buffer.

## The four phases

### Phase 1: Demand registration

`CStreaming::RequestModel` (C2) marks a model as "wanted" in its per-model status flags. The model ID maps to an entry in the streaming-info table (C31), which stores the IMG image index, sector offset, and sector count. The model is not yet queued for IO — it is only flagged as desired.

### Phase 2: Queue submission

`CStreaming::LoadAllRequestedModels` (the C2 frame-tick function) promotes demanded models to active IO by calling `CdStreamRead`:

```
CdStreamRead(imageIndex, sectorOffset, sectorCount, bufferPtr)
```

- `imageIndex` → the 32-slot image table index (C1.2§3) that maps to the file handle
- `sectorOffset` → the model's position within the IMG archive, in 2048-byte sectors
- `sectorCount` → number of sectors to read
- `bufferPtr` → the destination in the game's streaming buffer

The 48-byte request struct (C1.2§2) is filled and enqueued in the CdStream queue. The queue is bounded (⏳ — exact depth not traced), and if full, the submission fails silently and will be retried next frame.

### Phase 3: Asynchronous IO

A background thread (the CdStream IO thread, started during `CdStreamInit` at `0x406B20`) polls the queue and issues the actual `ReadFile` or `ReadFileEx` calls. The thread model means IO runs concurrently with the game loop — the main thread continues running frames while disk reads happen in the background.

When the read completes, the request's status byte in the 48-byte struct is updated to `1` (done). The game's streaming status flags for that model are also updated via a callback or poll.

### Phase 4: Model construction

After the bytes arrive in the buffer, `CStreaming` hands them to the appropriate store:
- `.dff` / `.txd` → `CTxdStore` / `CModelInfo` — RenderWare stream parsing
- `.col` → `CColStore` — collision parsing
- `.ipl` → world placement
- `.ifp` → animation store

The store parses the raw bytes out of the streaming buffer and builds the in-memory representation (a `RpClump`, a texture dictionary, a collision mesh). Only after this step can the model be used.

## The channel model: sequential reads per image

A key CdStream design constraint: **one active read per open image file at a time**. The 48-byte request struct has no concept of parallel IO to the same image. This is why the original PC game on a single spinning hard drive could stream effectively — it sequences reads in sector order to minimize seek distance.

If two subsystems simultaneously demand models from the same IMG, one read will queue behind the other. The order is determined by the queue's FIFO policy (⏳ — exact priority scheme not traced).

## The 32-image limit and its consequences

The 32-slot image table means SA can have at most 32 open IMG files simultaneously. The retail game opens 6 (three `gta3.img` variants for LOD, effects, etc., plus `player.img`, etc. — exact list ⏳). The remaining slots are available for mods.

Mods that add new `.img` files (custom content archives) consume image slots. Exceeding 32 slots causes `CdStreamOpenFile` to reject the extra files. Most modding tools that add content to the existing `gta3.img` avoid this problem by patching the existing archive rather than adding new ones.

## The sector handle and random-access guarantees

The packed sector handle `(imageIndex << 24) | sectorOffset` (C1.2§4) is what makes random access into a 2GB+ IMG file efficient. Since the IMG directory entry stores the sector offset as a 32-bit value, the entire archive content is accessible by a single 32-bit seek. The largest retail SA IMG is `gta3.img` at approximately 1.2 GB; at 2048 bytes/sector, the maximum addressable offset is `(2^24 - 1) × 2048 ≈ 34 GB`, so the 24-bit image index / 24-bit offset encoding has headroom well beyond any realistic IMG size.

## What this means for modders

- **Adding large models:** increasing a model's sector count requires patching the IMG directory entry's sector count field. The sector offset is also fixed in the directory — there is no fragmentation or free-space management at runtime. Rebuilding the IMG archive (with a tool like IMGtool or Alci's IMG editor) reorganizes directory entries and recalculates all offsets.
- **Streaming buffer size:** the streaming buffer must be large enough to hold the largest single model. SA's buffer size is a global (`0x8E4CB4` — ⏳, exact address per streaming-info derivation) that can be patched upward for mods with high-poly models.
- **Hot-reload:** there is no hot-reload — a model can only be loaded by the IO thread from an open image file. Mods that want to swap content at runtime must either use a custom ASI hook to intercept the `CdStreamRead` call, or replace the archive file between sessions.

**Previous:** [C1.2 — The CdStream layer](02-cdstream-layer.md)  
**Continue:** [C1.4 — Streaming and modding the IMG →](04-modding-streaming-and-img.md)
