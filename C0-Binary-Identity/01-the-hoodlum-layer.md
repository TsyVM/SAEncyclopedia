# C0.1 — Binary Identity: the SecuROM Wrapper & the HOODLUM Layer

> **The one-sentence version:** the `gta_sa.exe` in this working tree is the SecuROM-wrapped retail
> 1.0 US image with a HOODLUM crack section appended, and **492 function bodies do not live at their
> documented addresses** — they live in `.HOODLUM`, reached by a 5-byte `jmp` planted at each
> function's entry. Every signature scanner, prologue-copying hook, and static disassembly of this
> file is affected.

**Subsystem category:** Binary substrate (pre-engine)
**RE status:** Documented
**Confidence:** ✅ Verified against the shipped file
**SDK exposure:** Required — this is the input to the address resolver

---

## Why this is entry zero

SAEncyclopedia's first obligation is not to describe `CStreaming` or `CWorld`. It is to answer a
prior question that every later entry silently depends on:

> *Which bytes are we describing?*

MWEncyclopedia could take that for granted — `speed.exe` retail v1.3 is one file, and the whole book
is pinned to it (`ImageBase 0x400000`, `RVA == file offset`). GTA SA cannot. "1.0 US" names at least
two binaries that disagree about where code physically is, and the disagreement is not a footnote:
it is 492 functions wide. An encyclopedia that documents addresses without first documenting *which
image those addresses index* is publishing claims it cannot support.

So: entry zero is the binary's own identity.

---

## 1. What the file is

| Property | Value |
|---|---|
| Size | **14,383,616 bytes** |
| MD5 | `170b3a9108687b26da2d8901c6948a18` |
| SHA-1 | `185b73fbceaa05d66452691fc0d15c8d61b92a7e` |
| Machine | `0x14C` (i386) |
| ImageBase | `0x00400000` |
| Entry point | `0x00824570` (RVA `0x424570`) |
| PE timestamp | `2005-04-28 15:31:22Z` |
| Sections | **11** |
| ASLR | none — `DllCharacteristics = 0x0000` |

✅ *Verified:* all values read directly from the PE headers of the shipped file.

The `DllCharacteristics = 0` line matters more than it looks. There is no `DYNAMIC_BASE` bit, so the
image loads at its preferred base on every run. **GTA SA does not need the rebasing layer MWSDK
needs.** A `Va` in this encyclopedia is a real, absolute, run-to-run-stable address. That is a genuine
simplification relative to the Most Wanted work, and it should be exploited, not ignored.

### Section table

| # | Name | VA | Virtual size | Raw offset | Raw size | Characteristics |
|---|---|---|---|---|---|---|
| 1 | `.text` | `0x00401000` | 4,546,560 | 1,024 | 4,546,048 | `0x60000020` R/X |
| 2 | `_rwcseg` | `0x00857000` | 4,096 | 4,547,072 | 1,536 | `0x60000020` R/X |
| 3 | `.rdata` | `0x00858000` | 311,296 | 4,548,608 | 308,224 | `0x40000040` R |
| 4 | `.data` | `0x008A4000` | 4,169,728 | 4,856,832 | 262,144 | `0xC0000040` RW |
| 5 | `_TEXT_HA` | `0x00C9E000` | 69,632 | 5,118,976 | 68,608 | `0xC0000040` RW |
| 6 | `_rwdseg` | `0x00CAF000` | 4,096 | 5,187,584 | 512 | `0xC0000040` RW |
| 7 | `.rsrc` | `0x00CB0000` | 4,096 | 5,188,096 | 1,536 | `0x40000040` R |
| 8 | `.text` | `0x00CB1000` | 6,594,560 | 5,189,632 | 6,593,536 | `0xE0000020` **RWX** |
| 9 | `.init` | `0x012FB000` | 24,576 | 11,783,168 | 23,040 | `0xE0000020` **RWX** |
| 10 | `.data` | `0x01301000` | 2,445,312 | 11,806,208 | 2,442,240 | `0xE0000040` **RWX** |
| 11 | `.HOODLUM` | `0x01556000` | 135,168 | 14,248,448 | 135,168 | `0xE0000020` **RWX** |

Sections 1–7 are Rockstar's. Sections 8–11 are not.

---

## 2. Sections 8–10 are SecuROM

Strings recovered from section 10 (`.data`, raw `11,806,208 .. 14,248,448`):

| Needle | Occurrences in section 10 |
|---|---|
| `SecuROM` | **19** |
| `CmdLineExt` | **10** |
| `.securom` | 2 |
| `securom` | 2 |

✅ *Verified:* byte-count over the section's raw range. The same needles return **zero** hits in
sections 1–7 and in `.HOODLUM`.

