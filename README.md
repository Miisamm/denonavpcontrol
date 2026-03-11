# denonavpcontrol
> A plugin for the Lyrion Music Server to control a Denon or Marantz Audio/Video Receiver.

# Squeezebox Touch Volume Fix (Fork)

This fork fixes the Squeezebox Touch IR remote volume control maxing out Denon/Marantz receivers. With fixed output level enabled (required for this plugin), the Touch sends absolute volume values near 100, which the plugin's sqrt curve maps to near-max receiver volume. This fix bypasses the curve and sends direct 0.5dB steps to the AVR instead.

Tested with Marantz Cinema 70s, Squeezebox Touch, and SB3 Classic. SB3 and other players are unaffected by this patch.

Big Thanks to @SamInPgh for the original plugin!

https://github.com/SamInPgh/denonavpcontrol

- Miisamm

## Changelog

### v5.5
- **Web UI slider fix**: Slider clicks no longer drop volume — only Touch IR range (84-116) is intercepted, other values fall through to the original sqrt curve handler
- **Web UI +/- buttons fix**: Incremental volume commands now step 0.5dB like IR instead of hitting the sqrt curve at near-100 values
- **Slider reflects actual Denon volume**: LMS web UI slider shows the real receiver volume instead of being stuck at 100%
- **Poll sync for fixed-output players**: Volume polling now updates the slider for fixed-output players too, syncing with Marantz knob/remote changes

### v5.4
- **Poll sync**: Volume polling updates internal state for Marantz knob/remote sync
- **Trailing CR fix**: Strip `\r` from Denon volume responses before parsing — fixes wrong format detection for 2-digit values

### v5.3
- **IR acceleration**: Holding the remote ramps up step size (0.5dB → 1.0dB → 2.0dB → 4.0dB → 8.0dB), matching the Touch's built-in IR acceleration
- **Grid snapping**: Volume snapped to valid 0.5dB boundaries after acceleration math

### v5.2
- **Throttle**: 200ms minimum gap between Denon commands (max 5/sec) to prevent overwhelming the receiver

### v5.1
- **Initial fix**: Bypass sqrt curve for Touch with fixed output, step Denon volume directly in 0.5dB increments
- **Fixed `int()` bug** in existing `handleVolChanges()` — was stripping Denon format via `int("085") = 85`

### v4 (sqrt curve, abandoned)
- Format-aware Denon value storage (strings instead of `int()` which stripped "085" → 85)
- Loop to step SB until Denon value actually changes (handles sqrt flat spots)
- Abandoned: sqrt curve gives non-uniform steps (6dB jump at bottom, 0.5dB at top)

### v3 (intercept & convert, abandoned)
- First working approach — intercept near-100 values, convert to incremental AVR steps via sqrt curve
- Abandoned: `int()` format bug + sqrt flat spots + broken numeric Denon format comparison

### v1–v2 (rate-limiting, failed)
- Attempted to rate-limit volume commands by timestamp
- Failed: Touch always sends 99/101 (delta 0–2), never exceeding any rate limit

See [CHANGELOG-touch-volume-fix.md](CHANGELOG-touch-volume-fix.md) for full technical details.

---

## Introduction
This plugin will turn on and off a networked Denon or Marantz amplifier/receiver when an LMS player is turned on/off or a song is played. The plugin can also be configured to invoke one of the AVR's Quick Select modes or set the input source after powering it on, and will set the initial volume of the player to match the amplifier volume. An optional delay value can be specified before issuing commands after the AVR is powered on, as well as the maximum volume level that the amplifier should be set to when controlling the player. The user can also modify the AVR audio settings during playback when using the iPeng (iOS), Squeezer (Android) or Squeeze Ctrl (Android) client apps or the Material Skin LMS plugin interface.

The plugin uses the Denon/Marantz serial protocol over a wireless or wired network and therefore a network connection between the Lyrion server and the AVP/AVR must be available.

The plugin has been tested with numerous Denon and Marantz AVR's, and with the Squeezebox Receiver, Touch and software-based players such as Squeezelite. It should work with any Denon or Marantz AVR that supports the Denon/Marantz serial protocol over a network. The plugin has also been tested with Apple iOS devices (Touch, iPhone and iPad) using the iPeng application, Android devices using the Squeezer and Squeeze Ctrl applications, and the Material Skin plugin on any devices it supports.

## Details
  * Turns the AVR on when the user turns on a Lyrion player or starts playing a song
  * Puts the AVR in standby when the user turns the player off
  * Changes the volume of the AVR when the user changes the player volume
  * Sets the player volume to the default setting of the AVR when turned on
  * Optionally sets one of the Quick Select (Denon) or Smart Select (Marantz) modes, or selects an AVR Input Source when turned on
  * The user specifies the maximum volume the amplifier can be set to
  * The user can pick an optional delay time after powering the AVR on before playback begins
  * The user can select between Main, Zone 2, Zone 3 and Zone 4
  * Optionally synchronize the Lyrion player's volume to the AVR volume at track changes
  * Optionally control AVR audio settings using LMS menus available under the iPeng, Squeezer, Squeeze Ctrl or Material Skin applications during playback
  
# Installation
See the [Installation Instructions](https://github.com/aesculus/denonavpcontrol/wiki/Installation-Instructions)

# How to Use
See [How to Use](https://github.com/aesculus/denonavpcontrol/wiki/How-to-Use)
## License

The MIT License

Copyright (c) 2008-2024, Christopher Couper and contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

```
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
```
