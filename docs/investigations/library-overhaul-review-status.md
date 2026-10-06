# Library-Overhaul-Review: Stand der Fixes (2026-10-06)

Branch: `claude/exciting-feynman-5k0eby`. Er baut auf `origin/library-overhaul` auf
und lässt sich per Fast-Forward übernehmen. Die Findings selbst stehen in
`library-overhaul-review-2026-10-06.md` (C1, H1–H11, M1–M12, L1–L13).

## Erledigt

| Finding | Commit / Ergebnis |
|---|---|
| C1 Offline-NAS | Leerer Mountpoint gilt nicht als gesunder Root; ein Massen-Verschwinden wird als Ausfall gewertet, nicht als Löschung |
| H1/H2 Wishlist-Clear, Löschen | Owned Tracks bleiben monitored, leere Singles/Alben werden freigegeben; Checkbox „Don't download again“ (Standard aus) |
| H3/M4/M6/L1 Migration | Alt-Server-Zeilen werden gemergt, verwaiste Tracks mit Datei gerettet, Profile ohne eigene Bibliothek übernommen, keine Phantom-Singles |
| H5/M1/M2 Performance | Reconciler-Lookup ca. 90 ms statt 21 s, Queue im Leerlauf 0,1 ms, Hourly-Reconcile 66 s → 18,5 s |
| H6/M3 Jobs | Der Toggle setzt bzw. stoppt den Timer sofort (Musik und Video); alte Job-IDs werden gemappt, System-Automationen umgeschrieben bzw. entfernt |
| H8 | Der Track-Nummer-Fix prüft nur, was er geschrieben hat |
| H9 | Preview-Cleanup nutzt die Dateilänge (`duration_ms` im Tag-Cache, einmal per Mutagen-Header nachgelesen) |
| H11 | Ein exakter Stem-Match gewinnt vor Fuzzy-Treffern |
| M8 | Album-Jahr-Fix benennt den Ordner zurück, wenn das DB-Update scheitert |
| M9 | `suspect_album_tag`: „1 von N“ nur zusammen mit einem anderen Signal, Singles und eigene Greatest Hits ausgenommen |
| M12 `[dev]` | Live/Commentary-Cleaner: interlude/introduction sind keine Löschsignale mehr (cherry-pickbar) |
| H4 | `/api/library/manual-match` und `clear-match` schreiben über lib2 (`apply_manual_match`), admin-only, hinter der Migrations-Barriere |
| H7 (Teil) | Redownload-Metadatensuche auf lib2 wiederhergestellt (Tools, Operations, Album-Upgrade) |
| M10 | Papierkorb überall: Library-Delete, alle Repair-Fixes, Upgrade-, Enhance- und Redownload-Ersatz |
| M11 | Audioanalyse läuft in einem Kindprozess (`core/sample/isolated.py`); ein numpy-Pin hilft nicht |
| M5 | Globaler Banner bei laufender oder fehlgeschlagener Migration, mit Retry |
| L3 | Toter Legacy-Code in `embedded_id_reconcile` entfernt |
| L4 | Hauptdatei nach Qualitätsprofil (`profile_rank`); eine manuelle Wahl gewinnt |
| L5 | Tag-Vorschau zeigt auch Geschwisterdateien |
| L7 | Wishlist-Grund: „Missing from library“ bzw. „Quality upgrade wanted“ |
| L8 | Meldung der Automatic Search ist lesbar |
| L12 | Der Enrichment-Sweep importiert `web_server` nicht mehr |
| M7 | Schalter „Re-download files deleted outside SoulSync“ (`library.redownload_externally_deleted`, Standard an) unter Settings → Library Organize. Aus: der Scan schreibt für jeden Track ohne verbleibende Datei dieselbe explizite Unmonitor-Regel wie „Don't download again“ (`user_explicit`, damit ein Album-Toggle sie nicht überstimmt) |
| Extra | Bitrate in bps (Jellyfin) wird für Upgrades in kbps bewertet |
| Tests | Veraltete Tests angepasst; Blocklist-Test als `[dev]`-Commit |

Bewusst unverändert:
- **L2:** Migrierte Profile bekommen `none`, damit es keine Upgrade-Welle gibt.
- **L6:** Der Cover-Job meldet nur reparierbare Lücken.
- **L9:** Die übrigen Root-Dateien stammen von Upstream.
- **L10:** Nur zur Info, keine Aktion.

