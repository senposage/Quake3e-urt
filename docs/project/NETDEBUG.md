# Sporadic Vanilla-Server Disconnect — Debug Guide

> **Status:** OOB-disconnect guard added; bad-faith server protections in place; two real sessions captured and analysed.
> **Confirmed:** The vanilla URT4 server stops processing client usercmds mid-game and then floods ~125 OOB disconnect packets/second for 15–23 seconds while continuing to send snapshots. Root server-side trigger is still unknown.
> **Relates to:** PR #68 (serverTime safety cap), PR #109/110 (OOB guard fix + cmdTime logging), ongoing sporadics on the single populated vanilla URT4 server.

---

## How to capture a session log

```
set cl_netlog 1      // in console BEFORE connecting
connect <server>     // connect normally and play until disconnect
```

The log file is written to `<fs_basepath>/quake3/netdebug_YYYYMMDD_HHMMSS.log`.
Set `cl_netlog 2` for a heavier trace that includes per-snapshot state, all OOB
packets from the server, every userinfo send, and every console command the QVM
injects.

A `[DISCONNECT TRACE]` line is **always** printed to the console regardless of
`cl_netlog`, so even without a log file you will see the full state dump at
the moment of disconnect.

---

## Key cvars added for this investigation

| Cvar | Default | Description |
|------|---------|-------------|
| `cl_noOOBDisconnect` | `1` | When **1**, OOB `disconnect` packets from the server are logged but **ignored**. The normal `cl_timeout` mechanism handles cleanup if the server genuinely stops sending. Set to **0** to restore vanilla behaviour (immediate disconnect on OOB). |

---

## Log event reference

| Tag | Level | Meaning |
|-----|-------|---------|
| `CONNECT` | 1 | Gamestate received: sv_fps, sv_snapshotFps, seeded snapshotMsec, vanilla/forbids flags |
| `SVCMD` | 1 | Every new server command — seq numbers + first 80 chars of text |
| `SNAP` | 2 | Per-snapshot state: serverTime, ping, cmdTime (ps.commandTime), msgSeq, cmdSeq, relSeq, relAck |
| `DROP` | 1 | Netchan gap: server UDP packets that never arrived (maps to lagometer black bars) |
| `SNAP:DELTA_INVALID` | 1 | Delta-base snapshot was already invalid — should never happen |
| `SNAP:DELTA_OLD` | 1 | Delta-base too old to reconstruct — usually follows a DROP |
| `SNAP:DELTA_ENTITIES_OLD` | 1 | Entity ring buffer too old — rare |
| `WARN:RELOVERFLOW` | 1 | Client reliable-command buffer overflow recovery (relSeq − relAck wrapped) |
| `WARN:RELWND` | 1 | Reliable send window is ≥ 50 % full before a userinfo send — upstream packet loss suspected |
| `WARN:CMDCYCLED` | 1 | Server command cycled out of ring before cgame read it — precedes ERR_DROP |
| `TIMEOUT` | 1 | Timeout counter increment (logs counts 1-6; final fires DISCONNECT) |
| `CAP_RELEASE` | 1 | Safety cap released after ≥ 2 s server silence (rate-limited: 1/500 ms) |
| `RESET` | 1 | CL_AdjustTimeDelta hard-reset (deltaDelta > resetTime) |
| `FAST` | 1 | CL_AdjustTimeDelta fast-adjust (deltaDelta > fastAdjust) |
| `SNAP LATE` | 1 | Snapshot arrived later than 1.5× expected interval |
| `PING JITTER` | 1 | Measured ping changed by more than threshold in one snap |
| `QVM:SET_INTERCEPTED` | 1 | cgame QVM called trap_Cvar_Set on a cvar in the engine-side block list (`snaps`, `cg_smoothClients`, `net_qport`, `net_dropsim`, `cl_packetdelay`, `cl_packetdup`, `cl_timeout`, `in_mouse`, `cl_noOOBDisconnect`, `rate=0`) — silently ignored |
| `QVM:SET_BLOCKED` | 1 | cgame QVM called trap_Cvar_Set on a CVAR_PROTECTED cvar — blocked by engine |
| `QVM:CONSOLECMD` | 2 | Every console command injected by the cgame QVM via CG_SENDCONSOLECOMMAND |
| `QVM:CONSOLECMD_BLOCKED` | 1 | QVM tried to inject a dangerous command (`disconnect`, `quit`, `exit`, `exec`) — **blocked** |
| `SVCMD:DISCONNECT` | 1 | Server sent an in-band reliable `disconnect` command (different from OOB — carries a reason string) |
| `OOB:PACKET` | 2 | Every connectionless (OOB) packet received from the connected server |
| `OOB:DISCONNECT` | 1 | OOB `disconnect` packet arrived from server — fields show whether it was honored or ignored |
| `OOB:UNKNOWN` | 1 | Unknown connectionless packet from the connected server |
| `PKT:WRONG_SOURCE` | 1 | Sequenced packet received from a different address than the server |
| `USERINFO:SEND` | 2 | Userinfo reliable command sent to server mid-session — shows name, authc, authl, snaps |
| `DISCONNECT` | 1 | Full state dump immediately before any disconnect/ERR_DROP (also always console) |

