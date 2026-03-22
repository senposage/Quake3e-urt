# Quake3e-urt — Full Engine Overview

> **How to use this documentation:** Start here to find which file owns any piece of functionality, then follow the link to the relevant section doc for detailed function listings, data structures, and interaction notes.
>
> **Custom changes** made in this fork are marked **[CUSTOM]**. Everything else is stock Quake3e / ioquake3.
>
> **UrbanTerror integration features** (partially derived from FTWGL) are marked **[URT]** and guarded by compile-time flags: `USE_AUTH`, `USE_FTWGL`, `USE_SERVER_DEMO`, `USE_URT_DEMO`, `NO_DMAHD`.

---

## Repository Layout

```
code/
├── qcommon/          ← Core engine: memory, console, cvar, cmd, filesystem, math, crypto
│   ├── common.c/h    ← Com_Init, Com_Frame, memory zones, hunk, logging
│   ├── cvar.c        ← Console variable system
│   ├── cmd.c         ← Command buffer and execution
│   ├── files.c       ← Virtual filesystem (PAK/PK3), path resolution
│   ├── msg.c         ← Bit-level message serialization + delta compression
│   ├── net_chan.c     ← [CUSTOM] Reliable/unreliable channel layer over UDP
│   ├── net_ip.c      ← OS socket abstraction (IPv4 + IPv6)
│   ├── vm.c          ← QVM loader, symbol tables, dispatch
│   ├── vm_x86.c      ← x86-64 JIT compiler
│   ├── vm_aarch64.c  ← ARM64 JIT compiler
│   ├── vm_armv7l.c   ← ARMv7 JIT compiler
│   ├── vm_interpreted.c ← Bytecode interpreter fallback
│   ├── cm_load.c     ← BSP collision-map loader
│   ├── cm_trace.c    ← Ray/box/capsule trace engine
│   ├── cm_patch.c    ← Curved-surface patch collision
│   ├── cm_test.c     ← Point-in-solid tests, area portals
│   ├── q_shared.c/h  ← [URT] Shared types: FTWGL version string, BASEGAME, window titles
│   ├── q_math.c      ← Trigonometry, vector ops, matrix ops
│   ├── qcommon.h     ← [URT] URT_PROTOCOL_VERSION=70, URTDEMOEXT, cvar defaults
│   ├── huffman.c     ← Huffman compression for network packets
│   ├── md4.c/md5.c   ← Hash functions (pak checksums, auth)
│   └── unzip.c       ← Zlib/deflate for PK3 file reading
│
├── server/           ← Dedicated / listen server
│   ├── sv_main.c     ← [CUSTOM] Frame loop, sv_fps/sv_gameHz decoupling
│   ├── sv_init.c     ← [CUSTOM][URT] Cvar registration (subtick + auth cvars)
│   ├── sv_client.c   ← [CUSTOM][URT] Usercmd handling, multi-step Pmove, auth drop
│   ├── sv_snapshot.c ← [CUSTOM] Snapshot building, position extrapolation
│   ├── sv_antilag.c  ← [CUSTOM] Engine-side shadow antilag (new file)
│   ├── sv_antilag.h  ← [CUSTOM] Antilag public API (new file)
│   ├── sv_game.c     ← QVM game syscall handler (G_TRACE intercept)
│   ├── sv_world.c    ← Entity world sectors, SV_Trace, SV_LinkEntity
│   ├── sv_ccmds.c    ← [CUSTOM][URT] Server console commands, server demo, clientScreenshot
│   ├── sv_bot.c      ← Bot integration with server
│   ├── sv_net_chan.c ← Server-side netchan encode/decode
│   ├── sv_filter.c   ← IP ban/filter system
│   ├── sv_rankings.c ← Rankings / statistics tracking
│   └── server.h      ← [CUSTOM][URT] Server-internal types: auth field, demo fields
│
├── client/           ← Client (renders, input, network, sound)
│   ├── cl_main.c     ← [URT] Connection, URT demo recording, auth, cvar defaults
│   ├── cl_input.c    ← [CUSTOM] Key→usercmd, mouse, download pacing
│   ├── cl_cgame.c    ← [CUSTOM][URT] cgame QVM interface, time sync, cvar intercept, screenshot
│   ├── cl_parse.c    ← [CUSTOM] Network message parser, snapshot interval EMA
│   ├── cl_console.c  ← In-game console rendering and history
│   ├── cl_keys.c     ← Key binding management
│   ├── cl_scrn.c     ← [URT] Screen layout, net monitor widget (cl_netgraph)
│   ├── cl_ui.c       ← UI QVM interface
│   ├── cl_cin.c      ← Cinematic video (RoQ decoder)
│   ├── cl_jpeg.c     ← JPEG screenshot writer
│   ├── cl_avi.c      ← AVI video capture
│   ├── cl_net_chan.c  ← Client-side netchan (decode, demo filtering)
│   ├── cl_curl.c/h   ← HTTP download support (libcurl integration)
│   ├── snd_main.c    ← Sound system dispatcher (picks DMA or HD backend)
│   ├── snd_dma.c     ← [URT] Base PCM DMA sound backend (dmaHD integration hooks)
│   ├── snd_dmahd.c   ← [URT] HRTF/spatial HD sound backend (new file, guarded by NO_DMAHD)
│   ├── snd_dmahd.h   ← [URT] dmaHD public API (new file)
│   ├── snd_local.h   ← [URT] ch_side_t struct, channel_t union, sfx_t weaponsound
│   ├── snd_mem.c     ← [URT] Sound data loading (dmaHD memory + intercept)
│   ├── snd_mix.c     ← Audio sample mixing
│   ├── snd_adpcm.c   ← ADPCM codec
│   ├── snd_codec.c   ← Sound codec dispatcher
│   ├── snd_codec_wav.c ← WAV file codec
│   ├── snd_wavelet.c ← Wavelet codec (Quake 3 .wav format variant)
│   └── client.h      ← [CUSTOM][URT] clientActive_t: snapshotMsec, auth cvars, demoprotocol
│
├── renderer/         ← OpenGL renderer (legacy/fallback)
│   └── ... (unchanged from stock Quake3e)
│
├── renderervk/       ← Vulkan renderer (preferred on modern hardware)
│   └── ... (unchanged from stock Quake3e)
│
├── renderercommon/   ← Shared renderer utilities (both GL and VK use these)
│   └── ... (unchanged from stock Quake3e)
│
├── botlib/           ← Bot AI library (AAS pathfinding + behavior) — unchanged
│
├── game/             ← Interface headers for QVM modules
│   └── bg_public.h   ← Shared physics types (trajectory_t, playerState_t, entityState_t)
│
├── cgame/
│   └── cg_public.h   ← Engine→cgame (CG_* syscall IDs, cgame entry points)
│
├── ui/
│   └── ui_public.h   ← [URT] Engine→UI module: UI_AUTHSERVER_PACKET added
│
├── unix/             ← Linux/Unix platform layer
├── win32/            ← [URT] Windows platform layer: win_snd.c WASAPI/dmaHD compat
└── sdl/              ← SDL2 platform layer (input, window, audio)
```

