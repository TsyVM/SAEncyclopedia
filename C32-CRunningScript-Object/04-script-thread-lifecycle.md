# C32.4 — Script thread lifecycle

## Thread creation: START_NEW_SCRIPT

Opcode `0x004F` (`START_NEW_SCRIPT`) creates a new `CRunningScript` object and inserts it into the active thread list. The argument is a label (a 32-bit SCM offset pointing to the entry point of the new thread). Optional arguments after the label are placed into the new thread's local variables (locals 0, 1, 2, …) before it starts.

The creation process:
1. Allocate a 224-byte `CRunningScript` from the script thread pool (or a static array — exact allocation site ⏳)
2. Zero all local variables (`+0x3C..+0xBF`)
3. Set PC to the SCM offset given by the label argument
4. Clear the gosub stack depth counter
5. Copy any passed arguments into locals 0..N
6. Set `bActive = true`
7. Insert at the head of the `ProcessAllScripts` list

The new thread starts executing on the next call to `ProcessAllScripts` (i.e., the next frame). It does not execute in the same frame it was created.

## Mission threads: the +0xDC flag

The main mission system uses `CRunningScript` with a special modifier: when the `+0xDC` flag is set (`true`), the thread's local variable accesses (reads and writes) are redirected to the **mission-locals array at `0xA48960`** rather than to the thread's own `+0x3C` local block.

This is the mechanism that makes mission scripts "share" a common pool of variables accessible across the mission system. It also means:
- A mission thread's 34 local slots at `+0x3C` are unused for data (they may hold scratch or be zeroed)
- Reading local 0 in a mission thread reads from `0xA48960 + 0` rather than `this + 0x3C + 0`
- Writing to local 5 in a mission thread writes to `0xA48960 + (5 × 4) = 0xA48974`

The flag is set by the mission-start code path, not by SCM itself. Non-mission scripts always use their own local block.

## Thread termination: TERMINATE_THIS_SCRIPT

Opcode `0x004E` (`TERMINATE_THIS_SCRIPT`) terminates the calling thread:
1. Sets `bActive = false`
2. Removes the thread from the `ProcessAllScripts` list
3. The 224-byte allocation is returned to the pool (or freed — ⏳)

A thread that falls off the end of its SCM code without a `TERMINATE_THIS_SCRIPT` will read opcode bytes from whatever memory follows its code — this is an SCM authoring bug, and the compiler always emits a `TERMINATE_THIS_SCRIPT` as the last opcode of every script entry point.

## The special-case startup: the main script

The `main.scm` entry point is not started by `START_NEW_SCRIPT` — it is the root thread, created at game start by `CTheScripts::Init`. It runs for the life of the game and is never terminated (the final opcode of the game's SCM loop is a `WAIT 0` at the bottom of the main loop, yielding back each frame).

The mission script threads are created by `START_NEW_SCRIPT` called from within the main loop.

## The GOSUB / RETURN mechanism

Opcode `0x0050` (`GOSUB label`) is a subroutine call within a single script thread:
1. Push the current PC onto the gosub stack at `+0x18`
2. Increment the gosub depth counter
3. Jump PC to `label`

Opcode `0x0051` (`RETURN`) returns from a gosub:
1. Decrement the gosub depth counter
2. Pop the saved PC from the gosub stack
3. Resume execution at the saved PC

The gosub stack is 8 entries deep (8 × 4 bytes = 32 bytes at `+0x18..+0x37`). Nesting beyond 8 levels overwrites memory past the stack — the stack has no overflow guard. In practice, SCM scripts rarely nest more than 3–4 levels deep.

## Thread lifetime diagram

```
Game start
    │
    ▼
CTheScripts::Init → root thread created
    │
    ▼
ProcessAllScripts (each frame)
    ├── root thread runs
    │       ├── START_NEW_SCRIPT → new thread inserted at head
    │       └── WAIT 0 → yield
    │
    ├── new thread runs
    │       ├── executes logic
    │       ├── WAIT n → suspend until time elapses
    │       └── TERMINATE_THIS_SCRIPT → removed from list
    │
    └── (all other active threads...)
```

## The mission-locals array in depth

The mission-locals array at `0xA48960` holds 1024 dwords (4096 bytes), enough for multiple concurrent missions to share a variable namespace. The layout of this array is SCM-defined — the game's script compiler assigns specific indices to each mission's variables. Variables 0–33 correspond to the "local 0 through local 33" of any thread with `+0xDC` set.

A mod that patches `0xA48960` writes directly into the mission variable space. This is sometimes used to force mission completion conditions or skip mission objectives — but it requires exact knowledge of which variable index corresponds to which mission state, as the mapping changes between missions.

## Thread count limit

The total number of live script threads at any one time is bounded by the script thread pool. The exact pool size is ⏳ — not yet traced to its allocation site in `CTheScripts::Init`. In practice, the engine supports dozens of simultaneous threads (ambient ambient AI scripts, cutscene threads, mission threads, timer threads) without exhausting the pool in normal gameplay. Large mods with many `START_NEW_SCRIPT` calls can approach the limit.

**Previous:** [C32.3 — Script execution and the frame tick](03-script-execution-and-the-frame-tick.md)  
**Continue:** [C32.5 — Gosub stack, control flow, and SCM patterns →](05-gosub-stack-and-control-flow.md)