`CmdLineExt` is SecuROM's command-line-extension DLL family; `.securom` is the marker name the
protection uses for its own data. Three RWX sections totalling **9.06 MB** — larger than the entire
game code section — is the protection VM and its data. This is the difference between the 14.38 MB
retail image and the 5.19 MB "Compact" image the modding scene actually targets.

### The Compact-exe correspondence

Sections 1–7 occupy raw bytes `0 .. 5,189,632`. Section 7 (`.rsrc`) ends at raw
`5,188,096 + 1,536 = 5,189,632` — **exactly** the documented size of the GTA SA 1.0 US *Compact*
executable, and exactly where section 8 (the first SecuROM section) begins.

🟡 *Reasoned:* the Compact exe is the same seven Rockstar sections with the SecuROM layer removed.
The raw-span coincidence is exact and hard to explain otherwise, but this is **not** verified as a
byte-identical prefix — the PE header alone must differ (7 sections vs 11), and §3 shows `.text`
itself is patched. Confirming this properly requires diffing against a real Compact exe, which this
tree does not contain. Recorded as reasoned, not asserted.

---

## 3. `.HOODLUM` holds the stolen code — and the original imports

`.HOODLUM` is the crack. It contains two things.

### 3.1 The restored import table and the original PDB path

Readable ASCII in `.HOODLUM` includes the genuine Rockstar build path:

```
x:\SA_PC_SRC (pcv110 - final)\gta_source\MSVC PC files\D3D9 Final USA\gta_sa.pdb
```

alongside `WINMM.dll`, `vorbisfile.dll`, `WS2_32.dll`, `EAX.DLL`, `KERNEL32.dll` and their symbol
names (`timeGetTime`, `ov_open_callbacks`, `VirtualProtect`, …), plus the strings `San Andreas EU`,
`Rockstar North Ltd`, `DEFAULT`, `STANDARD`.

✅ *Verified:* extracted from the section's raw bytes.

SecuROM destroys the original import directory and resolves imports through its own stub; HOODLUM
rebuilds a working import table inside its own section. The PDB path is the original linker's, carried
along with it. `D3D9 Final USA` independently corroborates that the underlying build is the **US
D3D9** one — i.e. this is a 1.0 US image, regardless of the `San Andreas EU` product string sitting a
few bytes away.

### 3.2 The 492 relocated function bodies

This is the finding that changes how the SDK must be built.

Scanning `.text` (raw `1,024 .. 4,547,072`) for `E9 rel32` whose target lands inside `.HOODLUM`:

| Measurement | Value |
|---|---|
| Raw `E9`→`.HOODLUM` matches | 501 |
| …of which land on a **16-byte-aligned** address | **492** |
| Distinct targets | 501 (no target reused) |
| Stub address range | `0x00401000` .. `0x0049D310` |
| Target range in `.HOODLUM` | `0x01560930` .. `0x0157440D` (80,605 bytes of 135,168) |
| `E8` **calls** from `.text` into `.HOODLUM` | **0** |

✅ *Verified:* full linear scan of the section's raw bytes.

The 16-byte alignment is the tell. 492 of 501 matches sit exactly on a 16-byte boundary — MSVC's
function alignment for this build. These are **function entry points**, not incidental jumps. The
remaining 9 unaligned matches are mid-function and are treated separately (§3.3).

Independent confirmation that they are functions:

| Measurement | Value |
|---|---|
| Stubs that are the target of at least one `E8 call` in `.text` | **473 / 492 (96.1 %)** |
| Total direct call sites reaching a relocated function | **1,609** |
| Most-called relocated function | `0x0040EF50` — 46 call sites |

✅ *Verified.* 96 % of these addresses are directly called by name from ordinary game code. They are
functions, and the calls still work: the call lands on the stub, the stub jumps to the real body.

**Worked example — `CdStreamRead` @ `0x00406A20`:**

```
0x00406a20:  jmp 0x156c2c0                     ; <- 5-byte stub, the whole "function"
```

and at the target, in `.HOODLUM`:

```
0x0156c2c0:  mov  eax, dword ptr [0x8e3ffc]    ; CdStream array base
0x0156c2c5:  push ebx
0x0156c2c6:  mov  ebx, dword ptr [esp + 0x14]
0x0156c2ca:  push ebp
0x0156c2cb:  mov  ebp, dword ptr [esp + 0xc]
0x0156c2cf:  push esi
0x0156c2d0:  mov  esi, dword ptr [esp + 0x18]
0x0156c2d4:  lea  ebp, [ebp + ebp*2]           ; \ index * 48
0x0156c2d8:  mov  edx, esi                     ; /
0x0156c2da:  push edi
0x0156c2db:  shl  ebp, 4
0x0156c2de:  lea  edi, [ebp + eax]             ; &CdStream[index]
0x0156c2e2:  shr  edx, 0x18                    ; handle index = offset >> 24
0x0156c2e5:  mov  eax, dword ptr [edx*4 + 0x8e4010]
```

