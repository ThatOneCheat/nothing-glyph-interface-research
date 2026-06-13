# Nothing Glyph Interface — Reverse-Engineering Notes

Independent technical notes on how the Nothing Phone **Glyph Interface** and the **Glyph Matrix SDK 2.0**
work: the client SDK (`com.nothing.ketchum`), the system service that actually drives the LEDs
(`com.nothing.thirdparty`), the binder protocol between them, the authorization model, the matrix
rendering engine, the embedded font, and the Glyph Toys framework.

The full write-up is here:

- **[GLYPH_INTERFACE_DEEP_DIVE.md](GLYPH_INTERFACE_DEEP_DIVE.md)** — the complete analysis.

Build analysed: `com.nothing.thirdparty` **13.01.01** (versionCode 130101, compiled against
Android 16 / SDK 36), plus the official `glyph-matrix-sdk-2.0.aar`. Anything about *what is
enforced* is true for that build only and may change in later updates.

## Highlights

- The binder interface has grown to **10 methods** (full transaction map in the write-up).
- On Android 16, registration no longer checks an API key, a signing certificate, or a remote
  auth map — this matches Nothing's own documented change for that OS version.
- The Glyph Matrix is a small graphics stack: 3-layer compositing, image scale/rotate/threshold,
  scrolling text, and a circular progress renderer.
- Glyph Toys are separate apps the system binds and feeds button / always-on-display events.
- Device/limit reference, brightness scales, and several undocumented model codes.

## Prior work and credit

This stands on earlier community research and Nothing's own docs. Credit where it's due:

- [rec0de/glyph-api](https://github.com/rec0de/glyph-api) — the main earlier write-up of the
  unofficial Glyph API (an older build). The deep dive flags exactly where the current build
  differs from those notes.
- [Nothing-Developer-Programme/Glyph-Developer-Kit](https://github.com/Nothing-Developer-Programme/Glyph-Developer-Kit) — Nothing's official SDK and documentation.
- [SebiAi/custom-nothing-glyph-tools](https://github.com/SebiAi/custom-nothing-glyph-tools) — practical tooling for authoring Glyph compositions.

## Scope and legal

Interoperability and educational research, performed on hardware the author owns. **No proprietary
binaries or bulk decompiled source are included here** — only original analysis plus short quoted
snippets for commentary. Not affiliated with or endorsed by Nothing Technology Limited. Please read
**[DISCLAIMER.md](DISCLAIMER.md)** before reusing or redistributing.

## License

Original documentation in this repository is released under the [MIT License](LICENSE). All
trademarks and any quoted third-party code remain the property of their respective owners.
