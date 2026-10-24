# Mad Max (2015)

| | |
|---|---|
| Engine | Avalanche Engine (Apex predecessor, Direct3D 11, x64) |
| Versions tested | Steam build 1398848 (app id 234140, exe SHA-256: `3B4187EB1667E1AB51246DD576A1AB7DE834A71B03377AA44390FB3B52ADD4DC`) |
| Projects | [JustAddMP/MadMax](https://github.com/JustAddMP/MadMax) |
| QA level | none |
| Last updated | 2026-10-24 |

## Where things stand

No playable co-op or multiplayer mod exists for Mad Max today. The game was built strictly as a single-player title with closed binaries and no shipped multiplayer netcode. One community effort ([JustAddMP/MadMax](https://github.com/JustAddMP/MadMax)) has created a project repository with plans to build a co-op mod using their shared C++ `JustAddMP/core` framework, but code development has not yet started.

## Existing work

- **JustAddMP/MadMax** ([GitHub](https://github.com/JustAddMP/MadMax)): Planned 2-player co-op mod. Repository created October 2026; currently a placeholder README. Plans to build on [JustAddMP/core](https://github.com/JustAddMP/core), which provides D3D11 overlay, MinHook, and Steam P2P session handling.
- **M4 Co-op Concept** (Reddit `r/MadMaxGame`): Community design concept outlining a host/guest structure where the host drives the Magnum Opus and the guest controls Chumbucket (repairs, harpoon, sniper rifle, defense).
- **Same-engine reference projects**:
  - **JC2-MP** ([GitHub](https://github.com/jc2mp)): Just Cause 2 multiplayer mod. Mod scripts open source (MIT/BSD-2); core client closed.
  - **JC3-MP** ([Website](https://just-cause.mp/)): Nanos multiplayer mod for Just Cause 3 on Steam. Closed source.
  - **Gibbed.MadMax** ([GitHub](https://github.com/gibbed/Gibbed.MadMax)): Archive extraction (`.tab`/`.arc`), ADF property conversion, and XVM compiler tools (Zlib license).
  - **MadMaxFix** ([GitHub](https://github.com/Alexxlml/MadMaxFix)): Demonstrates working proxy DLL injection (`XInput9_1_0.dll`) and MinHook runtime memory patching for the current Steam build.

## How the game works (for co-op)

- **Engine:** Avalanche Studios proprietary engine (Avalanche Engine 3.0 / pre-Apex). Renders via Direct3D 11 in 64-bit.
- **Scripting:** Compiled XVM bytecode (`.xvmc`) and ADF (Avalanche Data Format) property graphs stored in `.arc` archives.
- **Netcode:** None shipped in retail game. A co-op mod requires building host authority and state synchronization over a P2P transport (e.g. SteamNetworkingMessages / Spacewar relay).
- **Save system:** Saved in `%ProgramFiles(x86)%\Steam\userdata\<account>\234140\remote\GameSave*.sav`. Host must act as world authority; guest syncs mission state, vehicle upgrades, and inventory.
- **Injection:** Engine loads DLLs from its local folder; `XInput9_1_0.dll` or `bink2w64.dll` proxying allows in-process initialization without external injectors.

## What we changed

Initial workspace setup and research cataloguing. No game binaries modified; isolated test install created and saves backed up and hashed.

## QA

No co-op mod code has been executed yet. QA level is `none`.

## How to play

Currently unplayable in native co-op. For couch/remote sharing without independent players, Steam Remote Play Together or Parsec allows streaming one instance.

## Known issues

None (no mod released yet).

## Next steps

1. Perform Ghidra static analysis of `MadMax.exe` to map entity classes (Player, Vehicle, Weapon, Harpoon) and entry points.
2. Build a baseline proxy DLL using MinHook to verify in-process hooks and D3D11 swap chain hooking on the test install.
3. Coordinate with the `JustAddMP` project maintainers to avoid duplicate effort.
