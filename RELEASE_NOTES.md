# Release Notes — v1.1

## Feature Highlights

### 🔊 Audio: Suppression & Combat Feedback

- **Per-source HF lowpass filter** replaces the old global volume duck. Nearby suppressed weapons now apply an individual high-frequency roll-off to each OpenAL source, producing a more natural "muffled" effect without cutting overall volume.
- **Tinnitus effect** — a short ringing tone is triggered when an explosion detonates close to the player.
- **Head-hit audio cue** — a distinct sound event fires on head-hit detection.
- **Health-fade audio** — audio quality degrades gradually as player health drops.
- **Dual suppressor detection** — a weapon is classified as suppressed when either the `g_gear` userinfo contains the `U` bit *or* the sound filename matches the `-sil` pattern (covers the Mac-11 and similar edge cases).
- **Enemy-suppressed volume category** — suppressed enemy weapons have their own independent mixer channel, separate from friendly suppressors.
- **Friendly-fire exclusion** — teammates' weapons are excluded from suppression ducking so they remain clearly audible.

### 🔊 Audio: Configurable Volume Groups

- **`s_alExtraVolEntry1`–`s_alExtraVolEntry8`** — eight independent per-slot cvars for user-adjustable loud sounds (replaces the single `s_alExtraVolList` string).
- **Per-sample dB cut** — append `:-N` to any entry to reduce that sample by N decibels (e.g. `hit.wav:-3`).
- Hit and kill sounds default to **–2.5 dB** to reduce ear fatigue in extended sessions.

### 🔊 Audio: Per-Category Volume Mixer

- **Category mixer (0–10 scale)** with independently adjustable channels: `weapon`, `suppressed`, `enemy-suppressed`, `world`, `ambient`, `UI`, and `music`.
- **EFX effects chain** — echo, chorus, reverb, and fire-impact processing added via the `alEFX` extension.
- **Grenade/explosion audio bloom** — volume briefly swells on grenade detonation for visceral impact feedback.
- **Incoming-fire reverb** — distant weapon fire gains a subtle reverb tail.
- **Loop fade transitions** — ambient and looping sounds cross-fade smoothly on start/stop instead of cutting abruptly.

### 🔊 Audio: DMA-style Slot Management

- The preemption/loop machinery has been replaced with a **DMA-style slot allocator** modelled on the software mixer's channel system, giving predictable behaviour across all game events.
- **Generous audio source pool** — the hard pool limit is raised so more concurrent sounds play before any preemption occurs.

### 🔊 Audio: New CVars & Commands

| Cvar / Command | Description |
|---|---|
| `s_alDirectChannels` | Control `AL_SOFT_direct_channels` per-source (bypass OpenAL panning for local sounds) |
| `s_alEFX` | Enable/disable EFX effects chain at runtime |
| `s_alLocalSelf` | Ambient loudness normalisation for first-person sounds |
| `s_alReset` | Runtime command to tear down and reinitialise the audio device |
| `s_alTrace` | Debug logging for OpenAL source allocation and lifecycle |
| `s_devices` | Console command — prints all available audio output devices |

### 🔊 Audio: OpenAL Soft 1.25.1 Integration

- **`soft_oal.dll` bundled** in Windows x64 CI artifacts — no separate OpenAL Soft install required.
- **Three-layer HRTF forcing** at engine, ALC-request, and extension-query levels for reliable headphone spatialisation.
- **Ambisonic panning** support added for multi-channel configurations.
- **EFX reverb/underwater** effects via `alEFX`.
- **Occlusion smoothing** with velocity look-ahead to prevent rapid filter jumps on fast-moving entities.
- **Inverse-clamped distance model** replaces the legacy linear falloff for physically correct attenuation.

### 🔊 Audio: dmaHD WASAPI Backend (Windows)

- dmaHD ported from the legacy DirectSound path to **WASAPI** for lower latency and better compatibility with modern Windows audio stacks.
- **Native format negotiation** via `GetMixFormat` — the engine matches the OS mixer's native sample rate and bit depth for **bit-perfect output** (no OS resampling, no APO interference).
- Correct **44,100 Hz enforcement** for systems whose OS mixer runs at 44.1 kHz.

### ⏱️ Client: Adaptive Timing (`cl_adaptiveTiming`)

- New **proportional mode** (`cl_adaptiveTiming 2`) — drift correction scales proportionally to the size of the timing error for faster equilibrium convergence.
- Adaptive timing is **automatically disabled on vanilla Q3 servers** (clean pass-through) and re-enables only when connecting to an FTWGL/URT server that sets `sv_allowClientAdaptiveTiming 1`.
- `sv_allowClientAdaptiveTiming` — new server cvar giving admins explicit control over whether connected clients may activate adaptive timing.

### 🎯 Server: Antilag Accuracy

- **`snapshotMsec` compensation** added to the fire-time formula — the rewind now accounts for the client-side interpolation offset (`cl_interp ≈ snapshotMsec`) so the rewound shadow position matches exactly what the shooter saw on screen.
- **Shadow recording order** moved to after `GAME_RUN_FRAME` so `shadow[T]` and `snapshot[T]` always hold positions from the same game tick, eliminating a one-tick positional mismatch.

### 🚀 CI / Release Pipeline

- New **`release.yml`** GitHub Actions workflow creates versioned release assets automatically on tag push.
- CI streamlined to **Windows x64 + Linux x86/x64** for release builds; ARM moved to a manual workflow.
- Both **client and dedicated-server binaries** are published as separate artifacts.

---

## Bug Fixes

### 🔊 Audio: Directionally sensitive crackling on busy servers

Three bugs in the multi-hop corridor probing and BSP area connectivity floor
features introduced distorted crackling audio.  The corruption was directionally
sensitive, scaled with the number of concurrent sound events, and was most
prominent on 32-player vanilla servers during heavy firefights.

- **BSP floor left stale multi-hop waypoint in `acousticPos`** — when the
  connectivity floor raised the occlusion fraction, `acousticPos` still held the
  last multi-hop waypoint (which is near the *listener*, not the source).  The
  projection step then placed the apparent source in a completely wrong direction
  that changed every trace tick.  Fixed: reset `acousticPos` to `srcOrigin` when
  the BSP floor wins so no gap-redirect is applied.

- **`acousticOffset` hard-assigned, causing abrupt HRTF direction jumps** —
  `occlusionGain` was already IIR-smoothed but `acousticOffset` (the HRTF
  redirect) was written directly at each trace tick (every 4–8 frames for
  medium/far sources).  When the best multi-hop corridor axis changed between
  ticks the offset snapped to a new direction, producing a spatial discontinuity
  that the HRTF convolution rendered as a directional click/crackle.  Fixed:
  IIR-blend `acousticOffset` toward the new target at step 0.35, matching the
  gain-opening rate.

- **Unbounded multi-hop trace burst with many concurrent sounds** — every new
  source starts with `occlusionTick = 0` so all sources born in the same frame
  trace together.  Each multi-hop run casts up to 16 `CM_BoxTrace` calls; with
  30+ weapon sounds in one tick this stacked 400–500 traces in a single audio
  frame, starving the OpenAL buffer fill thread.  Fixed: per-frame budget of 8
  multi-hop runs (`S_AL_MULTIHOP_BUDGET`), reset each `s_al_loopFrame`.  Sources
  beyond the budget skip the two-hop loop and use the near-source-probe result.
