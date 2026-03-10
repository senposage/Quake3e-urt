# Upstream Reimplementation Worklog

Branch: `upstream-reimpl` based on `upstream/main`

## Status: FTWGL Features In Progress

### Completed Phases (Subtick)

**Phase 1: Timing Infrastructure (server.h, sv_init.c, sv_main.c, sv_ccmds.c)**
- `sv.gameTime` / `sv.gameTimeResidual` in `server_t`
- All new cvars registered: sv_gameHz, sv_snapshotFps, sv_busyWait, sv_pmoveMsec, sv_smoothClients, sv_antiwarp*, sv_bufferMs, sv_velSmooth, sv_extrapolate
- sv_fps default 60, CVAR_SERVERINFO
- Dual nested frame loop (sv_fps outer, sv_gameHz inner)
- Antiwarp injection (modes 1+2 with decay)
- sv_busyWait spin in SV_FrameMsec
- SV_TrackCvarChanges for all new cvars + modified flag handling
- SV_MapRestart_f syncs gameTime
- Snapshot dispatch inside sv_fps loop

**Phase 2: Multi-Step Pmove (sv_client.c)**
- SV_ClientThink: multi-step loop when delta > sv_pmoveMsec
- Bot exclusion (NA_BOT check)
- No-Pmove state detection (intermission/pause safety)
- awLastThinkTime tracking for antiwarp

**Phase 3: Visual Smoothing (sv_snapshot.c, server.h)**
- 32-slot per-client ring buffer (svSmoothHistory)
- SV_SmoothInit / SV_SmoothRecord / SV_SmoothRecordAll
- SV_SmoothGetPosition (time interpolation)
- SV_SmoothGetAverageVelocity (windowed velocity)
- Entity state fixup in SV_BuildCommonSnapshot:
  - Phase 1: position source (sv_bufferMs delayed or live)
  - Phase 2: trajectory type (TR_LINEAR for sv_smoothClients, idle detection)
  - Bot exclusion from prediction
- Ring buffer recording in frame loop (after GAME_RUN_FRAME, before snapshots)

**Phase 4: Rate Override (sv_init.c)**
- sv_minRate default 500000, CVAR_PROTECTED

**Phase 5: Antilag System (sv_antilag.c, sv_antilag.h — NEW FILES)**
- Shadow position recording at sv_fps Hz
- Ring buffer per-client (up to 256 slots)
- Fire time: sv.time - ping/2
- Rewind/restore cycle in G_TRACE/G_TRACECAPSULE intercept
- CONTENTS_PLAYERCLIP filter (movement vs weapon traces)
- Bot exclusion from recording and rewind
- Debug/rate monitoring cvars
- Hooked into sv_game.c, sv_main.c, sv_init.c

**Phase 6: Net Fixes (net_chan.c, common.c)**
- NET_FlushPacketQueue: force-flush when delay cvars drop to 0
- cl_packetdelay / sv_packetdelay: CVAR_CHEAT → CVAR_TEMP
- Kept upstream packet queue infrastructure intact (Netchan_Enqueue preserved)

**Phase 7: Client-Side Basics (client.h, cl_parse.c, cl_main.c)**
- cl.snapshotMsec (EMA of snapshot intervals, 75/25 blend, clamped [8,100])
- cl.frameInterpolation field added
- cl_adaptiveTiming cvar registered (archive, default 1)
- snapshotMsec EMA measurement in CL_ParseSnapshot (before cl.snap = newSnap)
- Pre-seeds cl.snapshotMsec from sv_fps in CL_ParseServerInfo

**Phase 8: Adaptive Client Timing (cl_cgame.c)**
- slowFrac fractional accumulator (1/4ms units, eliminates ±1ms oscillation)
- Scaled resetTime/fastAdjust thresholds based on cl.snapshotMsec
- Proportional mode (cl_adaptiveTiming=2) for faster convergence
- ServerTime cap (adaptive only) with release at 1000ms drift
- frameInterpolation computation from prev/curr snapshot interval
- Scaled extrapolation threshold (snapshotMsec/3, range 3-16)
- Extrap check uses 1/4ms units for slowFrac visibility
- CG_CVAR_SET filter: blocks QVM from overriding snaps/cg_smoothClients
- Timedemo uses snapshotMsec instead of hardcoded 50ms

**Phase 9: Net Monitor Widget (cl_scrn.c, client.h)**
- 10-row overlay widget (cl_netgraph 1) with color-coded thresholds:
  - Snap rate/interval, ping with min/max, frame interpolation, frame time,
    server time delta, drops/extrap/cap, bandwidth, snap jitter, sequence, adjustments
- Configurable position (cl_netgraph_x/y) and scale (cl_netgraph_scale)
- 1-second rate aggregation with per-second display snapshots
- Public hooks: SCR_NetMonitorAdd{Incoming,Outgoing,Frametime,SnapInterval,
  CapHit,Extrap,Choke,TimeDelta,Ping,FastAdjust,ResetAdjust,SlowAdjust}
