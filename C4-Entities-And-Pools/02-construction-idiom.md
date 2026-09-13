# C4.2 — The Construction Idiom

> **The one-sentence version:** seventeen pools built by one repeated five-instruction pattern, each
> with its own template-instantiated constructor — which is what lets the table be read mechanically
> rather than interpreted.

[← C4.1 — The pool table](01-the-pool-table.md) · [Chapter 4 hub](C4-Entities-And-Pools.md) ·
[Next: C4.3 — The entity→model link →](03-entity-model-link.md)

**Confidence:** ✅ Verified

---

## 1. The pattern

```
00550f26: push 0x14                     ; sizeof(CPool) = 20
00550f28: call 0x82119a                 ; operator new
00550f2d: add  esp, 4
00550f30: mov  dword ptr [esp], eax
00550f34: test eax, eax
00550f3e: je   0x550f53                 ; new returned null -> skip ctor
00550f40: push 0x863d10                 ; -> "PtrNode Single"
00550f45: push 0x11170                  ; 70000
00550f4a: mov  ecx, eax
00550f4c: call 0x550180                 ; CPool<T>::CPool(count, name)
00550f51: jmp  0x550f55
00550f53: xor  eax, eax                 ; null pool
00550f55: ...
00550f5f: mov  dword ptr [0xb74484], eax
```

Repeated seventeen times with three things varying: the **count**, the **name pointer**, and the
**constructor address**. ✅ Verified.

## 2. Three details worth extracting

### The pool object is 20 bytes

`operator new(0x14)` — every pool, regardless of element type. The pool *object* is a small header; the
element **storage is allocated separately inside the constructor**, which is why a 70,000-entry pool
and a 16-entry pool both allocate 20 bytes here.

### Each pool has its own constructor address

`0x550180`, `0x550250`, `0x550320`, `0x5503F0`, `0x5504C0`, … — sequential, roughly 0xD0 apart. This is
`CPool<T>` instantiated once per element type, each copy differing only in the element size baked into
its allocation. The address progression is itself evidence of template instantiation rather than a
shared runtime-sized pool.

⏳ **Open:** the element sizes. Reading the allocation size out of each constructor would give
`sizeof(CPed)`, `sizeof(CVehicle)`, `sizeof(CBuilding)` and so on for free — seventeen struct sizes
from seventeen functions. That is a cheap, high-value pass and was not done here.

### Every allocation is null-checked, and failure is survivable

`test eax, eax; je` around every constructor call, with `xor eax, eax` storing a **null pool pointer**
on failure. Startup does not abort — it continues with a null pool, and the crash arrives later at
first use.

🟡 *Reasoned:* this is MSVC's standard non-throwing `operator new` pattern rather than a deliberate
resilience design. Either way the observable behaviour is the same, and it explains a class of
memory-pressure crashes that appear far from their cause.

## 3. Exception bookkeeping

The function opens with SEH setup and maintains a state index across the sequence:

```
00550f10: push -1
00550f12: push 0x83cc0b               ; exception handler
00550f17: mov  eax, dword ptr fs:[0]
00550f1e: mov  dword ptr fs:[0], esp
...
00550f36: mov  dword ptr [esp + 0xc], 0    ; state 0
00550f72: mov  dword ptr [esp + 0x10], 1   ; state 1
00550faa: mov  dword ptr [esp + 0x10], 2   ; state 2
00550fe2: mov  dword ptr [esp + 0x10], 3
0055101a: mov  dword ptr [esp + 0x10], 4
0055104f: mov  dword ptr [esp + 0x10], 5
```

The incrementing state index is the compiler tracking how many pools have been constructed, so an
exception mid-sequence unwinds exactly the ones already built. ✅ Verified.

It is also the most convenient thing in the function for an analyst: **the state index is a reliable
counter of pool number**, independent of the `push`/`call` pattern, and confirms the ordering of the
table in [C4.1](01-the-pool-table.md).

## 4. Why the idiom matters for extraction

The whole table in C4.1 was recovered by walking the disassembly and collecting `(name, count, global)`
triples mechanically. That is only sound because the pattern is rigid — same order every time, name
pushed before count, global written after the call.

This is worth noting as method: **a repeated construction idiom is a table in disguise.** When a
function repeats a fixed instruction shape N times with a few varying immediates, the varying
immediates *are* structured data and can be extracted by pattern rather than read by hand. The same
approach applies to any registration sequence in the binary.

The counterpart caution: pattern extraction only proves what the pattern captures. It gave the counts,
names and globals confidently, and it gave nothing at all about the pool object's internal layout —
which remains open in [C4.1 §4](01-the-pool-table.md) precisely because it is not part of this pattern.

---

### Key takeaways

- One five-instruction idiom, repeated **seventeen** times, varying only count, name and constructor.
- The `CPool` object is **20 bytes**; **element storage is allocated inside the constructor**, not here.
- **Each pool has its own constructor address**, spaced ~0xD0 apart — `CPool<T>` per element type.
- ⏳ Those constructors hold **seventeen struct sizes** (`sizeof(CPed)`, `sizeof(CVehicle)`, …) — a
  cheap, high-value pass not yet done.
- Allocation failure stores a **null pool** and continues; the crash surfaces later, far from the cause.
- The SEH **state index is an independent counter** confirming pool order.
- Method: **a repeated construction idiom is a table in disguise** — and proves only what it captures.

**Continue:** [C4.3 — The entity→model link](03-entity-model-link.md)
