# tmpbugs.md — Code Issues (re-impla branch)

Issues found while auditing `code/server/` against the documentation.
None of these are committed code fixes — awaiting go-ahead.

---

## BUG-1 — `sv_physicsScale` is a dead/orphaned cvar

**Severity:** Medium  
**Files:** `code/server/sv_antilag.c`, `code/server/sv_antilag.h`, `code/server/sv_init.c`, `docs/project/SERVER.md`, `docs/project/SV_ANTIWARP.md`, `docs/Example_server.cfg`

### What the code does

`sv_physicsScale` is registered in `SV_Antilag_Init`:

```c
// sv_antilag.c:299
sv_physicsScale = Cvar_Get( "sv_physicsScale", "3", CVAR_SERVERINFO );
```

Its `->modified` flag is cleared in `SV_Antilag_RecordPositions`:

```c
// sv_antilag.c:328-330
if ( sv_physicsScale->modified || sv_antilagMaxMs->modified || fpsChanged ) {
    SV_Antilag_ComputeConfig();
    sv_physicsScale->modified = qfalse;
```

But `sv_physicsScale->integer` is **never read**. `SV_Antilag_ComputeConfig` only uses `sv_fps` and `sv_antilagMaxMs`:

```c
// sv_antilag.c:85-98 (ComputeConfig)
int fps   = sv_fps ? sv_fps->integer : 40;
int winMs = sv_antilagMaxMs ? sv_antilagMaxMs->integer : SV_ANTILAG_HISTORY_WINDOW_MS;
sv_shadowTickMs = 1000 / fps;
sv_shadowHistorySlots = ( fps * winMs ) / 1000 + 1;
// sv_physicsScale->integer is NOT used here
```

There is no `physicsScale` loop in `sv_main.c:SV_Frame` either (it was removed in PR #35). Shadow history records at exactly `sv_fps` Hz.

### Impact

- Server admins set `sv_physicsScale 2` in their configs expecting 120 Hz shadow recording at 60 Hz server — they get 60 Hz. No error, no warning.
- `docs/Example_server.cfg` documents `sv_physicsScale` as controlling shadow Hz, which is now false.
- `docs/project/SERVER.md` and `docs/project/SV_ANTIWARP.md` contain accurate descriptions of the shadow system but do not flag this cvar as non-functional.

### Fix options

**Option A (recommended):** Restore the physicsScale multiplier in `ComputeConfig` and `RecordPositions`. Record `sv_fps × sv_physicsScale` shadow positions per second (using a sub-tick accumulator within the `sv_fps` loop, not a separate outer loop). This restores the intended higher-Hz shadow precision.

**Option B:** Remove `sv_physicsScale` from `sv_antilag.c` and `sv_init.c`. Update docs and `Example_server.cfg` to remove all references. Shadow Hz = `sv_fps` Hz.

---

## BUG-2 — `awLastThinkTime` not zeroed on `map_restart`

**Severity:** Low-Medium  
**Files:** `code/server/sv_ccmds.c` (`SV_MapRestart_f`), `code/server/sv_client.c` (`SV_ClientEnterWorld`)

### What the code does

`SV_ClientThink` (called when a real usercmd arrives from the network) sets:

```c
// sv_client.c:2265
cl->awLastThinkTime = sv.gameTime;
```

`SV_MapRestart_f` resets `lastUsercmd` for each active client but does **not** reset `awLastThinkTime`:

```c
// sv_ccmds.c — bottom of SV_MapRestart_f
for ( i = 0; i < sv.maxclients; i++ ) {
    client = &svs.clients[i];
    if ( client->state >= CS_PRIMED ) {
        Com_Memset( &client->lastUsercmd, 0x0, sizeof( client->lastUsercmd ) );
        client->lastUsercmd.serverTime = sv.time - 1;
        // ← awLastThinkTime NOT reset here
    }
}
```

`SV_ClientEnterWorld` (also called during restart for active clients) does **not** reset `awLastThinkTime` either.

### Why this matters

During `SV_MapRestart_f`, four settlement game frames run:

```c
sv.gameTime = sv.time;          // sync before restart
sv.gameTimeResidual = 0;
SV_RestartGameProgs();
for (i = 0; i < 3; i++) {
    sv.time += 100; sv.gameTime += 100;   // +300 ms
    VM_Call(gvm, GAME_RUN_FRAME, sv.gameTime);
}
sv.time += 100; sv.gameTime += 100;       // +100 ms more (= +400 ms total)
VM_Call(gvm, GAME_RUN_FRAME, sv.gameTime);
```

After the restart, `sv.gameTime` is ~400 ms ahead of the last recorded `awLastThinkTime` (set at the old `sv.gameTime` before restart). On the very first real game tick after clients reconnect:

```
awGap = sv.gameTime_new - awLastThinkTime_old  ≈ 400 ms
awTol = _gameMsec ≈ 16 ms  (auto, at sv_fps 60)
400 >> 16 → antiwarp fires for every client
```

All connected clients get one spurious antiwarp injection on the first tick after `map_restart`. In mode 2 this means their movement inputs are immediately zeroed (since 400 ms >> `awTol + awExtraMs + awDecayMs` = 182 ms at defaults).

Note: fresh map loads (`SV_SpawnServer`) are **not** affected — `svs.clients` is `memset` to zero, which zeros `awLastThinkTime`.

### Proposed fix

In `SV_ClientEnterWorld` (called for active clients during `map_restart`):

```c
// sv_client.c — SV_ClientEnterWorld, after client->oldServerTime = 0;
client->awLastThinkTime = 0;   // ← add this line
```

The existing guard `if ( awCl->awLastThinkTime == 0 ) continue;` in the antiwarp loop will then correctly skip the client until the first real usercmd arrives.

---

## BUG-3 — `sv_physicsScale` extern in `sv_antilag.h` is unused by callers

**Severity:** Low  
**Files:** `code/server/sv_antilag.h`

`sv_antilag.h` exports `sv_physicsScale`:

```c
extern cvar_t *sv_physicsScale;
```

No file that includes `sv_antilag.h` (`sv_main.c`, `sv_game.c`, `sv_init.c`) reads `sv_physicsScale->integer`. The extern declaration is only useful if BUG-1 option A is implemented (restoring the sub-tick physicsScale multiplier). If option B is chosen (remove the cvar), this extern should also be removed.

---

## NOTES (not bugs, design decisions)

### `sv_antilag` default is `0` (disabled)

By design — explicit opt-in. Server admins must set `sv_antilag 1` in their config. The `docs/Example_server.cfg` documents this. Not a bug, but new admins who don't read the docs will run with no engine antilag.

### `sv_physicsScale` default is `3` (but has no effect)

Interacts with BUG-1. The registered default of `3` is misleading since the value is not consumed.
