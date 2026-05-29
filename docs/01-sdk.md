# The Glyph Developer Kit SDK

Notes from the decompiled `glyph-matrix-sdk-2.0.aar` (package `com.nothing.ketchum`).

The SDK is the client library an app links against. It doesn't touch the LEDs itself — it packs
your instructions into a frame and ships them over binder IPC to the system service, which is
the thing that actually lights anything up. So treat this as the remote control; the service in
[02](02-system-service.md) is the device.

Everything below is confirmed against the decompiled source unless I say otherwise.

## Bugs worth knowing about

### `animate()` quietly does nothing
The breathing-light helper computes brightness with this expression (`GlyphManager`,
`DEFAULT_MAX_LIGHT` is 4096):

```java
light = 0 + ((DEFAULT_MAX_LIGHT / (period / 2)) * elapsed);
```

There are three problems stacked here. If you never call `.buildPeriod(...)`, `period` is 0, so
`period / 2` is 0 and the whole thing throws an arithmetic exception. Because `animate()` runs
on a single-thread executor and the returned future is never read, that exception goes nowhere —
no crash you can see, the lights simply stay dark. Even when you do set a period, the division is
integer division: once `period / 2` climbs past 4096 (i.e. any period longer than about 8.2
seconds) the term `4096 / (period/2)` truncates to 0 and brightness is pinned at 0 for the whole
cycle. And at ordinary values it's coarse — a 3000 ms period gives `4096 / 1500 == 2`, so
brightness steps in big jumps and never reaches full.

### Rendering text can hang the thread
`GlyphMatrixUtils.getLetterConfigs()` walks a string, and on one branch it does this:

```java
config = sLettersMap.get(Character.valueOf(Character.toLowerCase(ch)));
i = config == null ? i + 1 : 0;
```

A loop index should only move forward. This resets it to 0 every time a character is found in
the built-in map, so the loop never terminates and the app hangs or runs out of memory. The
catch: this branch only runs when the character wasn't already resolved as a bundled string
resource, so apps that ship the letter resources won't hit it. Still a real footgun on the
fallback path.

### `toggle()` doesn't toggle
It performs a single one-way write of the frame's colours and keeps no on/off state. Call it
again and the lights stay on — you have to call `turnOff()` yourself. The name suggests a flip;
the behaviour is "set these colours."

## Limits that will crash you

**Channel index.** `Builder.buildChannel(int)` writes straight into a list that was pre-sized to
the target device's channel count (`.set(channel, …)`). Ask for a channel that's valid on some
other model but out of range for the one you targeted and you get an unguarded
`IndexOutOfBoundsException`.

**Brightness.** Standard Glyph LEDs run 0–4096 (the builder default is 4000, and progress bars
use 800 as a floor). Matrix pixels run 0–4095. When you render *text* onto the matrix, anything
at or above 1024 snaps fully on and everything else snaps off.

**Matrix images.** The bitmap has to be a perfect square or you get `"Image need to be square"`,
and the scale has to be greater than zero or `"Scale should larger than 0"`.

**Progress display.** `displayProgress` forces a specific strip per model and throws otherwise:

| Internal model | Required channel | Error if you skip it |
|---|---|---|
| 20111 | `D1_1` (idx 7) | "Please choose D1_1 …" |
| 22111 | `C1_1` (3) or `D1_1` (25), not both | "Please choose C1 or D1 …" |
| 23111 | `C_1` (idx 0) | "Please choose C_1 …" |
| 24111 | A / B / C all accepted | (no error) |
| 25111 | `A_1` (idx 0) | "Please choose A_1 …" |

## Registration, from the client side

Two calls matter here. `registerSDK(key, device)` sends the API key from your manifest;
`registerMatrixSDK(device)` sends only the device name, no key. Whether that distinction means
anything depends entirely on what the service does with them, which is the subject of
[02](02-system-service.md).