- Net debug session logging (cl_netlog 0/1/2):
  - FAST/RESET/SNAP LATE/PING JITTER events
  - Periodic per-second stats at level 2
  - netgraph_dump command for full on-demand snapshot
- Laggot announce (cl_laggotannounce 1): 30-second ring buffer, auto-sends
  "[NET] ..." via say on packet loss/fast resets/snap jitter/extrap/low fps/choke
- Hooks wired into: cl_cgame.c, cl_parse.c, cl_input.c, cl_main.c

**Phase 10: Client Input Tweaks (cl_input.c)**
- Adaptive download throttle: snapshotMsec instead of hardcoded 50ms
- SCR_NetMonitorAddOutgoing hook after packet transmit

### Completed FTWGL Phases

**Phase 11: Branding & Protocol (q_shared.h, qcommon.h, ui_public.h)**
- Q3_VERSION = "FTWGL"
- CLIENT_WINDOW_TITLE = "FTWGL: Urban Terror"
- CONSOLE_WINDOW_TITLE = "FTWGL: UrT"
- BASEGAME = "q3ut4"
- AUTHORIZE_SERVER_NAME = "authorize.urbanterror.info"
- URT_PROTOCOL_VERSION = 70
- URTDEMOEXT = "urtdemo"
- UI_AUTHSERVER_PACKET enum (USE_AUTH guard)

**Phase 12: Cvar Defaults (cl_main.c, common.c, qcommon.h)**
- rate: "25000" → "90000"
- snaps: "40" → "60" + CVAR_PROTECTED
- cl_timeNudge: CVAR_TEMP → CVAR_PROTECTED
- cl_allowDownload: CVAR_ARCHIVE_ND → CVAR_ROM|CVAR_PROTECTED
- cl_mapAutoDownload: "0" → "1"
- cl_dlURL: "http://ws.q3df.org/maps/download/%1" → "http://urt.li/q3ut4"
- com_maxfps: add CVAR_PROTECTED
- com_maxfpsUnfocused: "60" → "0"
- com_blood: uncommented + variable declared + extern added

**Phase 13: Auth System (USE_AUTH)**
- server.h: `auth[MAX_NAME_LENGTH]` field in client_t, extern cvars for sv_authServerIP/sv_auth_engine, SV_Auth_DropClient declaration
- sv_main.c: cvar_t pointer declarations for sv_authServerIP, sv_auth_engine
- sv_init.c: sv_authServerIP (CVAR_TEMP|CVAR_ROM), sv_auth_engine (CVAR_ROM) registration
- sv_client.c: SV_Auth_DropClient implementation (~55 lines, separate reason/message params)
- client.h: serverInfo_t auth/password/modversion fields, extern cvars for cl_auth_engine/cl_auth/authc/authl
- cl_main.c: cvar declarations, AUTH:CL packet handler (forwards to UI_AUTHSERVER_PACKET), cvar registration
- ui_public.h: UI_AUTHSERVER_PACKET already present (added in Phase 11)
- All guarded with #ifdef USE_AUTH

**Phase 14: ClientScreenshot (USE_FTWGL)**
- sv_ccmds.c: SV_ClientScreenshot_f function + Cmd_AddCommand registration
- cl_cgame.c: clientScreenshot server command handler (screenshot silent with optional filename)
- cl_main.c: "ticket" userinfo cvar (CVAR_USERINFO|CVAR_TEMP)
- All guarded with #ifdef USE_FTWGL

**Phase 15: Server Demo (USE_SERVER_DEMO)**
- server.h: demo fields in client_t (recording, file, waiting, backoff, deltas), extern cvars, SVD_WriteDemoFile declaration
- sv_main.c: cvar_t declarations for sv_demonotice, sv_demofolder
- sv_init.c: cvar registration, demo cleanup on map load and shutdown
- sv_ccmds.c: Full demo system (~350 lines):
  - SVD_StartDemoFile: writes gamestate header + configstrings + baselines
  - SVD_WriteDemoFile: per-snapshot recording with svc_EOF append
  - SVD_StopDemoFile: writes trailer markers, closes file
  - SVD_CleanPlayerName: sanitizes names for filesystem
  - SV_StartRecordOne/All, SV_StopRecordOne/All
  - SV_StartServerDemo_f/SV_StopServerDemo_f command handlers
  - Command registration: startserverdemo, stopserverdemo
- sv_snapshot.c: forced full frames via exponential backoff, demo_waiting transition, SVD_WriteDemoFile call in SV_SendMessageToClient
- sv_client.c: demo field init in SV_DirectConnect, auto-stop in SV_DropClient and SV_Auth_DropClient
- USE_URT_DEMO conditional: URT protocol header, backward-play size markers, .urtdemo extension
- All guarded with #ifdef USE_SERVER_DEMO

