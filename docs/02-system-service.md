# The system service

Notes from a local jadx decompilation of `NtThirdParty.apk` (`com.nothing.thirdparty`,
version 13.01.01). This is the process on the phone that receives an app's frames over binder
and pushes them to the hardware.

The build is obfuscated with R8/ProGuard, so the class and method names are short and
meaningless (`f`, `ugf`, `tbz`, …). I recovered what each one does by following behaviour, not
by trusting names, and the name-to-method mapping is in [03](03-device-reference.md).

A note on prior work before the details: rec0de's [glyph-api](https://github.com/rec0de/glyph-api)
already documented most of this — the binding, the session IDs, the remote-config keys, the
signing-certificate check. The one place I land somewhere different is on whether that
certificate check is actually wired up, and I think that's a build difference. I've called it
out below rather than papering over it.

## Registration isn't enforced on this build

Every register entry point ends up returning success as long as the caller's package name
resolves, and the API key is never checked. In the binder stub, `registerSDK(key, device)`
forwards to the auth controller without ever looking at `key`:

```java
// registerSDK(key, device) — the key argument is ignored
if (device != null) return auth.kia(callerPkg, callerUid, device);
// registerMatrixSDK(device)
return auth.pbm(callerPkg, callerUid, device);
```

And both `kia` and `pbm` in the controller authorise on a single condition:

```java
if (callerPkg != null) { markAuthorized(callerPkg, uid); return true; }
return false;
```

So the `NothingKey` value an app puts in its manifest gets read by the SDK, sent across, and
discarded. This isn't a Matrix-specific shortcut the way it's sometimes described — none of the
register paths validate a key here.

## Two security mechanisms that look inert here

The utility class `f` contains exactly the pieces you'd expect to be the gatekeeper, and they're
real code:

- a SHA-1 hash of the caller's signing certificate, checked against a hardcoded 15-hex-character
  (60-bit) prefix, matched anywhere in the string;
- getters and setters for a `developer_end_time` preference, plus a reader for the global
  `nt_glyph_interface_debug_enable` setting.

The thing is, in this build I couldn't find a single call site for any of them inside the Nothing
packages. The only helpers from `f` that the service actually calls are the ones that fetch the
caller's package name and UID. So on 13.01.01 the certificate check appears to do nothing, and
there's no "expires after N hours" logic that I could find — there's no 48-hour constant or any
time-comparison wired to that preference anywhere in the APK.

rec0de documents the certificate check as active (and noted the partial-hash weakness, which is
a genuinely nice find). The most likely explanation for the difference is simply that their
analysis was an earlier build where it was still hooked up. Pin the version when you cite either
of us.

## How frames reach the hardware

The service is built on Android's standard Lights HAL. It resolves a `Light` by ID, opens a
`LightsManager` session, and pushes frames with `requestLights(... new LightState.Builder()
.setFrameColors(...))`. The session IDs it uses:

- **110** — the Glyph Composer's exclusive session
- **115** — third-party Glyph frames
- **121** — the hardware Glyph Matrix
- **123** — third-party "app matrix"

The Composer (`com.nothing.glyph.composer`) is special-cased throughout: it gets light 110 where
everyone else gets 115.

## Lifecycle and safety nets

An `IProcessObserver.onProcessDied` callback resets light 115 to off when a registered Glyph app
dies, so a crashed app doesn't leave the LEDs stuck on. Separately, a `ContentObserver` watches
the global `led_effect_enable` setting and closes every open light session the moment it goes to
zero — the system-wide Glyph off switch.

There's a hardcoded package list (`com.nothing.hearthstone`, `com.nothing.camera`, and the
`com.nothinglondon.*` first-party widgets — sandtimer, flygame, toys, lunarcycle, compass). On
this build it only gates `setGlyphMatrixTimeout`, which is narrower than it's sometimes described
as.

## Sensors, touch, lockscreen

The matrix controller (`eoj`) hooks two OEM sensor types — `65538`, which fires its action at a
value of `2.0f`, and `65541`, which fires at `1.0f` — and listens for a `"nothing.touch.event"`
broadcast carrying press, long-press, action-down and action-up. It also registers a
`KeyguardManager.KeyguardLockedStateListener` to react to lock and unlock. The auth controller
additionally holds a Firebase Realtime Database URL and a `foreground_app` observer, which lines
up with the remote-config key model that rec0de describes.

## Can a normal app drive the LEDs directly?

No. The system app uses `LightState.Builder().setFrameColors(int[])`, which is a real method — but
it's a hidden OEM API that isn't in the public Android SDK, so an app can't cleanly compile
against it. And opening a session on these lights needs `CONTROL_DEVICE_LIGHTS`, a
`signature|privileged` permission granted only to system/OEM-signed apps. A sideloaded app has
neither. The pattern is genuine; the door is locked.
