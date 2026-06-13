# Disclaimer & Legal Notice

> **Short version:** This repository contains independent, original research notes about how the
> Nothing Glyph Interface works, written for interoperability and education. It does **not**
> contain Nothing's app packages, SDK binaries, firmware, or bulk decompiled source. It is not
> affiliated with Nothing. This notice is not legal advice.

## 1. Not affiliated; trademarks

This is an unofficial, independent project. It is **not** affiliated with, authorized by,
sponsored by, or endorsed by **Nothing Technology Limited**. "Nothing", "Glyph", "Glyph
Interface", "Glyph Matrix", "Glyph Toys", "Phone (1)/(2)/(2a)/(3a)/(3)", "CMF", and related names
and logos are trademarks of their respective owners. They are used here only for accurate,
descriptive reference (nominative use) and do not imply any association or endorsement.

## 2. Purpose: interoperability and education

The analysis exists to document a publicly shipped device interface so that developers can build
**interoperable** software for hardware they own, and for general study of how the system works.
Examining and describing the behaviour of software for these purposes is widely recognized
(for example, the interoperability allowances in the EU Software Directive 2009/24/EC Art. 6, and
fair-use treatment of interface analysis in cases such as *Sega v. Accolade* and
*Sony v. Connectix* in the United States).

## 3. What is NOT included here (on purpose)

To respect the rights of others, this repository deliberately omits:

- Nothing application packages (`.apk`), SDK archives (`.aar`), `.dex`, or other binaries;
- any complete or bulk decompiled / disassembled source of Nothing's software;
- cryptographic keys, signing certificates, or anything enabling repackaging or impersonation of
  a signed app;
- any tool whose purpose is to circumvent a technological protection measure.

What **is** included is original prose, tables, and diagrams, plus **short excerpts** of code or
constants quoted strictly for commentary, criticism, and interoperability. The included font image
was regenerated from the SDK's **publicly distributed** plain-text string resources for
documentation purposes.

## 4. No circumvention, no harm

The notes describe a documented, app-facing binder interface and observable system behaviour. They
do **not** provide a way to bypass device security, escape the application sandbox, gain
privilege, unlock paid features, or modify a device. Statements about what the system does or does
not enforce are observations about a specific software build, not instructions to defeat a
protection measure.

## 5. Accuracy and build scope

Findings are tied to specific software versions (notably `com.nothing.thirdparty` 13.01.01 and
`glyph-matrix-sdk-2.0`). Vendor software changes; behaviour described here may differ on other
builds. The material is provided **"as is"**, without warranty of any kind. Use it at your own
risk. The author is not responsible for damage, bricked devices, voided warranties, or violations
of any agreement that result from acting on this information.

## 6. Your own agreements

If you downloaded an SDK or accepted a developer agreement or terms of service from Nothing, those
terms may contain their own restrictions (including on reverse engineering). Those are a contract
between you and the vendor and are independent of copyright. Review them before relying on this
material, and only analyse software on devices and accounts you own.

## 7. Not legal advice

This document is provided for general information only and is **not legal advice**. Laws on reverse
engineering, fair use / fair dealing, and contract terms vary by country and change over time. If
you are concerned about your specific situation, consult a qualified lawyer in your jurisdiction.

## 8. Good-faith takedown

This project is published in good faith. If you are a rights holder and believe any specific
content here exceeds fair use or otherwise infringes your rights, please open an issue or contact
the maintainer, and the specific material will be reviewed and removed or revised promptly.