**Phase 16: UT Demo Format (USE_URT_DEMO)**
- CL_Record_f: URT header write (modversion + protocol + 2 reserved ints), skip ext stripping
- CL_CompleteRecordName: .urtdemo extension for tab completion
- CL_CheckDemoProtocol: new function, determines protocol from demo file extension
- CL_ReadDemoMessage: backward-play length marker skip + verification, zero-size URT end check
- CL_WalkDemoExt: .urtdemo extension for URT_PROTOCOL_VERSION entries
- CL_DemoNameCallback_f: .urtdemo file recognition in demo listing
- CL_CompleteDemoName: .urtdemo tab completion alongside .dm_??
- CL_PlayDemo_f: .urtdemo extension detect, URT header read + protocol verify, demoprotocol-based compat
- CL_StopRecord_f: .urtdemo extension (already done in Phase 15)
- CL_WriteDemoMessage/CL_WriteGamestate: backward-play markers (already done in Phase 15)
- common.c: demo_protocols includes URT_PROTOCOL_VERSION (already done in Phase 15)
- client.h: demoprotocol field (already done in Phase 15)
- All guarded with #ifdef USE_URT_DEMO

**Phase 17: dmaHD Sound (NO_DMAHD)**
- snd_dmahd.c (NEW, ~1467 lines): Full HRTF/spatial sound system, copied from master
- snd_dmahd.h (NEW): Public interface (dmaHD_LoadSound, dmaHD_Enabled, dmaHD_Init)
- snd_local.h: ch_side_t struct (vol/offset/bass/reverb per side), channel_t unions (leftvol/rightvol + ch_side_t), sodrot vec3_t, sfx_t weaponsound field
- snd_dma.c: #include snd_dmahd.h, removed static from s_soundStarted/s_soundMuted/listener_number/loopSounds/s_knownSfx/s_numSfx/s_mixahead + S_GetSoundtime/S_ScanChannelStarts/S_UpdateBackgroundTrack, s_khz default 44 when dmaHD enabled, dmaHD_Init call at end of S_Base_Init
- snd_mem.c: #include snd_dmahd.h, dmaHD memory allocation (2*1536 fixed), dmaHD_LoadSound intercept in S_LoadSound
- win32/win_snd.c: dmaHD_Enabled declaration, WASAPI disabled when dmaHD active, force DirectSound driver, force 44100Hz/stereo/16-bit
- Makefile: NO_DMAHD=0 flag, -DNO_DMAHD conditional, snd_dmahd.o in client build
- All guarded with #ifndef NO_DMAHD

### Pending FTWGL Phases

**Phase 18: Multiview (USE_MV, USE_MV_ZCMD)**
- NOT found in current codebase — needs fresh implementation
- Most complex feature, protocol extensions across many files
- svc_multiview/svc_zcmd opcodes
- Per-player snapshot data, LZSS compression
- mvjoin/mvleave/mvfollow commands

### Files Modified (24 + 4 new)
Server: server.h, sv_main.c, sv_init.c, sv_client.c, sv_snapshot.c, sv_ccmds.c, sv_game.c, sv_antilag.c (NEW), sv_antilag.h (NEW)
Client: client.h, cl_cgame.c, cl_parse.c, cl_input.c, cl_main.c, cl_scrn.c, snd_local.h, snd_dma.c, snd_mem.c, snd_dmahd.c (NEW), snd_dmahd.h (NEW)
Platform: win32/win_snd.c
Common: common.c, net_chan.c, q_shared.h, qcommon.h, ui_public.h
Build: Makefile (upstream base + sv_antilag.o + snd_dmahd.o entries)

### Not Implemented (Low Priority / Cosmetic)
- Cmd_SetDescription calls (doesn't exist in upstream)
- SCR_FontWidth/DrawFontChar/DrawFontText (cls.fontFont doesn't exist in upstream)
- cl_maxpackets default change (125→60) — cosmetic preference
- CL_WritePacket repeat parameter removal — keep upstream signature
- net_ip.c cosmetic changes (Sys_StringToSockaddr simplified signature)

### sv_fps > 20 Warning — NOT FOUND
- User reports a warning on server startup when sv_fps > 20
- Searched all server code, client code, common code — not found in engine
- May come from QVM game code at runtime (UT QVM)

### Important Notes
- Keep close to upstream — don't refactor existing upstream infrastructure
- Dedicated server: guard client-only code with #ifndef DEDICATED
- Use sv.maxclients (not sv_maxclients->integer) per upstream convention
- SV_ClientEnterWorld takes no usercmd param in upstream
- Need build tools to test compilation
- Auth system is primarily QVM-side; engine provides hooks only
- ClientScreenshot and Multiview NOT found in codebase — need fresh impl
- Server demo reference: ET:Legacy sv_demo.c (2403 lines)

### Research Notes (from agent exploration)
- Auth encryption: custom obfuscation in auth_shared.c (QVM-side)
- Auth packets: AUTH:CLIENT (accept/reject), AUTH:MSG (say/drop)
- Auth server sends to: SV_Authserver_Send_Packet("CLIENT"/"HEARTBEAT"/"PLAYERS"/"SHUTDOWN"/"BAN")
- Demo ops enum: demo_endDemo, demo_EOF, demo_endFrame, demo_configString, etc.
- Demo uses delta compression with previous frame storage
- Demo recording hooks at: client commands, server commands, configstrings, userinfo, frame end
