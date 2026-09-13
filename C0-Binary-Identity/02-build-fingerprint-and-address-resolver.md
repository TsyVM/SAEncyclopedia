# C0.2 — The Build Fingerprint & the Address Resolver

> **The one-sentence version:** because two files both correctly called "1.0 US" disagree about where
> 492 function bodies live, an address in SAEncyclopedia is not a number — it is a **pair** (entry,
> body) resolved against an **identified build**, and this entry specifies the identification scheme,
> the pair model, and the resolver that SASDK is built on.

**Subsystem category:** Binary substrate (pre-engine)
**Depends on:** [C0.1 — Binary Identity: the SecuROM Wrapper & the HOODLUM Layer](01-the-hoodlum-layer.md)
**RE status:** Documented (design); backing data ✅ Verified
**SDK exposure:** This *is* the SDK's foundation layer
**Data artifact:** [`RE-Data/data/hoodlum_relocation_map.json`](../RE-Data/data/hoodlum_relocation_map.json) — 492 relocations, generated

---

## 1. The problem, stated precisely

C0.1 established the finding. Here is the consequence in one line of pseudocode — the thing every
existing GTA SA toolkit does:

```cpp
auto CdStreamRead = reinterpret_cast<CdStreamReadFn>(0x406A20);   // "1.0 US"
```

On a Compact exe this points at `CdStreamRead`'s first instruction. On the binary in this tree it
points at `jmp 0x156C2C0`. **Both are correct.** Calling through it works in both cases. But:

- disassembling 32 bytes there gets you a function in one case and a thunk in the other;
- comparing a byte signature there matches in one case and not the other;
- copying 5 bytes for a trampoline yields real instructions in one case and a position-dependent
  `E9` in the other.

The single number `0x406A20` is doing two jobs and only succeeding at one of them. It is a **call
target**. It is not reliably a **code location**. Conflating those is the defect; separating them is
the whole design.

---

## 2. The two-address model

Every function symbol in SAEncyclopedia carries two addresses.

| Field | Meaning | Stability |
|---|---|---|
| **`entry_va`** | Where callers call. The documented, community-consensus address. | Same on every 1.0 US build ✅ |
| **`body_va`** | Where the instructions actually are. | **Build-specific** |

For 99 %+ of functions these are equal and the distinction costs nothing. For the 492 in
[the relocation map](../RE-Data/data/hoodlum_relocation_map.json) they differ, and the distinction is
the difference between working and silently wrong.

Consumers pick by intent, and the SDK should make them say which they mean:

| Task | Wants |
|---|---|
| Call the function | `entry_va` |
| Install a detour (callers must route through it) | `entry_va` |
| Read/verify a prologue; disassemble; scan a signature | `body_va` |
| Patch an instruction inside the function | `body_va` |
| Compute a call graph | both — edges arrive at `entry_va`, live in `body_va` |

> ✅ *Verified:* on this build, `entry_va == body_va` for all functions except the 492 recorded in the
> map, where `body_va` lies in `.HOODLUM` (`0x01560930 .. 0x0157440D`).

### Worked example

```
CdStreamRead
  entry_va  0x00406A20    ← call here; 6 direct call sites in .text reach it
  body_va   0x0156C2C0    ← disassemble/sign here
  build     1.0-US-HOODLUM
```

On a Compact build the same symbol would record `entry_va == body_va == 0x00406A20`. The *symbol* is
build-independent; its address *binding* is not. That separation is what lets one encyclopedia
describe more than one binary honestly.

---

## 3. Identifying the build

A resolver cannot pick a binding without knowing which image it is looking at. Two contexts, two
methods, one answer.

### 3.1 Offline (a file on disk)

Definitive: **SHA-1 of the file.** For this build:

```
sha1  185b73fbceaa05d66452691fc0d15c8d61b92a7e
md5   170b3a9108687b26da2d8901c6948a18
size  14,383,616
```

### 3.2 Runtime (an image already mapped into the process)

A DLL injected into a running game cannot cheaply hash the file — it has the *loaded image*, which is
not byte-identical to the file (sections are expanded to their virtual sizes, the IAT is written).
Hashing 14 MB at every startup is also rude.

The fingerprint must therefore be computed from something that is (a) present in memory, (b) cheap,
(c) discriminating. The PE section headers satisfy all three: they sit at a fixed offset from
`ImageBase`, they are ~440 bytes, and they encode exactly the structural difference between builds —
section count, names, virtual addresses, sizes, characteristics.

**Definition.** `section_layout_sha256` = SHA-256 over, for each section in order: the 8-byte name,
then `VirtualAddress`, `VirtualSize`, `Characteristics` as little-endian `u32`; finally the section
count as `u32`.

For this build:

```
section_layout_sha256  3fe96d15aa882270d46fdf945cc77fbf49788921762e746770deeb828683117e
short id               3fe96d15
section_count          11
SizeOfImage            0x01177000
```

✅ *Verified:* computed from the shipped file's section table.

This alone separates the 11-section SecuROM+HOODLUM image from a 7-section Compact image before a
single byte of code is read.

### 3.3 Probe bytes — the cheap confirmation