---

## What each field in `DISCONNECT` means

```
[HH:MM:SS] DISCONNECT  reason="<string>"
  snapT=<N>  svrT=<N>  dT=<N>  ping=<N>
  cmdSeq=<N>  relSeq=<N>  relAck=<N>  relWnd=<N>
  snapMs=<N>  vanilla=<0|1>  forbidsAdaptive=<0|1>
  timeout=<N>  silenceMs=<N>  capHits=<N>  oobIgnored=<N>
```

| Field | What to look for |
|-------|-----------------|
| `reason` | Server-supplied string — should identify why the server dropped us |
| `ping=999` | `ps.commandTime` frozen — server has stopped processing our usercmds (see §2h). **This is a symptom of a server-side kick decision, not a cause.** It appears in the log one snap before the OOB disconnect flood begins. |
| `ping` high (>400) | High-ping kick if `sv_maxPing` is set on the server |
| `silenceMs` large | Server stopped sending packets before the disconnect — network outage or server-side kick |
| `silenceMs` small | Server sent the disconnect recently — deliberate server action (possibly spurious) |
| `capHits` high | Safety cap was firing often — `cl.serverTime` was being pushed ahead repeatedly |
| `relWnd` | `relSeq − relAck` — fill level of our reliable send window (max 64). High value means the server is not acking our commands fast enough |
| `relSeq - relAck` large | Server not acknowledging our reliable commands — upstream packet loss |
| `timeout > 0` | We were already timing out before the disconnect arrived |
| `vanilla=1` | Vanilla server path confirmed |
| `oobIgnored` | Number of OOB disconnect packets silently ignored this session (see `cl_noOOBDisconnect`) |

---

## What `OOB:DISCONNECT` means vs `DISCONNECT`

`OOB:DISCONNECT` fires when an **out-of-band** `disconnect` packet arrives.
`DISCONNECT` fires when the engine actually tears down the connection.

With `cl_noOOBDisconnect 1` (default) these are **two separate events**:
- `OOB:DISCONNECT honored=0` — packet arrived, was logged, was **ignored**
- `clc.lastPacketTime` is updated (the OOB packet counts as a sign of server
  liveness), so the `cl_timeout` clock runs from the last packet of *any* kind
- No `DISCONNECT` line follows unless the server stops sending *all* packets
  (including OOB) and `cl_timeout` fires

With `cl_noOOBDisconnect 0` they fire in sequence:
- `OOB:DISCONNECT honored=1` — packet arrived
- `DISCONNECT reason="OOB disconnect from server"` — engine disconnected

An `SVCMD:DISCONNECT` is a **different path** — the server sent a reliable
in-band `disconnect` command. This carries an explicit reason string and is
**always honoured** regardless of `cl_noOOBDisconnect`.

---

## Suspect list and current working theories

### 1. Black-bars-then-disconnect (netgraph correlated)

**Sequence:**  
`DROP` events → `SNAP:DELTA_OLD` → extrapolation → `FAST` or `RESET` adjustment → disconnect.

**Mechanism being logged:** Each `DROP` maps to one UDP packet loss. If the drop
knocks out a snapshot, `CL_ParseSnapshot` invalidates the delta chain
(`SNAP:DELTA_OLD`) which forces full-snapshot retransmit. During this gap the
client extrapolates; the lagometer shows black bars. When the burst resumes,
`CL_AdjustTimeDelta` may fire a `FAST` adjust which momentarily pushes
`cl.serverTime` ahead, causing the safety cap to fire repeatedly.

