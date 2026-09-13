# C52.5 — Pause states and bIsFrozen

## The two flags and what they gate

`CTimer` has two flags that together gate whether the physics integration advances:

| VA | Name | Default | Role |
|---|---|---|---|
| `0xB7CB48` | `bIsFrozen` | 0 | Set to 1 during loading, some cutscene states, and explicit pause |
| `0xB7CB49` | `bGameStarted` | 0 → 1 | Set to 1 when the world is fully loaded and gameplay begins |

`CTimer::Update` at `0x560D2E` tests both at the start of its clamp logic. If `bIsFrozen != 0` OR `bGameStarted == 0`, the timestep is written as the near-zero frozen value (`0x3C23D70A ≈ 0.008`) instead of the actual elapsed time. This value is not zero — it is small enough that the world effectively stands still, but it allows continuous-integration code to avoid divide-by-zero.

## Who writes bIsFrozen

`bIsFrozen` is written by several sites in `.text` — these are the confirmed callsites:

| Who | What it does | Direction |
|---|---|---|
| `CGame::InitialiseWhenReady` | Streaming load complete → unfreeze | 0 |
| `CLoadingScreen::Start/Stop` | Entering/leaving loading screen | 1 / 0 |
| Pause menu handler | Menu opened/closed | 1 / 0 |
| Mission-pass / fail handler | Brief post-mission freeze | 1 / 0 |
| Cutscene system (`CRecordDataForGame`) | Some scripted freeze states | 1 / 0 |

The pattern is always direct byte writes to `0xB7CB48`. Any mod can read or write this address to freeze or unfreeze the simulation.

## The loading screen state in detail

During the initial game load (pre-world-ready), `bGameStarted` at `0xB7CB49` is 0. Even if `bIsFrozen` were 0, the timestep would still be forced to the frozen value because `bGameStarted == 0` trips the same branch. This two-flag design means the game's first frame after a load has the simulation frozen by default — the world doesn't suddenly simulate the full load duration when the screen clears.

`bGameStarted` is set to 1 exactly once, when `CGame::InitialiseWhenReady` determines the world is ready to simulate. After that it remains 1 for the lifetime of the process — it is never reset to 0 except on returning to the main menu, where a full re-init sets it back to 0 until the next load completes.

## The frozen timestep value

When the freeze condition is met, `ms_fTimeStepNonClipped` is written with `0x3C23D70A`:

```
0x3C23D70A as a float = 0.00999999... ≈ 0.01
```

This is slightly above 0 — just barely above the minimum clamp (`0.005` at rdata `0x858C18`). The reason it is not zero:

1. Code that divides by `ms_fTimeStep` would crash on divide-by-zero if the value were ever exactly 0.
2. Animation and camera blending need to advance slightly even during a freeze so they can reach their target poses rather than snapping.
3. Some timer countdowns that gate the unfreeze sequence itself need to tick even during the freeze.

The result is that even in a frozen state, the world ticks at approximately 1/67 of normal rate — slow enough to be invisible, fast enough to be safe.

## The pause menu: frozen vs paused

When the pause menu is open, `bIsFrozen = 1`. The render loop continues to run (the world is visible behind the menu), but `CGame::Process`'s physics/AI/timer updates are gated by `bIsFrozen`. From the engine's perspective, the world is frozen but not stopped:

- **Render:** continues normally (the last simulation state is rendered indefinitely)
- **Physics:** `ms_fTimeStep ≈ 0.01` — effectively stopped
- **Scripts:** SCM continues to run (the script interpreter's own loop is separate from CGame::Process's physics gate), but most script opcodes that do gameplay things check their own conditions
- **Audio:** continues independently of CTimer

**Important for modders:** if you set `bIsFrozen = 1` in your mod, the physics and AI stop, but the render and audio keep running. This is the correct pattern for a "slow-motion" or custom pause effect. Do not stop the render loop to freeze the game — freeze the timer instead.

## Implementing slow motion

The "correct" slow-motion implementation in SA uses the timer directly. There are two approaches:

**Approach 1 — Override the timestep via hook:**
Hook `CTimer::Update` at `0x560D2E`. After it writes `ms_fTimeStep`, read the written value and multiply it by your slow-motion factor (0.5 = half speed, 0.1 = 10% speed). Write the result back to `0xB7CB5C`. This is the cleanest approach — it scales all physics, timers, and AI uniformly.

**Approach 2 — Override the clamp constants:**
The max clamp constant at rdata `0x858C14` (2.0) limits how large `ms_fTimeStep` can get. For slow motion, you do not want a larger clamp — you want a smaller value. Instead, patch the write site at `0x560DA2` to load a different constant. This is less flexible than Approach 1.

For slow motion use Approach 1 — it correctly scales `ms_fTimeStepNonClipped` as well, keeping the camera interpolation proportional.

## Implications for custom loading screens

Custom loading screens that hook `CGame::InitialiseWhenReady` to extend the loading screen must ensure `bGameStarted` is eventually set to 1 and `bIsFrozen` is cleared. Failing to clear `bIsFrozen` after a custom load sequence will leave the game's physics frozen — the world renders but nothing moves, and this often looks like a successful load that "hangs" silently.

The sequence to safely end a custom freeze:
1. Ensure all streaming assets are loaded (check `CStreaming::ms_bIsAlreadyDoingAMission` or equivalent)
2. Write 0 to `bIsFrozen` (`0xB7CB48`)
3. Verify `bGameStarted` (`0xB7CB49`) is 1

## Key takeaways

- `bIsFrozen` (`0xB7CB48`) gates `CTimer::Update`'s clamp — set it to 1 to pause physics without stopping the render or audio.
- `bGameStarted` (`0xB7CB49`) is the world-ready flag — it must be 1 for normal simulation; it is only ever 1 after `CGame::InitialiseWhenReady` signals completion.
- The frozen timestep value (`≈ 0.01`) is deliberately not zero — it keeps the engine numerically safe and allows slow-but-nonzero advance of certain systems.
- Script `WAIT` timers use `ms_nTimeInMilliseconds` (real-time) and are unaffected by `bIsFrozen`.
- Slow-motion is best implemented by hooking `CTimer::Update` and scaling the written `ms_fTimeStep` value.

**Previous:** [C52.4 — How ms_fTimeStep propagates](04-timestep-consumers.md)  
**Up:** [C52 — CTimer and the Game Loop](C52-CTimer-And-Game-Loop.md)
