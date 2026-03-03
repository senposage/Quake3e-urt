# Merge Memory - Upstream Sync Progress

## Project Overview
- **Repo**: senposage/Quake3e-subtick (fork of ec-/Quake3e)
- **Purpose**: Quake3e engine with subtick networking, antilag, antiwarp modifications
- **Upstream**: ec-/Quake3e (target sync tag: 46add7d)
- **Build system**: GNU Make (Makefile), also has CMakeLists.txt and MSVC solutions

## What This Fork Adds (8 patch files in /patches/)
1. `0001-subtick-frame-loop.patch` - Modified server frame loop (sv_main.c)
2. `0002-subtick-cvars.patch` - Subtick cvars (sv_init.c)
3. `0003-subtick-pmove.patch` - Player movement changes (sv_client.c)
4. `0004-subtick-extrapolation.patch` - Extrapolation system (sv_snapshot.c) - largest patch
5. `0005-subtick-maprestart.patch` - Map restart handling (sv_ccmds.c)
6. `0006-subtick-antilag.patch` - Antilag system (sv_antilag.h - new file)
7. `0007-subtick-client-jitter.patch` - Client jitter handling (client/client.h)
8. `0008-subtick-net-dropsim.patch` - Network drop simulation (qcommon/net_ip.c)

## Previous Merge Work (PR #38 - MERGED)
- Title: "Sync with upstream ec-/Quake3e latest tag (46add7d)"
- Replaced ~390 files with upstream content
- Re-applied 25 fork patches
- 443 files changed, 57276 additions, 63289 deletions
- **Known unresolved issues from PR #38 description:**
  - ~93 corrupted files (got "404: Not Found" from raw GitHub downloads)
  - Build errors (linker error, snd_dmahd.c corruption)
  - API mismatches possibly remaining

## Current State (PR #39 - THIS PR)
- Branch: copilot/continue-merge-process
- Base: latest tag commit d22f38b
- **Build status: SUCCEEDS** (client + dedicated server build OK on Linux x86_64)
- Only deprecation warnings from curl API (pre-existing, not our issue)
- No files found containing "404: Not Found" corruption
- No suspiciously small/empty source files found

## Build Configuration (from Makefile)
- USE_SDL=0, USE_VULKAN=1, USE_OPENGL=0, USE_VULKAN_API=1
- USE_AUTH=1, USE_URT_DEMO=1, NO_DMAHD=0, USE_FTWGL=1
- USE_SERVER_DEMO=1, USE_RENDERER_DLOPEN=0
- Output names: ftwgl-{gitrev}.x64 (client), quake3e.ded-{gitrev}.x64 (server)

## What Still Needs Investigation
- [ ] Compare fork source files against upstream ec-/Quake3e at tag 46add7d to find remaining differences
- [ ] Verify all 8 fork patches are properly applied in current code
- [ ] Check if any upstream files are still missing or wrong
- [ ] Check dedicated server build
- [ ] Identify what "continue the merge" specifically means - more upstream changes to integrate?

## Key Directories
- `code/server/` - Server-side code (most fork changes are here)
- `code/client/` - Client-side code
- `code/qcommon/` - Common/shared code
- `code/game/` - Game logic (QVM)
- `code/cgame/` - Client game logic
- `code/renderervk/` - Vulkan renderer
- `code/renderer/` - OpenGL renderer
- `code/renderer2/` - OpenGL2 renderer
- `code/libjpeg/` - Bundled libjpeg
- `code/unix/` - Unix/Linux platform code
- `code/win32/` - Windows platform code
