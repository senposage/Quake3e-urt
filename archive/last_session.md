# Last Session Briefing
**Updated:** 2026-03-02
**Branch:** `master`

> This file is rewritten at the end of every session. Read it first.

---

## Status

All engine work complete and stable. Documentation restructured this session.

---

## This Session — What Was Done

### 1. sv_antiwarpExtra cvar (NEW)
Added configurable extrapolation window for `sv_antiwarp 2`. Previously hardcoded at `2 × tolerance`.

- **sv_antiwarpExtra** (default 0 = auto = awTol): how long full movement inputs are kept before decay starts.
- At sv_fps 60 with defaults: auto = 16ms. Total extrapolation = tolerance(16ms) + extra(16ms) = 32ms.
- User tested `sv_antiwarpExtra 50` + `sv_antiwarpDecay 225` with `net_dropsim 50` — good results for extreme loss.
- Files: `server.h`, `sv_init.c`, `sv_main.c`

### 2. sv_smoothClients default changed to 1
Was `0`. Now defaults to TR_LINEAR trajectory. Updated in `sv_init.c` and all docs.

### 3. Documentation Restructure
**Problem:** sv_gameHz 0 is the recommended setup but docs led with sv_gameHz 20 as if it were the default. Combination matrix was confusing.

**Changes:**
- **CVARS.md**: Reorganized into sections — Recommended Setup, Core, Antiwarp, Antilag, Smoothing, Legacy. Each smoothing cvar now notes its behavior at sv_gameHz 0. Combination table simplified.
- **SV_GAMEHZ.md** (NEW): Created `docs/project/SV_GAMEHZ.md` with full sv_gameHz legacy reference — version compatibility matrix, position correction pipeline, feature relevance matrix, configuration profiles.
- **CLAUDE.md**: Added recommended setup block at top of engine section. Simplified cvar descriptions.
- **SERVER.md**: Added antiwarp cvars to custom cvar table. Updated call graph comment.
- **FLOWCHART.md**: Section 2 (Clock System) now leads with sv_gameHz 0, sv_gameHz 20 moved to labeled legacy subsection. Both sections clarify sv_fps is safe at any value when sv_antiwarp is on OR g_antiwarp is off. Antiwarp box expanded with phase detail. Section 4 and 12 updated with sv_gameHz 0 context.
- **sv_init.c**: Updated cvar descriptions — sv_gameHz leads with "0 = recommended", sv_extrapolate and sv_bufferMs note their no-op behavior at sv_gameHz 0.

### 4. cl_packetdelay recovery fix (from earlier in session)
`NET_FlushPacketQueue` in `net_chan.c` — force-flush all queued packets when both delay cvars drop to 0. Previously, packets queued at old delay kept releasing on schedule, inflating cl.snapshotMsec EMA and preventing recovery without reconnect.

---

## Key Files Modified This Session

| File | Change |
|------|--------|
| `code/server/server.h` | Added `sv_antiwarpExtra` extern |
| `code/server/sv_init.c` | Registered `sv_antiwarpExtra`, changed `sv_smoothClients` default to 1, updated cvar descriptions |
| `code/server/sv_main.c` | Added `sv_antiwarpExtra` cvar definition + usage in antiwarp loop, modified flag clearing |
| `code/qcommon/net_chan.c` | Fixed `NET_FlushPacketQueue` force-flush on delay=0 |
| `code/qcommon/net_ip.c` | `net_dropsim` CVAR_CHEAT → CVAR_TEMP |
| `code/qcommon/common.c` | `cl_packetdelay` CVAR_CHEAT → CVAR_TEMP |
| `archive/CVARS.md` | Full restructure |
| `archive/CLAUDE.md` | Added recommended setup, updated cvar table |
| `docs/project/SV_GAMEHZ.md` | NEW — legacy sv_gameHz reference |
| `docs/project/SERVER.md` | Added antiwarp cvars, updated call graph |
| `docs/project/FLOWCHART.md` | Updated clock system, antiwarp detail, sv_gameHz 0 context |

---

## No Open Questions

All planned work from this session is complete. No pending tasks.
