# Shadow Antilag — Changelog & Review Notes

Reference document for manual review of engine-side shadow antilag fixes
applied in **PR #35** (merged), **PR #36**, and **PR #37**.

All changes are in `code/server/sv_antilag.c` unless noted otherwise.

---

## Table of Contents

1. [Architecture recap](#architecture-recap)
2. [PR #35 — Fire time & shadow history config fixes](#pr-35--fire-time--shadow-history-config-fixes)
3. [PR #36 — sv.time / svs.time domain fix](#pr-36--svtime--svstime-domain-fix)
4. [PR #37 — Fire time formula revision (ping/2)](#pr-37--fire-time-formula-revision-ping2)
5. [Combined before/after summary](#combined-beforeafter-summary)
6. [Review checklist](#review-checklist)

---

## Architecture recap

```
GAME TICK  (sv_gameHz Hz)        → QVM sees this. level.time advances here.
SHADOW RECORDING  (sv_fps Hz)    → Engine only. Records entity positions
                                    into svShadowHistory[] ring buffer.
HIT REGISTRY  (on G_TRACE)      → Engine intercepts trap_Trace from QVM.
                                    Rewinds entities → runs trace → restores.
                                    QVM never sees rewound state.
```

Two independent clocks:
- **`sv.time`** — resets on map change; sent to clients in snapshots
- **`svs.time`** — persists across map changes; wall-clock server time

Clients calibrate their `serverTime` estimate from snapshots, so
`lastUsercmd.serverTime` is in the **`sv.time`** domain.

---

## PR #35 — Fire time & shadow history config fixes

**Merged:** 2026-03-03 · **Files:** `sv_antilag.c`, `sv_antilag.h`, `sv_main.c`

### Fix 1: Fire time over-rewind (primary)

**Bug:** `SV_Antilag_GetClientFireTime` used `svs.time - cl->ping` as the
rewind target. `cl->ping` is the full round-trip time, but the client's
perception of server time is approximately half RTT in the past. A 40 ms
ping player got a 40 ms rewind when the correct value is ~20 ms.

```c
// BEFORE (original)
fireTime = svs.time - cl->ping;

// AFTER (PR #35)
fireTime = cl->lastUsercmd.serverTime;
```

**Why `lastUsercmd.serverTime`:**
- Set from the usercmd *before* `GAME_CLIENT_THINK` fires
- Available during weapon traces (QVM sees it as `AttackTime`)
- Already incorporates `cl_timeNudge` / `cl_autoNudge` from the client
- Makes `ut_timenudge` correctly irrelevant (shadow rewind overrides FIFO)

### Fix 2: Bots excluded from shadow history recording

**Bug:** The recording loop filtered out bots (`NA_BOT`), so human players
shooting **at** bots got zero lag compensation — bot positions were never
in the shadow history ring buffer.

```c
// BEFORE (original) — in SV_Antilag_RecordPositions
for ( i = 0; i < sv_maxclients->integer; i++ ) {
    client_t *cl = &svs.clients[i];
    if ( cl->state != CS_ACTIVE ) continue;
    if ( cl->netchan.remoteAddress.type == NA_BOT ) continue;  // ← WRONG
    SV_Antilag_RecordClient( i, svs.time );
}

// AFTER (PR #35) — bots are recorded
for ( i = 0; i < sv_maxclients->integer; i++ ) {
    client_t *cl = &svs.clients[i];
    if ( cl->state != CS_ACTIVE ) continue;
    // No NA_BOT filter — bots must be recorded so humans can rewind them
    SV_Antilag_RecordClient( i, svs.time );
}
```

Note: bots are still **skipped as shooters** in `InterceptTrace` (they
have no lag to compensate). The fix only affects recording targets.

### Fix 3: sv_fps change detection race

**Bug:** `sv_fps->modified` is cleared by `SV_TrackCvarChanges` before
`SV_Antilag_RecordPositions` runs, so `sv_fps` changes were never detected.

```c
// BEFORE
if ( sv_physicsScale->modified || sv_antilagMaxMs->modified || sv_fps->modified )

// AFTER — track integer value directly
static int sv_antilag_lastFpsValue = 0;
int currentFps = sv_fps ? sv_fps->integer : 40;
qboolean fpsChanged = ( currentFps != sv_antilag_lastFpsValue );
if ( sv_physicsScale->modified || sv_antilagMaxMs->modified || fpsChanged )
```

### Fix 4: Stale history on cvar round-trip

**Bug:** Ring buffer flush was conditional on slot count changing. A
round-trip like `sv_fps 60→40→60` keeps the same slot count but leaves
entries with wrong timestamp spacing, producing incorrect interpolation.

```c
// BEFORE — only flush if slot count changed
if ( sv_shadowHistorySlots != oldSlots ) {
    Com_Memset( sv_shadowHistory, 0, sizeof( sv_shadowHistory ) );
}

// AFTER — always flush on any timing change
Com_Memset( sv_shadowHistory, 0, sizeof( sv_shadowHistory ) );
```

### Fix 5: Duplicate-timestamp entries from physicsScale loop

**Bug:** `SV_Frame` called `RecordPositions` in a `physicsScale` loop
(e.g. 3×), but all calls used the same `svs.time`, filling 2/3 of the
ring buffer with duplicate timestamps and reducing effective history
coverage to ~175 ms instead of 200 ms (24 slots / 3 per tick = 8 unique
timestamps spanning 7 × 25 ms).

```c
// BEFORE (sv_main.c) — recorded 3× per tick with same timestamp
for ( _s = 0; _s < _scale; _s++ ) {
    SV_Antilag_RecordPositions();
}

// AFTER (sv_main.c) — record once per engine tick
SV_Antilag_RecordPositions();
```

### Fix 6: Config/recording rate mismatch

**Bug:** After removing the physicsScale loop, `ComputeConfig` still
calculated slots for `fps × scale` Hz instead of the actual `fps` Hz
recording rate.

```c
// BEFORE
int shadowHz = fps * scale;
sv_shadowTickMs = 1000 / shadowHz;
sv_shadowHistorySlots = ( shadowHz * winMs ) / 1000;

// AFTER — based on fps alone
sv_shadowTickMs = 1000 / fps;
sv_shadowHistorySlots = ( fps * winMs ) / 1000 + 1;
```

### Fix 7: Off-by-one in slot count

**Bug:** `(Hz * windowMs) / 1000` under-counts by 1 because N slots span
N−1 intervals. At defaults (40 Hz, 200 ms) this gave 8 slots = 175 ms
instead of 9 slots = 200 ms.

```c
// BEFORE
sv_shadowHistorySlots = ( shadowHz * winMs ) / 1000;

// AFTER — +1 to cover full window
sv_shadowHistorySlots = ( fps * winMs ) / 1000 + 1;
```

---

## PR #36 — sv.time / svs.time domain fix

**Status:** Open · **Files:** `sv_antilag.c`

### The core problem

PR #35 correctly changed the fire time formula to
`cl->lastUsercmd.serverTime`, but all the **surrounding comparisons**
(clamps, fallback, recording timestamps) still used `svs.time`.

Since `lastUsercmd.serverTime` is calibrated from snapshots (which send
`sv.time`), the value is in the `sv.time` domain. After a map change the
two clocks diverge:

```
svs.time = 500100      // persisted from previous map
sv.time  = 408         // reset on map load
fireTime = ~400        // from lastUsercmd.serverTime, calibrated via sv.time

svs.time - fireTime = 499700  >>  maxRewind (200)
→ every shot clamped to 200 ms rewind regardless of actual ping
```

### Fix 1: Recording timestamps — `svs.time` → `sv.time`

Shadow history must be in the same time domain as `fireTime` so the
interpolation lookup finds correct entries.

```c
// BEFORE (after PR #35)
SV_Antilag_RecordClient( i, svs.time );

// AFTER (PR #36)
SV_Antilag_RecordClient( i, sv.time );
```

### Fix 2: Fire time clamps — `svs.time` → `sv.time`

```c
// BEFORE — fallback and clamps used svs.time
if ( shooterNum < 0 || shooterNum >= sv_maxclients->integer )
    return svs.time;
...
if ( svs.time - fireTime > maxRewind )
    fireTime = svs.time - maxRewind;
if ( fireTime > svs.time )
    fireTime = svs.time;

// AFTER — all in sv.time domain
if ( shooterNum < 0 || shooterNum >= sv_maxclients->integer )
    return sv.time;
...
if ( sv.time - fireTime > maxRewind )
    fireTime = sv.time - maxRewind;
if ( fireTime > sv.time )
    fireTime = sv.time;
```

### Fix 3: Save/restore simplification

**Before:** `RewindAll` called `SV_Antilag_GetMostRecentPosition` to
get the "true" position from shadow history, saving that instead of the
entity's actual current position. This was originally to override FIFO
state, but:
- For bots (no FIFO shift), shadow-history "true pos" == current pos
- For humans, the FIFO has already shifted them; saving FIFO state is
  correct because `G_UnTimeShiftAllClients` expects to undo exactly that

**After:** Save `gent->r.currentOrigin` directly. Removed
`SV_Antilag_GetMostRecentPosition()` entirely.

```c
// BEFORE — two-path save via shadow history lookup
if ( SV_Antilag_GetMostRecentPosition( i, trueOrigin, trueAbsmin, trueAbsmax ) ) {
    sv_shadowSaved[i].saved = qtrue;
    VectorCopy( trueOrigin, sv_shadowSaved[i].origin );
    ...
} else {
    sv_shadowSaved[i].saved = qtrue;
    VectorCopy( gent->r.currentOrigin, sv_shadowSaved[i].origin );
    ...
}

// AFTER — always save current entity position
sv_shadowSaved[i].saved = qtrue;
VectorCopy( gent->r.currentOrigin, sv_shadowSaved[i].origin );
VectorCopy( gent->r.absmin,        sv_shadowSaved[i].absmin );
VectorCopy( gent->r.absmax,        sv_shadowSaved[i].absmax );
```

### Fix 4: No-history fallback simplification

**Before:** When no shadow history existed for `targetTime`, the code
force-wrote the saved position back into the entity and re-linked it
(a no-op write that still cost a `SV_LinkEntity` call).

**After:** Just leave the entity at its current position. The trace runs
against whatever position the QVM has set.

### Fix 5: Debug output — `svs.time` → `sv.time`

```c
// BEFORE
int rewindMs = svs.time - fireTime;
Com_Printf( "... (svs.time=%d fireTime=%d)\n", ..., svs.time, fireTime );

// AFTER
int rewindMs = sv.time - fireTime;
Com_Printf( "... (sv.time=%d fireTime=%d)\n", ..., sv.time, fireTime );
```

### Intentionally NOT changed: Rate tracking

`SV_Antilag_NoteSnapshot` uses `svs.time` to measure snapshot send rate.
This is correct — it measures wall-clock intervals, not game time. The
rate tracking system is unrelated to the rewind calculations.

---

## PR #37 — Fire time formula revision (ping/2)

**Status:** Open · **Files:** `sv_antilag.c`, `sv_antilag.h`, `Example_server.cfg`

### The core problem

PR #35 changed the fire time from `svs.time - cl->ping` to
`cl->lastUsercmd.serverTime`. While `lastUsercmd.serverTime` is in the
correct `sv.time` domain, it includes the client's **local interpolation
delay** (cl_timeNudge, cl_autoNudge, snapshot buffering) on top of the
network latency. This produces excessive rewind:

```
ping = 45 ms        → expected half-RTT rewind ≈ 22 ms
lastUsercmd rewind  → actual rewind = 69–77 ms  (1.5× full RTT)
```

The extra ~47 ms comes from the client's interpolation pipeline:
- `cl_autoNudge` adds approximately `snapInterval / 2` (~16 ms)
- Snapshot smoothing / `CL_AdjustTimeDelta` adds another ~16–25 ms
- These offsets are correct for the QVM's FIFO antilag (which operates in
  the `level.time` domain where they cancel out), but in the shadow system
  they double-up because shadow history already records at `sv.time`

The result: hits feel delayed and spongy compared to the QVM's own FIFO
antilag. Reverting to `sv_antilagEnable 0` restores responsiveness because
the FIFO system's `level.time` domain absorbs the interpolation delay
naturally.

### Fix 1: Fire time — `lastUsercmd.serverTime` → `sv.time - ping/2`

```c
// BEFORE (after PR #35 + PR #36)
fireTime = cl->lastUsercmd.serverTime;

// AFTER (PR #37)
fireTime = sv.time - cl->ping / 2;
```

**Why `sv.time - cl->ping / 2`:**
- `cl->ping` is the full round-trip time, measured by the engine
- `ping / 2` approximates one-way latency (server → client)
- Shadow history records at `sv.time`, so rewinding by one-way latency
  gives the position the client was seeing when they fired
- Does not include client interpolation delay → rewind is tight and
  responsive
- For bots (ping = 0): `sv.time - 0 = sv.time` → no rewind (correct,
  bots are already skipped as shooters in `InterceptTrace`)

**Example improvement (45 ms ping player):**

| Formula | Rewind | Feel |
|---------|--------|------|
| `svs.time - cl->ping` (original) | 45 ms, wrong domain | broken after map change |
| `cl->lastUsercmd.serverTime` (PR #35) | 69–77 ms | spongy, delayed |
| `sv.time - cl->ping / 2` (PR #37) | 22 ms | responsive, accurate |

### Fix 2: Header cvar name mismatch

`sv_antilag.h` declared `sv_antilagTraceLog` and `sv_snapRateLog`, but
`sv_antilag.c` defines `sv_antilagDebug` and `sv_antilagRateDebug`.
Fixed the header externs to match the implementation.

```c
// BEFORE (sv_antilag.h)
extern cvar_t *sv_antilagTraceLog;
extern cvar_t *sv_snapRateLog;

// AFTER
extern cvar_t *sv_antilagDebug;
extern cvar_t *sv_antilagRateDebug;
```

### Fix 3: Example config cvar names

`docs/Example_server.cfg` referenced the stale header names. Updated to
match the actual cvar names registered by `SV_Antilag_Init()`.

---

## Combined before/after summary

| Area | Original code | After PR #35 | After PR #36 | After PR #37 |
|------|--------------|--------------|--------------|--------------|
| **Fire time formula** | `svs.time - cl->ping` | `cl->lastUsercmd.serverTime` | *(unchanged)* | **`sv.time - cl->ping / 2`** |
| **Fire time clamps** | vs `svs.time` | vs `svs.time` | vs **`sv.time`** | *(unchanged)* |
| **Recording timestamp** | `svs.time` (bots excluded) | `svs.time` (bots included) | **`sv.time`** (bots included) | *(unchanged)* |
| **sv_fps detection** | `sv_fps->modified` (race) | Integer value tracking | *(unchanged)* | *(unchanged)* |
| **History flush** | Only if slot count changed | Always on timing change | *(unchanged)* | *(unchanged)* |
| **Recording loop** | `physicsScale×` per tick | 1× per tick | *(unchanged)* | *(unchanged)* |
| **Slot calculation** | `fps × scale` Hz, off-by-one | `fps` Hz, +1 | *(unchanged)* | *(unchanged)* |
| **Save/restore** | Via `GetMostRecentPosition` | Via `GetMostRecentPosition` | Direct `gent->r.currentOrigin` | *(unchanged)* |
| **No-history fallback** | Force-write saved pos back | Force-write saved pos back | Leave entity at current pos | *(unchanged)* |
| **Debug output time** | `svs.time` | `svs.time` | **`sv.time`** | *(unchanged)* |
| **Header cvar names** | `sv_antilagTraceLog` | *(unchanged)* | *(unchanged)* | **`sv_antilagDebug`** |

---

## Review checklist

When reviewing these changes, verify:

- [ ] Fire time uses `sv.time - cl->ping / 2` (half-RTT, not
      `lastUsercmd.serverTime` which includes interpolation delay)
- [ ] `sv.time` is used consistently in all shadow antilag paths that
      compare against fire time or shadow history timestamps
- [ ] `svs.time` is **only** used in `NoteSnapshot` rate tracking (wall-clock)
- [ ] Bots are recorded into shadow history (no `NA_BOT` filter in
      `RecordPositions`) but skipped as shooters (in `InterceptTrace`)
- [ ] `ComputeConfig` slot count uses `sv_fps` Hz (not `fps × scale`)
      with `+1` to avoid off-by-one
- [ ] History is always flushed on any timing config change (not just
      when slot count differs)
- [ ] Recording happens once per engine tick (no `physicsScale` loop in
      `SV_Frame`)
- [ ] `RewindAll` saves `gent->r.currentOrigin` directly — no
      `GetMostRecentPosition` lookup
- [ ] `RestoreAll` restores saved position unconditionally after trace
- [ ] The QVM contract is never violated: `level.time` is never touched,
      entities are restored before control returns to the VM
- [ ] Header `sv_antilag.h` externs match `sv_antilag.c` definitions
      (`sv_antilagDebug`, `sv_antilagRateDebug`)