**What to look for in logs:**  
`DROP` count ≥ 1 → `SNAP:DELTA_OLD` within the same second → `FAST` or
`RESET` within 1-2 snapshots → `DISCONNECT`.

---

### 2. Clean-connection disconnect (no netgraph warning)

**This is the harder case.** No DROP, no SNAP:DELTA_OLD, no TIMEOUT — yet the
server sends a disconnect. Possible causes:

#### 2a. Server-side reliable-command overflow  
The vanilla server queues print/cs-update reliable commands for our client.
If the server sends a burst of commands (e.g. many player-join events) and
our client's acks do not reach the server fast enough (even 1-2% packet loss
upstream), the server's `cl->reliableSequence − cl->reliableAcknowledge`
reaches `MAX_RELIABLE_COMMANDS` (128) and the server calls
`SV_DropClient("reliable overflow")`.

**What to look for:**  
`reason="reliable overflow"` in `DISCONNECT`. High `capHits` or elevated
`silenceMs` at time of disconnect. `WARN:RELOVERFLOW` in our own buffer
(though that is client→server direction, not server→client).

#### 2b. Frametime hitch → CL_AdjustTimeDelta FAST/RESET → cap fires  
A system-level hitch (OS scheduler, GC, disk, v-sync stall) causes
`cls.realtime` to jump by 100–400 ms in a single frame. For a remote client
`Com_ModifyMsec` allows up to **5 000 ms** without clamping (line ~4195 in
`common.c`). The jump in `cls.realtime` propagates into `cl.serverTime`
immediately, the safety cap fires, and on the next snapshot `CL_AdjustTimeDelta`
fires a `FAST` adjust. This is **benign by itself** — the cap prevents negative
ping — but if `sv_maxPing` is tight on the server, the transient high ping
during the FAST-adjust phase could trigger a ping-kick.

**What to look for:**  
A `FAST` or `RESET` log line with `deltaDelta` ≈ 100–500 followed within
1-3 snapshots by `DISCONNECT` with `ping` close to `sv_maxPing`. Also
look for `capHits` spiking in the second before disconnect.

The **5 000 ms remote-client clamp** in `Com_ModifyMsec` (`common.c:4195`)
is worth revisiting: upstream Quake3e uses the same value but the URT vanilla
server's QVM ping-kick threshold may be lower than typical servers. Consider
whether adding a tighter client-side hitch guard (e.g. 200 ms) when `cl_maxfps`
is set would reduce transient ping spikes — **but only change this once we
have log evidence that hitches are the cause**.

> **Note on ping=0 in logs:** `ping=0` (same-frame command match) can appear when
> a snapshot arrives late while the safety cap is active — that specific case is a
> calculation artefact.  `ping=999` is **not** an artefact: it means
> `ps.commandTime` has frozen and is a reliable indicator that the server has
> stopped processing our commands (see §2h).  Check the `cmdTime` field in
> adjacent SNAP lines — a frozen value confirms the freeze.  Then look for
> `OOB:DISCONNECT` or `SVCMD:DISCONNECT` to identify the kick mechanism.

#### 2c. QVM cvar intercept side-effects  
Our engine silently ignores `trap_Cvar_Set("snaps", …)` calls from the cgame
QVM. The URT4 vanilla cgame.qvm almost certainly sets `snaps` once on map load
to match `sv_fps`. With our fork, `snaps` stays at 60 when the QVM reads it
back. This is harmless for a 20 Hz server (server clamps delivery to `sv_fps`
anyway), but if the QVM makes any decision based on the current value of
`snaps` — for example computing expected snapshot intervals for a timeout check
— the wrong value could matter.

**What to look for:**  
`QVM:SET_INTERCEPTED` lines in the log. Note how many times and with what
value the QVM tries to set `snaps`. If the QVM keeps retrying repeatedly
(dozens of times per second) it may indicate a logic loop triggered by the
unexpected value.

