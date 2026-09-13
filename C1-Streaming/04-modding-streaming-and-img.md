# C1.4 — Modding the streaming pipeline and IMG archives

## What modders can change without patching the exe

The streaming pipeline's behavior is controlled by data files (the `.img` archives and their directory entries) and a handful of runtime globals. The IMG format itself is read-only at runtime — there is no write path in `CdStreamWrite`. All modding is therefore:
1. Offline archive editing (rebuild the `.img` with modified contents)
2. Runtime patching of streaming limits (via ASI, before `CdStreamInit` runs)

## Modifying model content: the IMG editor workflow

1. Open `gta3.img` (or whichever archive contains the target model) in an IMG editor
2. Extract the target `.dff`, `.txd`, or `.col` file
3. Modify it (in a DFF editor, TXD workshop, etc.)
4. Re-import: the editor replaces the old archive entry with the new file, recalculating the sector offset and count
5. Save the archive — this rewrites the 8-byte header (VER2 magic + entry count) and all 32-byte directory entries before the data payload

The runtime effect: on next game load, `CStreaming` reads the updated directory entries, sees the new sector offset and count, and the IO path streams the new bytes.

**Critical:** the game caches some streaming-info data at boot. A running game cannot pick up changes to the `.img` without a restart.

## Patching the streaming buffer for high-poly mods

SA's streaming buffer is large enough for the biggest model in the base game. High-poly replacement mods (upscaled vehicles, HD character models) can exceed this buffer and cause `CStreaming::LoadAllRequestedModels` to silently fail to load the model.

To raise the buffer:
```cpp
// Find the streaming buffer size global (address varies by mod setup)
// Patch before CStreaming::Initialise runs
const DWORD kStreamBufSizeAddr = 0x8E4CB4; // ⏳ verify for 1.0 US
VirtualProtect((void*)kStreamBufSizeAddr, 4, PAGE_EXECUTE_READWRITE, &old);
*(DWORD*)kStreamBufSizeAddr = new_size_in_bytes;
VirtualProtect((void*)kStreamBufSizeAddr, 4, old, &old);
```

Tools like `SA Limit Adjuster` handle this transparently based on a config file.

## Adding new IMG archives

To add a mod's content in a separate archive without modifying `gta3.img`:

1. Build a new VER2 IMG (8-byte header + 32-byte directory entries + data) using an IMG editor
2. Call `CdStreamOpenFile` with the path to the new archive — this consumes one of the 32 image slots
3. Register models from the new archive via `CStreamingInfo` patches that point to the new image index

The practical limit is that SA only opens its archives during `CStreaming::Initialise`. Adding archives from an ASI requires hooking `Initialise` or calling `CdStreamOpenFile` directly before any model loading.

## The model ID range and streaming-info table (C31)

Each model ID (0–20,000+) has a corresponding entry in the streaming-info table (C31). The streaming-info entry contains:
- Image index (which `.img` file)
- Sector offset and count
- Load status flags

To add a completely new model (new model ID), a mod must:
1. Add the new file to an `.img` archive
2. Write a streaming-info entry for the new model ID
3. Register the model in `peds.ide`, `vehicles.ide`, or `objs.ide` (C11)

The IDE file (C11) is how the model ID appears in the game world — the streaming system only knows "sector range," not "what kind of thing this is."

## CdStream error states and what causes them

| State | Cause | Observable effect |
|---|---|---|
| Read timeout | File handle invalid (closed or wrong path) | Model never finishes loading; spinning wheel in loading screen |
| Buffer overflow | Model sector count > buffer size | Model silently not loaded; invisible prop in world |
| Queue full | Too many pending requests (>32 simultaneous) | Requests deferred to next frame; temporary pop-in |
| Image slot exhausted | >32 images opened | Extra archives silently not opened |

The game has no user-facing error reporting for streaming failures — all failures are silent (the model simply does not appear). This makes diagnosing streaming issues the most common and hardest-to-debug class of SA modding problem.

## The relationship to CStreaming (C2) and CModelInfo (C3)

The CdStream layer (C1.2) is the lowest level — raw bytes, sectors, IO threads. Above it:
- `CStreaming` (C2) decides which models are needed, when to stream them in/out, and prioritizes requests
- `CModelInfo` (C3) takes the streamed bytes and constructs the RenderWare objects the game renders

A mod that hooks `CdStreamRead` gets control before the bytes are read. A mod that hooks `CModelInfo::AddModel` gets control after the bytes are parsed. The hook point determines what the mod can change about the streamed content.

**Previous:** [C1.3 — The streaming request lifecycle and channel model](03-streaming-cache-and-request-lifecycle.md)  
**Up:** [C1 — Streaming](C1-Streaming.md)
