# README-Standardisierung mit Badge-Zeile (14 public Repos)

Datum: 2026-07-09 · Status: approved

## Ziel

Alle 14 öffentlichen Repos von `7onnie` erhalten einen standardisierten README-Header
im BLEUnlock-Stil inkl. Buy-Me-a-Coffee-Badge.

## Ziel-Layout

```html
<h1 align="center">RepoName</h1>

<p align="center">Einzeiler-Beschreibung (Quelle: GitHub-Repo-Description)</p>

<!-- BADGES:START -->
<p align="center">
  <a href="…actions/workflows/X.yml"><img alt="CI" src="…badge.svg"></a>
  <a href="…releases/latest"><img alt="Latest release" src="https://img.shields.io/github/v/release/7onnie/R?sort=semver"></a>
  <a href="…releases"><img alt="Total downloads" src="https://img.shields.io/github/downloads/7onnie/R/total.svg"></a>
  <a href="LICENSE"><img alt="License" src="https://img.shields.io/github/license/7onnie/R"></a>
  <a href="https://buymeacoffee.com/7onnie"><img alt="Buy me a coffee" src="https://img.shields.io/badge/Buy%20me%20a%20coffee-%E2%98%95-FFDD00?logo=buymeacoffee&logoColor=black"></a>
</p>
<!-- BADGES:END -->
```

## Badge-Regeln (pro Repo automatisch)

| Badge | Bedingung |
|---|---|
| CI | Workflows vorhanden UND mindestens ein abgeschlossener Run (Fork-Workflows sind oft deaktiviert) |
| Latest Release | ≥1 Release, `?sort=semver` |
| Total Downloads | ≥1 Release-Asset |
| Lizenz | SPDX-ID erkannt (kein NOASSERTION) |
| Buy me a coffee | immer, `#FFDD00`, Logo-Slug `buymeacoffee` (Verfügbarkeit bei shields.io am 2026-07-09 verifiziert) |

## Entscheidungen (inkl. Gemini-Consensus-Review)

- `align="center"`-HTML rendert auf github.com zuverlässig; auf npmjs.com kann die
  Zentrierung degradieren (linksbündig) — akzeptiert.
- Forks: Upstream-Titel + Upstream-Badges (auch npm-Badges aufs Original-Paket)
  werden ersetzt — im eigenen Repo irreführend. Logos/Bilder und restlicher Inhalt
  bleiben. Fork-Hinweis mit Upstream-Link bleibt/wird ergänzt.
- BLEUnlock: bestehende Zeile bleibt, nur BMAC-Badge + Marker ergänzt —
  auch in README.ja.md und README.cn.md.
- READMEs fehlen bei GiveMacOSRights + homebrew-tap → minimal neu anlegen.
- Leere Repo-Descriptions (GiveMacOSRights, hello-world) werden formuliert und
  auch als GitHub-Description gesetzt.
- Marker `<!-- BADGES:START/END -->` machen spätere Badge-Updates idempotent
  scriptbar.
- Umsetzung per Contents-API auf Default-Branch, Commit-Message:
  `docs: standardize README header with badge row`.
- Reihenfolge: Testlauf auf `hello-world`, Sichtprüfung, dann restliche 13.
