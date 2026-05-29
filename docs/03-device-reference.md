# Device reference

Lookup tables pulled from the SDK's `Glyph.java` and the service's light handling. All values
are straight from the code.

## Device codes and Glyph channel counts

| Model code | Internal code | Glyph channels |
|---|---|---|
| `A063` | 20111 | 15 |
| `A065` / `AIN065` | 22111 | 33 |
| `A142` / `A142P` | 23111 / 23113 | 26 |
| `A059` | 24111 | 36 |
| `A069` | 25111 | 6 |
| `A024` | 23112 (Matrix) | 625 (25×25) |
| `A069P` | 25111p (Matrix) | 169 (13×13) |

The retail names (Phone 1, 2, 2a, 3a, and so on) don't appear in the code at all — only the
model codes above do. The channel counts are exact; mapping a code to a marketing name is
guesswork that gets less reliable for newer models.

## Brightness ranges

| | Off | Max | Notes |
|---|---|---|---|
| Standard Glyph LED | 0 | 4096 | builder default 4000; progress floor 800 |
| Matrix pixel | 0 | 4095 | in text mode, ≥1024 snaps on, else off |

## Light-session IDs (system service)

| ID | Purpose |
|---|---|
| 110 | Glyph Composer, exclusive |
| 115 | third-party Glyph frames |
| 121 | hardware Glyph Matrix |
| 123 | third-party app matrix |

## Binder method mapping (SDK ↔ obfuscated service)

| Transaction | SDK method | Obfuscated impl |
|---|---|---|
| 1 | setFrameColors | `tbz` |
| 2 | openSession | `lgi` |
| 3 | closeSession | `ziu` |
| 4 | register | `yum` |
| 5 | registerSDK | `nhp` |
| 6 | registerMatrixSDK | `kbv` |
| 7 | setMatrixColors | `rwe` |
| 8 | setGlyphMatrixTimeout | `eur` |
| 9 | setAppMatrixColors | `qae` |
| 10 | closeAppMatrix | `tez` |

These names are specific to the 13.01.01 build and will change with any rebuild — obfuscation
remaps them every release. Match by behaviour, not by name.
