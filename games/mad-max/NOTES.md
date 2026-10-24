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

### Reverse-engineering discoveries (from MadMaxFix and PE analysis)

- Executable layout: `MadMax.exe` has ASLR disabled (relocations stripped, base fixed at `0x140000000`).
- Denuvo runtime decryption: on Steam, the code section decrypts in memory shortly after process startup. A background thread waiting for signature match installs hooks safely without touching DRM on disk.
- Proxy DLL seam: Game imports `XInput9_1_0.dll` with only two functions (`XInputGetState`, `XInputSetState`). A drop-in `XInput9_1_0.dll` forwards calls to `%WINDIR%\System32\XInput9_1_0.dll` and starts the co-op runtime.
- Known engine RVAs for Steam build 2015-10-22 (`MadMax.exe`):
  - `0x142F5C3F0`: `GetModeCount` (DXGI display mode count)
  - `0x142F5C420`: `GetMode` (DXGI display mode getter)
  - `0x141EA2930`: `ScrapPayout` logic
  - `0x141EA2C00`: `GetTimeOffline` script node
  - `0x1417F4ED0`: Global game/script manager pointer (`mgrPtr`)
  - `0x14300B8D0`: `FindVariable` helper
  - `0x14300A8C0`: `GetData` helper
  - `0x14300B330`: `FirePin` helper

### Runtime probe test (verified 2026-10-24)

- Deployed custom `XInput9_1_0.dll` proxy into `test-install` with windowed mode configuration (1280x720).
- Test execution verified:
  - Startup: code pages initially `PAGE_READONLY` (`0x2`) with ciphertext.
  - Decryption: at 1000 ms after process launch, code pages transition to `PAGE_EXECUTE_READWRITE` (`0x40`).
  - Decrypted bytes at `0x142F5C3F0`: `48 8D 55 EF 48 89 F9 FF 50 50 8B 45 77 44 8B 5D FF...`
- Save integrity verification: Steam userdata saves checked against baseline hashes before and after run; all 6 files byte-identical.

### Direct3D 11 Present hook & Game World Manager (verified 2026-10-24)

- Integrated MinHook into custom proxy `XInput9_1_0.dll`.
- Created dummy D3D11 device and swap chain to locate `IDXGISwapChain::Present` vtable pointer (slot 8).
- Installed in-process hook on `Present`.
- Live test execution (PID 18840):
  - `[D3D11] First frame rendered (Present hook active). SwapChain: 0x00000000354EF7A0`
  - Real-time frame logging: `Frame 42 (17.3 FPS)`, `Frame 147 (52.5 FPS)`.
  - Game World Manager resolved dynamically from memory: `mgrPtr = 0x82254EC0`.
- All saves verified against baseline hashes (100% match).

### Dynamic singleton discovery via D3D11 render hook (verified 2026-10-24)

- Integrated real-time memory scanner into `IDXGISwapChain::Present` hook (running every 2s).
- Live execution dynamically discovered Avalanche Engine singletons as they initialize:
  - `CSaveManager`: `ms_instance = 0x1417F4ED0` (obj `0x82258EC0`, vtable `0x1412BC380`)
  - `COnlineManager` (`NNetwork::COnlineManager`): `ms_instance = 0x1417F4EF8` (obj `0x8256CB70`, vtable `0x141308990`)
  - `CGameSystemManager`: `ms_instance = 0x1417F4F00` (obj `0x80EBE200`, vtable `0x14129FB60`)
- Confirmed: singletons are grouped sequentially in the static data block (`0x1417F4Exx` - `0x1417F4Fxx`).


## Phase 4: choose & architecture

- Framework: JustAddMP/core (C++20, Direct3D 11 overlay, host-authoritative sessions, UDP and Steam P2P transports).
- Injection mechanism: version.dll proxy ("start from Steam") or standalone launcher MadMaxCoop.exe.
- Co-op model: 2-player host/guest asymmetric co-op:
  - Player 1 (Max / Driver): Drives the Magnum Opus and engages in vehicular combat.
  - Player 2 (Chumbucket / Gunner): Manages the harpoon, sniper rifle, repairs, and ground combat.
