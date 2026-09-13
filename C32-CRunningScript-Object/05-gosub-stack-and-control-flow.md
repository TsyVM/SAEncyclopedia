# C32.5 — Gosub stack, control flow, and SCM patterns

## Control flow in the SCM bytecode

GTA:SA's SCM (Script Control Module) bytecode is a flat binary stream. Control flow is entirely by absolute or relative offsets — there are no symbolic labels in the compiled binary. The five primary control-flow opcodes:

| Opcode | Name | Effect |
|---|---|---|
| `0x0002` | `JUMP` | Unconditional jump to absolute SCM offset |
| `0x004D` | `JUMP_IF_FALSE` | Conditional jump if last condition was false |
| `0x0050` | `GOSUB` | Push PC, jump to subroutine |
| `0x0051` | `RETURN` | Pop PC, resume after GOSUB |
| `0x004E` | `TERMINATE_THIS_SCRIPT` | Thread exits |

High-level SCM source constructs (`IF`, `WHILE`, `FOR`, `REPEAT`) all compile down to `JUMP` and `JUMP_IF_FALSE` with fixed offsets.

## The gosub stack in depth

The gosub stack lives at `+0x18` within the `CRunningScript` 224-byte object (C32.1) — 8 × 4 = 32 bytes. The stack grows downward (or upward — exact direction ⏳), tracking return addresses. The depth counter (a separate byte field in the struct, exact offset ⏳) prevents underflow on mismatched `RETURN` calls.

### Stack depth in practice

Standard SA mission scripts call subroutines for:
- Camera cutscene setup
- Checkpoint creation
- Objective-text display
- Fade-in/fade-out sequences

These commonly use 2–4 levels of nesting. The 8-level limit is generous for SA's own scripts. Modders writing complex procedural logic may approach the limit if they use deeply nested gosub chains.

### Stack overflow behavior

When `GOSUB` is called at depth 8 (stack full), there is no bounds check in the disassembly — the depth counter is incremented and the return address is written past the end of the stack region at `+0x37`. This overwrites the bytes at `+0x38` onward, which in the 224-byte layout is within the local variables block (`+0x3C` starts locals). A single overflow corrupts local 0's value; a second overflow corrupts local 1, etc.

The corruption is silent — no crash immediately, but the script will read garbage from those locals and behave unpredictably. The game's own SCM compiler generates scripts that never exceed depth 8, but a mod using GOSUB heavily must stay within 8 levels.

## The condition register

SA's SCM has a per-thread condition register (exact field offset ⏳) that stores the result of the most recent comparison or boolean opcode. This register feeds `JUMP_IF_FALSE`. Each comparison opcode (`IS_CHAR_DEAD`, `IS_CAR_IN_AREA_2D`, arithmetic comparisons, etc.) writes a 1-bit true/false into this register.

The `AND_*` / `OR_*` prefixes to opcodes (SCM compiler syntax) are compiled as a sequence of tests that aggregate results via AND or OR into the condition register before the conditional jump.

## The local variable file: 34 slots in 3 groups

The 34 × 4 = 136 bytes at `+0x3C..+0xBF` serve three distinct purposes within SA's scripting system:

| Slots | Purpose |
|---|---|
| 0–31 (32 slots) | General-purpose integer/float/handle variables |
| 32–33 (2 slots) | **Timer variables** — polled by `WAIT_UNTIL_TIMER_REACHES_VALUE` |

The two timer slots (32 and 33) are the SCM system's per-thread countdown timers. They count upward in milliseconds (auto-incremented by `Process` each frame by the elapsed time) and reset to zero when their opcode target is reached. Scripts use them for "do this task within N seconds" patterns.

## Global variables

Beyond the 34 per-thread locals, SCM has a global variable space at `0xA49960` (the main SCM globals array). These are shared across all threads and persist across thread lifetimes. Globals 0..N are assigned at compile time by the Rockstar script compiler.

The distinction:
- **Locals** `+0x3C..+0xBF` (or `0xA48960` for mission threads) — thread-private, reset on thread create
- **Globals** `0xA49960..` — shared across all threads, persist until game end or explicit reset

## Common SCM patterns and their bytecode

### "Wait for player to enter zone" loop

```scm
:wait_loop
    WAIT 0
    IS_PLAYER_IN_AREA_2D 0, -100.0, -200.0, 100.0, 200.0 false
    JUMP_IF_FALSE @wait_loop
```

Bytecode structure: `JUMP 0001 0000 0001` (wait one frame) → condition opcode → `JUMP_IF_FALSE target`.

### "Run for at most 30 seconds" with timer

```scm
    LOCAL_32 = 0          ; reset timer slot 32
:timed_loop
    WAIT 0
    IS_TIMER_32_PAST 30000
    JUMP_IF_FALSE @work_code
    JUMP @timeout
:work_code
    ...
    JUMP @timed_loop
:timeout
    ...
```

Timer slot 32 auto-increments each frame; `IS_TIMER_32_PAST` checks if it has exceeded the threshold.

### Gosub for reusable logic

```scm
    GOSUB @setup_camera
    GOSUB @spawn_actors
    GOSUB @play_cutscene
    TERMINATE_THIS_SCRIPT

:setup_camera
    ...
    RETURN

:spawn_actors
    ...
    RETURN
```

Each `GOSUB` pushes the current PC and jumps; each `RETURN` restores it. The maximum 8-deep chain means up to 8 nested `GOSUB`s can be active simultaneously per thread.

## Interaction with the entity pools (C53)

Scripts that create peds, vehicles, or objects via `CREATE_CHAR`, `CREATE_CAR`, `CREATE_OBJECT` are allocating from the C53 entity pools. The returned handle is a 32-bit packed value (C53 generational handle) stored in a script local or global variable. When the script later calls an opcode on this handle, the opcode validation calls `CPool::GetAt` with the handle — if the entity has been deleted (pool slot reused, version counter changed), `GetAt` returns null and the opcode typically does nothing.

This is why SA scripts robustly handle mission vehicles being destroyed by the player before the mission ends: the vehicle handle in the script local simply no longer resolves, and the script's vehicle-check opcodes return "false" for a dead handle.

## Why no recursive scripts

SCM `GOSUB` is the closest thing to a function call in SA scripts, but there is no way to pass arguments to a gosub target or receive return values (locals are shared within the thread, not scoped per-call). True recursion would require either explicit stack management in locals (using locals as a manual stack) or `START_NEW_SCRIPT` to create a new thread for each recursive call. Neither is practical in SA's script system, so all game logic is iterative, not recursive.

**Previous:** [C32.4 — Script thread lifecycle](04-script-thread-lifecycle.md)  
**Up:** [C32 — CRunningScript](C32-CRunningScript.md)
