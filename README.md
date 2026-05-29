# Nothing Glyph Interface — Reverse-Engineering Notes

![License](https://img.shields.io/badge/license-MIT-blue)
![Platform](https://img.shields.io/badge/platform-Nothing%20OS-black)
![Decompiler](https://img.shields.io/badge/tooling-jadx-orange)

Notes from taking apart the Nothing Phone "Glyph" LED interface and writing down how it
actually works. Two pieces are covered: the public **Glyph Developer Kit** SDK
(`com.nothing.ketchum`) and the on-device system service that drives the LEDs
(`com.nothing.thirdparty`, shipped as `NtThirdParty.apk`).

This isn't a from-scratch discovery — others have mapped large parts of this before (see
[Prior work](#prior-work) for credit). I went through it independently against a recent build,
checked every claim against the decompiled code, and wrote down what held up, what didn't, and
what's specific to the version I looked at.

**Build analyzed:** `com.nothing.thirdparty` **13.01.01** (versionCode `130101`, compiled
against Android 16 / SDK 36). Anything I say about "what's enforced" is true of that build and
may not hold on older or newer ones.

> No Nothing binaries or bulk decompiled source live in this repo. There are only my notes plus
> short code excerpts quoted as evidence. If you want to confirm any of it, you grab the SDK and
> APK yourself and decompile them locally — [`docs/04-reproduce.md`](docs/04-reproduce.md) walks
> through it.

## What's new here, versus prior work

The service-level basics were mapped before me (see [Prior work](#prior-work)). What this repo
adds on top of that:

- a **newer build** — `13.01.01` (Android 16 / SDK 36), where registration and the
  signing-certificate check behave differently from what older notes describe;
- coverage of the **Glyph Matrix** light paths (session IDs 121/123, `registerMatrixSDK`), which
  postdate most existing classic-Glyph documentation;
- a catalogue of **SDK-level bugs** (`animate()`, `getLetterConfigs()`, `toggle()`) I haven't
  seen written up elsewhere;
- a **verify-don't-trust** approach: every claim is scoped to the build and marked by how
  strongly the code supports it.

If you already know rec0de's work, those four points are the reason to read on.

## What I found

A few things stood out on this build.

The SDK has some genuinely broken code. The breathing-light helper (`animate()`) divides by a
period that defaults to zero, and because it runs on a fire-and-forget background task the
crash is swallowed and the lights just never come on. The text-to-matrix routine has a loop
counter that resets to zero mid-loop, so certain strings hang the thread. And `toggle()` is a
misnomer — it only ever writes "on," it never flips state. Details in
[`docs/01-sdk.md`](docs/01-sdk.md).

On the service side, the part I spent most time on, the interesting result is about
registration. All three register entry points return success, and the `NothingKey` an app
declares in its manifest is read by the SDK, sent over, and then dropped on the floor — the key
isn't validated anywhere I could find. A SHA-1 signing-certificate check and a "developer end
time" debug mechanism do exist in the code, but in this build I couldn't find any call site for
them; they look inert here. That last point differs from earlier write-ups (see
[Prior work](#prior-work)), which is exactly why version matters. Full reasoning in
[`docs/02-system-service.md`](docs/02-system-service.md).

The rest is mechanics: the Glyph rides on Android's standard Lights HAL, with light-session IDs
110 (Composer), 115 (third-party), 121 (Matrix) and 123 (third-party Matrix); there's a
process-death watchdog that clears the LEDs when a registered app dies; and a global
`led_effect_enable` switch that tears every session down. You can't drive any of it from an
ordinary app — the system uses a hidden OEM lights API behind a privileged permission
(`CONTROL_DEVICE_LIGHTS`), so a sideloaded app can neither compile against it nor hold it.

## Repository layout

```
docs/
  01-sdk.md              SDK bugs, limits, brightness ranges, progress rules
  02-system-service.md   service behaviour: registration, light IDs, watchdogs, sensors
  03-device-reference.md device codes, channel counts, light IDs (quick lookup)
  04-reproduce.md        how to obtain the SDK/APK and verify everything yourself
  05-methodology.md      how the analysis was done and how confidence was assigned
```

## Prior work

Most of the service-side groundwork was done before me, and the overlap is large enough that
credit is owed up front:

- **[rec0de/glyph-api](https://github.com/rec0de/glyph-api)** — the most complete prior
  documentation of the unofficial Glyph Light API. Covers the `GlyphService` binding, the
  session IDs, the remote-config/API-key model, and the signing-certificate check (including the
  observation that only part of the SHA-1 hash is matched). If you want one reference, start
  there. Where my notes and theirs disagree, it is most likely a build difference, and I've
  flagged those spots.
- **[Nothing-Developer-Programme/Glyph-Developer-Kit](https://github.com/Nothing-Developer-Programme/Glyph-Developer-Kit)**
  and **[GlyphMatrix-Developer-Kit](https://github.com/Nothing-Developer-Programme/GlyphMatrix-Developer-Kit)**
  — Nothing's official SDKs and docs.
- **[SebiAi/custom-nothing-glyph-tools](https://github.com/SebiAi/custom-nothing-glyph-tools)**
  — tools for authoring Glyph compositions for the official Composer; a different layer of the
  stack but the most practical thing to actually use.

For what this repo adds on top of the above, see
[What's new here](#whats-new-here-versus-prior-work).

## Scope and legality

This is interoperability and educational research on software running on hardware I own. No
proprietary binaries or bulk decompiled source are redistributed; short excerpts appear for
commentary and criticism. Nothing here modifies a device or unlocks paid functionality. Not
affiliated with Nothing Technology Limited.

## License

Original documentation is released under the [MIT License](LICENSE). Quoted code excerpts from
third-party software remain the property of their respective owners and are included for
commentary only.