#### 2d. CVAR_PROTECTED cvar blocked by QVM  
`cl_timeNudge` is `CVAR_PROTECTED` in our fork (upstream: `CVAR_TEMP`). If
the vanilla URT QVM attempts to set `cl_timeNudge` (which some mods do to
tune timing), `Cvar_SetSafe` will block it and print a yellow console warning.
The QVM receives no error indication and simply continues with the old value.
Depending on what the QVM expects `cl_timeNudge` to be, this could lead to
incorrect timing behaviour at the QVM level.

**What to look for:**  
`QVM:SET_BLOCKED` lines. Pay attention to which cvar is blocked and what
value the QVM wanted.

#### 2e. AUTH userinfo fields on vanilla server  
`authc` and `authl` are `CVAR_USERINFO` in our fork. Every client sends
`\authc\0\authl\` in the userinfo string to the server. The vanilla URT4
server QVM receives this and may react to `authc=0` as "unauthenticated user"
if it has any auth-gating logic, kicking with a reason like "not authenticated"
or "auth required".

**What to look for:**  
`reason` string in `DISCONNECT` containing "auth". Also check whether the
disconnect happens consistently on connect (within the first few seconds)
which would point to a connect-time auth check rather than a gameplay event.
Cross-reference with `USERINFO:SEND` entries at level 2 to see if a mid-session
userinfo update preceded the disconnect.

#### 2f. Spurious OOB disconnect (undocumented server behaviour)  
Some vanilla URT servers send an OOB `disconnect` packet as an undocumented
server-side action with no matching in-band goodbye and no clear pattern.
The client previously disconnected immediately. With `cl_noOOBDisconnect 1`
(now the default) the packet is logged as `OOB:DISCONNECT honored=0` and
ignored; the connection continues until `cl_timeout` fires if the server
genuinely stops sending.

**What to look for:**  
`OOB:DISCONNECT honored=0` entries. If `oobIgnored` in the final `DISCONNECT`
is > 0, at least one of these spurious packets was received this session.
Note the `silenceMs` in the `OOB:DISCONNECT` entry — if small (< 200 ms) the
server was still actively sending snapshots immediately before the OOB packet,
which strongly suggests spurious / undocumented behaviour rather than a real
kick.

#### 2g. Bad-faith server injecting commands via cgame  
A malicious or buggy server could send reliable commands that cause the cgame
QVM to call `trap_SendConsoleCommand` with `disconnect`, `quit`, or `exec`.
The engine now **blocks** these (logged as `QVM:CONSOLECMD_BLOCKED`).
Similarly, `trap_Cvar_Set` attempts to zero `rate` or change `net_qport` are
intercepted and logged.

**What to look for:**  
`QVM:CONSOLECMD_BLOCKED` entries. Any such entry means the server attempted
to force a client action. Cross-reference with surrounding `SVCMD` entries to
identify which server command triggered the QVM call.

#### 2h. Server stopped processing our usercmds before kicking (confirmed in captured sessions)

**Confirmed sequence from both netdebug logs:**

1. `ps.commandTime` (`cmdTime` in SNAP line) freezes — server stops stamping
   our usercmds into the snapshot.
2. As we send new outgoing packets their `p_serverTime` advances past the
   frozen `ps.commandTime`. Once all 32 `outPackets` slots are newer, the
   ping calculation finds no match → `ping=999` in the log.
3. One snap later (~50 ms), the first OOB disconnect packet arrives
   (`silenceMs=56–64 ms`). The OOB flood runs at ~125 packets/second.
4. Game data stops. `cl_timeout` would eventually fire but the server accepts
   a new connection first.

The jump from normal ping (31 ms) to `ping=999` in a **single snap** is the
observable fingerprint: it cannot be jitter or a cap artefact — only a frozen
or reset `ps.commandTime` can exhaust all 32 history slots in one frame.

**What we do not yet know:** what server-side condition causes it to stop
processing our commands at that moment. Candidates — pending QVM source
review:
- `sv_maxPing` kick triggered by a transient period where `usercmd.serverTime`
  ran ahead of `sv.time` (the earlier `ping=263→247→0` blip in log 2 is a
  candidate precursor)
- Per-client auth or reliable-overflow check firing mid-session
- Something specific to the URT4 game QVM's `ClientThink`

**What to look for:**  
In a level-2 log: two or more consecutive SNAP lines with the same `cmdTime`
value, immediately followed by the first `OOB:DISCONNECT`. The `cmdTime` freeze
precedes `ping=999` by definition; find the freeze to find the trigger moment.

---

## Priority reading order for the first log

1. Jump straight to the last `DISCONNECT` block.
2. Read `reason` — if it contains any server-supplied text, that is the primary clue.
3. Check `oobIgnored` — if > 0, look for preceding `OOB:DISCONNECT` entries.
4. Check `silenceMs` — if > 500 ms, there was a network gap before the disconnect.
5. At level 2: scan SNAP lines backwards for a **frozen `cmdTime`** — two or more
   consecutive snaps with the same value. That is the server-side kick moment.
   `ping=999` appears in the same snap or the one immediately after.
6. Check `ping` — `ping=999` confirms `ps.commandTime` froze (§2h); `ping=0` on an
   isolated snap can be a calculation artefact, ignore it.
7. Scroll back ~5 seconds to find the last `DROP`, `SNAP:DELTA_*`, or `OOB:PACKET` burst.
8. Look for any `QVM:SET_INTERCEPTED`, `QVM:SET_BLOCKED`, or `QVM:CONSOLECMD_BLOCKED` events.
9. Look for `FAST`, `RESET`, or `SVCMD:DISCONNECT` events in the window before disconnect.
10. At level 2: check `USERINFO:SEND` to see if a mid-session userinfo update preceded the disconnect.

---

## Files changed for this instrumentation

| File | What was added |
|------|---------------|
| `code/client/cl_scrn.c` | `SCR_LogConnectInfo`, `SCR_LogServerCmd`, `SCR_LogSnapState`, `SCR_LogTimeout`, `SCR_LogNote`, `SCR_LogPacketDrop`, `SCR_LogCapRelease`, `SCR_LogOOBPacket`, `SCR_LogOOBDisconnect`, `SCR_LogUserinfoSend`, `SCR_OOBIgnoredIncrement`, `SCR_LogDisconnect`; `netMonCapHitsSession`+`netMonOOBIgnoredSession` per-connection counters; `cmdTime` (`ps.commandTime`) added to `SCR_LogSnapState` output |
| `code/client/client.h` | Declarations for all new log functions; `cl_noOOBDisconnect` extern; `SCR_LogSnapState` signature updated |
| `code/client/cl_parse.c` | Call sites: `SCR_LogConnectInfo`, `SCR_LogServerCmd`, `SCR_LogDisconnect` (early disconnect), `SCR_LogSnapState` (now passes `ps.commandTime`), `SCR_LogPacketDrop`, `SCR_LogNote` × 4 (delta failures + reloverflow); fixed stale "cap will be disabled" comment |
| `code/client/cl_cgame.c` | `SVCMD:DISCONNECT` note for in-band disconnect; `CG_SENDCONSOLECOMMAND` guard (log all, block dangerous); `CG_CVAR_SET` expanded intercept list (`net_qport`, `rate=0`, `net_dropsim`, `cl_packetdelay`, `cl_packetdup`, `cl_timeout`, `in_mouse`, `cl_noOOBDisconnect`) |
| `code/client/cl_main.c` | `cl_noOOBDisconnect` cvar (`CVAR_ARCHIVE\|CVAR_PROTECTED`); `SCR_LogOOBPacket` at top of `CL_ConnectionlessPacket`; `SCR_LogOOBDisconnect` in OOB disconnect handler; `SCR_LogUserinfoSend` + `WARN:RELWND` in `CL_CheckUserinfo` |

---

## Research history

Chronological record of every PR and investigation step related to the sporadics.

| PR / commit | Date | What was done |
|-------------|------|---------------|
| **PR #68** `fix-vanilla-server-connection-issues` | 2026-03-12 | First attack: added a universal `cl.serverTime` safety cap (clamps `cl.serverTime` to not exceed the last received `snap.serverTime + cap_margin`) for vanilla-server connections. Also added the first version of the OOB disconnect handler in `cl_main.c`: when an OOB `disconnect` arrives, immediately call `Com_Error(ERR_SERVERDISCONNECT)`. This stopped the immediate-disconnect symptom but the sporadics continued. |
| **PR #71** `increase-servertime-cap` | 2026-03-13 | Cap margin raised from 1 ms to 2 ms, then to a proportional `snapshotMsec / 4` margin (2–8 ms depending on `sv_fps`). Added a 2-second backstop: if silence exceeds 2 s, release the cap entirely. Documented in `CLIENT.md`. |
| **PR #107** `standby-before-next-steps` | 2026-03-14 | Added full `cl_netlog 1/2` infrastructure (`SCR_Log*` family of functions), the per-connection counters, `DISCONNECT` trace block logged to both console and file, and the initial version of this NETDEBUG.md. Goal: capture real session data. |
| **PR #108** `analyze-svcmd-logs` | 2026-03-14 | Analysed SVCMD logs from earlier test sessions. Ruled out in-band `SVCMD:DISCONNECT` as the cause — the server never sent a reliable-command disconnect. Confirmed pattern is always OOB. |
| **PR #109** `fix-oob-guard-issue` | 2026-03-15 | **Critical bug fix**: the OOB disconnect handler was returning `qfalse` after ignoring the packet. `CL_PacketEvent` only calls `clc.lastPacketTime = cls.realtime` when the handler returns `qtrue`. With `qfalse` the `cl_timeout` clock started from the last *in-band* snapshot, so the client would timeout ~30 s after the OOB flood began regardless of `cl_noOOBDisconnect`. Fixed to return `qtrue` so OOB packets count as liveness evidence. |
| **PR #110** `debug-random-disconnect-issue` | 2026-03-15 | Added `cmdTime` (`ps.commandTime`) to the `SNAP` log line. Corrected the documentation: `ping=999` is a **symptom** of `ps.commandTime` freezing, not a calculation artefact. Added the §2h section (server stops processing usercmds) as a confirmed scenario. Analysed the two captured `netdebug_20260314_*.log` files — see §Captured session analysis below. |

---

## Captured session analysis (2026-03-14)

Two back-to-back sessions were captured with `cl_netlog 2` on the same vanilla URT4 server
(`|RFA| RisenFromAshes.us`) on 2026-03-14. Both ended in spontaneous disconnects.
Log files: `netdebug_20260314_202819.log` (session 1) and `netdebug_20260314_203024.log` (session 2).

### Session metrics

| Metric | Session 1 | Session 2 |
|--------|-----------|-----------|
| Log file | `netdebug_20260314_202819.log` | `netdebug_20260314_203024.log` |
| Wall-clock start | 20:28:19 | 20:30:24 |
| Server snapshot Hz | 20 Hz (snapshotMsec=50) | 20 Hz (snapshotMsec=50) |
| Server type | vanilla=1 forbidsAdaptive=1 | vanilla=1 forbidsAdaptive=1 |
| Auth response | `no account` | `no account` |
| Last good ping | 31 ms (t=102850, msg=1972) | 32 ms (t=227750, msg=1985) |
| Ping jump to 999 | msg=1972→1973 (single snap) | msg=1985→1986 (single snap) |
| cmdSeq frozen at | 406 (relSeq=relAck=17) | 352 (relSeq=relAck=9) |
| Server time at kick | svrT≈104842 ms into server uptime | svrT≈229742 ms into server uptime |
| Gap last-snap → first OOB silenceMs | 56 ms | 64 ms |
| OOB flood rate | ~120 pkt/s | ~124 pkt/s |
| Total OOB packets until reconnect | 2743 | 1974 |
| OOB flood duration | ~22.8 s | ~15.9 s |
| Session duration before kick | ~98.6 s | ~99.3 s |
| Notable precursor | Ping blip 263→247→0 ms at t≈100100 ms (~2.75 s before kick) | None observed |

### Confirmed sequence in both sessions

```
1.  Normal gameplay: ping 31–40 ms, snapshots at 20 Hz, no drops, no delta errors.

