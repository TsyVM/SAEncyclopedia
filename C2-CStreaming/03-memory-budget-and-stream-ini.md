# C2.3 — The Memory Budget & `stream.ini`

> **The one-sentence version:** the streaming budget is a plain-text integer multiplied by 1024 and
> stored in one global — the single most-patched number in San Andreas modding, and it is four
> instructions.

[← C2.2 — The streaming-info record](02-streaming-info-record.md) · [Chapter 2 hub](C2-CStreaming.md) ·
[Next: C2.4 — The request path →](04-the-request-path.md)

**Confidence:** ✅ Verified

---

## 1. The file

`stream.ini` ships in the game root, 197 bytes, and is the only plain-text tuning file the streaming
system reads:

```
memory		13500
devkit_memory	13500
vehicles	12
pe_lightchangerate	0.0005
pe_lightingbasecap	0.35
pe_lightingbasemult	0.5
pe_leftx	16
pe_topy		16
pe_rightx	16
pe_bottomy	16
dontbuildpaths
```

The `pe_*` keys are not handled by this parser — they belong to a different consumer and are ignored
here.

## 2. The parser

At `0x005BCCD0`. Structure: open the file, read lines, skip `#` comments and empty lines, tokenise on
the separator set `" ,\t"` (string at `0x0086A8C8`), then compare the key against a chain of literals.

```
005bccd7: push 0x86a8cc                ; "stream.ini"
005bccde: call 0x538900                ; open
005bcd00: mov  cl, byte ptr [eax]
005bcd02: cmp  cl, 0x23                ; '#'  -> skip line
005bcd0b: test cl, cl                  ;  \0  -> skip line
005bcd13: push 0x86a8c8                ; " ,\t"
005bcd19: call 0x82244b                ; strtok
```

✅ *Verified.*

## 3. The four keys it recognises

| Key | String VA | Destination | Transform |
|---|---|---|---|
| `memory` | `0x0086A8C0` | `0x008A5A80` | `atoi(v) << 10` |
| `devkit_memory` | `0x0086A8B0` | `0x008A5A80` | `atoi(v) << 10`, sets override flag |
| `vehicles` | `0x0086A8A4` | `0x008A5A84` | `atoi(v)` raw |
| `dontbuildpaths` | `0x0086A894` | `0x0096F016` (byte) | set to `1`, no value |

### The budget

```
005bcd45: call 0x82258e            ; atoi
005bcd4d: shl  eax, 0xa            ; × 1024
005bcd50: mov  dword ptr [0x8a5a80], eax
```

**`memory 13500` → 13,824,000 bytes → 13.18 MiB.** ✅

The unit is **kilobytes**, which is worth stating plainly because the file gives no hint: a reader
seeing `13500` with no suffix might reasonably guess bytes, megabytes, or sectors. It is `<< 10`.

### The `devkit_memory` override

The two memory keys write the *same* global, but the second one also sets `bl = 1`:

```
005bcd40: test bl, bl
005bcd42: jne  0x5bcd5a            ; already overridden -> ignore 'memory'
...
005bcd7d: mov  bl, 1               ; devkit_memory wins from here on
```

✅ *Verified.* `bl` starts zero (`xor bl, bl` at `0x005BCCDC`). The precedence is **order-independent**:
whichever line appears, `devkit_memory` wins, because `memory` checks the flag before writing and
`devkit_memory` sets it after. A developer build could raise the budget without editing the shipping
value.

## 4. What the budget governs

`0x008A5A80` is the ceiling; `0x008E4CB4` is the running total, adjusted by `cdSize × 2048` on every
load and unload ([C2.2 §2](02-streaming-info-record.md)). When the total would exceed the ceiling, the
streamer evicts.

⏳ **Open:** the eviction routine itself — the comparison site and the victim-selection order — was not
reached in this pass. What is established is the ceiling, the accumulator, and the units both use.

## 5. Why this number is the one everyone patches

13.18 MiB was a reasonable working set in 2004 and is absurdly small now. Raising it is the single
highest-impact streaming change available, which is why essentially every memory mod does it.

Two cautions the disassembly makes clear:

- **The value is read once, at parse time.** Patching `0x008A5A80` after startup changes the ceiling but
  does not retroactively change anything already evicted. Tools that write the global live must do it
  early.
- **It is a `u32` of bytes.** `memory` values above 4,194,303 overflow the `<< 10`. The practical ceiling
  from the ini is ~4 GiB, but a 32-bit process will fail long before that — and the accumulator at
  `0x008E4CB4` is also `u32`, so the two agree.

The SDK should expose this as `SA::Streaming::BudgetBytes()` reading the live global, never as a
constant — the same rule as pool capacities in
[C4](../C4-Entities-And-Pools/C4-Entities-And-Pools.md), and for the same reason: another mod may have
changed it.

---

### Key takeaways

- The budget is **`stream.ini memory × 1024`**, stored at **`0x008A5A80`**; shipped value gives
  **13,824,000 bytes**.
- The unit is **kilobytes** — invisible in the file, explicit in the `shl eax, 0xa`.
- **`devkit_memory` always beats `memory`**, order-independently, via a flag checked before one write
  and set after the other.
- Only **four keys** are handled here: `memory`, `devkit_memory`, `vehicles`, `dontbuildpaths`. The
  `pe_*` keys belong elsewhere.
- The value is **read once at parse time**; live patches must land early.
- Expose it through the SDK as a **runtime read**, never a constant.

**Continue:** [C2.4 — The request path](04-the-request-path.md)
