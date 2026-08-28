# Actions-Policy: Minuten sparen, ohne Releases zu verlieren

Gültig ab 2026-08-17 für alle Repos dieses Accounts, die GitHub Actions nutzen.
Anlass: das Monatskontingent war aufgebraucht. Ergänzt [BUILD-POLICY.md](BUILD-POLICY.md)
(arm64-only), die aus demselben Grund entstand.

**Wer eine neue Action baut oder eine bestehende anfasst, arbeitet diese Datei ab.**

---

## Die zwei Zahlen, aus denen alles folgt

| | |
|---|---|
| **Runner-Faktor** | Linux **1×**, Windows **2×**, **macOS 10×**. Eine macOS-Minute kostet wie zehn Linux-Minuten. |
| **Mindest-Abrechnung** | **1 Minute je JOB** — nicht je Lauf. Ein Job, der 8 Sekunden läuft, kostet auf macOS **10 abgerechnete Minuten**. |

★★★ Die zweite Zahl ist die, die man übersieht. Gemessen am 2026-08-17: die
`check-version`-Jobs dreier App-Repos liefen 2–10 Sekunden (ein `grep` auf eine
Versionsnummer) und kosteten zusammen rund **870 abgerechnete Minuten im Monat**.

⇒ **Einen Workflow in zwei Jobs zu teilen ist auf einem teuren Runner teurer als einer —
es sei denn, der billige Teil zieht auf `ubuntu-latest` um.** Genau das ist die Regel.

---

## Die Pflichtteile

### 1. `timeout-minutes` auf JEDEN Job. Ohne Ausnahme.

⚠⚠ **Der teuerste Einzelposten, den es gibt.** GitHubs Standard ist **360 Minuten**. Auf
einem macOS-Runner sind das **3600 abgerechnete Minuten aus einem einzigen hängenden Lauf** —
mehr als ein ganzes Monatskontingent. Bis 2026-08-17 stand in **keinem** Job dieses Kontos
ein Limit.

Und Hänger sind real: eine Testsuite dieses Accounts stand nachweislich 66 Minuten still,
ohne zu enden.

```yaml
jobs:
  release:
    runs-on: macos-latest
    timeout-minutes: 25        # 3-8x der gemessenen Laufzeit, nicht knapper
```

★ Grosszügig setzen. Ein zu enges Limit macht aus einem langsamen Lauf einen roten, und das
kostet mehr Vertrauen als es Minuten spart.

### 2. Der teure Runner trägt NUR, was ihn wirklich braucht

Alles, was `grep`, `gh`, `jq`, `git`, `curl` oder Versionsvergleiche tut, gehört auf
`ubuntu-latest`. Auf macOS gehört nur: `swiftc`/`swift build`, `xcodebuild`, `codesign`,
`lipo`, `hdiutil`, `pkgbuild`, `plutil`, `brew`, `security`, `launchctl`.

```yaml
jobs:
  check-version:
    runs-on: ubuntu-latest     # ← der Torwächter ist billig
    timeout-minutes: 5
  release:
    needs: check-version
    if: needs.check-version.outputs.changed == 'true'
    runs-on: macos-latest      # ← nur der Build ist teuer
    timeout-minutes: 25
```

### 3. `concurrency` — aber die Richtung hängt vom Zweck ab

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true     # TEST-Workflows: ja
```

⚠⚠ **In RELEASE-Workflows `cancel-in-progress: false`.** Einen laufenden Release-Lauf
abzubrechen kann ein halb veröffentlichtes Release hinterlassen — Tag da, Asset nicht.
Warten ist dort richtig, Abbrechen nicht.

### 4. Der Auslöser so eng wie möglich

```yaml
on:
  push:
    branches: [main]
    paths: ['Sources/Foo/Updater.swift']   # nur die Datei, die die Version trägt
  workflow_dispatch:                        # Handauslösung als Notausgang
```

⚠ **Vorher nachsehen, ob schon ein Filter da ist.** In den drei App-Repos dieses Kontos stand
`paths` bereits auf einer einzigen Datei — ein zusätzliches `paths-ignore` hätte dort nichts
getan ausser Konfiguration zu erzeugen, die man später für wirksam hält.

### 5. arm64-only

Siehe [BUILD-POLICY.md](BUILD-POLICY.md). Der zweite Bogen plus `lipo` verdoppelt genau die
Minute, die am meisten kostet.

### 6. Ein Release je Arbeitsblock, nicht je Commit

Auf dem Feature-Branch beliebig committen — dort löst nichts aus. Version-Bump und Merge nach
`main` erst, wenn ein Block fertig ist. Fünf Releases an einem Tag sind ein Fehler, kein Fleiss.

⚠ Die Action vergleicht die Version im Code mit dem letzten Release-Tag: **ohne Bump kein
Release.** Der Bump gehört deshalb in den *letzten* Commit des Blocks.

### 7. Kein mehrzeiliges Argument in einem `run: |`-Block. Und: jede Datei durch den Parser.

⚠⚠ **Der teuerste Fehler dieser Liste — nicht in Minuten, sondern in Auslieferung.**
Ergänzt am 2026-08-29, nachdem er ein halbes Jahr unentdeckt lief.

Release-Notes, Commit-Messages, JSON-Payloads: alles mit Zeilenumbrüchen **niemals** als
mehrzeiliges Shell-Argument in einen Block-Scalar schreiben. Der Generator vom März 2026 tat
genau das:

```yaml
        run: |
          gh release create "v$VERSION" \
            --notes "Automatisches Release.