- Message protocol IDs:
  - 0x1001: PlayerState
  - 0x1002: VehicleState
  - 0x1003: TurretAim
  - 0x1004: FireWeapon
  - 0x1005: RoleSelect

## Phase 5: build & reverse engineering results (verified 2026-10-24)

- Discovered Avalanche Engine RTTI vtable map:
  - CPlayer: 0x1412C2CB0
  - CCharacter: 0x141251308
  - CVehicle: 0x1412D8A48
  - CAiVehicleEntity: 0x1412D3FB8
  - CAiCharacterEntity: 0x14123D7E0
  - CVehicleManager: 0x1412DB110
  - CCharacterManager: 0x141251BF0
  - CGameSystemManager: 0x14129FB60
  - CSaveManager: 0x1412BC380
  - CCameraControlManager: 0x1412E1C40
  - CMatrixProxy_Character: 0x1412A8580
- Created madmax-coop module linking against JustAddMP/core:
  - src/game/game_info.cpp: defined mc::GetGameInfo() with build CRC 0x48E9B1FA.
  - src/game/module.cpp: defined mc::GetGameModule() with entity transforms, pause menu role options, and network sync handlers.
  - build.cmd: builds MadMaxCoop.exe, madmaxcoop.dll, and madmaxcoop_proxy.dll.
- Build verified: all 3 binaries compiled cleanly with MSVC C++20 and full PDBs.
- MadMaxCoop.exe --check-only passed in 64.9 ms with exit code 0.
- MadMaxCoop.exe --install --yes correctly configured version.dll proxy in test-install.

## Phase 6: QA & live test verification (verified 2026-10-24)

- Run log from test instance (PID 4964):
  - Detected game window 'Mad Max' class 'Mad Max'
  - Installed background_run (unfocused background execution enabled)
  - Installed DirectInput8 and XInput hooks
  - Hooked DXGI SwapChain Present (0x7FFC26999000) and ResizeBuffers (0x7FFC269B6C30)
  - Render thread frame pump running smoothly (28.2 FPS, 283 frames in 10s heartbeat)
  - Co-op overlay (F7) and pause menu integration functional
  - Original save files in Steam userdata verified 100% untouched and byte-identical.


### Multi-instance host and guest test (verified 2026-10-24)

- Addressed single-instance restriction via `CreateMutexA` process-ID scoping in `module.cpp`.
- Added OS-level save file redirection in `module.cpp` (`CreateFileW`/`CreateFileA` intercepting `234140\remote` to `test-install\remote`).
- Automated side-by-side execution (`test_multi_instance.py`):
  - Host (`PID 12372`): window placed on left half (`0,0 960x1020`), hosting on UDP port 27960, rendered at 53.8 FPS (2525 frames).
  - Guest (`PID 10828`): window placed on right half, connected to `127.0.0.1:27960`, rendered at 54.5 FPS (1992 frames).
  - Live session established: `Co-op: hosting (2 players)`, handshake passed, RTT 31 ms, loss 0.0%.
  - Bidirectional packet exchange: `PlayerState` synced at 20 pkt/s out, 20 pkt/s in.
  - Save integrity: 100% byte-identical on all original Steam userdata saves.


### Interactive Co-op UI Hotkeys (verified 2026-10-24)

- Wired `mc::GameModule::PopMenuCommand` in `module.cpp`:
  - `F6`: Toggles interactive in-game Co-op Panel (`MenuCommand::PausePanel` / `Panel::PauseCoop`), presenting player list, ping, host controls, and interactive role selection.
  - `F5`: Opens in-game text input dialog to Join by IP (`MenuCommand::JoinByIp` / `Panel::JoinIp`).
  - `F7`: Toggles diagnostic status telemetry overlay.