✅ *Verified by disassembly.* The relocated body is **ordinary, unobfuscated Rockstar code**, and it
addresses the game's own globals at their normal 1.0 US virtual addresses — `0x008E3FFC` (the
`CdStream` array), `0x008E4010` (the stream file-handle table), `0x008E4898`. The neighbouring stub at
`0x00406460` resolves to `0x0156CD80` and touches the same `0x008E3FFC` array with the same
`lea eax,[eax+eax*2]; shl eax,4` idiom.

Two facts fall out of that, and both are load-bearing:

- **The data/global address space is untouched.** Only code was moved. Every documented `0x008…`
  global is still valid.
- **`CdStream` entries are `0x30` (48) bytes.** ✅ Verified by the index arithmetic in two independent
  relocated bodies. A structural fact, recovered as a side effect of a packing analysis — which is
  exactly how the encyclopedia is supposed to accrete.

### 3.3 The computed-jump veneers

Beyond the entry stubs, `.text` reads data *out of* `.HOODLUM` at runtime:

| Measurement | Value |
|---|---|
| Absolute dwords in `.text` pointing into `.HOODLUM` | **453** |
| Distinct `.HOODLUM` addresses referenced | 357 |
| 64 KB blocks of `.text` containing at least one | **61 of 70** |

Expected count from chance alone is ≈0.14 (4.55 MB scanned × 132 KB window / 2³²), so this is
structural, not coincidence. ✅ *Verified.*

A worked instance, immediately after the stub at `0x004045B0`:

```
0x004045b0:  jmp  0x1567820                    ; the relocated function
0x004045b5:  push eax
0x004045b6:  pushfd
0x004045b7:  clc
0x004045b8:  mov  eax, dword ptr [0x15607b8]   ; <- .HOODLUM data slot
0x004045bd:  adc  eax, dword ptr [0x5cafc6]    ; combined with a game-image dword
0x004045c3:  popfd
0x004045c4:  xchg dword ptr [esp], eax         ; overwrite return address
0x004045c7:  jmp  0x5980fd
```

The target is *computed at run time* from a `.HOODLUM` constant, then substituted for the return
address. Control flow through these sites is not statically resolvable by reading `.text` alone.

⏳ **Open:** whether these veneers are SecuROM's original obfuscation left in place, or HOODLUM's
emulation of what SecuROM's VM would have returned, is not established here. What *is* established is
that they exist, that there are 453 of them, and that they span 87 % of `.text`. The provenance
question is bounded and stated rather than guessed at.

### 3.4 Correction — not every target is a whole restored function

*Added while writing [C1.2](../C1-Streaming/02-cdstream-layer.md); §3.2's worked example was accurate
but not representative, and the generalisation drawn from it was too broad.*

§3.2 showed `CdStreamRead`'s relocated body and described it as "ordinary, unobfuscated Rockstar code."
That is true of that function and of most, but **not all**, of the 492. Classifying every target by
disassembling its extent:

| Class | Count | Share |
|---|---:|---:|
| Restored function body | 382 | 77.6 % |
| Short stub (≤ 40 bytes) | 109 | 22.2 % |
| Protection thunk — computed call with a fabricated return address | 1 | 0.2 % |

🟡 *Reasoned:* the classification is heuristic (it keys on a computed `call`/`jmp` through a register
in the first 14 instructions, and on extent). The counts are reproducible; the class boundaries are a
judgement.

The verified counter-example is `CdStreamShutdown`. Its entry `0x00406360` jumps to `0x015700D0`,
which is **32 bytes** of:

```
015700d0: mov  eax, [0x8e4010]
015700d5: push 0
015700d7: push eax
015700d8: push 0x401005              ; fabricated return address
015700dd: mov  eax, 0xa7f8f095
015700e2: pushfd
015700e3: xor  eax, 0xa6ece915       ; -> 0x01141980, inside the SecuROM .text
015700e8: popfd
015700e9: call eax
015700eb: ret
```

The function's **real body never left `.text`** — it is at `0x00406370`, sixteen bytes past its own
entry, and reads exactly as expected ([C1.2 §3](../C1-Streaming/02-cdstream-layer.md)). Only the head
was taken.

Separately, **200 of 492 bodies contain a `jmp` back into `.text`**, and in **none** of those cases
does the target land within 64 bytes of the function's own entry — the destinations are scattered
thousands of bytes away. Code is woven between the two sections rather than cleanly partitioned.

**What survives unchanged:** the relocation map itself. `entry_va → body_va` is a statement about where
control goes, and it is correct for all 492 regardless of what is found at the destination. What needed
narrowing was the claim about *what* the destination contains.