The layout hash identifies the *shape*. A short probe list confirms the *content* and, importantly,
degrades usefully: if the layout hash is unrecognised (a build nobody has catalogued), probes still
report which structural hazards apply.

| VA | Bytes on this build | What it discriminates |
|---|---|---|
| `0x00401000` | `E9 7B 19 16 01` | first relocated function — `E9` ⇒ relocation scheme present |
| `0x00406A20` | `E9 9B 58 16 01` | `CdStreamRead` relocated |
| `0x004087E0` | `53 8B 5C 24 0C` | a function *not* relocated — real prologue, sanity anchor |
| `0x0049D310` | `E9 BB 4D 0C 01` | last relocated function |
| `0x005324A0` | `8B 44 24 04 50` | in-place function above the relocation range |

✅ *Verified:* read from the shipped file.

A resolver that sees `E9` at `0x00401000` and `0x00406A20` but real code at `0x004087E0` knows it is
on a HOODLUM-class image even if the exact hash is new to it.

### 3.4 Build identity is a hash, not a version string

Restating C0.1 §4.5 as a rule the schema enforces: **`"1.0 US"` is never a key.** It is a human label
attached to a key. The key is `sha1` offline and `section_layout_sha256` at runtime. Any API that
accepts a version *string* as its discriminator has reintroduced the defect this entry exists to
remove.

---

## 4. The address database

Following the Most Wanted methodology and the Constitution's DRY rule: **JSON is the single
authoritative source; headers are build artifacts.** No address is ever hand-typed into a `.hpp`.

```
SAEncyclopedia/
└── RE-Data/
    └── data/
        ├── builds.json                    ← known builds, keyed by hash
        ├── symbols.json                   ← build-independent symbol catalogue
        ├── bindings/
        │   ├── 3fe96d15.json              ← per-build entry/body bindings
        │   └── <compact>.json             ← (when a Compact exe is available)
        ├── hoodlum_relocation_map.json    ← ✅ generated, 492 entries
        └── signatures.json                ← patterns + the section they scan
                      │
                      ▼   tools/gen_sa_db.py
        SASDK/include/sasdk/game/sa_db.inl  (generated, never hand-edited)
```

### 4.1 The symbol record

A symbol is build-independent. It names a thing and records what is known about it.

```jsonc
{
  "id": "CdStreamRead",
  "kind": "function",
  "subsystem": "Streaming/CdStream",
  "signature": "int __cdecl(int index, void* buffer, unsigned offset, unsigned size)",
  "re_status": "documented",       // unknown | partial | documented | verified | sdk_ready
  "confidence": "verified",        // verified | reasoned | open
  "evidence": "C0-Binary-Identity/01-the-hoodlum-layer.md#32"
}
```

### 4.2 The binding record

A binding attaches a symbol to one build. This is the only place addresses live.

```jsonc
{
  "build": "3fe96d15",
  "symbol": "CdStreamRead",
  "entry_va": "0x00406A20",
  "body_va":  "0x0156C2C0",
  "relocated": true,
  "body_section": ".HOODLUM",
  "direct_call_sites": 6,
  "confidence": "verified"
}
```

The 492 relocated bindings for this build are already generated and verified — see the map artifact.
Its per-entry fields are `entry_va`, `body_va`, `body_extent`, `direct_call_sites`, `entry_bytes`.

> 🟡 *Reasoned:* `body_extent` is the gap to the next body start, an **upper bound** on the function's
> true size, not a measured end. Median 64 bytes, min 16, max 3,392. It is adequate for bounding a
> disassembly window and inadequate for anything that needs an exact extent; treated as a hint, and
> labelled as one in the data.

---

## 5. The resolver

```
resolve(symbol_id, which ∈ {Entry, Body}) -> Result<Va>
```

**Algorithm.**

1. **Identify the build once**, at load, and cache it: read the section headers at `ImageBase`,
   compute `section_layout_sha256`, look it up in `builds.json`.
2. **Known build** → return the binding's `entry_va` or `body_va`. Done; no scanning, no heuristics.
3. **Unknown build** → run the probe list to classify the hazard, then fall back to §6 signature
   scanning, and *report* that the address was derived rather than looked up. A derived address is
   never silently presented as a verified one.
4. **Unresolvable** → a typed error. Never a plausible-looking wrong number.

Notes that follow from C0.1:

- **No rebasing.** `DllCharacteristics == 0`, so the image loads at `0x00400000` every run. The
  resolver still reads the actual base and adds the delta — the delta is simply always zero here.
  Costing nothing, it removes a whole class of future breakage if a build ever ships with ASLR.
- **`Result<T>`, not exceptions or sentinels** — consistent with the Constitution's error-handling
  rule and with MWSDK's `std::expected` vocabulary.

### 5.1 What SASDK exposes

Per the project brief, developers should not touch addresses at all. The resolver sits *underneath*
the engine wrappers; almost no plugin author should ever name it:

```cpp
// Level 1 — what a plugin author writes. No addresses, no builds, no hazards.
SA::Streaming::RequestModel(id);
SA::Events::OnStreamingLoad += &my_handler;

// Level 2 — explicit, still safe.
auto va = SA::Address::Entry("CdStreamRead");        // Result<Va>
auto hk = SA::Hook::Install("CdStreamRead", &detour, &original);

// Level 3 — raw, for research and tooling.
auto body = SA::Address::Body("CdStreamRead");       // 0x0156C2C0 on this build
auto info = SA::Build::Current();                    // id, hashes, hazards
```

`SA::Hook::Install` takes the **symbol**, not an address, precisely so the entry/body decision is made
by the SDK and not by the caller. That is the hooking-philosophy point from the brief applied to the
substrate: the SDK hides addresses, calling conventions, trampolines, *and build differences*.

---

## 6. Signature scanning, corrected

Signature scanning is the standard defence against version drift, and C0.1 §4.2 showed it fails here
in a nasty way: all 492 stubs begin with the same byte, so a short pattern anchored at a relocated
function's entry returns *hundreds of false positives* rather than an honest miss.

Three rules make it sound:

1. **Author every signature against `body_va`.** A pattern is a statement about a function's
   instructions. The stub is not the function.
2. **Scan every executable section, not just `.text`.** On this build the search space is
   `.text ∪ .HOODLUM`. A scanner hardcoded to `.text` cannot find 492 functions no matter how good the
   pattern is.
3. **Require uniqueness.** A pattern that matches more than once across the scanned range is a failed
   pattern — return an error, do not return the first hit. This single rule converts the
   `E9`-false-positive failure mode from *silent corruption* into *a build-time error*.

```jsonc
{
  "symbol": "CdStreamRead",
  "pattern": "A1 ?? ?? ?? ?? 53 8B 5C 24 14 55 8B 6C 24 0C 56",
  "scan": ["executable"],      // never ".text" alone
  "expect": "unique",
  "anchors": "body"
}
```

✅ *Verified:* that pattern matches the body at `0x0156C2C0` byte-for-byte and occurs **exactly once**
across `.text ∪ .HOODLUM` (0 hits in `.text`, 1 in `.HOODLUM`). It is a worked demonstration that a
body-anchored, all-sections, uniqueness-checked signature resolves a relocated function correctly on
this build — the failure mode from C0.1 §4.2 disappears entirely once all three rules are applied.

---

## 7. Status vocabulary

The project brief requires a status on every item. Two orthogonal axes, kept separate because they
answer different questions:

| `re_status` | Meaning |
|---|---|
| `unknown` | Named or suspected; nothing established |
| `partial` | Some structure recovered, gaps stated |
| `documented` | Mechanism explained and usable |
| `verified` | Proven against the shipped bytes, reproducible |
| `sdk_ready` | Verified **and** bound on ≥1 catalogued build **and** exposed through an API |

| `confidence` | Meaning |
|---|---|
| ✅ `verified` | Confirmed against real bytes |
| 🟡 `reasoned` | Well-supported inference consistent with all evidence |
| ⏳ `open` | Located and bounded, not decoded |

`sdk_ready` deliberately requires a *binding*, not just knowledge. A function can be perfectly
understood and still not be SDK-ready, because nobody has recorded where it lives on the build in
front of the user. That gate is the whole point of this entry.

Current state of this substrate: **`documented` / ✅ verified**, 492 bindings generated, one build
catalogued.

---

## 8. Open items

- ⏳ **Compact-exe bindings.** The second binding table cannot be written without a Compact exe to
  measure. Until then SASDK catalogues exactly one build and says so, rather than shipping a guessed
  table. The schema already has the slot.
- ⏳ **Veneer semantics.** The 453 computed-jump sites (C0.1 §3.3) are counted and located but their
  provenance and exact dispatch rule are undecoded. They do not affect entry/body resolution — no
  veneer sits at a function entry — but they will affect any future automated call-graph extraction.
- 🟡 **`body_extent` exactness.** Upper bound only (§4.2). Tightening it needs a real function-boundary
  analysis over `.HOODLUM`, which is tractable and not yet done.
- ⏳ **Whether other 1.0 US variants exist in the wild** with a *third* relocation scheme. The schema
  assumes nothing; each new hash is simply a new binding file.

---

### Key takeaways

- An address is a **pair** — `entry_va` (call here) and `body_va` (read code here) — resolved against
  an identified build. They differ for exactly 492 functions on this binary.
- **Build identity is a hash.** Offline: SHA-1. At runtime: SHA-256 over the section headers
  (`3fe96d15…` here), confirmed by 5 probe bytes. A version *string* is a label, never a key.
- **JSON is the source of truth**; `sa_db.inl` is generated. No address is ever hand-typed.
- The resolver returns a **typed error** for unknown builds and flags derived addresses as derived —
  it never manufactures a plausible number.
- **Signatures anchor on `body_va`, scan all executable sections, and must be unique** — the
  uniqueness rule is what turns the 492-way `E9` collision into a build error instead of silent
  corruption.
- `sdk_ready` requires a binding on a catalogued build, not merely understanding.

**Next:** `C1 — Streaming: CdStream, CStreaming & the IMG Model` — the first engine subsystem, and the
one the relocation map already reaches into (`CdStream` stride `0x30`, array at `0x008E3FFC`).