- **Foo.txt**: per Copy-Paste in tomedo einfügen.
"
```

Die Notes-Zeilen stehen auf **Spalte 0**. Eine nicht-leere Zeile auf Spalte 0 **beendet den
Block-Scalar** — der Parser sieht danach Markdown auf Top-Level, und die Datei ist ungültiges
YAML.

**Stattdessen `printf` + `--notes-file`:**

```yaml
        run: |
          printf '%s\n' \
            'Automatisches Release.' \
            '' \
            '- **Foo.txt**: per Copy-Paste in tomedo einfügen.' \
            > notes.md
          gh release create "v$VERSION" Foo.txt --notes-file notes.md
```

Die Anführungszeichen liegen innerhalb der Einrückung, es entsteht keine Zeile auf Spalte 0.
In PowerShell dasselbe Problem mit dem Here-String `@" … "@`, dessen Terminator auf Spalte 0
stehen muss — dort ein String-Array plus `[IO.File]::WriteAllLines(...)`.

**Vor jedem Commit an einem Workflow:**

```
ruby -ryaml -e 'YAML.load_file(".github/workflows/release.yml")'
```

Kein Output = gültig. Zwei Sekunden.

#### Die Diagnose-Signatur — daran erkennt man es im Bestand

Zwei Dinge treten zusammen auf:

1. Runs mit **0 s Laufzeit und `failure`**, ohne Log (`gh run view --log-failed` → „log not
   found"), Annotation „This run likely failed because of a workflow file issue"
2. Die Runs feuern **auch auf Feature-Branches**, obwohl `branches: [main]` dasteht

★★★ Punkt 2 ist der eigentliche Verräter — und der Grund, warum es so lange lief: GitHub kann
die `branches`/`paths`-Filter nicht anwenden, wenn es die Datei nicht parst. Man sieht rote
Läufe auf Branches, auf denen laut Konfiguration nichts laufen dürfte, und liest das als „die
Action ist halt zickig" statt als „die Datei ist kaputt".

**Gemessen am 2026-08-25: 22 Repos betroffen, jedes seit dem 9./10. März ohne ein einziges
Release.** Wo der Code weitergelaufen war, klaffte die Lücke entsprechend — ARD-Utils stand bei
Release v0.1, im Code bei v1.2; ManageUsers v0.3.0 gegen v0.5.5; StandardAP v0.1 gegen v0.8.
Kosten entstanden keine (es startet ja kein Job) — der Schaden war, dass die Auslieferung
stillstand und niemand ein Signal bekam.

#### Workflow-Dateien brauchen SSH, nicht `gh api`

Dem `gh`-Token fehlt der `workflow`-Scope. Ein `PUT /repos/…/contents/.github/workflows/…`
antwortet mit einem **404** — nicht mit „fehlende Berechtigung", sondern mit „nicht gefunden",
obwohl ein `GET` auf denselben Pfad die Datei liefert. Über `git push` per SSH geht es. Das
kostet sonst jedes Mal eine Viertelstunde Fehlersuche an der falschen Stelle.

---

## Der Bauplan zum Abschreiben

```yaml
name: Release on Version Change

on:
  push:
    branches: [main]
    paths: ['Sources/MeinTool/Updater.swift']
  workflow_dispatch:

permissions:
  contents: write

# Release-Workflow ⇒ warten statt abbrechen (siehe Punkt 3).
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  check-version:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    outputs:
      version: ${{ steps.version.outputs.version }}
      changed: ${{ steps.check.outputs.changed }}
    steps:
      - uses: actions/checkout@v4
      - id: version
        run: |
          VERSION=$(grep 'let kAppVersion' Sources/MeinTool/Updater.swift \
            | grep -oE '"[0-9]+\.[0-9]+\.[0-9]+"' | tr -d '"')
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
      - id: latest
        env: { GITHUB_TOKEN: "${{ secrets.GITHUB_TOKEN }}" }
        run: |
          TAG=$(gh release view --repo "$GITHUB_REPOSITORY" --json tagName -q .tagName 2>/dev/null || true)
          echo "latest=${TAG#v}" >> "$GITHUB_OUTPUT"
      - id: check
        run: |
          if [ "${{ steps.version.outputs.version }}" != "${{ steps.latest.outputs.latest }}" ]
          then echo "changed=true"  >> "$GITHUB_OUTPUT"
          else echo "changed=false" >> "$GITHUB_OUTPUT"
          fi

  release:
    needs: check-version
    if: needs.check-version.outputs.changed == 'true'
    runs-on: macos-14          # Apple Silicon, siehe BUILD-POLICY.md
    timeout-minutes: 25
    steps:
      - uses: actions/checkout@v4
      - run: swift build -c release --arch arm64
      - name: arm64-only verifizieren
        run: |
          BIN=.build/arm64-apple-macosx/release/MeinTool
          lipo -info "$BIN"
          if lipo -info "$BIN" | grep -q x86_64
          then echo "::error::x86_64-Slice — verstoesst gegen BUILD-POLICY.md"; exit 1
          fi
      - run: codesign --force --deep --sign - "$BIN"
      # … Assets schnueren, Release anlegen
