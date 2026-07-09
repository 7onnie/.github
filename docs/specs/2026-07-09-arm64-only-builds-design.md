# ARM64-only macOS-Builds (Ablösung Universal-Binaries)

Datum: 2026-07-09 · Status: approved · Policy: [BUILD-POLICY.md](../../BUILD-POLICY.md)

## Ziel

macOS-Binaries werden nur noch für arm64 gebaut. Betroffen: `mysides`
(Swift-PM-CLI + Homebrew-Tap) und `BLEUnlock` (Xcode-App mit In-App-Updater).
Motivation: kaum noch Intel-Macs, halbierter Compile-Aufwand auf teuren
macOS-Runnern.

## Entscheidungen (inkl. Gemini-Consensus-Review)

- Zentralform: Policy-Doc (`BUILD-POLICY.md` im `.github`-Repo) + lokale
  Umsetzung. Reusable Workflow verworfen — nur 2 Repos mit grundverschiedenen
  Builds (YAGNI).
- Build-Flags: `swift build --arch arm64` bzw. `xcodebuild ARCHS=arm64
  ONLY_ACTIVE_ARCH=NO`; eingebettete Helper-Targets (Launcher.app) erben das.
- Formel-Syntax `depends_on arch: :arm` verifiziert; Intel-User erhalten bei
  brew install/upgrade eine saubere Fehlermeldung und behalten die installierte
  Version.
- Hardware-Erkennung: `sysctlbyname("hw.optional.arm64")` — prüft die physische
  Maschine, nicht die Prozess-Architektur (Rosetta-sicher). Auf Intel existiert
  der Key nicht → sysctl schlägt fehl → Intel.
- Erwartete Ersparnis: 20–40 % der Workflow-Gesamtzeit (Konsens aller
  3 Review-Varianten; die vollen 50 % gelten nur für den reinen Compile-Anteil).

## Rollout

**mysides (einphasig, kein Auto-Updater):** x86_64-Build + lipo entfernt,
Tarball `mysides-<ver>-arm64.tar.gz`, Verifikations-Step (`lipo -info` ohne
x86_64), Formel arch-restricted. Wirkt ab dem nächsten `v*`-Tag.

**BLEUnlock (zweiphasig — kritischer Konsens-Fund: der Guard muss die Nutzer
VOR dem ersten arm64-only-Release erreichen, sonst zieht ein Intel-Mac per
Auto-Update eine nicht startende Version):**

1. **Phase 1 (v1.15.5, Universal):** Intel-Guard im Updater —
   `UpdateCheckResult.intelUnsupported`, Guard am Anfang von `checkUpdate()`,
   manueller Check zeigt lokalisierten Hinweis (Base/de/ja/zh-Hans),
   automatischer Check bleibt still und räumt Pending-Updates weg.
   Asset-Namen (`BLEUnlock-<tag>.dmg/.zip`) enthalten kein "universal" und
   bleiben unverändert — der Updater bricht nicht an Namen.
2. **Phase 2 (Folge-Release):** `ARCHS=arm64 ONLY_ACTIVE_ARCH=NO` in
   release.yml + test-build.yml, Verifikations-Step via `lipo -info` (App-Binary
   und eingebettete Launcher.app), README-Plattform-Badge → „macOS 11+
   (Apple Silicon)". Erst NACH verifiziertem Phase-1-Release committen.

## Verifikation

- Phase 1: Release-Workflow v1.15.5 grün; `lipo -info` des DMG-Inhalts zeigt
  weiterhin beide Architekturen (letztes Universal-Release).
- Phase 2: Workflow-Step schlägt fehl, wenn das Produkt eine x86_64-Slice
  enthält; Stichprobe nach erstem arm64-Release.
