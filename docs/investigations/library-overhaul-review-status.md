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
| Extra | Bitrate in bps (Jellyfin) wird für Upgrades in kbps bewertet |
| Tests | Veraltete Tests angepasst; Blocklist-Test als `[dev]`-Commit |

Bewusst unverändert:
- **L2:** Migrierte Profile bekommen `none`, damit es keine Upgrade-Welle gibt.
- **L6:** Der Cover-Job meldet nur reparierbare Lücken.
- **L9:** Die übrigen Root-Dateien stammen von Upstream.
- **L10:** Nur zur Info, keine Aktion.

## Offen

1. **M7: Einstellung „Extern gelöschte Dateien neu laden“ (Standard: an).**
   Entschieden, aber noch kein Code.
   - Ansatz: Config-Key, z. B. `library.redownload_externally_deleted`.
   - `core/library2/scan.py` sammelt in `rescan_files` die Tracks, die
     `missing_confirmed` geworden sind (`presence`, `on_presence_change`).
   - Ist die Option aus: für diese Tracks, wenn keine nutzbare Datei mehr da ist,
     `monitor_rules.record_rule(conn, 'track', id, False, PROVENANCE_FILE)`, dazu
     `lib2_tracks.monitored=0` und `recompute_wanted`.
   - Settings-UI: Schalter unter Library.
2. **H7/H10 Enhanced View im Artist-Detail:** Library-Artists werden auf `/library`
   umgeleitet, die Seite ist auf diesem Branch also unerreichbar. Die Dateien kommen
   über dev-Merges, deshalb nicht angefasst, um Merge-Konflikte zu vermeiden.
   Tag-Preview, ReplayGain, Source-Info und Sync rufen dort weiter entfernte Routen auf.
3. **Release Notes:**
   - Migrationsdauer (ca. 9 min bei 200k Tracks; Downloads und Automationen in der
     Zeit pausiert).
   - Papierkorb statt Löschen.
   - Extern gelöschte Dateien werden neu geladen (Lidarr-Verhalten).
   - Hauptdatei folgt dem Profil.
4. **Kompletter Testlauf zum Schluss fehlt.** Zuletzt:
   - vitest: 10.011 von 10.012 bestanden; der Fehlschlag ist A07 unter Volllast,
     danach mit längerem Timeout versehen.
   - library2: alle 18 Fehlschläge behoben, aber kein Gesamtlauf mehr danach.
   - Unter Python 3.13 stürzen die echten Analysen in `tests/sample` im Kindprozess ab
     (erwartet); mit 3.11 wie in CI laufen alle durch.

   Vorschlag: `PYTHON=.venv/bin/python bash scripts/run_tests_chunked.sh`
   mit Python 3.11 und `cd webui && npx vitest run`.

## Hinweise
- Commits mit dem Präfix `[dev]` sind unabhängig vom Overhaul und lassen sich per
  Cherry-Pick nach `dev` übernehmen.
- Nicht nach `library-overhaul` gepusht; das ist deine Entscheidung (Fast-Forward
  möglich).