2.  [cmdTime freeze]  ps.commandTime stops advancing.  The client continues sending
    usercmds but the server stamps none of them into the snapshot.

3.  [ping → 999]  Over the next ~50 ms the 32 outPacket history slots all advance
    past the frozen ps.commandTime.  On the first snap where no slot matches,
    ping reads as 999.  Transition is instantaneous — one snap goes from 31 ms
    to 999 ms with no intermediate values.

4.  [OOB flood begins ~56–64 ms after the last snapshot]  The server sends OOB
    `disconnect` packets at ~120–124 pkt/s while simultaneously continuing to
    send game snapshots (snapshots carry on arriving for a few more frames).

5.  [Snapshots stop]  In-band game data ceases entirely.  The OOB flood keeps
    arriving; `cl_noOOBDisconnect 1` keeps it ignored.  The client keeps sending
    outgoing packets at normal rate (~125 pkt/s, ~3100 B/s).

6.  [Reconnect]  The server's previous session slot is freed (presumably when the
    server accepts the client's reconnect attempt), the CONNECT event fires, and
    a new session begins on the same server without any user action.
```

### Key observations

**1. Single-snap ping transition is the fingerprint.**  
In both sessions, ping goes from 31–32 ms to 999 in exactly one snapshot interval
(50 ms).  The only mechanism that exhausts all 32 `outPackets` history slots that
fast is a frozen `ps.commandTime`.  Jitter, cap artefacts, or packet loss cannot
do this — they produce intermediate values.

**2. Server kicked while game data was still flowing.**  
`silenceMs=56–64 ms` at `sessionCount=1` (the *first* OOB packet of the flood)
means the last in-band packet from the server was only ~60 ms before the OOB
arrived at the client.  The last snapshot's server time (`snapT=104850` / `snapT=229750`)
matches the server's clock at kick time (`svrT=104842` / `svrT=229742`) to within
8 ms.  The server decided to kick while it was still actively sending snapshots —
this is a deliberate server-side kick, not a network timeout or server crash.

**3. `cl_noOOBDisconnect 1` worked as intended.**  
Without the guard the client would have disconnected immediately at `sessionCount=1`.
Instead it stayed connected, the OOB flood was absorbed, and the server eventually
freed the slot.  The client reconnected automatically.  No user action was needed.

**4. Reliable window was clean at kick time.**  
`relSeq=relAck` in both sessions (window=0) rules out a reliable-command overflow
on the *client→server* path as the trigger.

**5. No in-band `SVCMD:DISCONNECT` in either session.**  
The server never sent a reliable `disconnect` command.  The kick mechanism is
entirely OOB.  There is no server-supplied reason string.

**6. No DROP, no SNAP:DELTA_OLD, no TIMEOUT, no FAST/RESET in either session.**  
The connection was clean from the network perspective.  The kick was not caused by
packet loss or a timing artefact.

**7. Ping blip precursor in session 1 (unconfirmed link).**  
Two snapshots at t=100100 and t=100150 showed ping=263→247 ms, then immediately
dropped to ping=0 (a calculation artefact after the safety cap absorbed the jump),
2.75 seconds before the fatal ping=999 at t=102900.  This transient could indicate
the server briefly stopped or delayed stamping a usercmd, then resumed — possibly
the same condition that then triggered the full freeze 2.75 s later.  Session 2
had no such precursor.

### What the server-side trigger could be (still unknown)

The server stops processing our usercmds precisely at the kick moment.  Candidates
for what causes it in the URT4 game QVM `ClientThink`:

| Candidate | Evidence for | Evidence against |
|-----------|-------------|-----------------|
| `sv_maxPing` kick on transient high ping | Ping blip precursor in session 1 (263→247 ms is within sv_maxPing range on many servers) | Session 2 had no precursor; sv_maxPing is usually 150–200 ms |
| Per-client auth gate (no account = kick after N seconds) | Both sessions lasted ~99 s, a suspiciously consistent duration; auth reports `no account` on both connects | No `SVCMD:DISCONNECT` reason string; other `no account` clients remain connected longer on the same server |
| Reliable-overflow on server→client path | Not visible on our side — we only see client→server reliable window | Would normally produce `reason="reliable overflow"` in SVCMD:DISCONNECT |
| URT4 QVM per-session inactivity or anti-cheat timer | Could fire at ~99 s regardless of activity | No evidence; client was actively sending inputs throughout |
| Server-side usercmd `serverTime` ahead of `sv.time` | Safety cap prevents our `cl.serverTime` from running ahead, but the gap is small | The cap margin is only 2–8 ms; unlikely to cause a kick |

**Next step:** Obtain or decompile the vanilla URT4 `qagame.qvm` to audit
`ClientThink` / `ClientEndFrame` for any ~99-second or auth-triggered kick logic.
Cross-reference with the `[auth] no account` handling path.

---

