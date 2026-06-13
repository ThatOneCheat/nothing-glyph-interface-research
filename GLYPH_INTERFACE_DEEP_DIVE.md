# Nothing Glyph Interface — A Reverse-Engineering Deep Dive

> An independent, ground-up analysis of how Nothing's **Glyph Interface** and **Glyph Matrix SDK 2.0** actually work. It walks through the client SDK, the on-device system service, the binder protocol, the auth model (now mostly gutted), the rendering engine, the Glyph Toys framework, and a handful of things Nothing never bothered to document.
>
> **Sources analysed**
> - Official SDK: `glyph-matrix-sdk-2.0.aar` (package `com.nothing.ketchum` + `com.nothing.thirdparty`), from [Nothing-Developer-Programme/Glyph-Developer-Kit](https://github.com/Nothing-Developer-Programme/Glyph-Developer-Kit).
> - On-device system service: `com.nothing.thirdparty` **v13.01.01** (`versionCode 130101`, compiled against Android 16 / SDK 36), pulled from a physical Nothing Phone (3) over ADB.
> - Cross-referenced against the community write-up [rec0de/glyph-api](https://github.com/rec0de/glyph-api). See [§12](#12-what-changed-since-the-community-write-up) for a full account of what has changed since it was written.
>
> **Method:** I decompiled the `.aar` and the system APK with jadx. Every constant, transaction code, and device string quoted below was read directly out of the decompiled bytecode and, where possible, re-verified programmatically. This is original analysis rather than a restatement of the community repo; that repo described a materially older build, and I give a side-by-side comparison at the end.

---

## Table of contents

1. [TL;DR](#1-tldr)
2. [Architecture at a glance](#2-architecture-at-a-glance)
3. [The binder protocol (`IGlyphService`)](#3-the-binder-protocol-iglyphservice)
4. [The on-device service (`com.nothing.thirdparty.GlyphService`)](#4-the-on-device-service-comnothingthirdpartyglyphservice)
5. [The authorization model — and why it's now wide open](#5-the-authorization-model--and-why-its-now-wide-open)
6. [Device map, channel counts & the undocumented models](#6-device-map-channel-counts--the-undocumented-models)
7. [Brightness, color encoding & per-device channel remapping](#7-brightness-color-encoding--per-device-channel-remapping)
8. [The legacy "Glyph strip" API (`GlyphManager`)](#8-the-legacy-glyph-strip-api-glyphmanager)
9. [The Glyph Matrix engine (`GlyphMatrixManager` + rendering)](#9-the-glyph-matrix-engine-glyphmatrixmanager--rendering)
10. [The embedded "NDot" matrix font](#10-the-embedded-ndot-matrix-font)
11. [The Glyph Toys framework & network comms](#11-the-glyph-toys-framework--network-comms)
12. [What changed since the community write-up](#12-what-changed-since-the-community-write-up)
13. [Limits, gotchas & a minimal client](#13-limits-gotchas--a-minimal-client)
14. [Appendix: constant reference](#14-appendix-constant-reference)

---

## 1. TL;DR

The Glyph hardware is driven through Android's system Lights HAL (`android.hardware.lights.LightsManager`). That API needs `CONTROL_DEVICE_LIGHTS`, which is locked to system apps, so Nothing ships a privileged broker to stand in front of it: `com.nothing.thirdparty` / `GlyphService`, running as `android.uid.system`. It relays your color arrays to the HAL, and your app talks to it over AIDL/Binder.

The SDK you integrate (`com.nothing.ketchum.*`) is a thin client wrapped around a 10-method binder interface. Everything interesting happens on the server side.

On the current Android 16 build, the historical gatekeeping is gone. There used to be an API key, a signing-certificate fingerprint check, and an over-the-air JSON "auth map." None of that is enforced anymore: `register()` returns `true` for any caller. The only thing standing between an app and the Glyphs is a permission, `com.nothing.ketchum.permission.ENABLE`, declared at `normal` protection level, which Android auto-grants to anything that asks. Nothing says as much in its own docs: *"The API key restriction has been removed starting from Android B (Android 16)."*

The Glyph Matrix (the round dot-matrix on Phone 3) is a small graphics stack in its own right. It does three-layer compositing, bitmap scale/rotate/threshold, a scrolling text marquee, a circular progress-arc renderer, and ships a built-in 5×6 dot font.

Glyph Toys are independent apps. They expose a service that the system binds and feeds button, AOD, and notification events, and they paint the matrix through a dedicated "app matrix" light channel.

A few undocumented odds and ends turn up along the way: a 13×13 matrix device (`A069P`), four server-only device codes (`A012`, `A013`, `A143`, `A215`), a hardcoded allowlist of first-party Toy apps, a dead signing-key "backdoor" substring left over from the old auth path, and a Firebase Realtime Database endpoint.

---

## 2. Architecture at a glance

```
┌──────────────────────────┐         bindService(                       ┌─────────────────────────────────────┐
│  Your app                │         action = com.nothing.thirdparty     │  com.nothing.thirdparty  (GlyphService) │
│  ──────────────────────  │           .bind_glyphservice)               │  runs as android.uid.system (uid 1000)  │
│  com.nothing.ketchum     │  ───────────────────────────────────────▶  │                                       │
│    GlyphManager          │                                             │   IGlyphService.Stub (10 transactions)│
│    GlyphMatrixManager    │  ◀───────  IGlyphService (AIDL/Binder) ────▶ │     register* / openSession           │
│    GlyphFrame / Matrix*  │            int[] color frames               │     setFrameColors / setMatrixColors  │
└──────────────────────────┘                                             │     setAppMatrixColors / timeout      │
        ▲                                                                 │              │                        │
        │ requires <uses-permission                                       │              ▼                        │
        │  com.nothing.ketchum.permission.ENABLE/>  (normal)              │   android.hardware.lights.LightsManager│
        │                                                                 │   LightState.setFrameColors(int[])    │
        │                                                                 └──────────────┬────────────────────────┘
        │                                                                                ▼
        │                                                                        Glyph LEDs / dot matrix
        │
   Glyph Toys (separate apps) ── bound by the system, fed button/AOD events ──▶ paint via "app matrix" channel (light id 123)
```

A few things about this picture matter for everything that follows.

You never touch the LEDs directly. You hand the broker an `int[]` of brightness values. It validates very little, remaps the array for the physical device, and forwards it to the Lights HAL.

The broker is a single persistent system-uid service (`android:persistent="true"`, `directBootAware="true"`). It owns all Glyph state and arbitrates between callers using a small set of "light sessions."

And the SDK is disposable. The contract is a stable AIDL interface, so you can talk to the service with a hand-written binder proxy and skip the `.aar` entirely (see [§13](#13-limits-gotchas--a-minimal-client)).

---

## 3. The binder protocol (`IGlyphService`)

The interface descriptor is the string `com.nothing.thirdparty.IGlyphService`. In SDK 1.x and the community era it had four methods. The current interface has ten:

| TX # | AIDL method | Args | Returns | Purpose |
|-----:|-------------|------|---------|---------|
| 1 | `setFrameColors` | `int[]` | void | Set legacy Glyph **strip** brightness array |
| 2 | `openSession` | — | void | Open a lights session for this caller |
| 3 | `closeSession` | — | void | Close the caller's session |
| 4 | `register` | `String` | boolean | **Legacy** register (old API-key path) |
| 5 | `registerSDK` | `String apiKey, String device` | boolean | Current `GlyphManager` registration |
| 6 | `registerMatrixSDK` | `String device` | boolean | Current `GlyphMatrixManager` registration |
| 7 | `setMatrixColors` | `int[]` | void | Paint the **Glyph Matrix** (dot grid) |
| 8 | `setGlyphMatrixTimeout` | `boolean` | void | Toggle matrix idle-timeout (restricted, see §11) |
| 9 | `setAppMatrixColors` | `int[]` | void | Paint the **app/Toy** matrix layer |
| 10 | `closeAppMatrix` | — | void | Tear down the app/Toy matrix layer |

Transactions 5 through 10 are all new since the community analysis. The parcel formats are plain AIDL: write the interface token, then the args (`writeIntArray`, `writeString`, `writeInt` for the boolean). Replies are `writeNoException()` plus an optional boolean.

### The obfuscation map (server side)

The shipped service is R8/Proguard-obfuscated; each class carries a `go/retraceme <hash>` marker. The `IGlyphService.Stub` method names are scrambled, but the `onTransact` switch lets you pin each obfuscated name to its public AIDL meaning byte-for-byte:

| TX # | Obfuscated server method | = public AIDL method |
|-----:|--------------------------|----------------------|
| 1 | `tbz(int[])` | `setFrameColors` |
| 2 | `lgi()` | `openSession` |
| 3 | `ziu()` | `closeSession` |
| 4 | `yum(String)` | `register` |
| 5 | `nhp(String,String)` | `registerSDK` |
| 6 | `kbv(String)` | `registerMatrixSDK` |
| 7 | `rwe(int[])` | `setMatrixColors` |
| 8 | `eur(boolean)` | `setGlyphMatrixTimeout` |
| 9 | `qae(int[])` | `setAppMatrixColors` |
| 10 | `tez()` | `closeAppMatrix` |

The bind target is fixed:

```
package   = com.nothing.thirdparty
action    = com.nothing.thirdparty.bind_glyphservice
component = com.nothing.thirdparty/.GlyphService
```

---

## 4. The on-device service (`com.nothing.thirdparty.GlyphService`)

The manifest tells you most of the story before you read a line of code:

```xml
<manifest package="com.nothing.thirdparty"
          android:sharedUserId="android.uid.system"
          android:versionCode="130101" android:versionName="13.01.01"
          android:compileSdkVersion="36">          <!-- Android 16 -->
  <uses-permission android:name="android.permission.CONTROL_DEVICE_LIGHTS"/>
  <uses-permission android:name="android.permission.INTERNET"/>
  <uses-permission android:name="android.permission.WRITE_SETTINGS"/>
  <uses-permission android:name="android.permission.INTERACT_ACROSS_USERS_FULL"/>
  <permission     android:name="com.nothing.ketchum.permission.ENABLE"/>   <!-- no protectionLevel ⇒ "normal" -->

  <application android:persistent="true" android:directBootAware="true"
               android:name="...matrix.toys.GlyphToysApplication">
    <service android:name=".GlyphService"
             android:permission="com.nothing.ketchum.permission.ENABLE"
             android:exported="true">
      <intent-filter><action android:name="com.nothing.thirdparty.bind_glyphservice"/></intent-filter>
    </service>
    ...
  </application>
</manifest>
```

`sharedUserId="android.uid.system"` is the load-bearing line. The broker shares the system UID (1000), and that is what lets it call the otherwise-forbidden `CONTROL_DEVICE_LIGHTS` Lights HAL. It's signed with the platform key, so you can't replicate or repackage it.

The exported service is guarded by `com.nothing.ketchum.permission.ENABLE`, declared with no `protectionLevel`. Android defaults that to `normal`, which means it's granted automatically at install time to any app that lists it, with no user prompt and no signature match. That one detail is the entire access-control story (see [§5](#5-the-authorization-model--and-why-its-now-wide-open)).

### How a frame reaches the LEDs

`GlyphService.onCreate()` grabs the system `LightsManager`, builds a per-device color adapter (`ocs`), registers an `IProcessObserver` so it can reset the LEDs when a client process dies, and watches the global setting `led_effect_enable`. When you call `setFrameColors` or `setMatrixColors`, the relevant handler:

1. resolves your package from `Binder.getCallingUid()`,
2. looks up (or lazily opens) a `LightsManager.LightsSession` keyed by a light id,
3. runs your `int[]` through the device adapter (`ocs.snw`) to reorder channels if you target a different model than the physical phone,
4. issues `session.requestLights(new LightsRequest.Builder().addLight(light, new LightState.Builder().setFrameColors(colors).build()).build())`.

### Light-id scheme (the "session map")

The service multiplexes callers onto distinct hardware light ids:

| Light id | Used for |
|---------:|----------|
| `110` | the first-party **Glyph Composer** (`com.nothing.glyph.composer`) — given its own bucket |
| `115` | **normal third-party apps** (legacy strip frames) |
| `121` | the **Glyph Matrix** when an "always-on/extended" mode is active |
| `123` | the **app / Toy matrix layer** (`setAppMatrixColors` / `closeAppMatrix`) |

`led_effect_enable` (a `Settings.Global` flag) is a global kill switch. When it flips to 0 the service closes every session and blanks the LEDs, and matrix writes are silently dropped (`if (this.luc) …`).

---

## 5. The authorization model — and why it's now wide open

More has changed here than anywhere else since the community write-up, so it's worth being precise.

### What the *old* model did (community era)

The historical `AuthController.register(pkg, apiKey, uid)`:
- accepted system UID (1000) unconditionally;
- otherwise looked the package up in an `mAuthMap` delivered over the air as JSON (`RemoteConfigController`);
- and returned true only if both the supplied API key and a SHA-1 of the caller's signing certificate matched the values in that map.

There was also a special case that let `com.nothing.glyph.composer` skip registration entirely, as long as its signing-cert fingerprint *contained* the substring `95E1F157FE98518`. That's a weak check, since it only constrains 60 of the certificate's 160 bits.

### What the *current* (Android-16) model does

The current `AuthController` (obfuscated class `com.nothing.thirdparty.ugf`) registration path is, in full:

```java
public boolean pbm(String pkg, int uid, String targetDevice) {   // registerMatrixSDK path
    if (pkg != null) {
        tay(pkg, uid, 2);                 // addAlreadyAuth(pkg, uid, type=2)  -> authorized = true
        this.tay.get(pkg).jye(targetDevice);   // remember the target device
        return true;                      // <-- always succeeds for any non-null package
    }
    return false;
}
public boolean kia(String pkg, int uid, String targetDevice) {    // registerSDK path
    if (pkg != null) { tay(pkg, uid, 2); this.tay.get(pkg).jye(targetDevice); return true; }
    return false;
}
```

All three binder entry points collapse onto this:

- `registerSDK(apiKey, device)` calls `kia(callingPkg, uid, device)`. The `apiKey` argument is read off the parcel and then never used.
- `registerMatrixSDK(device)` calls `pbm(callingPkg, uid, device)`.
- Legacy `register(device)` calls `kia(callingPkg, uid, "A065")`, registering you against a constant model.

Nowhere in the registration path is there an API-key comparison, a certificate-fingerprint check, or a remote auth-map lookup. Registration succeeds for any caller with a non-null package name. That lines up with Nothing's own note in the developer kit: *"The API key restriction has been removed starting from Android B (Android 16). You no longer need to apply for an API key."*

There's a second point that matters just as much. `openSession`, `setFrameColors`, and `setMatrixColors` don't re-check authorization or foreground state inside the binder methods. The "must be in the foreground" enforcement that the community repo documented isn't present in these paths on this build. The service still tracks the current foreground app through a `com.nothing.kafka.EventObserver` on `foreground_app`, and it still resets a client's LEDs when its process dies, but the decompiled code doesn't reject background `setFrameColors` calls.

### The leftover "backdoor" is now dead code

The magic fingerprint substring still physically exists:

```java
// com.nothing.thirdparty.f  (formerly "Utils")
private static final String ncb = "95E1F157FE98518";
public static boolean snw(Context c, String pkg) { return ziu(c, pkg).contains(ncb); }  // checkFingerprint
```

But nothing calls `snw()` anymore. Grep across the whole decompiled service and you find the constant only at its definition. The same goes for the developer-mode helpers in that class (`nt_glyph_interface_debug_enable`, the `developer_end_time` 48-hour timer). They're vestigial remnants of the old auth path. The live 48-hour debug expiry is now handled by a separate `com.nothing.thirdparty.developer.DeveloperJobService` (`BIND_JOB_SERVICE`).

### Net effect

On a current Phone (3), an ordinary third-party app can drive the Glyphs with no key and no signature. Declare `com.nothing.ketchum.permission.ENABLE`, bind the service, call `register*()` (which returns true), `openSession()`, then `setFrameColors()` / `setMatrixColors()`. The brute-force-the-signing-key idea from the community repo is no longer necessary, because the door it tried to pick has been removed.

> **Responsible-use note.** This is a factual description of the shipped access-control design, written for interoperability research, not an exploit. Nothing here lets you steal another app's credentials, escape the app sandbox, or gain privilege. The broker still runs as system, and you still only get to push brightness arrays to lights the OS already lets the broker control. Don't ship background LED spam; Nothing can tighten this again, and historically did.

---

## 6. Device map, channel counts & the undocumented models

Each Nothing phone is identified by `Build.MODEL`. The SDK and the service both carry a model table, and combining them gives the complete picture, including codes the public README never lists.

| Model code | Marketing name | Glyph type | Channels / matrix |
|------------|----------------|-----------|-------------------|
| `A063` | Phone (1) | strips | 15 |
| `A065` / `AIN065` | Phone (2) | strips | 33 |
| `A142` | Phone (2a) | strips | 26 |
| `A142P` | Phone (2a) Plus | strips | 26 |
| `A059` | Phone (3a) / (3a) Pro | strips | 36 |
| `A069` | Phone (4a)\* | strips | 6 |
| **`A024`** | **Phone (3)** | **dot matrix** | **25 × 25 = 625** |
| **`A069P`** | (4a-class, matrix variant)\* | **dot matrix** | **13 × 13 = 169** |

\* `A069` / `A069P` appear in the SDK 2.0 client (`DEVICE_25111` / `DEVICE_25111p`) but not in the README's documented tables.

Four more codes exist only inside the on-device service (`com.nothing.thirdparty.qfd`), with no SDK or README counterpart. They're almost certainly other or forthcoming Nothing and CMF hardware the broker is being prepped for:

```
A012, A013, A143, A215
```

The matrix length is the key new constant. `Common.getDeviceMatrixLength()` returns 25 for `A024` (Phone 3) and 13 for `A069P`, and everything in the matrix engine is sized `length × length`.

### Per-model channel index maps

For the strip phones, each LED "zone" has a fixed array index. The full maps live in `com.nothing.ketchum.Glyph.Code_2xxxx`. The highlights:

- **Phone (1) — 15 ch:** `A1=0, B1=1, C1..C4=2..5, E1=6, D1_1..D1_8 = 7..14` (D1_1 bottom → D1_8 top).
- **Phone (2) — 33 ch:** `A1=0, A2=1, B1=2, C1_1..C1_16 = 3..18, C2..C6 = 19..23, E1=24, D1_1..D1_8 = 25..32`.
- **Phone (2a/2a+) — 26 ch:** `C_1..C_24 = 0..23, B=24, A=25`.
- **Phone (3a) — 36 ch:** `C_1..C_20 = 0..19, A_1..A_11 = 20..30, B_1..B_5 = 31..35`.
- **Phone (4a) — 6 ch:** `A_1..A_6 = 0..5`.

The Glyph Matrix devices don't use these zone maps. They take a flat `length×length` array indexed row-major, `index = y*length + x`.

---

## 7. Brightness, color encoding & per-device channel remapping

There are two different brightness scales, and the gap between them trips people up:

| Surface | Range | Notes |
|---------|-------|-------|
| Legacy strip API (`GlyphFrame`) | **0 – 4096** | `GlyphFrame.DEFAULT_LIGHT = 4000`; manager clamps to `DEFAULT_MAX_LIGHT = 4096`, with a `DEFAULT_MIN_LIGHT = 800` floor used by the progress renderer. Effectively 12-bit PWM. |
| Matrix public API (`GlyphMatrixObject.setBrightness`) | **0 – 255** | 8-bit, default `255`. |
| Matrix **on the wire** (`setMatrixColors`) | **0 – 4095** | The engine rescales 8-bit → 12-bit: `value12 = (value8 * 4095) / 255` (`MAX_BRIGHTNESS = 4095`, `BRIGHTNESS_MULTIPLIER = 16`). |

A "color" in this system isn't RGB. The Glyph LEDs are single-color (white), and every value in the `int[]` is just a per-zone or per-dot brightness. When the engine ingests an actual bitmap it first converts to luminance (`0.299R + 0.587G + 0.114B`) and then to brightness.

### Channel remapping (`ocs` + `adapter.*`)

An app can target a model different from the phone it's running on, so the service keeps a set of remapping tables, one per device family, under `com.nothing.thirdparty.adapter`:

| Adapter class | Device | Array size |
|---------------|--------|-----------:|
| `pjy` | `A063` (Phone 1) | 15 |
| `ugf` | `A065`/`AIN065` (Phone 2) | 33 |
| `lmr` | `A142*` (Phone 2a) | 26 |
| `ztk` | `A059` (Phone 3a) | 36 |
| `bnv` | `A024` (Phone 3) | — |

`ocs.snw(int[] colors, String targetDevice)` reorders and resizes the incoming array so a frame authored for, say, a Phone (2) renders sensibly on the physical device. The active adapter is chosen at startup from `SystemProperties.get("ro.product.model")`.

---

## 8. The legacy "Glyph strip" API (`GlyphManager`)

`GlyphManager` is the client for the strip phones (1/2/2a/3a/4a). The lifecycle:

```
getInstance(ctx) → init(callback)  // binds the service
   onServiceConnected → register() / register(device)   // returns boolean
   openSession()
   ... toggle / animate / displayProgress / setFrameColors / turnOff ...
   closeSession(); unInit()
```

You build frames with `GlyphFrame.Builder`:

```java
GlyphFrame.Builder b = mGM.getGlyphFrameBuilder();
GlyphFrame f = b.buildChannel(Glyph.Code_22111.C1_4)   // light one zone
                .buildChannelA()                        // or a whole zone group
                .buildPeriod(3000).buildCycles(2).buildInterval(10)
                .build();
mGM.animate(f);
```

Three behaviours are worth flagging, because they run client-side rather than in hardware:

`animate(frame)` is a software breathing effect. The manager runs a background thread that ramps each lit channel `0 → 4096 → 0` over `period` ms in roughly 10 ms steps, for `cycles` iterations, separated by `interval`. Under the hood it's a loop calling `setFrameColors` over and over.

`displayProgress(frame, percent, reverse)` maps a 0–100 value onto a specific progress zone (D1 on Phone 1; C1 or D1 on Phone 2; the C arc on 2a/3a; the A column on 4a). It lights an integer number of segments plus a fractional last segment, with the `800` minimum-brightness floor. `displayProgressAndToggle` does the same thing and leaves the rest of your frame lit.

`register()` sends `Common.getAppKey(ctx)` (the `NothingKey` manifest meta-data) as the API key, which, per §5, the service ignores on Android 16.

---

## 9. The Glyph Matrix engine (`GlyphMatrixManager` + rendering)

This is the new subsystem for the Phone (3) round dot-matrix, and it does a lot more than set an array of pixels. The client side is a small composition and rasterization library that produces the flat `length×length` `int[]` that eventually goes to `setMatrixColors`.

### Objects, frames & layers

- **`GlyphMatrixObject`** is one drawable: either a square `Bitmap` or a text string, plus `positionX/Y`, `scale` (%), `orientation` (degrees), `brightness` (0–255), `reverse` (invert), and a `marqueeType` (`NONE`/`FORCE`/`AUTO`) with an optional range.
- **`GlyphMatrixFrame`** composites up to three objects in z-order: Top → Mid → Low. `render()` rasterizes each layer to a `length×length` array and merges them, with the topmost non-zero pixel winning (`combineArrays`).
- **`GlyphMatrixFrameWithMarquee`** is a frame that animates. A `Handler` ticks every `duration` ms, advancing scrolling text by `step` pixels and emitting updated frames through an `OnDataUpdateListener`.

### Rasterization (`GlyphMatrixUtils`)

`convertToGlyphMatrix(...)` does the heavy lifting for bitmaps:

1. require a square source bitmap (`width == height`, else `IllegalArgumentException`);
2. apply `scale` (`Matrix.postScale`), then `orientation` (`Matrix.postRotate` into an enlarged canvas);
3. sample the bitmap onto the `length×length` grid, converting each pixel to luminance and then to a 0–4095 brightness scaled by the object's `brightness`;
4. optionally apply `reverse` (invert) and a 1-bit threshold mode (anything ≥ 1024 → full on, else off) for hard dithered output;
5. `shiftArray()` applies the X/Y offset.

Text rendering (`renderText` → `getLetterConfigs`) walks the string, looks up each character's dot pattern, and blits it. The marquee variant offsets the whole run each tick.

### Two utilities worth noting

`generateMatrixProgress(size, …, icon)` draws a circular progress arc sized to the round matrix. It computes per-row spans of the inscribed circle, fills an arc proportional to a 0–100 value, dithers the partial cells in a checkerboard, and can stamp a small icon bitmap in the center. This is the radial loader you see on the Phone 3 matrix.

There are also built-in arrow sprites, `ARROW_3x2` and `ARROW_5x6`, plus morphological `dilation()` and `erosion()` helpers. In other words, the engine expects tiny, low-res art, and it includes primitives to fatten or thin strokes so they read on a sparse LED grid.

### Matrix manager lifecycle

```
GlyphMatrixManager.getInstance(ctx).init(cb);
mgr.register(Glyph.DEVICE_23112);          // → registerMatrixSDK("A024")
mgr.setMatrixFrame(frame);                 // → setMatrixColors(frame.render())
mgr.setAppMatrixFrame(frame);              // → setAppMatrixColors(...) — the Toy layer (light 123)
mgr.setGlyphMatrixTimeout(true);           // restricted; see §11
mgr.closeAppMatrix(); mgr.turnOff();
```

---

## 10. The embedded "NDot" matrix font

The matrix text renderer ships its own micro-font. Characters are stored as comma-separated binary rows. The string `"0000,0110,1001,1111,1001,1001"`, for instance, is the letter **a**, parsed into a `LetterMatrix` of lit `Point`s. There are three sources, checked in order:

1. a per-app override string resource `letter_<char>_<style>` (so an app can ship its own font and style variants via `setTextStyle`),
2. the SDK's bundled resource `letter_<char>` (61 glyphs in `res/values/values.xml`),
3. a hardcoded fallback map in `GlyphMatrixUtils` (36 glyphs).

The bundled set is 61 glyphs: 26 letters, 10 digits, and 9 specials (`colon, dash, dot, deg, exclamation, percent, plus, underscore, space`), plus 10 "tall" (7-row) variants and 6 lowercase variants. Special characters map through a small table (`'°' → "deg"`, `':' → "colon"`, `'!' → "exclamation"`, and so on). The default typeface name referenced for vector text is `NDot55All` (Nothing's "Ndot" dot font).

Here is the actual bundled font, rendered straight from the SDK resources (verification asset, regenerated from `values.xml`):

![NDot matrix font atlas](ndot_font_atlas.png)

Each glyph is 4–5 dots wide by 6 rows tall (digits and letters), with `LETTER_SPACING = 1` dot between characters. On the 25×25 Phone 3 matrix that leaves room for about 4 fixed characters per line, which is why longer strings auto-marquee.

---

## 11. The Glyph Toys framework & network comms

Glyph Toys are the headline Phone 3 feature, and they live almost entirely outside the documented SDK. The model goes like this.

A Toy is a separate app that exposes a service. The system binds it and sends `Message`s (`GlyphToy.MSG_GLYPH_TOY = 1`, payload key `"data"`) carrying lifecycle status (`"prepare"` / `"start"` / `"end"`) and events:

| Constant | Value | Meaning |
|----------|-------|---------|
| `EVENT_CHANGE` | `"change"` | user cycled to the next toy |
| `EVENT_AOD` | `"aod"` | always-on-display tick |
| `EVENT_ACTION_DOWN` | `"action_down"` | **Glyph Button** pressed |
| `EVENT_ACTION_UP` | `"action_up"` | Glyph Button released |

The toy paints by calling back into the matrix via `setAppMatrixColors` (light id 123), and the system tears it down with `closeAppMatrix`.

Toys are tracked in a content provider, `content://com.nothing.glyphtoyprovider/glyph_toy` (a SQLite table), with a sibling URI `…/active_aod_toy_name` that returns the currently selected always-on toy.

First-party toys are hardcoded. `setGlyphMatrixTimeout` is gated to a fixed allowlist:

```
com.nothing.hearthstone, com.nothing.camera,
com.nothinglondon.sandtimer, com.nothinglondon.flygame,
com.nothinglondon.toys, com.nothinglondon.lunarcycle, com.nothinglondon.compass
```

(`com.nothinglondon.*` are Nothing's own published toy apps.) `GlyphTriggeredState` further wires trigger types `100` and `200` to `com.nothing.hearthstone.battery.glyph.GlyphMatrixBatteryService`, the battery toy.

For AOD and "Flip to Glyph," the UI strings confirm an always-on toy mode shown at reduced brightness, toggled by flipping the phone face-down, plus a configurable idle timeout. That last one is what `setGlyphMatrixTimeout` controls.

### How it talks to the network

The service holds `INTERNET` + `ACCESS_NETWORK_STATE` and has two outward channels.

First, a Toys catalog / install API (Retrofit). The interface is:

```
@GET("toy/installation/{toyId}")  ApiResult<ToyData> getToys(toyId, queryMap)
```

`ToyData → ToyDownload` carries `id, toyName, creator, description, version, deviceType, deploymentMethod, animationUri, avatarUri, playStoreLink, status`. So a toy is discovered as metadata, including a Play Store link and a `creator` field (note the UI string *"(Community developed)"*), then installed. There's a deep link for this too: `glyphtoy://com.nothing.thirdparty/…` (a `BROWSABLE` `VIEW` intent on `ToysTransparentActivity`). The base URL isn't a plaintext constant in this class; it's supplied by Nothing's shared `com.nothing.net` networking layer and remote config.

Second, a Firebase Realtime Database endpoint, hardcoded in the AuthController:

```
https://nothingdevice-aafad-default-rtdb.asia-southeast1.firebasedatabase.app/
```

The current registration path no longer reads from it, so it now looks like a legacy or secondary channel. It's also exactly the kind of over-the-air config store the community write-up predicted Nothing would use to distribute the old auth map.

### Other components in the APK

`DeveloperJobService` (the live 48-hour debug-mode timer), `BootReceiver` (`NT_BOOT_COMPLETED`), `ResourceContentProvider` + `GlyphToyProvider` (exported providers), and a set of preview, manager, and settings activities (`ToysPreviewActivity`, `ToysManagerActivity`, `AodToySelectActivity`, `ToyTimeoutSettingsActivity`). The whole thing is a normal AndroidX/Kotlin app that happens to run as system.

---

## 12. What changed since the community write-up

The [rec0de/glyph-api](https://github.com/rec0de/glyph-api) write-up was a strong piece of work, but it described a 2022-era Phone (2) build. The table below is what this analysis of the current Android-16 build adds or corrects, so the research stands on its own rather than restating theirs.

| Topic | Community write-up (old build) | This analysis (current build) |
|-------|-------------------------------|-------------------------------|
| Interface size | 4 methods (`setFrameColors`, `open/closeSession`, `register`) | **10 methods** — adds `registerSDK`, `registerMatrixSDK`, `setMatrixColors`, `setGlyphMatrixTimeout`, `setAppMatrixColors`, `closeAppMatrix` |
| Auth | API key **+** cert fingerprint **+** OTA JSON auth map, all enforced | **No enforcement** — `register*()` returns true for any package; key/fingerprint/map checks gone |
| Foreground rule | `setFrameColors` rejected background callers | not enforced in the binder methods in this build |
| Composer "backdoor" | `95E1F157FE98518` fingerprint check actively used | constant still present but **dead code** — no caller |
| Glyph Matrix | not covered (didn't exist) | full engine documented: compositing, marquee, progress arc, font |
| Glyph Toys | not covered | full framework: events, button, AOD, content provider, downloadable toys |
| Light routing | single session | **four light ids** (110/115/121/123) |
| Networking | speculated (firebase, cert pinning) | confirmed Firebase RTDB URL + a Retrofit Toys API |
| Device coverage | Phone 1/2 | adds 2a, 2a+, 3a, 3, 4a-class + **undocumented `A012/A013/A143/A215`, `A069P` 13×13** |
| Brightness | "probably 33 values" | precise: strips 0–4096, matrix 0–255 public → 0–4095 wire |

The single biggest takeaway: the elaborate lock the community repo spent its time trying to pick has been removed by Nothing. The interesting frontier is no longer "how do I get authorized." It's "what can the Matrix and Toys subsystem actually do."

---

## 13. Limits, gotchas & a minimal client

**Hard limits and rules**

- Nothing devices, Android 14+ only. The Matrix subsystem is gated on a platform feature flag (`NtFeaturesUtils.isSupport(105)`), so matrix calls no-op on non-matrix phones.
- You must hold `com.nothing.ketchum.permission.ENABLE` and bind the exact component in §3.
- Frames are brightness arrays, not RGB. Strip length is per-model (§6); the matrix is `length²` row-major.
- The global `led_effect_enable` setting and the Toy idle-timeout can blank you at any time. A dying process gets its LEDs reset by the service's `IProcessObserver`.
- `setGlyphMatrixTimeout` only works for the first-party allowlist (§11); third-party callers are logged and ignored.
- `animate()` and the marquee are CPU-side loops on a single-thread executor. Overlapping calls cancel the previous task (`stopCurrentTask`).
- Matrix source bitmaps must be square, or you get an `IllegalArgumentException`.

**Talking to the service without the SDK.** Since it's plain AIDL, a minimal client is just a binder proxy. Conceptually (Kotlin, abbreviated; this is the documented public contract, written from scratch):

```kotlin
val intent = Intent("com.nothing.thirdparty.bind_glyphservice").apply {
    component = ComponentName("com.nothing.thirdparty", "com.nothing.thirdparty.GlyphService")
}
bindService(intent, conn, BIND_AUTO_CREATE)
// in onServiceConnected(binder):
fun txInt(code: Int, data: IntArray) {
    val p = Parcel.obtain(); val r = Parcel.obtain()
    try {
        p.writeInterfaceToken("com.nothing.thirdparty.IGlyphService")
        p.writeIntArray(data)
        binder.transact(code, p, r, 0); r.readException()
    } finally { p.recycle(); r.recycle() }
}
// register (tx5) → openSession (tx2) → setMatrixColors (tx7) → closeSession (tx3)
```

with manifest:

```xml
<uses-permission android:name="com.nothing.ketchum.permission.ENABLE"/>
<meta-data android:name="NothingKey" android:value="test"/>  <!-- ignored on A16, kept for compat -->
```

For debugging on older builds you may still need:

```
adb shell settings put global nt_glyph_interface_debug_enable 1   # auto-disables after 48h
```

---

## 14. Appendix: constant reference

**SDK / service versions** — SDK constant `Common.NOTHING_SDK_VERSION = 140101`; on-device service `versionCode 130101` (`13.01.01`), compiled against SDK 36 (Android 16).

**Brightness** — `GlyphFrame.DEFAULT_LIGHT = 4000`; `GlyphManager.DEFAULT_MAX_LIGHT = 4096`, `DEFAULT_MIN_LIGHT = 800`; `GlyphMatrixUtils.MAX_BRIGHTNESS = ON = 4095`, `OFF = 0`, `BRIGHTNESS_MULTIPLIER = 16`, `LETTER_SPACING = 1`; `GlyphMatrixObject` default brightness `255`, scale `100`, orientation `0`.

**Light ids** — `110` Composer · `115` third-party strips · `121` matrix (extended) · `123` app/Toy matrix.

**Matrix sizes** — `A024` → `25` (625 dots) · `A069P` → `13` (169 dots).

**Glyph Toy framework** — `MSG_GLYPH_TOY = 1`, data key `"data"`; status `prepare`/`start`/`end`; events `change`/`aod`/`action_down`/`action_up`.

**Key strings** — bind action `com.nothing.thirdparty.bind_glyphservice`; permission `com.nothing.ketchum.permission.ENABLE`; manifest key `NothingKey`; settings `led_effect_enable`, `nt_glyph_interface_debug_enable`; provider `content://com.nothing.glyphtoyprovider/glyph_toy`; deep link scheme `glyphtoy://`; dead fingerprint substring `95E1F157FE98518`; Firebase `https://nothingdevice-aafad-default-rtdb.asia-southeast1.firebasedatabase.app/`.

---

*Analysis performed for interoperability and educational research on hardware the analyst owns. All trademarks belong to Nothing Technology Ltd. No proprietary code is redistributed here — only independently-derived descriptions of the public binder contract and observable behaviour.*
