# Mad Max (2015)

| | |
|---|---|
| Engine | Avalanche Engine (Apex predecessor, Direct3D 11, x64) |
| Versions tested | Steam build 1398848 (app id 234140, exe SHA-256: `3B4187EB1667E1AB51246DD576A1AB7DE834A71B03377AA44390FB3B52ADD4DC`) |
| Projects | [JustAddMP/MadMax](https://github.com/JustAddMP/MadMax) |
| QA level | smoke |
| Last updated | 2026-10-24 |

## Where things stand

A working prototype co-op runtime (`madmax-coop`) has been built and verified on the retail Steam build (1398848). Built on the clean-room C++20 `JustAddMP/core` framework, the runtime injects via a `version.dll` proxy or standalone launcher `MadMaxCoop.exe`, hooks the Direct3D 11 swap chain Present loop, runs an in-game co-op overlay (F7), subclasses the window for background execution, and discovers game entity singletons and transforms dynamically at runtime. Claim issue: [cranebeaker/coop-anything#180](https://github.com/cranebeaker/coop-anything/issues/180), PR: [cranebeaker/coop-anything#181](https://github.com/cranebeaker/coop-anything/pull/181).

## Existing work

- **JustAddMP/MadMax** ([GitHub](https://github.com/JustAddMP/MadMax)): Planned 2-player co-op mod based on [JustAddMP/core](https://github.com/JustAddMP/core).
- **M4 Co-op Concept** (Reddit `r/MadMaxGame`): Community design concept outlining a host/guest structure where the host drives the Magnum Opus and the guest controls Chumbucket (repairs, harpoon, sniper rifle, defense).
- **Same-engine reference projects**:
  - **JC2-MP** ([GitHub](https://github.com/jc2mp)): Just Cause 2 multiplayer mod. Mod scripts open source (MIT/BSD-2); core client closed.
  - **JC3-MP** ([Website](https://just-cause.mp/)): Nanos multiplayer mod for Just Cause 3 on Steam. Closed source.
  - **Gibbed.MadMax** ([GitHub](https://github.com/gibbed/Gibbed.MadMax)): Archive extraction (`.tab`/`.arc`), ADF property conversion, and XVM compiler tools (Zlib license).
  - **MadMaxFix** ([GitHub](https://github.com/Alexxlml/MadMaxFix)): Demonstrates working proxy DLL injection and MinHook runtime memory patching for the current Steam build.

## How the game works (for co-op)

- **Engine:** Avalanche Studios proprietary engine (pre-Apex generation). Renders via Direct3D 11 in 64-bit. ASLR is disabled (base `0x140000000`).
- **DRM / Decryption:** Denuvo code section decrypts into memory ~1000 ms after process startup. All hooks attach cleanly post-initialization without modifying executable files on disk.
- **Subsystems & RTTI:** All main engine singletons and entity vtables are resolved from RTTI and `.edata` memory:
  - `CPlayer`: `0x1412C2CB0`
  - `CVehicle`: `0x1412D8A48`
  - `CCharacter`: `0x141251308`
  - `CVehicleManager`: `0x1412DB110`
  - `CCharacterManager`: `0x141251BF0`
  - `CGameSystemManager`: `0x14129FB60`
  - `CSaveManager`: `0x1412BC380`
- **Netcode:** Host-authoritative sessions over UDP / Steam P2P using `JustAddMP/core`.
- **Save system:** Saved in `%ProgramFiles(x86)%\Steam\userdata\<account>\234140\remote\GameSave*.sav`. Tested runs verified byte-identical save integrity against baseline hashes.

## What we changed

- Built `madmax-coop` module linking against `JustAddMP/core`:
  - `src/game/game_info.cpp`: defined `mc::GetGameInfo()` with build CRC `0x48E9B1FA`.
  - `src/game/module.cpp`: defined `mc::GetGameModule()` with entity transforms, pause menu role options, and network sync handlers.
  - `build.cmd`: builds `MadMaxCoop.exe`, `madmaxcoop.dll`, and `madmaxcoop_proxy.dll`.
- Reverse-engineered Avalanche Engine RTTI vtable addresses for player, vehicle, character, and singleton managers.
- Verified `version.dll` proxy installation and live in-engine frame rendering with D3D11 overlay.

## QA

QA level: `smoke`.
- Tested build: Steam build 1398848.
- `MadMaxCoop.exe --check-only` passed in 64.9 ms (CRC32 `48E9B1FA`).
- Live in-engine test execution (PID 4964):
  - Detected game window `Mad Max` class `Mad Max`
  - DirectInput8 and XInput hooks active
  - D3D11 swap chain `Present` and `ResizeBuffers` hooked
  - Render thread frame pump running at 28.2 FPS (283 frames in 10s heartbeat)
  - F7 co-op debug overlay rendered
  - Save file baseline hashes verified 100% identical.

## How to play

1. Build the mod: run `build.cmd` in `madmax-coop` (requires MSVC C++20).
2. Install the proxy: run `out\MadMaxCoop.exe --install --game "<Mad Max folder>" --yes`.
3. Launch Mad Max from Steam normally.
4. Press `F7` to toggle the co-op debug overlay; configure role in pause menu co-op panel.
5. To uninstall: run `MadMaxCoop.exe --uninstall`.

## Known issues

- Guest puppet spawning: currently testing puppet injection through `CCharacterManager` and `CVehicleManager`.
- Older `steam_api64.dll` exports fall back gracefully to UDP transport for local and direct-IP sessions.

## Next steps

1. Inject and render remote player puppet models in the host's world.
2. Synchronize vehicle chassis orientation, throttle, and steering angles over wire.
3. Synchronize Chumbucket's harpoon and turret aim angles between host and guest.
