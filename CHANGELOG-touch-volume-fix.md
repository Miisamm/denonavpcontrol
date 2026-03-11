# DenonAvpControl — Touch Volume Fix Changelog

## The Problem

Squeezebox Touch with "fixed output level" enabled (required for DenonAvpControl) pins its internal volume at 100. Volume up/down sends absolute values near 100 (e.g., `99`/`101`), which the plugin's sqrt curve maps to near-max Denon volume. Every press = max volume blast.

| Player | Volume commands | Plugin path | Result |
|---|---|---|---|
| SB3 Classic | Incremental (`+2`, `-2`) | Incremental handler | Correct stepping |
| SB Touch (fixed output) | Absolute near 100 (`99`, `101`) | Falls to absolute handler | Maps to near-max |

## Denon Protocol Volume Format

The Denon/Marantz protocol uses a mixed 2-digit/3-digit format:

| Format | Example | Meaning | Valid values |
|---|---|---|---|
| 2-digit | `"06"`, `"33"`, `"60"` | Whole dB (6.0, 33.0, 60.0) | `00`–`98` |
| 3-digit ending in 5 | `"085"`, `"325"`, `"595"` | Half dB (8.5, 32.5, 59.5) | `005`–`985` |

**Critical**: These formats cannot be compared numerically! `"085"` (8.5dB) > `"60"` (60.0dB) as numbers (85 > 60) but 8.5dB < 60.0dB in reality.

To normalize: 2-digit × 10, 3-digit as-is → "tenths-of-dB" (e.g., `"06"` → 60, `"085"` → 85, `"60"` → 600).

## Version History

### v1–v2: Rate-limiting (failed)

Attempted to rate-limit volume commands by timestamp. Failed because the Touch doesn't send accelerating absolute values — it always sends 99 or 101 (±1 from pinned 100). Delta between commands is always 0-2, never exceeding any rate limit.

### v3: Intercept & convert to incremental via sqrt curve

First working approach. Intercepts near-100 absolute values from fixed-output players, converts to incremental AVR steps using `calculateSBVolume`/`calculateAvrVolume`, sends directly to AVR (bypassing timer delay), resets player back to 100.

**Issues discovered:**
- `int()` on Denon values strips leading zeros (`"085"` → `85`), causing `calculateSBVolume` to misinterpret format → volume jumps to max at low volumes
- Sqrt curve has flat spots at high volumes where adjacent SB values map to same Denon value → stuck between two values
- Numeric comparison of Denon format strings breaks ordering (`"085" > "60"` but 8.5dB < 60dB) → max volume clamp fires incorrectly, jumping low volumes to max

### v4: Format-aware storage

Fixed `int()` bug — store Denon values as strings to preserve format. Added loop to step SB until Denon value actually changes (handles sqrt flat spots). Removed broken numeric Denon comparison for max volume clamping.

**Sub-versions:**
- **v4.1**: Extended SB range from 1–99 to 0–100 (min volume 0.0dB, max 60.0dB)
- **v4.2**: Removed broken max volume enforcement (Denon format breaks `>` comparison)

**Remaining issue:** Sqrt curve gives non-uniform steps — 6dB jump at bottom (SB 0→1), 0.5dB at top (SB 98→99).

### v5: Direct Denon stepping (bypasses sqrt curve entirely)

Complete rewrite of the stepping logic. Instead of converting through the sqrt curve (`calculateSBVolume`/`calculateAvrVolume`), steps the Denon volume directly in 0.5dB increments:

1. Normalize `curVolume` to tenths-of-dB (2-digit × 10, 3-digit as-is)
2. Add/subtract step (5 tenths = 0.5dB)
3. Snap to valid Denon 0.5dB boundary (multiple of 5)
4. Convert back to Denon protocol format (÷10 if whole dB, 3-digit if half)
5. Clamp at 0 and `(80 + maxVol) × 10`

