# Last Session Briefing
**Updated:** 2026-03-01  
**Branch:** `copilot/analyze-netlog-data`

> This file is rewritten at the end of every session.  Read it first.
> Full session history lives in `archive/docs/debug-session-*.md`.

---

## Status

**INTERP/EXTRAP oscillation: FIXED.**  
User shared a netlog confirming zero EXTRAP events across the entire session.
The `snap.serverTime - 1` cap is working; fI peaks at 0.938 (= 15/16) and never crosses 1.0.

**Remaining symptom: intermittent top-line chop in the lagometer.**  
The log did not contain enough data to diagnose it.  The new fields added this session
(see below) will tell us whether it is a client frame-time spike or snap delivery jitter.

---

## What the Netlog Showed

Session ran at 125 Hz then switched to 62 Hz via `\rcon sv_fps 60`.

| Phase | snap | fI range | dT swing | EXTRAP |
|-------|------|----------|----------|--------|
| 125 Hz | 8 ms | 0.500–0.625 | ±1 ms | 0 |
| 62 Hz (drifting) | 16 ms | 0.000–0.938 | 9 ms | 0 |
| 62 Hz (settled) | 16 ms | 0.125–0.688 | ±1 ms | 0 |

Key observations:
- **No DELTA FAST or RESET events** logged — `CL_AdjustTimeDelta` was never triggered.
- **fI=0.938** confirms the `-1` cap firing at 62 Hz ceiling (15/16 = 0.9375). ✓
- **Settled 62 Hz oscillation amplitude is 10× larger than ±1 ms dT drift** — structural
  beat between render frame rate and snap rate, not a timing bug.
- The top-line chop is unrelated to EXTRAP.  The log had no frame-time or snap-jitter
  data, so the cause remains unknown.

---

## New Diagnostic Fields Added This Session

`cl_netlog 2` STATS lines now include two new fields at the end:

```
[HH:MM:SS] STATS  snap=62Hz  ping=40ms  fI=0.625(INTERP)  dT=-21189ms
           drop=0/s  in=4677B/s  out=4163B/s  ft=16/23ms  snapgap=2ms
```

| Field | Meaning | What a bad value tells us |
|-------|---------|--------------------------|
| `ft=avg/max` | Client frame time: average / peak this second (ms) | `max` spike > 2× `avg` → OS jitter pushed `cl.serverTime` past a snap boundary → causes top-line chop |
| `snapgap=Xms` | Peak deviation of measured snap interval from `snapshotMsec` (ms) | Large value → snaps arriving late or bunched → chop is network-side |

**Files changed:** `cl_scrn.c`, `cl_parse.c`, `cl_main.c`, `client.h`

---

## What to Look for Next Time the Chop Occurs

Enable `cl_netlog 2` before play.  When chop is seen, look at the STATS lines bracketing it:

- **`ftmax` spike (e.g. `ft=16/45ms`)** → client frame hiccup is the cause.
  Next step: investigate OS scheduler / vsync / frame-limiter.
- **`snapgap` spike (e.g. `snapgap=18ms` at 62 Hz)** → a snap arrived a full interval late.
  Next step: check network path jitter or server tick consistency.
- **Neither spikes** → something else; add per-frame serverTimeDelta logging.

---

## Diagnostic Quick Reference

| Tool | How to activate | What it shows |
|------|----------------|---------------|
| `cl_netgraph 1` | in-game cvar | Live overlay: snap Hz, ping, smoothed fI, dT, drop, in/out, seq |
| `cl_netlog 1` | in-game cvar | Continuous log: console cmds + FAST/RESET delta events |
| `cl_netlog 2` | in-game cvar | Level 1 + per-second stats incl. `ft` and `snapgap` |
| `netgraph_dump` | console command | Point-in-time: full timing state + all server cvars |

Log location: **game folder** → `netdebug_YYYYMMDD_HHMMSS.log`

---

## Key Files

| File | Role |
|------|------|
| `code/client/cl_cgame.c` | serverTime cap, extrapolation detection, AdjustTimeDelta |
| `code/client/cl_scrn.c` | net monitor widget, log hooks, netgraph_dump, STATS line |
| `code/client/cl_parse.c` | snapshotMsec EMA, snap-interval measurement + jitter hook |
| `code/client/cl_main.c` | per-frame frametime hook |
| `code/client/client.h` | SCR_* declarations |
