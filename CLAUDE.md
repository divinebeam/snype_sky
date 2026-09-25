# SNYPE_SKY

StarCraft: Brood War Remastered custom (EUD) map, written in **epScript (`.eps`)** — a JS-like language that compiles through euddraft/eudplib into map triggers. SNYPE_SKY is a rework of the old SNYPE map (`reference/SNYPE_OLD/`).

## Build / toolchain

- Project file: `snype_sky.e3s` (EUD Editor 3 v0.19.6, binary .NET format — never edit by hand).
- Input map `snype_sky.scx` → output `snype_sky_compiled.scx`. `.scx` files are binary; don't edit them.
- euddraft plugins enabled in the project: `[MSQC]`, `[unlimiter]`, `[eudTurbo]`.
- Compiling and testing happen in the EUD Editor 3 GUI and in-game. Claude cannot build or run the map, so check code carefully against the reference files and tell the user what needs testing.
- Map locations are defined in `snype_sky.scx` and listed as `L_` constants in `main.eps`. `$L("name")` of a location that doesn't exist fails the build, so only use existing ones (ask the user to add new locations in the map editor).
- All `.eps` source files go in the project root, next to `snype_sky.scx`. Each new file must be added to the e3s project in EUD Editor.

## Reference folder (`reference/`) — read-only

| Path | What it is |
|---|---|
| `epsFunctions.eps` | API reference: signatures and docs for all built-in triggers, conditions, eudplib functions, `StringBuffer`, `EUDArray`, `PVariable`, and more. **Check here first for any built-in.** |
| `ETK/` | Third-party utility library (ETKUnit = CUnit field getters/setters by EPD, ETKUtils = locations/sprites/damage helpers, ETKConstants, ETKTimer). |
| `DREAD/` | The user's **most recent and cleanest** project (`main.eps`, `common.eps` helpers). Use it as the style reference. |
| `KNIGHTS/` | Large single-file project (items, projectiles, bosses, `switch`, `py_` imports). |
| `SNYPE_OLD/` | The original SNYPE: multi-file layout (menu, projectiles, respawn, score, sound, mouse, coord, effects). Useful for game design and systems, but it is early code with weaker patterns. |
| `*/notes.txt` | The user's TODO, bug, and idea lists. `SNYPE_OLD/notes.txt` has SNYPE design ideas (perks, equipment, modes). |

The old projects contain bad patterns, such as long copy-pasted `if (p == 0) … else if (p == 1)` chains, reading `array[playerID]` over and over, and giant files. Borrow their ideas, not those patterns.

## epScript essentials

- **Values are unsigned 32-bit integers only.** There are no floats or strings as values. Negative numbers wrap around (`-1 == 0xFFFFFFFF`). Check for "negative" with `x >= 0x80000000` (or `> 4000000000`). Use fixed-point math for fractions (the user scales by 10000; HP is stored ×256).
- Module files: `import common as Cmn;` then `Cmn.Func()`. Folders use dots: `import ETK.ETKUtils as Utils;`. `import py_sys;` / `py_range(...)` reach Python and run at compile time (`foreach(i : py_range(n))` unrolls the loop).
- `var` is a runtime variable. `const` is a binding that cannot be reassigned (it can still hold a runtime value, e.g. `const x = CUnit(ptr).posX;`). Multiple returns work: `const x, y = GetPos();`.
- Global arrays: `EUDArray(n)`, `PVariable()` (8 slots, one per player), `[a, b, c]` literals. Arrays are fixed size.
- `function f(unit: TrgUnit) : TrgString` adds type hints. `object Foo { var a; function m() {} };` defines structs.
- Control flow: `if/else`, `for`, `while`, `break/continue`, `switch(x) { case 1: … break; }`, `foreach(u : EUDLoopCUnit())`.
- Trigger actions and conditions are called like functions (`CreateUnit(1, U_X, L_MAIN, $P1)`; `if (Bring($P8, AtLeast, 1, U_ANY, L_5X5))`).
- Names: `$L("loc name")` is a location, `$U("Unit Name")` is a unit ID, `$P1`..`$P12` are players (0-indexed: `$P1 == 0`), plus `Force1`, `AllPlayers`.
- Strings: `Db("text")` makes a static byte buffer. `StringBuffer()` with `.printf/.printfAt(line, fmt, …)` handles on-screen text. `sprintf(dst, fmt, …)` and `eprintf` exist for debugging. Format codes: `{}` is a number, `{:s}` is a string, `{:n}` is a player name, `{:c}` is a player color. Color and alignment codes are escapes (`\x04` white, `\x12` right-align, `\x13` center, …).
- Memory: `dwread/dwwrite`, `*_epd` variants, `EPD(addr)`, `MemoryXEPD`, `SetMemoryX`, `CUnit(ptr)` with fields like `.posX`, `.posY`, `.playerID`, `.unitType`, `.hp`, `.topSpeed`.
- Geometry: `atan2(y, x)` returns degrees, `lengthdir(len, angle)` returns `dx, dy`, `sqrt`, `div(a, b)` returns quotient and remainder. Map units are pixels; 1 tile = 32 px. `setloc(loc, x, y)` moves a location.

## Execution model

- Entry points in the main module: `onPluginStart()` runs once at game start, `beforeTriggerExec()` / `afterTriggerExec()` run every game frame (timers use frames: 24 ≈ 1 s on Fastest, so 72 = 3 s and 1440 = 1 min).
- Per-player logic: `EUDPlayerLoop()(); … EUDEndPlayerLoop();` loops over active players with `getcurpl()` set to each one. Trigger actions like `DisplayText` and `PlayWAV` target the current player (`setcurpl`/`getcurpl`).
- **Desync safety:** `IsUserCP()` / `getuserplayerid()` / raw mouse and screen memory are *local* to each client. Inside those branches, only change display state (text, sounds, local vars). Never change game state there, or the game desyncs.
- **Input** comes from the MSQC plugin, which writes key and mouse state into registered PVariables: `EUDRegisterObjectToNamespace("key_d", keypress_d);` paired with MSQC lines such as `NotTyping; KeyDown(D): key_d, 1` / `KeyUp(D): key_d, 2` and `Mouse: <location>`. Mouse position is read from per-player mouse locations. MSQC settings live in the e3s, so the user edits them in EUD Editor. The matching variables are in the Input section of `main.eps`; if the user changes the MSQC config, update that section to match.

## Conventions (follow DREAD style)

- Constants in UPPER_SNAKE with a prefix: `L_` locations, `U_` unit IDs, `IMAGE_`, `ISCRIPT_`, `MAX_…`.
- Per-player state: `const p_<name> = PVariable();` Wrap updates with side effects in setters (`set_p_power(playerID, v)`).
- Put shared math and utility helpers in a separate module (like `common.eps`).
- At the top of a player loop, read a player's array values into local `const`/`var` once instead of re-indexing `p_x[playerID]` throughout. The user has marked this as a goal.
- Loop over players or use lookup arrays instead of repeating an 8-way if-chain.
- Keep modules focused (menu, projectiles, respawn, score, sound, and so on) rather than one huge `main.eps`.