**Benefits:**
- Consistent 0.5dB steps across entire range (0.0–60.0dB)
- No sqrt curve, no `calculateSBVolume`/`calculateAvrVolume` calls
- Changing maxVol plugin setting just changes the clamp — no breakage
- Simpler code, fewer edge cases

**Sub-versions:**

#### v5.1: Direct Denon stepping
- Initial implementation of direct stepping
- Step of 5 tenths-of-dB = 0.5dB per press
- Works correctly across full range

#### v5.2: + 200ms throttle
- Added 200ms minimum gap between sends to Marantz
- Prevents overwhelming the Marantz when Touch IR acceleration sends 10+ commands/sec
- Commands arriving within 200ms of last send are silently dropped (player reset to 100)

#### v5.3: + IR acceleration multiplier + grid snapping
- Uses Touch IR acceleration (`abs(val - 100)`) as step multiplier:

| val from Touch | Deviation | Step size | Typical timing |
|---|---|---|---|
| 99/101 | 1 | 0.5dB | Single press / start of hold |
| 98/102 | 2 | 1.0dB | ~2s hold |
| 96/104 | 4 | 2.0dB | ~3s hold |
| 92/108 | 8 | 4.0dB | ~4s hold |
| 84/116 | 16 (cap) | 8.0dB | ~5s+ hold |

- Acceleration capped at 16× (8dB per step)
- Grid snapping: after applying acceleration, result is snapped to nearest valid 0.5dB boundary (multiple of 5 in tenths-of-dB), rounding in direction of travel
- Fixes v5.3-pre crash where off-grid `curVolume` (from Marantz feedback or previous invalid sends) produced invalid Denon protocol values like `"047"`, `"052"`
- Combined with 200ms throttle: max 5 commands/sec, each carrying acceleration-appropriate step size

#### v5.4: + volume poll sync fix + trailing CR fix

Two bugs fixed that caused volume jumps when the Marantz volume was changed externally (knob, other remote) or on startup:

**Bug 1 — Volume poll fighting with Touch volume pin:**
- The volume poll (`handleVolumeRequest` → `updateSqueezeVol`) was calling `handleVolSet()` to sync the Touch's server-side volume to match the Marantz. This changed the Touch from its pinned 100 to an SB equivalent (e.g., 44), so the next IR press sent values far from 100 (e.g., 43/45). The patch interpreted `abs(45 - 100) = 55` as max acceleration going the wrong direction → massive volume jump.
- **Fix**: In `updateSqueezeVol`, skip `handleVolSet` for fixed-output players. The poll still updates `curVolume` (so the patch knows the real Marantz volume), but the Touch stays pinned at 100.

**Bug 2 — Trailing `\r` in Denon volume values:**
- `DenonAvpComms.pm` line 1355: `substr($buf,2,3)` extracts 3 chars from the Marantz response. For 2-digit volumes like `"MV59\r"`, this gives `"59\r"` (trailing carriage return). For 3-digit like `"MV075\r"`, it correctly gives `"075"` (the `\r` is at position 5, not included).
- The trailing `\r` made `length("59\r") = 3`, causing the patch to take the 3-digit path: `int("59") = 59` → treated as 5.9dB instead of 59.0dB → volume jumped from 59.0 to 6.5dB on next step.
- **Fix**: Strip non-digit characters from `curVolume` before the length check: `$curRaw =~ s/[^0-9]//g;`

#### v5.5: + web UI slider/buttons fix + slider reflects Denon volume

Four issues fixed with LMS web interface interaction:

**Bug 1 — Web UI slider drops volume on every click:**
- The Touch IR intercept checked `$volAdjust != 100`, catching ALL volume values including web UI slider positions (e.g., 50). Any slider value < 100 → `direction = -1` (always down), `accel = abs(50 - 100) = 50` capped to 16 → 8dB drop per click regardless of slider position.
- **Fix**: Restrict Touch IR intercept to range 84–116 only (`$volAdjust >= 84 && $volAdjust <= 116 && $volAdjust != 100`). Values outside this range (web UI slider 0–83) fall through to the original sqrt curve handler in `handleVolChanges`, which correctly maps slider position to Denon volume.