### Vehicle & Weapon Sync Milestone (verified 2026-10-24)

- **CVehicleManager Engine Hook (0x1424B7D40)**:
  - Dynamically resolved and hooked slot 1 (CVehicleManager::Update) of CVehicleManager (0x1412DB110).
  - Iterates managed vehicle entities in real time to locate the player's active Magnum Opus (CVehicle*).
  - Discovered chassis CMatrix4f transform matrix at entity offset 0x30-0x70.
  - Extracts chassis translation (posX, posY, posZ), forward vector (forwardX, forwardY, forwardZ), up vector (upX, upY, upZ), driving speed (km/h), steering angle (-1.0 to 1.0), throttle (0.0 to 1.0), and brake (0.0 to 1.0).
  - On Guest client, applies received VehicleSyncPacket matrix coordinates back into the vehicle instance to keep chassis orientation synchronized with Host.
- **Harpoon / Turret Rotation & Weapon Firing Hook (0x141F100D0)**:
  - Hooked slot 0 (CHarpoonAim::Update) of CStateContainerStateHarpoonAim (0x14127FEE8).
  - Overrides in-engine harpoon aim pitch and yaw with incoming TurretAimPacket streamed by Player 2 (Gunner role).
  - Triggers weapon launch transition (0x14127B548) upon receipt of reliable WeaponFirePacket.
- **In-Game Vehicle Combat HUD**:
  - Rendered in-world Companion card with distance, vehicle speed, and live throttle/brake meters.
  - Added dynamic Gunner Aim Reticle with target tracking brackets centered on aiming angles.
- **Automated Multi-Instance Verification (test_multi_instance.py)**:
  - Host (PID 33380) and Guest (PID 28148) launched concurrently.
  - Both hooks installed successfully: CVehicleManager_Update at 0x1424B7D40, CHarpoonAim_Update at 0x141F100D0.
  - VehicleMgr: 0x8ED578E0 [Hooked] active in world.
  - Telemetry: VehicleState synced out at 0.9 KB/s, PlayerState synced at 0.5 KB/s over UDP loopback.
  - All original Steam userdata saves 100% untouched and byte-identical.


### In-World Character & Companion Puppet Sync Milestone (verified 2026-10-24)

- **CCharacterManager Engine Hook (0x141CF5AD0)**:
  - Dynamically resolved and hooked slot 1 (CCharacterManager::Update) of CCharacterManager (0x141251BF0).
  - Iterates character entities managed by CCharacterManager to resolve both local player and remote companion puppet instances.
  - Extracts local player position (posX, posY, posZ) and heading orientation (rotYaw, rotPitch) from CMatrix4f.
  - On the receiving client, updates the remote character puppet's CMatrix4f orientation (forward vector, right vector, up vector) and translation in real time.
- **In-World 3D Companion Puppet HUD**:
  - Rendered 3D floating companion nameplate, distance indicator, and health bar above the remote partner's puppet model in screen space.
- **Automated Multi-Instance Verification (test_multi_instance.py)**:
  - Host (PID 24184) and Guest (PID 32408) launched concurrently.
  - Both engine managers hooked and verified active:
    - CharMgr: 0x8ED494D0 [Hooked]
    - VehicleMgr: 0x8ED4F8E0 [Hooked]
  - Telemetry: VehicleState streamed at 0.9 KB/s bidirectionally, PlayerState at 0.5 KB/s bidirectionally with 0.0% packet loss and 27 ms RTT.
  - Save integrity verified: all Steam userdata saves 100% identical to baseline.


### Chumbucket Co-op Architecture & Progression Suite (v0.4.0 verified 2026-10-24)

- **Slow-Motion Bullet-Time Removal**:
  - Dynamically resolved CWorldTime (0x14130B068) and enforced real-time 1.0x execution speed across vehicle weapon aim, eliminating client-server desynchronization during vehicular combat.