**What this adds:** `body_va` is where to start reading, not a guarantee of a self-contained function.
A consumer that disassembles `body_va .. body_va + body_extent` and stops has, for at least 200 of
them, only part of the picture. C0.2's `body_extent` caveat (upper bound, not a measured end) now has
a second reason to exist.

---

## 4. What this means

### 4.1 Static disassembly of this file is incomplete by construction

Load this executable into IDA or Ghidra and 492 functions decompile as one-line thunks. Their bodies
are 17 MB away in a section the tool will not associate with them, and 453 control-flow edges resolve
to computed targets. Any address table, call graph, or xref count derived from a naive load of this
file is wrong in a way that will not announce itself.

### 4.2 Signature scanning breaks at exactly 492 known points

This is the practical consequence for SASDK. A byte pattern anchored at a function's prologue is the
standard way to survive version differences — and for these 492 functions the prologue on *this*
binary is `E9 xx xx xx xx`, while on a Compact exe it is the real first instruction. The same
signature cannot match both. Worse, all 492 stubs share the *same* first byte, so a short pattern
will produce hundreds of false positives rather than a clean miss.

### 4.3 Inline hooking mostly survives — but not for free

A conventional 5-byte detour written over a stub overwrites exactly the `jmp` and nothing else, which
is well-behaved. The hazard is the **trampoline**: a hooking library that copies the original 5 bytes
and executes them elsewhere must *relocate* that `E9`, because its rel32 is position-dependent.
MinHook and Detours do this correctly. Hand-rolled trampolines and anything that memcmp's a prologue
against an expected constant do not.

### 4.4 `gta-reversed` will not run on this binary

The project requires the Compact exe and states the requirement in bytes: exactly **5,189,632**. This
file is 14,383,616. Its mechanism — replacing in-game functions with reimplemented ones at fixed
addresses — assumes the function *is* at that address. For 492 of them here, it is not.

### 4.5 The version key is not "1.0 US"

"1.0 US" is not a sufficient identifier. Two files both correctly described as 1.0 US disagree about
the physical location of 492 function bodies. **The identity of a build is its hash, not its marketing
version.** SASDK's address database must therefore be keyed on a *binary fingerprint* — size plus
section-layout hash plus a small set of probe bytes — and must record, per build, which functions are
relocated and where to. A resolver that asks "is this 1.0 US?" is asking the wrong question; it must
ask "is this image `170b3a91…`?" and then consult a per-build relocation map.

That is the design consequence, and it is why this entry exists before any entry about the engine.

---

## 5. Reproducing every number above

All measurements come from linear scans of the shipped file — no tooling beyond a PE header parse, a
byte scan, and a disassembler.

```python
import struct
d = open('gta_sa.exe','rb').read()
TEXT_VA, TEXT_RAW, TEXT_RS = 0x401000, 1024, 4546048
HOOD_LO, HOOD_HI = 0x1556000, 0x1556000 + 135168

stubs = []
for i in range(TEXT_RS - 5):
    if d[TEXT_RAW + i] != 0xE9:
        continue
    rel = struct.unpack_from('<i', d, TEXT_RAW + i + 1)[0]
    src = TEXT_VA + i
    dst = src + 5 + rel
    if HOOD_LO <= dst < HOOD_HI and src % 16 == 0:
        stubs.append((src, dst))

assert len(stubs) == 492
```

Section-to-file mapping used throughout (raw offset = section raw base + (VA − section VA)):

| Section | VA base | Raw base |
|---|---|---|
| `.text` | `0x00401000` | 1,024 |
| `.rdata` | `0x00858000` | 4,548,608 |
| `.data` | `0x008A4000` | 4,856,832 |
| `.HOODLUM` | `0x01556000` | 14,248,448 |

---

### Key takeaways

- This `gta_sa.exe` is **retail 1.0 US + SecuROM (sections 8–10) + HOODLUM (section 11)**; MD5
  `170b3a91…`, 14,383,616 bytes, **no ASLR** — absolute addresses are stable.
- **492 function bodies live in `.HOODLUM`**, reached by a 16-byte-aligned `E9` stub at the documented
  address. 473 of them are directly called from `.text` across 1,609 call sites.
- **Only code moved.** Relocated bodies address the game's globals at their normal 1.0 US VAs — the
  data address space is intact. (Recovered in passing: `CdStream` stride = `0x30`.)
- **453 computed-jump veneers** read `.HOODLUM` data from 61 of 70 64 KB blocks of `.text`; those
  control-flow edges are not statically resolvable.
- Consequences: static disassembly is incomplete, **signature scanning breaks at 492 known points**,
  trampolines must relocate the stub's rel32, and `gta-reversed` cannot run here.
- **Design rule for SASDK:** key the address database on a *binary fingerprint*, not on a version
  string, and carry a per-build relocation map.

**Next:** `C0.2 — The Build Fingerprint & Address Resolver` — turning §4.5 into a concrete schema.
