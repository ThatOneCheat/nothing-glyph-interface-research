# Reproducing this

Nothing proprietary ships in this repo, so to check any of the claims you fetch the files
yourself and decompile them locally. It all runs on your own machine against software on a
device you own.

## What you need

- A JRE, version 17 or newer (jadx needs it). `java -version` to check.
- [jadx](https://github.com/skylot/jadx). The `-all` jar can run headless through its CLI entry
  point, which is what the commands below use.
- The Glyph Developer Kit `.aar`, from Nothing's official
  [Glyph-Developer-Kit](https://github.com/Nothing-Developer-Programme/Glyph-Developer-Kit).
- `NtThirdParty.apk`, pulled from a Nothing device you own:

  ```
  adb shell pm path com.nothing.thirdparty
  adb pull /data/app/.../base.apk NtThirdParty.apk
  ```

## Decompiling the SDK

An `.aar` is just a zip; jadx reads it directly.

```
java -cp jadx-gui-<ver>-all.jar jadx.cli.JadxCLI -d sdk_out glyph-matrix-sdk-2.0.aar
```

Then read, under `sdk_out/sources/com/nothing/ketchum/`:

- `GlyphManager.java` for `animate()`, `toggle()` and the `displayProgress` rules
- `GlyphMatrixUtils.java` for `getLetterConfigs()`, the square-image and scale checks, and the
  text threshold
- `GlyphFrame.java` for the `buildChannel()` index behaviour and default brightness
- `Glyph.java` for device codes, channel counts and matrix sizes

## Decompiling the service

```
java -cp jadx-gui-<ver>-all.jar jadx.cli.JadxCLI -d apk_out --show-bad-code NtThirdParty.apk
```

The claims to check, all under `apk_out/sources/com/nothing/thirdparty/`:

- `GlyphService.java` — the binder stub. Find where it opens `LightsManager` sessions for IDs
  110/115/121/123, the `IProcessObserver`, and the `led_effect_enable` `ContentObserver`.
- the auth-controller class — confirm both register helpers return true whenever the caller
  package is non-null, and that the key argument is never read.
- the utility class holding the SHA-1 prefix and `developer_end_time` — then grep the whole tree
  to see whether anything in the Nothing packages actually calls those methods:

  ```
  grep -rn "developer_end_time" apk_out/sources/com/nothing/
  ```

- confirm there's no 48-hour timer by searching for the constant in milliseconds (expect nothing):

  ```
  grep -rn "172800000" apk_out/sources/
  ```

## Check your version

Everything here is for `versionName 13.01.01` / `versionCode 130101`. Your APK's version is in
`apk_out/resources/AndroidManifest.xml`. Note it down in anything you publish — behaviour, and
especially what's enforced, changes between builds.

One shortcut: `classes.dex` is often stored uncompressed inside the APK, so a plain text search
over the raw `.apk` will surface a lot of strings before you decompile anything. Handy for a
quick sanity check.