**Bug 2 — Web UI +/- buttons hit sqrt curve at near-100:**
- LMS web UI volume buttons send incremental commands (`"+2"`, `"-2"`). In Perl, `"+2" >= 84` evaluates as `2 >= 84` → false, so these bypassed the Touch IR block and fell through to the sqrt curve handler. With the player pinned at 100, `calculateAvrVolume(100)` → near-max Denon volume. Volume up = max, volume down mapped 98 to some high-ish value (e.g., 58.5dB).
- **Fix**: Convert incremental values to absolute before the range check: `$volAdjust = 100 + $volAdjust` (Perl auto-converts `"+2"` to numeric 2). So `"+2"` → 102, `"-2"` → 98 — both in the 84–116 Touch IR range, correctly handled as 0.5dB steps.

**Bug 3 — Slider stuck at 100%:**
- After every Touch IR command, `handleVolSet($client, 100, 1)` reset the LMS player volume to 100. The web UI slider reflects the player volume, so it was permanently stuck at ~100%.
- **Fix**: Instead of resetting to 100, set to `calculateSBVolume($client, $curVolume)` — the reverse-mapped Denon volume. Slider now shows the real receiver volume percentage (maxVol = 100%).
- **Key discovery**: Touch firmware keeps its internal volume pinned at 100 regardless of what the server reports. IR always sends values centered on 100 (99/101 + acceleration), so changing the server-side volume doesn't break Touch IR stepping.

**Bug 4 — Volume poll didn't update slider for fixed-output players:**
- v5.4 had deliberately skipped `handleVolSet` in `updateSqueezeVol` for fixed-output players to keep them pinned at 100. This meant the 5-second volume poll couldn't update the slider to reflect Marantz knob/remote changes.
- **Fix**: Re-enable `handleVolSet` in `updateSqueezeVol` for all players. Since we now set the mapped Denon value (not 100), this correctly syncs the slider with the Marantz every poll cycle without breaking Touch IR.

## Code Location

Patch is in `commandCallback()` in `Plugin.pm`, inserted before the `volAdjust == 100` firmware bug guard. The patch:

1. Converts incremental +/- values to absolute (centered on 100)
2. Detects fixed-output player + volume in Touch IR range (84–116, not 100)
3. Throttles (200ms gap)
4. Calculates direction and acceleration from Touch val
5. Normalizes curVolume to tenths-of-dB
6. Steps with acceleration, snaps to grid, clamps to range
7. Converts to Denon protocol format and sends directly via `SendNetAvpVol`
8. Sets player volume to `calculateSBVolume(curDenonVol)` for slider reflection
9. Returns (no fall-through to existing absolute/incremental paths)

Also fixed `int()` bug in existing `handleVolChanges()` function (line ~2613) — same format-stripping issue affected the SB3/web UI path at low volumes.

## Deployment

```bash
# Plugin location in Docker LMS on MiisammSyno
/volume3/docker/lms/config/cache/InstalledPlugins/Plugins/DenonAvpControl/Plugin.pm

# Backups on server
Plugin.pm.bak   # original unpatched
Plugin.pm.v42   # v4.2 (last sqrt-based version)
Plugin.pm.v51   # v5.1 (direct stepping, no throttle)
Plugin.pm.v52   # v5.2 (+ throttle)
Plugin.pm.v53   # v5.3 (+ acceleration + grid snap)
Plugin.pm.v54   # v5.4 (+ poll sync fix + trailing CR fix)
Plugin.pm.v55   # v5.5 (+ web UI slider/buttons fix + slider reflects Denon vol)
```

## Fork

https://github.com/Miisamm/denonavpcontrol — branch `fix/touch-volume-acceleration`
