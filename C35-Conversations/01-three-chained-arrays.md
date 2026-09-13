# C35.1 — Three Chained Arrays

> **The one-sentence version:** the conversation system's state is three static arrays laid consecutively in
> memory — a 14-slot per-ped table, a 12-node conversation tree, a 50-entry line pool — and each is sized
> by the address of the next, because `0x9691D8 + 14×28 = 0x969360`, `0x969360 + 12×44 = 0x969570`, so the
> three pack end-to-end with no gap.

**Subsystem category:** Peds / audio — memory layout
**Depends on:** [C35 hub](C35-Conversations.md), [C34.2](../C34-Vehicle-Recording/02-the-playback-engine.md)
(the chained next-global proof)
**RE status:** Documented
**Confidence:** ✅ for the three geometries and the chain · 🟡 for per-record fields

---

## 1. The reset path names two of the arrays

`CConversations::Clear` (`entry_va 0x0043A7B0`) reinitialises the system, and it walks two of the three
arrays back to back:

```
; per-ped state table
0156DFF3  mov  eax, 0x9691E0                ; (record base 0x9691D8 = eax-8)
0156DFFC  mov  [eax-8]/[eax-4] = -1 ; [eax],[eax+4],[eax+8] = 0
0156E00A  add  eax, 0x1C                     ; stride 28
0156E00D  cmp  eax, 0x969368                 ; until end
; dialogue-line array
0156E014  mov  eax, 0x969578                ; (record base 0x969570 = eax-8)
0156E01B  mov  [eax-8](byte),[eax],[eax+2](word),[eax+4],[eax+8],[eax+0xC] = 0
0156E02E  add  eax, 0x18                     ; stride 24
0156E031  cmp  eax, 0x969A28                 ; until end
```

The first loop clears **28-byte** records from `0x9691D8`, the second clears **24-byte** records from
`0x969570`. Counting the iterations: `(0x969368 − 0x9691E0)/0x1C = 14` and `(0x969A28 − 0x969578)/0x18 = 50`.

## 2. The node table, and the chain

The middle array — the conversation-node tree — is named by `SetUpConversationNode` (`0x0043A870`), which
appends a node at the current build index:

```
0156A7C0  mov  ecx, dword ptr [0x9691C8]    ; build index (nodes so far)
0156A7CA  imul ecx, ecx, 0x2C               ; * 44
0156A7D0  add  ecx, 0x969360                 ; node array base
...
0156A899  inc  eax ; mov [0x9691C8], eax     ; ++build index
```

Node base `0x969360`, stride `0x2C` = **44 bytes**, and the build index (nodes appended so far) is the global
at `0x9691C8`. Now the three bases and sizes chain:

```
per-ped :  0x9691D8 + 14 × 0x1C = 0x969360   = node base
nodes   :  0x969360 + 12 × 0x2C = 0x969570   = line base
lines   :  0x969570 + 50 × 0x18 = 0x969A20
```

Each array ends exactly where the next begins. The per-ped table's capacity (**14**) is fixed by the node
base; the node table's capacity (**12**) is fixed by the line base; and the line array is **50** from its own
clear loop. This is [C34](../C34-Vehicle-Recording/C34-Vehicle-Recording.md)'s chained next-global proof one
link longer — three consecutive arrays, two boundary equalities, both closing to the byte with no assumption.

## 3. The per-ped record points into the line array

The arrays are not independent: `RemoveConversationForPed` (`0x0043A960`) shows a per-ped record holding an
index into the line pool:

```
0156ADCA  mov  edx, 0x9691E0                ; per-ped array
0156ADD5  mov  eax, dword ptr [edx-8]        ; the record's first field = a line index
0156ADD8  lea  ecx, [eax + eax*2]            ; index * 3
0156ADDB  lea  ecx, [ecx*8 + 0x969570]       ; * 8  ->  index * 24 + line base
```

So a per-ped record's first field is a **line index** (`index × 24 + 0x969570` addresses the line record),
and clearing a ped's conversation resets that link to −1. The per-ped table is the *who is talking* set, the
line array is the *what is said* pool, and the node table is the *tree that sequences them*.

## 4. The build index and the reset

`StartSettingUpConversation` (`0x0043A840`) begins assembling a conversation: it zeroes the build index at
`0x9691C8` (so `SetUpConversationNode` appends from node 0) and sets the active flag at `0x9691D0`. `Clear`
(§1) is the full teardown. Together they bound the node table's lifetime: a conversation is built up to 12
nodes, walked, then cleared.

---

### Key takeaways

- Three consecutive static arrays: **per-ped 14 × 28 B @ `0x9691D8`**, **nodes 12 × 44 B @ `0x969360`**,
  **lines 50 × 24 B @ `0x969570`**.
- They **chain**: `0x9691D8 + 14×28 = 0x969360` and `0x969360 + 12×44 = 0x969570` — each array sized by the
  next's base, both closing to the byte.
- A per-ped record's first field is a **line index** (`×24 + 0x969570`); the build index at `0x9691C8`
  counts nodes appended.

**Next:** [C35.2 — The conversation node and its text](02-the-node-and-its-text.md).
