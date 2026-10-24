# Mad Max (2015) - working notes

Session 1: 2026-10-24. Hub commit at start: 5b86e5bb10ce864ab9872f2589651f73ffcae5db.

## Game identity (rule 5)

| Field | Value |
|---|---|
| Title | Mad Max (Avalanche Studios, 2015) |
| Store | Steam, app id 234140 |
| Steam build id | 1398848 (from appmanifest_234140.acf) |
| Main exe | `<game dir>\MadMax.exe`, 76,618,184 bytes, no version resource |
| SHA-256 of MadMax.exe | 3B4187EB1667E1AB51246DD576A1AB7DE834A71B03377AA44390FB3B52ADD4DC |
| Engine | Avalanche Engine (pre-Apex generation). Derived: uses `.arc`/`.tab` archives, `.adf` data, `.xvmc` bytecode |
| OS / arch | Windows, x64, Direct3D 11 |
| Install size | 32.0 GB (31.97 GB, 113 files) |

## Phase 0: triage

### Observations (checked this session)

- DIRECTORY.md and `games/` have no Mad Max entry (checked 2026-10-24).
- Install layout: `archives_win64`, `patch_win64`, `dlc_win64` holding 46 `.arc` + 46 `.tab`
  pairs; `MadMax.exe`; `*_F.dll` "fragment" DLLs (EffectFragments_F, FragmentBundle_F,
  GuiFragmentBundle_F, GuiXvmFragments_F, IOFragments_F, ProfileContext_F);
  `fmodex64.dll`, `fmod_event64.dll`, `bink2w64.dll`, `steam_api64.dll`, `dbghelp.dll`,
  `Shaders_F.shader_bundle`, `settings.ini`.
- The tested install is unmodded (owner confirmed). Only `steam_appid.txt` (contains `234140\n\0`)
  and `settings.ini` changed after install; derived: both written by the game itself.
- Save location: `%ProgramFiles(x86)%\Steam\userdata\<account>\234140\remote\GameSave*.sav`. Backed up and hashed before any testing.
- No shipped multiplayer of any kind. No script source in the install (closed binary).
- Existing repos:
  - https://github.com/JustAddMP/MadMax: README only ("Status: planned. Work hasn't started yet."), last pushed 2026-10-17.
  - https://github.com/JustAddMP/core: Active framework (pushed 2026-10-21). Provides MinHook, D3D11 Present hook overlay, Steam P2P/UDP networking with host-authoritative sessions, lobbies, Scaleform hooks.
  - https://github.com/gibbed/Gibbed.MadMax: Zlib license, pushed 2017-07-19. Tools for .tab/.arc extraction, ADF conversion, XVM decompile/compile.
  - https://github.com/Alexxlml/MadMaxFix: XInput9_1_0.dll proxy DLL using MinHook. Proof that DLL proxy injection works on current Steam build.
  - https://github.com/x0reaxeax/MadMaxCrashFix: MIT license, dinput8.dll / XInput9_1_0.dll crash fix for display modes > 256.

### Derived

- Avalanche Engine uses Direct3D 11 and loads proxy DLLs placed in the game root (`XInput9_1_0.dll`, `dinput8.dll`, `bink2w64.dll`).
- Game is physics-heavy (vehicular combat) and single-player only; requires host authority with state sync (driving, harpoon, combat, world state).
- Triage rating: "Closed binary, no MP, no scripting" -> Build from zero / extend framework path.

### Unknowns

- Whether Denuvo is active on current Steam build 1398848.
- Exact vehicle and player object structures in memory (to locate via Ghidra / runtime probe).
- Extent to which `dropzone` loose file override works in this specific build.

## Phase 1: research

### Existing co-op attempts

| Project | Link | License | Last activity | Really syncs | Notes |
|---|---|---|---|---|---|
| JustAddMP Mad Max | https://github.com/JustAddMP/MadMax | none stated | 2026-10-17 | nothing (planned) | Placeholder repo. Plans to use JustAddMP/core. |
| M4 (Reddit concept) | r/MadMaxGame | none | unknown | nothing | Community design proposal for Max + Chumbucket 2-player co-op. |

Searched across GitHub, Nexus Mods, ModDB, Reddit, Steam Community, YouTube, Russian, German, and Polish forums. No released playable or alpha mod exists.

### Same-engine projects

| Project | Game | Link | Status | Notes |
|---|---|---|---|---|
| JC2-MP | Just Cause 2 | https://github.com/jc2mp | playable | Avalanche Engine 2.0; server scripts open source (MIT/BSD-2), core client closed. |
| JC3-MP (nanos) | Just Cause 3 | https://just-cause.mp/ | released | Avalanche Engine (Apex); closed-source client and server on Steam (app 619910). |

## Phase 2: workspace setup (completed 2026-10-24)

- Isolated test copy: `C:\Users\gm491\coop\mad-max\test-install` (31.97 GB, 113 files, byte-identical to Steam install).
- Save backup: `local\save-backup-20261024` (hashes recorded in `local\save-hashes-20261024.txt`).
- Ghidra: 12.2 installed at `C:\Users\gm491\coop\tools\ghidra_12.2_PUBLIC` (verified SHA-256 `2c2cd776db3c1f6a097ba418922378ac3649cadd5e844a6f5f088ae628a7be67`). Tested headless with JDK 25 (`EclipseAdoptium.Temurin.25.JDK`).
- Git & gh: Git installed; GitHub CLI installed at `C:\Program Files\GitHub CLI\gh.exe` (pending user login).
- Compiler: VS Build Tools pending manual administrator winget run.

## Phase 3: understand (next)

1. Run Ghidra headless analysis on `test-install\MadMax.exe`.
2. Map proxy DLL entry point (`XInput9_1_0.dll` or `bink2w64.dll`).
3. Scan exe strings for debug symbols, netcode stubs, and console commands.