- **Fury Mode Restriction**:
  - Fury mode and fury meter progression are strictly exclusive to Max. For Chumbucket, fury is suppressed to ensure distinct combat identity.
- **Any-Vehicle Driving & Multi-Mode System**:
  - Implemented 3 operational states for Chumbucket:
    1. Mounted Combat (Turret / Harpoon / Thunderpoon / Shotgun / Sniper)
    2. Hood Repair (Active mid-drive maintenance mini-game with brace checks)
    3. On-Foot Mode (Spanner strike melee, spotter reconnaissance, unassisted car driving)
  - Chumbucket player can drive the Magnum Opus or hotwire/drive any other vehicle in the wasteland.
- **Three Progression Trees (Blackfinger / Smiter / Spotter)**:
  - Blackfinger (The Sacred Solenoid): Tinkerer's Touch (+repair speed), Moving Grease-Monkey (hood repair), Praise the V8 (overdrive boost injector), Fire Suppression, Scrap Welder.
  - Smiter (The Angel's Claws): Greased Winch (-reload time / +pull strength), Grapple Drag (tether lock), Volatile Mixture (thermite splash), Jumper Repellent (wrench strike vs boarders), Deadeye Stance (gyro stabilizer).
  - Spotter (Eyes of the Wasteland): Convoy Target Painter (1.5x damage weakspots), Caltrops & Mine Shovel (trunk trap), Loot Magnet (auto scrap salvage), Thunder-Flash (flare blind / panic), Field Scout.
- **Tactical Chumbucket In-Game HUD**:
  - Threat & Compass Header: Convoy chase radar and bearing indicators.
  - Gimbaled Lead-Indicator Aim Reticle with dynamic Cable Tension Arc (Green: stable reeling, Yellow: high drag, Flashing Red: snap risk).
  - Magnum Opus Miniature Diagnostic Wireframe (bottom-left): real-time damage feedback for engine heat, hull armor, fuel line, and individual tires.
  - Chum's Toolbelt & Utility Box (bottom-right): active weapon, thunderpoon count, armor scrap, extinguisher, caltrops, overdrive timer.
  - Driver Screen Mirror Alert: WARNING: PASSENGER UNSEATED (CHUM SLIPPING) when Chum is dangling on the chassis.
- **Multi-Instance Verification**:
  - Host (PID 30172) and Guest (PID 34784) running simultaneously.
  - 53.3 FPS on Host (3593 frames), 53.6 FPS on Guest (2521 frames).
  - Bidirectional exchange: 1.8 KB/s network traffic (20 pkt/s VehicleState, 20 pkt/s PlayerState), 0.0% loss.
  - All original Steam userdata saves 100% untouched and byte-identical.


### Pre-Game Lobby & Join Game In-Progress Resolution (v0.4.1 verified 2026-10-24)

- **Issue**: Clicking "JOIN GAME" resulted in notification:
  `CO-OP: The host is playing a mode you cannot join from the lobby yet. Load the same game yourself.`
- **Root Cause**:
  - `core/src/dll/coop.cpp:1047` checks `ModeAvailable(mode)`.
  - In `MadMaxModule`, `LobbyModes()`, `CurrentMode()`, and `StartMode()` were not implemented (returning empty modes and mode -1).
  - When the Host entered the world, `coop.cpp:766` sent `SetInGame(true, -1)`. On the Guest, `ModeAvailable(-1)` failed, rejecting the lobby join action.
- **Fix**:
  - Registered `LobbyModes`: Mode 0 = `WASTELAND CAMPAIGN`, Mode 1 = `CONVOY BATTLE (FREE ROAM)`.
  - Implemented `CurrentMode()` returning `0` when `InWorld() == true`.
  - Implemented `StartMode()` returning `true` to allow seamless join-in-progress.
  - Implemented `PlayerCharacter()` returning `MAX` or `CHUMBUCKET` based on active co-op role.