## Bewusst offen

- **H7/H10 Enhanced View im Artist-Detail:** geprüft und unerreichbar:
  `$source/$id.tsx` leitet jede `library`-URL auf `/library` um. Die Dateien kommen
  über dev-Merges, deshalb nicht angefasst, um Merge-Konflikte zu vermeiden.
  Tag-Preview, ReplayGain, Source-Info und Sync rufen dort weiter entfernte Routen auf.

## Release Notes (Entwurf für PR #1062 / Discord)

```
- The first start upgrades the library: about 9 minutes per 200k tracks on a local
  disk, longer on a NAS. Downloads, sync and automations wait until it is done; a
  banner on every page shows the progress and offers a retry if it fails.
- Nothing is deleted outright anymore. The Library delete dialog, every repair fix
  and the file an upgrade, enhance or redownload replaces go to the deleted-files
  folder and can be restored until retention clears them.
- A monitored track whose file was deleted outside SoulSync is downloaded again,
  like Lidarr does. Settings > Library Organize > "Re-download files deleted outside
  SoulSync" turns that off.
- When a track has several files, the quality profile decides which one is primary.
  A file you picked by hand stays primary.
```

## Testlauf

Lokal am 2026-10-06, Python 3.14, in einem Worktree ohne `config/config.json`
(im Haupt-Checkout gehen einige Download-Tests mit der echten Konfiguration ans Netz
und hängen):

- **vitest:** 10.012 von 10.012 bestanden; `npm run check` 0 Fehler.
- **pytest (chunked):** alles grün bis auf
  - `test_filter_keys_match_write_path_keys`: auch auf `upstream/dev` rot;
  - `test_vanilla_globals_resolve` (35×): Worktree ohne `webui/node_modules`, im
    Haupt-Checkout 35/35 grün;
  - `tests/imports/test_rematch_hints_seam.py` (5×): echte Regression aus L4, die
    handgebaute Tabelle hatte keine `profile_rank`-Spalte; behoben (2e0dcf0b7),
    `tests/imports` danach 1262/1262.
- `tests/sample` (188) läuft auch mit 3.14 komplett durch.

## Dev-Branch

`fix/dev-bugs-from-overhaul-review`, nur lokal (Worktree `../SoulSync-devfix2`),
Basis `upstream/dev` 24c5ef0ef (3.5.1). Jeder Fix ist gegen dev geprüft, jeder Test
schlägt ohne den Fix auf dev fehl:

1. Live/Commentary-Cleaner löscht keine Interludes/Introductions mehr (M12)
2. Exakter Stem-Match gewinnt vor Fuzzy (H11); auf dev hing das Ergebnis von der
   Verzeichnisreihenfolge ab, der neue Test erzwingt beide Reihenfolgen
3. Tools-Job-Toggle setzt bzw. stoppt den Timer sofort (H6, ohne die fork-eigene
   Job-ID-Umbenennung)
4. Album-Jahr-Fix benennt den Ordner zurück, wenn das DB-Update scheitert (M8)
5. `suspect_album_tag`: „1 von N“ nur noch als Zusatzsignal, Singles und eigene
   Greatest Hits ausgenommen (M9; dev nutzt `albums.record_type`)

Volltest auf dem Dev-Branch: grün bis auf `test_filter_keys_match_write_path_keys`
(auf dev schon rot), `test_vanilla_globals_resolve` (kein `node_modules` im Worktree)
und `test_title_match_guard::test_single_word_spelling_variants_not_regressed`
(Titel-Matching, von keinem Commit berührt, allein grün: Reihenfolge-Flake).

Nicht übernommen:
- Blocklist-Test: auf dev ohne Änderung grün.
- M11 (Analyse im Kindprozess): lokal mit Python 3.14 / numpy 2.5.3 kein Absturz,
  Docker nutzt 3.11; ohne Nachweis keine Architekturänderung für dev.
- M10 (Papierkorb überall): Verhaltensänderung, hängt am lib2-Delete-Journal.
- Bitrate in bps: dev misst für Upgrades die echte Datei per mutagen.
- H8, H9, L5, L7, L8, L12, L13: nur Library v2.

## Hinweise
- Commits mit dem Präfix `[dev]` sind unabhängig vom Overhaul; der Dev-Branch oben
  enthält die übernommenen Fixes ohne Präfix.
- Nichts gepusht, keine PR.
