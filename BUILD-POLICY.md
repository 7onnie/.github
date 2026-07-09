# Build-Policy: macOS-Binaries

Gültig ab 2026-07-09 für alle Repos dieses Accounts, die macOS-Binaries bauen.

## Regel

**macOS-Binaries werden nur noch für `arm64` (Apple Silicon) gebaut — keine
Universal-Binaries, keine x86_64-Slices.**

Begründung: Es gibt kaum noch Intel-Macs im Einsatz, und der doppelte
Compile-Pass (x86_64 + `lipo`) verbrennt teure macOS-Runner-Minuten in den
GitHub Actions.

## Anforderungen an Build-Workflows

| Punkt | Vorgabe |
|---|---|
| Runner | Apple-Silicon-Runner (`macos-14` oder neuer) |
| Swift PM | `swift build -c release --arch arm64` |
| xcodebuild | `ARCHS=arm64 ONLY_ACTIVE_ARCH=NO` als CLI-Override |
| Verifikation | `lipo -info` auf das Produkt — Build schlägt fehl, wenn `x86_64` enthalten ist |
| Asset-Naming | `<tool>-<version>-arm64.tar.gz` (nie mehr `-universal`) |
| Homebrew-Formeln | `depends_on arch: :arm` — Intel-User bekommen eine klare Fehlermeldung |
| Mindest-macOS | implizit 11.0+ (arm64 existiert erst ab Big Sur) |

## Apps mit Auto-Updater

Bevor das erste arm64-only-Release erscheint, MUSS ein letztes
Universal-Release ausgeliefert werden, dessen Updater einen **Intel-Guard**
enthält: physische Hardware per `sysctlbyname("hw.optional.arm64")` prüfen
(nicht die Prozess-Architektur — x86_64-Prozesse unter Rosetta melden
arm64-Hardware korrekt) und auf Intel-Macs keine Updates mehr anbieten,
stattdessen Hinweis „letzte Intel-kompatible Version".

Umgesetzt in BLEUnlock v1.15.5 (`checkUpdate.swift`, `isAppleSiliconHardware()`).

## Umgesetzte Repos

- **mysides** — arm64-only ab dem nächsten Tag nach 2026-07-09; Formel im Tap arch-restricted
- **BLEUnlock** — Phase 1 (Intel-Guard, noch Universal): v1.15.5; Phase 2 (arm64-only): ab dem Folge-Release