---

## Compile-Time Feature Flags

The re-impla branch adds several opt-in feature flags:

| Flag | Default | Enables |
|------|---------|---------|
| `USE_AUTH` | defined | Auth server integration (server + client hooks) |
| `USE_FTWGL` | defined | Client screenshot command, ticket userinfo |
| `USE_SERVER_DEMO` | defined | Server-side demo recording |
| `USE_URT_DEMO` | defined | URT demo format (`.urtdemo`, backward-play markers) |
| `NO_DMAHD` | 0 (not set) | When set: disables dmaHD, falls back to standard DMA sound |

---

## Subsystem Dependency Graph

```
                    ┌─────────────────────────────┐
                    │         Com_Frame()          │
                    │   (qcommon/common.c)         │
                    └──────────┬──────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
         SV_Frame()       CL_Frame()        Renderer
         (server)         (client)         (tr_init.c)
              │                │
         ┌────┴────┐      ┌────┴────┐
         │  Game   │      │ cgame   │
         │  QVM    │      │  QVM    │
         └────┬────┘      └────┬────┘
              │                │
         ┌────▼────┐      ┌────▼────┐
         │ Botlib  │      │ Sound   │
         └────┬────┘      └─────────┘
              │
         ┌────▼────────────────────┐
         │  qcommon (all use it)   │
         │  CM, VM, FS, NET, MSG   │
         └─────────────────────────┘
```

---

## Startup Sequence

