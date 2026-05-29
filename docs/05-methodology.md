# Methodology

How the notes in this repo were produced, and how I decided what to claim.

## Process

I decompiled both targets locally with jadx — the SDK `.aar` and the system APK — and read the
relevant classes directly rather than working from summaries. For the SDK that's
straightforward, since the code is only lightly processed. The service is obfuscated, so for
anything there I worked from behaviour: trace what a method actually calls, what binder
transaction it answers, which Lights HAL session it touches, and only then attach a meaning to
its scrambled name. Where I name an obfuscated method in these notes, that name is a label for
the behaviour I observed, not something I trusted from the decompiler.

The questions that needed the most care were the "is X enforced?" ones. For those, finding the
relevant code isn't enough — a check that exists but is never called tells you nothing about
runtime behaviour. So for the signature check and the debug timer I went looking for call sites
across the whole package and reported what I found (none, on this build) rather than assuming
the presence of the code implied it runs.

## Confidence

I tried to keep three things separate in the writing:

- what's directly confirmed in the decompiled code,
- what's a reasonable inference but not nailed down (for example, that the unused signature
  check is a build difference rather than a permanent removal), and
- what I couldn't determine from these files at all.

When something is an inference or a guess, I say so in the text instead of stating it flatly.

## Versioning

The single most important caveat is the build number. Everything about enforcement —
registration, the certificate check, the debug timer — is a statement about
`com.nothing.thirdparty` 13.01.01 specifically. Obfuscated names also reshuffle every release.
Anyone extending this should re-pull the APK, record its version, and re-verify rather than
assuming these notes still describe the current ROM.

## Cross-checking

I compared my conclusions against existing public documentation, chiefly rec0de's
[glyph-api](https://github.com/rec0de/glyph-api). Most of it agreed. The notable disagreement —
whether the signing-certificate check is live — I've attributed to a build difference and
flagged in [02](02-system-service.md) rather than treating my reading as the last word.