```

---

## Was NICHT hilft (geprüft, damit es niemand zweimal prüft)

* ⛔ **Self-hosted Runner auf einem MacBook.** Rechnerisch der grösste Hebel (self-hosted
  Minuten sind kostenlos), praktisch untauglich: ein Laptop ist unterwegs, auf Akku und nur
  mit Unterbrechungen online. Ein Runner nimmt Aufträge nur an, solange er wach ist — sonst
  bleibt der Lauf hängen und fällt nach rund 24 h aus. Man pusht ein Release und merkt Tage
  später, dass keins entstand. **Nur mit einem dauerhaft laufenden Mac** vertretbar, und dann
  nur für **private** Repos (bei öffentlichen führt ein fremder Pull Request Code auf dem
  Rechner aus), mit eigenem macOS-Benutzer und `git clean -fdx` als erstem Schritt.
* ⛔ **macOS-Builds auf einen Linux-Server verlagern.** Es gibt keinen Swift-SDK für
  Darwin-Ziele, der auf Linux läuft (Apples `swift-sdk-generator` kennt macOS nur als *Host*),
  AppKit/SwiftUI/WebKit sind nicht cross-kompilierbar, und Apples SDK-Lizenz verbietet die
  Nutzung auf Nicht-Apple-Hardware ausdrücklich. ★ Die Notarisierung ist dabei *nicht* der
  Blocker — wer ad-hoc signiert, könnte das sogar auf Linux (`rcodesign`).
* ⚠ **Build-Caching** (`actions/cache` auf `.build`): bei Build-Zeiten unter ~5 Minuten liegt
  die Ersparnis im Sekundenbereich, die Abrechnung rundet aber auf ganze Minuten — der Gewinn
  ist unbestimmt, und ein veralteter `.build` kann ein falsches Release-Artefakt erzeugen.
  ⇒ Erst erwägen, wenn ein Build regelmässig **über 5 Minuten** liegt, und dann nur die
  Abhängigkeits-Checkouts cachen, nicht die Übersetzungsergebnisse.

---

## Prüfliste für eine neue oder geänderte Action

- [ ] **Die Datei parst** — `ruby -ryaml -e 'YAML.load_file(".github/workflows/x.yml")'` schweigt
- [ ] Kein mehrzeiliges Argument in einem `run: |`-Block (Notes über `printf` + `--notes-file`)
- [ ] Jeder Job hat `timeout-minutes`
- [ ] Kein Job auf macOS/Windows, der dort nichts Plattformspezifisches tut
- [ ] `concurrency` gesetzt — `cancel-in-progress` **nur** in Test-Workflows
- [ ] `on:`-Filter so eng wie möglich, `workflow_dispatch` als Notausgang
- [ ] Bei macOS-Binaries: `--arch arm64` **und** ein `lipo`-Tor, das bei x86_64 scheitert
- [ ] Keine Matrix auf einem teuren Runner
- [ ] Artefakt-Aufbewahrung geprüft (Standard 90 Tage kostet Speicher, keine Minuten)

Der Bestand lässt sich jederzeit nachmessen — das Prüfskript liegt in
`7onnie/ApiHub` unter `scripts/actions-audit.py` und meldet je Job, was fehlt.
Es prüft `timeout-minutes` seit dem 2026-08-29 auf **jedem** Job (vorher nur auf Runnern ab
Faktor 2 — es schwieg damit über 40 von 43 Fällen) und meldet eine nicht parsende Datei als
`⛔ … laeuft NIE` samt Hinweis auf die Spalte-0-Ursache.

⚠ Beim Einsammeln **nicht** `repos/$R/actions/workflows` benutzen: diese API führt gelöschte
Workflows weiter als `active` (gemessen 6 statt 1 bei CDImport). Der Abruf läuft in einen 404,
es entsteht eine leere Datei, und der Prüfer meldet dafür nichts — was sich wie „geprüft und
sauber" liest. Die Contents-API listet, was wirklich da ist; der Aufruf im Skript-Kommentar ist
entsprechend korrigiert.