```
main() → Com_Init(commandLine)
  Sys_Init()               ← OS-specific init (console, affinity)
  Com_InitZoneMemory()     ← heap zones
  Com_InitHunkMemory()     ← hunk allocator
  Cvar_Init()              ← cvar hash table
  Cmd_Init()               ← command table, built-in commands
  FS_InitFilesystem()      ← PAK/PK3 search paths (BASEGAME="q3ut4") [URT]
  NET_Init()               ← sockets
  Netchan_Init()           ← channel layer
  VM_Init()                ← QVM dispatch setup
  SV_Init()                ← server cvars, antilag init [CUSTOM]
                           ← auth cvars if USE_AUTH [URT]
  CL_Init()                ← client cvars, renderer, sound, UI
                           ← auth cvars, net monitor cvars if USE_FTWGL [URT]

Com_Frame() loop (called by Sys_ConsoleInputEvent / main loop)
  CL_Frame(msec)           ← input, time sync, cgame, renderer
  SV_Frame(msec)           ← game logic, snapshots, antilag
```

---

## Section Documents

| File | Contents |
|------|----------|
| `QCOMMON.md` | Memory, console, cvar, cmd, filesystem, math, huffman, hash |
| `NETWORKING.md` | NET sockets, Netchan protocol, MSG bit-packing, LZSS compression |
| `COLLISION.md` | CM map loading, trace engine, patches, area portals |
| `QVM.md` | QVM virtual machine, JIT compilers, bytecode, syscall contract |
| `SERVER.md` | Full server subsystem — all files, all functions, all custom changes |
| `CLIENT.md` | Full client subsystem — input, time sync, cgame, UI, console, demo |
| `SOUND.md` | DMA and HD sound backends, mixing, codecs |
| `RENDERER.md` | OpenGL and Vulkan renderers, scene graph, shaders |
| `BOTLIB.md` | AAS pathfinding, AI behaviors, elementary actions |
| `GAME_INTERFACES.md` | QVM syscall tables: g_public.h, cg_public.h, bg_public.h |
| `SV_GAMEHZ.md` | sv_gameHz reference: legacy compatibility, position correction |
| `SV_ANTIWARP.md` | sv_antiwarp deep-dive: modes 1/2, decay timeline, cvars, interaction with antilag |

---

## Key Global State

| Variable | Type | Owner | Purpose |
|---|---|---|---|
| `sv` | `server_t` | sv_main.c | Per-map server state: time, gameTime, entity arrays |
| `svs` | `serverStatic_t` | sv_main.c | Persistent server state: clients[], time, nextHeartbeat |
| `gvm` | `vm_t *` | sv_main.c | Game QVM handle |
| `cl` | `clientActive_t` | cl_main.c | Active client game state: serverTime, snaps, snapshotMsec |
| `clc` | `clientConnection_t` | cl_main.c | Connection state: netchan, serverAddress, demo recording |
| `cls` | `clientStatic_t` | cl_main.c | Persistent client state: realtime, rendererStarted |
| `cgvm` | `vm_t *` | cl_cgame.c | cgame QVM handle |
| `uivm` | `vm_t *` | cl_ui.c | UI QVM handle |
| `cm` | `clipMap_t` | cm_load.c | Loaded BSP collision model |
| `tr` | `trGlobals_t` | tr_init.c | Renderer state (either GL or VK version) |
| `botlib_export` | `botlib_export_t *` | sv_game.c | Botlib function table |

---

## Custom Changes Summary (This Fork)

All engine changes relative to stock Quake3e, organized by phase:

### Subtick / Timing (Phases 1–10)

| File | Change |
|------|--------|
| `server/sv_main.c` | Dual-rate frame loop (sv_fps outer + sv_gameHz inner); antilag recording; snapshot dispatch inside loop |
| `server/sv_init.c` | New cvars: sv_gameHz, sv_snapshotFps, sv_busyWait, sv_pmoveMsec, sv_extrapolate, sv_smoothClients, sv_bufferMs, sv_velSmooth |
| `server/sv_client.c` | Multi-step Pmove (sv_pmoveMsec), bot exclusion, sv_snapshotFps policy in SV_UserinfoChanged |
| `server/sv_snapshot.c` | Engine-side position extrapolation; TR_LINEAR smoothing; sv_bufferMs ring buffer; sv_velSmooth averaging |
| `server/sv_antilag.c` | **New file** — entire engine-side shadow antilag system |
| `server/sv_antilag.h` | **New file** — antilag public API header |
| `server/sv_ccmds.c` | SV_MapRestart_f: sync sv.gameTime before restart |
| `client/client.h` | Added `snapshotMsec` to `clientActive_t` |
| `client/cl_parse.c` | Snapshot interval EMA measurement; bootstrap from sv_snapshotFps configstring |
| `client/cl_cgame.c` | Proportional time-sync thresholds; serverTime clamp; QVM cvar intercept for snaps/cg_smoothClients |
| `client/cl_input.c` | Download pacing uses snapshotMsec instead of hardcoded 50ms |
| `qcommon/net_ip.c` | net_dropsim changed CVAR_TEMP → CVAR_CHEAT |

