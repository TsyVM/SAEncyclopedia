# C32.3 — Script execution and the frame tick

## Where scripts run in the game loop

The GTA:SA main loop (C52) drives script execution once per frame. `CTheScripts::ProcessAllScripts` is called from the game-update path, and it walks every live `CRunningScript` thread in turn. Each thread runs until it either:
1. Exhausts its per-frame opcode budget, or
2. Hits a `WAIT` opcode (opcode `0x0001`), which stores the wait duration in the thread and returns control

This cooperative multi-tasking model means scripts never truly run in parallel — they interleave within one game frame.

## CTheScripts::ProcessAllScripts

The function walks a linked list (or indexed array — the exact container type is ⏳) of live `CRunningScript` objects and calls `Process` on each:

```
CTheScripts::ProcessAllScripts:
  lea  esi, [script_pool_head]      ; first thread
  test esi, esi
  jz   .done
.loop:
  push esi                          ; this pointer = CRunningScript*
  call CRunningScript::Process
  mov  esi, [esi + offset_next]     ; advance to next thread
  test esi, esi
  jnz  .loop
.done:
```

The `offset_next` field within `CRunningScript` (exact offset ⏳) is how the linked list walks. Critically: `CRunningScript::Process` may **delete** the current thread (if it terminates), which means the walk must cache the `next` pointer before calling `Process`. The disassembly confirms this defensive pattern.

## The per-frame opcode budget

Each call to `CRunningScript::Process` runs opcodes in a tight loop until:

```
sub [frame_opcode_budget], 1
jz  .yield         ; budget exhausted — yield control
```

The default budget is set to a constant (⏳ — not yet traced to its exact address) before the loop starts. When the budget reaches zero, the current thread stops mid-instruction-stream and resumes at the same PC next frame.

**Consequences for modders:**
- Very long opcode sequences (thousands of fast opcodes) will be spread across multiple frames
- A tight infinite `WHILE` loop in a script will consume the entire frame budget every frame, delaying other scripts
- Time-sensitive checks (like "did the player enter this zone this frame?") should use `WAIT 0` periodically to yield to other threads

## The WAIT opcode: 0x0001

`WAIT n` (opcode `0x0001`, one argument: milliseconds to wait) is the primary yield mechanism:

1. The current thread stores `ms_nTimeInMilliseconds + n` in its wait-until field (an absolute timestamp)
2. Control returns immediately to `ProcessAllScripts`
3. On the next frame, `CRunningScript::Process` checks: if `ms_nTimeInMilliseconds < wait_until`, skip this thread and move to the next
4. Once the wall-clock time passes `wait_until`, execution resumes at the opcode after `WAIT`

`WAIT 0` is a single-frame yield — the thread skips the remainder of this frame's budget and resumes at the start of next frame. It is the idiomatic way to make a long loop cooperative.

## CRunningScript::Process in outline

The Process method is the core execution engine:

```
Process(this):
  ; Check WAIT timer
  if (ms_nTimeInMilliseconds < this->wait_until):
      return

  ; Run opcodes until budget or WAIT
  budget = kDefaultBudget
  loop:
    opcode = fetch word at [this->PC]
    this->PC += 2
    dispatch(opcode, this)
    budget -= 1
    if (budget == 0): break
    if (this->bActive == false): break    ; thread terminated
```

The `dispatch` function is a large switch on the opcode number. It reads any arguments from the SCM byte-stream starting at the current PC (advancing PC past each argument), performs the opcode's action, and returns. Certain opcodes (`WAIT`, `GOSUB`, `RETURN`, `START_NEW_SCRIPT`, `TERMINATE_THIS_SCRIPT`) manipulate the thread's own state (PC, gosub stack, active flag).

## The frame-tick relationship to CTimer

`ms_nTimeInMilliseconds` (CTimer's millisecond counter, C52) is what `WAIT` compares against. This means:
- If the game is paused (`bIsFrozen = true`), `ms_nTimeInMilliseconds` does not advance, and WAIT timers do not expire — scripts are effectively frozen too
- If the game runs slow (below 30fps), `ms_nTimeInMilliseconds` still advances by wall-clock time, so scripts executing `WAIT 1000` will correctly wait one real second regardless of frame rate

This is a deliberate design choice: script timers are wall-clock-based, not frame-count-based. The implication is that scripts are frame-rate independent for timing, but frame-rate dependent for opcode throughput (more frames = more opcodes per second can be executed).

## Multiple threads and execution order

`ProcessAllScripts` walks threads in list order. The list head is the most recently created thread. This means:
1. Newer threads execute before older threads within the same frame
2. The main mission script (started at game load) is near the tail — it runs after all dynamic threads
3. Player-controlled scripts (from `START_NEW_SCRIPT`) inserted at the head run first

This ordering matters for scripts that share global variables: the first thread to run in a frame "wins" that frame's global state. Scripts that must coordinate should use explicit synchronization opcodes rather than assuming execution order.

## The relationship to CRunningScript memory layout (C32.1 and C32.2)

Each `CRunningScript` object participating in `ProcessAllScripts` is 224 bytes (0xE0). The per-thread state that `Process` reads and writes every frame:

| Field offset | Field | Used by Process |
|---|---|---|
| `+0x14` | PC (program counter) | Fetch opcode, advance past args |
| `+0x18..+0x37` | Gosub stack (8 × dword) | `GOSUB`/`RETURN` opcodes |
| `+0x3C..+0xBF` | Local variables (34 × dword) | Arguments and temporaries |
| `+0xDC` | Mission-locals flag | Redirects local reads to 0xA48960 |
| Wait-until field | Absolute ms timestamp | WAIT comparison |
| bActive flag | Thread alive/dead | Loop termination |

**Previous:** [C32.2 — The 224-byte CRunningScript layout](02-crunningscript-layout.md)  
**Continue:** [C32.4 — Script thread lifecycle →](04-script-thread-lifecycle.md)