### URT / Urban Terror Integration (Phases 11–17)

| File | Change | Guard |
|------|--------|-------|
| `qcommon/q_shared.h` | `Q3_VERSION="FTWGL"`, `CLIENT_WINDOW_TITLE`, `CONSOLE_WINDOW_TITLE`, `BASEGAME="q3ut4"` | — |
| `qcommon/qcommon.h` | `URT_PROTOCOL_VERSION=70`, `URTDEMOEXT="urtdemo"`, rate/snaps/maxfps defaults | — |
| `ui/ui_public.h` | `UI_AUTHSERVER_PACKET` enum entry | `USE_AUTH` |
| `client/cl_main.c` | rate→90000, snaps→60+PROTECTED, cl_allowDownload→ROM, cl_dlURL→urt.li, URT demo recording, auth packet handler, ticket cvar | `USE_AUTH`, `USE_URT_DEMO`, `USE_FTWGL` |
| `qcommon/common.c` | com_maxfps PROTECTED, com_maxfpsUnfocused→0, com_blood, demo_protocols list | — |
| `server/server.h` | auth[] field in client_t, demo recording fields in client_t | `USE_AUTH`, `USE_SERVER_DEMO` |
| `server/sv_main.c` | sv_authServerIP/sv_auth_engine cvar pointers | `USE_AUTH` |
| `server/sv_init.c` | sv_authServerIP (CVAR_TEMP\|ROM), sv_auth_engine (CVAR_ROM) registration | `USE_AUTH` |
| `server/sv_client.c` | SV_Auth_DropClient (~55 lines, separate reason/message params), demo auto-stop on drop | `USE_AUTH`, `USE_SERVER_DEMO` |
| `server/sv_ccmds.c` | Server demo system (~350 lines): SVD_StartDemoFile, SVD_WriteDemoFile, SVD_StopDemoFile, startserverdemo/stopserverdemo commands; SV_ClientScreenshot_f | `USE_SERVER_DEMO`, `USE_FTWGL` |
| `server/sv_snapshot.c` | SVD_WriteDemoFile hook, demo_waiting/backoff logic | `USE_SERVER_DEMO` |
| `client/cl_cgame.c` | clientScreenshot server command handler | `USE_FTWGL` |
| `client/cl_main.c` | CL_Record_f URT header, CL_CheckDemoProtocol, CL_ReadDemoMessage backward-play skip | `USE_URT_DEMO` |
| `client/cl_scrn.c` | Net monitor widget (cl_netgraph 1/0), laggot announce, net session logging | — |
| `client/client.h` | auth/password/modversion in serverInfo_t, cl_auth*/cl_authc/authl cvar externs, demoprotocol field | `USE_AUTH`, `USE_URT_DEMO` |
| `client/snd_local.h` | ch_side_t struct, channel_t vol/offset unions, sfx_t weaponsound field | `NO_DMAHD` |
| `client/snd_dma.c` | dmaHD_Init call at end of S_Base_Init, removed static from shared symbols | `NO_DMAHD` |
| `client/snd_mem.c` | dmaHD memory allocation (2×1536), dmaHD_LoadSound intercept | `NO_DMAHD` |
| `client/snd_dmahd.c` | **New file** — full HRTF/spatial HD sound system (~1467 lines) | `NO_DMAHD` |
| `client/snd_dmahd.h` | **New file** — dmaHD public API | `NO_DMAHD` |
| `win32/win_snd.c` | dmaHD: WASAPI uses `GetMixFormat` for native bit-perfect output (no resampling); DirectSound fallback 48 kHz 16-bit PCM | `NO_DMAHD` |
| `Makefile` | NO_DMAHD=0 flag, sv_antilag.o + snd_dmahd.o build entries | — |

### Not Yet Implemented

| Feature | Phase | Notes |
|---------|-------|-------|
| Multiview | Phase 18 | `USE_MV` / `USE_MV_ZCMD` — protocol extensions, per-player snapshots, LZSS compression. Most complex feature, not yet implemented. |
