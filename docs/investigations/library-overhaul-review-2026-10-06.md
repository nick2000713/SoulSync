# Review: PR „Library overhaul“ (nick2000713/SoulSync#1, `library-overhaul` → `dev`)

Stand: 2026-10-06 · Head `0094b3a7` · Umfang: 2718 Dateien, +532k / −84k Zeilen, 1357 Commits

---

## Kurzfazit

Die **Migration selbst** (Import der Altdaten) ist robust. Crash und Resume sind an jedem Checkpoint verifiziert, kaputte Daten lassen sie nicht abstürzen, und ein zweiter Start ist ein No-Op. Die schweren Probleme liegen **danach**, im Zusammenspiel von Monitoring, Wishlist, Jobs und Dateisystem.

- **Kritisch:** Ist das NAS beim Scan nicht gemountet (leerer Mountpoint), landet nach zwei Tagen die **komplette Bibliothek in der Wishlist** und wird neu heruntergeladen (C1, reproduziert).
- **Monitoring-Semantik weicht von Lidarr und Nutzererwartung ab:** „Clear All“ demonitored vorhandene Tracks (Upgrades sterben), leere Singles bleiben monitored, und Löschen in der Bibliothek lädt sofort neu herunter.
- **Tools:** Der Track-Nummer-Fix gilt praktisch nie als erfolgreich. Der Preview-Detektor ist durch lib2 blind geworden. Die Reorganize-Vorschau in der Artist-Ansicht zeigt etwas anderes als das Apply. `suspect_album_tag` produziert False Positives. Die Dateisuche beim Download importiert die Live-Version statt des Originals.
- **CI ist rot:** 20 Python-Tests (überwiegend veraltete Tests nach den letzten Commits, 1 echter Bug) und 1 Vitest.

**Empfehlung: Nicht mergen, bevor C1 und H1–H11 behoben sind.**

---

## Wie geprüft wurde

| Was | Ergebnis |
|---|---|
| **Upgrade-Simulation:** DB mit `dev`-Code erzeugt, realistisch befüllt (2 Media-Server, Compilation, Unicode, toter Pfad, Wishlist, Watchlist, Findings, Automation, 2 Profile), mit PR-Code migriert | erfolgreich, Details unten |
| **Crash-Test:** Prozess per `os._exit` an jedem Checkpoint gekillt, neu gestartet | identisch mit sauberem Lauf ✅ |
| **Kaputte Legacy-Daten** (dangling ids, leere Titel, Text-IDs, kaputtes JSON) | kein Absturz ✅, aber stilles Verwerfen (M4) |
| **Große Bibliothek** 2k/20k/200k | Migration ≈ 8 min 48 s |
| **Alle 35 Tools-Jobs** über den echten `RepairWorker._run_job` auf einer migrierten DB mit **echten Audiodateien** (ffmpeg-generiert: falsche Tags, falsche Tracknummer, 3-s-Preview, MP3-als-FLAC, abgeschnittene Datei, Live-Track, Orphan, leerer Ordner, MP3+FLAC-Geschwister) | mehrere Bugs, siehe H8–H10, M9–M12 |
| **Fix-Aktionen** ausgeführt (track_number, empty_folder, orphan, path_mismatch, unwanted_content, dead_file) und Platte + DB nachgeprüft | H8 |
| **Retag** Vorschau → Schreiben → Vorschau mit FLAC+MP3 | funktioniert ✅ (L-Hinweis) |
| **Reorganize** Vorschau (neu vs. Artist-Ansicht) gegen Apply-Pfad | H10 |
| **Offline-NAS-Simulation** (Mountpoint leer) mit `dead_file_cleaner` + `monitoring_list_reconcile` | **C1** |
| **Wishlist „Clear All“, Library-Delete, Watchlist-Remove** auf der migrierten DB | H1, H2 |
| **Benchmarks** (400k Tracks): Worker-Queue, Library-Listen, Acquisition-Reconciler, Hourly-Reconcile, Artwork | H5, M1, M2; Library-Laden ok |
| Statische Analyse (pyflakes + ruff-Bugregeln, 623 Python-Dateien) | sauber ✅ |
| Frontend↔Backend-Routen-Abgleich, entfernte Routen dev→PR | H7 |
| Frontend `tsc`, `vitest`, `build` / Backend `scripts/run_tests_chunked.sh` (wie CI) | siehe „Tests“ |

**Nicht vollständig geprüft:** CSS, Video/Podcast/Audiobook/Sample-Studio im Detail, jede einzelne React-Komponente, Downloads mit echten Clients (Soulseek/Usenet/Tidal; im Sandbox kein Netz).

---

## 🔴 Kritisch

### C1 · Offline-NAS: komplette Bibliothek wird als fehlend bestätigt und neu heruntergeladen
**Ort:** `core/library2/paths.py:230` (`missing_path_root_is_healthy`), dazu `core/library2/scan.py:36` (`MISSING_CONFIRMATION_SCANS = 2`), `core/repair_jobs/dead_file_cleaner.py`, `core/library2/monitor_sync.py` (`reconcile_track_wishlist`)

**Reproduziert:** Musikordner weg, Mountpoint-Verzeichnis leer (der typische Zustand bei nicht gemountetem NFS/SMB/USB in Docker). Danach `dead_file_cleaner` zweimal laufen lassen (täglich, **standardmäßig an**) und dann `monitoring_list_reconcile` (stündlich, **standardmäßig an**).
Ergebnis: `file states: missing_confirmed × 9/9` → `wishlist rows: 9` → die Wishlist-Verarbeitung lädt **die gesamte Bibliothek neu**.

**Ursache:** „Gesund“ heißt nur `os.path.isdir(root)`. Ein leerer Mountpoint besteht diesen Test. Genau den Fall soll der Code laut eigenem Kommentar verhindern („An unmounted share makes every one of its files look deleted“). Dass nach der Migration **jeder vorhandene Track monitored** ist, macht das zur Download-Lawine.

**Fix (mehrstufig):**
1. Ein Root gilt als ungesund, wenn er leer ist oder wenn ein großer Anteil (z. B. > 10 %) seiner katalogisierten Dateien in einem Scan fehlt. In dem Fall den Lebenszyklus nicht weiterschalten (Circuit Breaker) und den Admin benachrichtigen.
2. Optional: Marker-Datei (z. B. `.soulsync-root`) oder `os.path.ismount` für konfigurierte Roots.
3. Ein Massen-Zugang zur Wishlist durch einen Reconcile (z. B. > 50 Tracks auf einmal) erfordert eine Bestätigung.

---

## 🔴 Hoch – vor dem Merge beheben

### H1 · Wishlist „Clear All“ demonitored vorhandene Tracks und lässt leere Singles/Alben monitored *(dein Fund, reproduziert)*
**Ort:** `database/music_database.py:14711` (`clear_wishlist`), gleiche Logik beim Einzel-Entfernen: `core/library2/monitor_sync.py:442` (`demonitor_lib2_tracks_for_removed_wishlist`)

- Jeder Track in der Wishlist bekommt `monitored=0` plus eine User-Regel. Das gilt auch für Tracks **mit Datei**, die nur als **Upgrade-Kandidat** drin stehen. Ergebnis `wanted=0, reason=track_explicit`: **Upgrades sind danach dauerhaft aus.**
- Das **Album-/Single-Flag bleibt unverändert.** Die Single „Debut“ (0 Dateien, alle Tracks demonitored) bleibt `monitored=1` und sichtbar, weil die Sichtbarkeit „Dateien ODER Intent“ ist.
- **Zu deiner Frage:** Ja, Tracks unter dem Cutoff ihres Profils landen in der Wishlist (`upgrade_candidate_track_ids` → `monitoring_list_reconcile`), sofern die `upgrade_policy` Upgrades erlaubt.

**Soll-Verhalten (Lidarr):** Clear/Entfernen entfernt nur den Download-Wunsch. Tracks **mit Datei** bleiben monitored; für den aktuellen Upgrade-Versuch genügt ein Ignore- oder Snooze-Eintrag. Tracks ohne Datei werden demonitored. Danach jedes betroffene Album bzw. jede Single prüfen: keine Datei, kein gewollter Track und Intent nicht vom Nutzer gesetzt (Provenance ≠ `user`) ergibt `album.monitored=0`.

### H2 · Löschen in der Bibliothek löst sofortigen Re-Download aus
**Ort:** `api/library_v2.py:4176` (`_reproject_after_file_removal`), Dialog in `webui/src/routes/library/-ui/library-v2-page.tsx` (~4174)

Nach der Migration ist jeder vorhandene Track monitored (`track_rule:file_import`). Die Single „Creep“ in der Bibliothek zu löschen erzeugte **sofort einen neuen Wishlist-Eintrag**. Auf `dev` war Löschen endgültig. Der Dialog sagt nur „Monitoring and Wanted state are recalculated“.
**Fix:** Eine Checkbox „Nicht erneut herunterladen (Unmonitor)“, bei ganzen Alben/Artists standardmäßig an (wie Lidarrs „Delete files + Unmonitor“).

### H3 · Migration übernimmt Altdaten inaktiver Media-Server → doppelte Alben, dieselbe Datei an zwei Tracks
**Ort:** `core/library2/importer.py` (kein Filter auf den aktiven Server, `_server_identity` :344)

`dev` zeigte nur den aktiven Server (`get_library_artists`: `a.server_source = active_server`), und `clear_server_data` löscht absichtlich nur den aktuellen Server. Wer je den Server gewechselt hat, hat unsichtbare Alt-Zeilen. Der Importer übernimmt alle. In der Simulation erschien „OK Computer“ zweimal, und **dieselbe Datei** hing an zwei `lib2_track_files`-Zeilen. `dedup_repair` meldete nur `album_review: 1`.
**Fix:** Inaktive Server nur übernehmen, wenn die Datei nicht schon vom aktiven Server belegt ist (oder per Pfad mergen), plus Dedup bzw. Unique-Regel auf `lib2_track_files.path`.

### H4 · Manueller Match / „Clear Match“ schreiben mit lib2-IDs in die Legacy-Tabellen
**Ort:** `web_server.py:11013` (`library_manual_match`), `:11120` (`library_clear_match`); Aufrufer `webui/static/enrichment-manager.js:1406`, `webui/src/routes/artist-detail/-artist-detail.enrich-match.ts:148/177`

Der „Unmatched“-Browser liefert lib2-IDs (`core/enrichment/unmatched.py` liest `lib2_*`), der Endpoint führt `UPDATE artists|albums|tracks … WHERE id=<lib2-id>` aus. Folge: 404, oder die **Provider-ID landet auf einer fremden Legacy-Zeile**; die UI meldet trotzdem „Matched ✓“. Die Endpoints stehen auch nicht hinter der Migrations-Barriere.
**Fix:** Auf `PUT/DELETE /api/library/v2/<entity>s/<id>/manual-match` umstellen (existiert bereits).

### H5 · Acquisition-Monitor scannt alle 15 s die komplette Bibliothek im Dateisystem
**Ort:** `core/acquisition/reconciler.py:86` (`_known_index_paths`), aufgerufen aus `:584`; Trigger `core/acquisition/client_monitor.py` (`run_once`, 15 s, Monitor startet immer)

Bei jedem Lauf werden alle `lib2_track_files`-Pfade geladen und einzeln über das Dateisystem aufgelöst, **auch bei 0 offenen Grabs**. Gemessen **21,8 s pro Aufruf bei 400k Dateien** (lokale Disk; auf einem NAS viel mehr).
**Fix:** Bei 0 offenen Grabs sofort zurückkehren; sonst nur die Pfade der betroffenen Grabs prüfen.

### H6 · Job im Tools-Tab einschalten → bis zum Neustart keine geplanten Läufe
**Ort:** `core/repair_worker.py:731` (`set_job_enabled`)

Der Toggle setzt nur `automations.enabled`, ruft aber nie `engine.schedule_automation()` bzw. `cancel_automation()` auf (anders als `core/automation/api.py:245`). Die meisten Jobs werden deaktiviert geseedet, ein Einschalten wirkt also erst nach einem Neustart. Aus und wieder an: Der Timer feuert, sieht `enabled=0` und kehrt ohne Neuplanung zurück; danach läuft der Job nie wieder.

### H7 · Entfernte Backend-Routen werden von der Admin-„Enhanced View“ noch aufgerufen (404)
**Ort:** `webui/src/routes/artist-detail/-artist-detail.tags-rg.ts`, `-artist-detail.redownload.ts:65`, `-artist-detail.manage-actions.ts:100`, `-ui/artist-meta-panel.tsx:87`, `-ui/enhanced-bulk-bar.tsx:36`

Betroffen: `track/<id>/tag-preview`, `…/write-tags`, `tracks/tag-preview-batch`, `tracks/write-tags-batch`, `track|album/<id>/analyze-replaygain`, `tracks/analyze-replaygain-batch`, `track/<id>/redownload/search-metadata`, `track/<id>/source-info`, `artist/<id>/sync`. Alle sind im PR entfernt; übrig sind nur verwaiste `/status`-Routen.
**Fix:** Auf die lib2-Endpoints umstellen (`/api/library/v2/<entity>/<id>/tag-preview`, `/api/library/v2/tracks/<id>/source-info` …) oder die Buttons entfernen.

### H8 · Track-Nummer-Fix wird praktisch nie als erfolgreich verbucht
**Ort:** `core/library2/validation.py:249` (`finding_verified`, Zweig `track_number_mismatch`), Aufruf in `core/repair_worker.py:2448`

**Reproduziert:** Der Fix schreibt `2/3` korrekt (`track_number: correct`), die Verifikation verlangt aber zusätzlich `disc_number == 'correct'` und `total_tracks == 'correct'`. Der Fix schreibt keine Disc-Nummer, und `total_tracks` liest der Validator als `unknown`. Ergebnis: `success: false`, „Repair is incomplete or could not be verified; finding remains pending“. Bei **jeder Datei ohne Disc-Tag** bleibt das Finding ewig offen, und „Fix All“ meldet nur Fehlschläge.
**Fix:** Nur die Felder verifizieren, die der Fix ändert (`track_number`, und `total_tracks` nur, wenn er sie schreibt). Alternativ schreibt der Fix `discnumber` und `totaltracks` mit.

### H9 · `short_preview_track` ist durch lib2 blind geworden
**Ort:** `core/repair_jobs/short_preview_track.py:146-149`, Datenquelle `core/library2/maintenance_subjects.py:84` (`t.duration`)

Der Job filtert über `lib2_tracks.duration ≤ 30 s`. In lib2 ist das die **erwartete Provider-Dauer** (Tracklisten aus `core/library2/completeness.py:563`), nicht die Dateilänge; `lib2_track_files` hat gar keine Dauer-Spalte. Eine 30-s-Preview an einem Track mit 238 s Katalogdauer wird **nie geprüft**, und das ist genau der Fall, für den es den Job gibt. Im Test: 3-s-Datei, Katalog 238 s → `scanned=0`. Auf `dev` kam `tracks.duration` vom Media-Server (also aus der Datei).
**Fix:** Dateidauer in `lib2_track_files` speichern (beim Import oder Tag-Scan) und darauf filtern; zusätzlich Datei- gegen Katalogdauer vergleichen.

### H10 · Reorganize-Vorschau in der Artist-Ansicht ≠ Apply
**Ort:** `webui/src/routes/artist-detail/-artist-detail.reorganize.ts:201` → `web_server.py` `/api/library/album/<id>/reorganize/preview` → `core/library_reorganize.preview_album_reorganize` (**Provider-Planer**). Das Apply läuft über die Queue → `core/reorganize_runner.py:294` mit `catalogue_preview_fn` (**Katalog-Planer**).

**Belegt:** Für dasselbe Album liefert die neue Vorschau `planned` mit Zielpfaden, die Artist-Ansicht dagegen `no_source_id` und leere Pfade. Mit konfigurierten Providern kann die Vorschau andere Pfade zeigen, als das Apply tatsächlich verschiebt.
**Fix:** Die Artist-Ansicht auf `/api/library/v2/albums/<id>/reorganize/preview` umstellen und den Legacy-Preview entfernen.

### H11 · Download-Dateisuche greift „ANGEL (Live).flac“ statt „ANGEL.flac“
**Ort:** `core/downloads/file_finder.py` (Fix #1366 in diesem PR); der eigene Test `tests/downloads/test_file_finder.py::test_extensionless_encoded_title_prefers_closest_stem` ist **rot**

Liegen nach einem Download zwei ähnliche Dateien im Ordner (z. B. bei einem Album- oder Discography-Download), wählt die Suche für den Key `id||PRYVT - ANGEL` die **Live-Version**, die dann importiert und getaggt wird. Eine frühere Match-Stufe (Substring) gewinnt vor dem exakten Stem-Vergleich.
**Fix:** Ein exakter Stem-Match muss vor allen Fuzzy- und Substring-Stufen gewinnen.

---

## 🟠 Mittel

| # | Problem | Ort | Beleg / Fix |
|---|---|---|---|
| **M1** | Enrichment-Queue: Leerlauf = Full-Scan pro Worker und Tick | `core/library2/worker_queue.py:115` | 0,52 s/Aufruf bei 400k; 16 Worker × alle 10 s ≈ 0,8 Kerne Dauerlast. Fix: Cursor „max. geprüfte id“. |
| **M2** | Stündlicher `monitoring_list_reconcile` braucht ≈ 66 s ohne Änderungen | `core/library2/monitor_sync.py:1015` | Alle vorhandenen monitored Tracks werden einzeln gemirrort. Fix: Adds in SQL auf „ohne nutzbare Datei“ + Upgrade-Kandidaten vorfiltern. |
| **M3** | `run_repair_job` mappt alte Job-IDs nicht; Automationen entfernter Jobs bleiben | `core/automation/handlers/run_repair.py:33` | `JOB_ID_MIGRATIONS` vor der Prüfung anwenden; Retired-Automationen aufräumen. *(Betrifft nur frühe `:library2`-Image-Nutzer.)* |
| **M4** | Legacy-Zeilen mit verwaisten Referenzen werden still verworfen | `core/library2/importer.py:1962`, `:1681` | Tracks mit echter Datei verschwinden ohne Log; ihre Dateien werden „Orphans“ mit „Löschen“-Option. Fix: Platzhalter importieren oder loggen und anzeigen. |
| **M5** | Fehlgeschlagene Migration hält die App unbegrenzt im Upgrade-Modus | `core/library2/migration_gate.py` | 105 Endpoints → 409, Automationen aus; der Fehler steht nur auf der Library-Seite. Fix: globaler Banner + Retry. |
| **M6** | Mehrbenutzer: Watchlist/Wishlist von Profil ≠ 1 fließen nicht in lib2-Monitor-Regeln | Importer | Bewusst entscheiden und dokumentieren. |
| **M7** | Außerhalb von SoulSync gelöschte Dateien werden automatisch neu geladen | `core/library2/scan.py:36` | Lidarr-konform, aber neu gegenüber `dev`; in die Release Notes oder als Option. |
| **M8** | Album-Jahr-Reparatur: Ordner umbenannt, DB-Update ohne Rollback | `core/repair_jobs/album_release_year_repair.py:317-365` | Bei DB-Fehler zurückbenennen oder Move-Journal nutzen. |
| **M9** | `suspect_album_tag` (neuer Job) mit False Positives | `core/repair_jobs/suspect_album_tag.py` | Im Test markiert: Compilation mit Album-Artist „Various Artists“ (prüft Track- statt Album-Artist), Single 1/1 („einziger Track“), 1-von-N-Alben (bei Wishlist-Downloads der Normalfall). |
| **M10** | Papierkorb wird umgangen | `core/imports/pipeline.py:2450` (Upgrade: `os.remove`), Library-Delete (`unlink`), Repair-Fixes `unwanted_content`/`short_preview`/`acoustid_mismatch`/`lossy_converter` | Nur `corrupt_audio` und `fake_lossless` nutzen `.deleted`. Lidarr legt ersetzte Dateien in den Recycle Bin. Fix: einheitlich über den Papierkorb (konfigurierbar). |
| **M11** | Native Audioanalyse im Server-Prozess | `core/repair_jobs/bpm_backfill.py` → `core/sample/analyze.py` (librosa/numba) | Mit numpy 2.5 (Python ≥ 3.12) **Segfault, der den ganzen Server killt** (reproduziert). Docker/CI (3.11, numpy 2.4) ok. numpy/numba sind ungepinnt. Fix: pinnen und/oder Analyse im Subprozess. |
| **M12** | Live/Interview-Heuristik löscht endgültig *(bestand schon auf `dev`)* | `core/repair_jobs/live_commentary_cleaner.py` | `\binterlude\b` gilt als Interview, `\bintroduction\b` als Spoken Word (Klassik!), Live-Alben werden über den Albumtitel komplett markiert; „Fix All“ löscht ohne Papierkorb. |

---

## 🟡 Niedrig / Info

- **L1 Phantom-Single:** Ein Wishlist-Eintrag ohne Album-ID/-Typ erzeugt ein neues Album mit Typ `single` (mit vollständigem Payload korrekt).
- **L2 `upgrade_policy`-Default** `acceptable` → `none` (`core/quality/migrate_to_profiles.py:50`): Absicht bestätigen.
- **L3 Toter Legacy-Code:** `core/library/embedded_id_reconcile.py:83/438`.
- **L4 Primäre Datei** wird fest „lossless zuerst“ gewählt (`core/library2/track_files.py:71`), nicht nach dem Qualitätsprofil; Ersetzen beim Import entscheidet dagegen nach Profil (#1270).
- **L5 Retag-Vorschau** zeigt nur die primäre Datei. Falsche Tags einer MP3-Geschwisterdatei sind unsichtbar, werden aber überschrieben.
- **L6 `missing_cover_art`** meldet nur, wenn ein Provider ein Cover findet; ohne Netz/Treffer bleibt ein Album ohne Cover unsichtbar.
- **L7 Upgrade-Kandidaten** stehen in der Wishlist mit `failure_reason = "Download failed"`, obwohl nie ein Versuch lief.
- **L8 „Automatic Search“**-Meldung: `` `${started} Missing tracks…` `` ergibt „Wishlist processing started Missing tracks and …“.
- **L9 Repo-Hygiene:** `pr.md`, `pr_description.md`, `pr discord.md`, `IMPORT_PAGE_PLAN.md`, `discovery.md`, `VIDEO_SIDE_BEST_IN_CLASS_ROADMAP.md` im Root; `library-v2-page.tsx` hat **12.165 Zeilen**; `web_server.py` und `music_database.py` je ≈ 24k Zeilen.
- **L10 Image-Größe:** `librosa`/`onnxruntime` für alle Nutzer.
- **L11 Migrationsdauer:** ≈ 9 min bei 200k Tracks lokal (NAS länger); in dieser Zeit sind Downloads, Sync und Automationen gesperrt. Gehört in die Release Notes.
- **L13 Import ohne Cover:** `enhance_file_metadata` gibt `False` zurück, wenn kein Cover gefunden wird → Downloads werden als „nicht angereichert“ markiert (siehe Tests).
- **L12** Ein Repair-Job importiert beim Lauf `web_server` (die ganze App mit allen Workern startet). Im Produktivbetrieb unkritisch, macht Jobs aber schwer isoliert testbar.

---

## ✅ Was gut gelöst ist (verifiziert)

- **Migration:** Crash/Resume identisch, idempotent, robust gegen Datenmüll; `missing_suspected` bei unerreichbaren Pfaden (aber siehe C1); keine Downloads direkt nach dem Upgrade; Post-Import holt Tracklisten nur für unvollständige, nicht monitored Alben.
- **Retag:** Vorschau → Schreiben → leere Vorschau, inklusive MP3-Geschwisterdatei.
- **Reorganize (neue Library-Seite):** Vorschau und Apply nutzen denselben Katalog-Planer; Copy-then-Delete; Findings werden mit umgezogen.
- **Mehrere Versionen pro Track:** Trigger garantieren genau eine primäre Datei, manuelle Wahl gewinnt, Verschieben nimmt Geschwisterdateien mit.
- **Upgrade beim Import:** Snapshot-Prüfung, Längen-Schutz gegen Previews, Profil-basierter Vergleich, Rollback, falls die alte Datei nicht entfernt werden kann.
- **Watchlist entfernen** demonitored nur die Artist-Ebene; vorhandene Tracks bleiben für Upgrades monitored.
- **Riskante Tools** sind standardmäßig aus oder im Dry-Run.
- **Library-Laden:** Listen 40–90 ms bei 400k Tracks (warm); Artwork mit 256-px-Thumbs, `immutable`-Cache plus `?v=mtime`, Cold-Path asynchron, `loading="lazy"`. **Hier ist kein Performance-Problem.**
- **Migrations-Barriere:** alle 105 Endpoint-Namen gültig.

---

## Empfohlene Reihenfolge

1. **C1** (Massen-Re-Download bei Offline-NAS): zuerst, denn es betrifft jeden NAS-Nutzer.
2. **H1, H2** (Monitoring-Semantik bei Clear, Entfernen und Löschen) und **H3** (Dubletten durch alte Server-Zeilen).
3. **H8, H9, H10, H11** (Tools, die falsch oder gar nicht arbeiten).
4. **H4, H6, H7** (harte Bugs, kleine Fixes) und **H5, M1, M2** (Performance).
5. CI grün machen (veraltete Tests aktualisieren), danach M- und L-Punkte, Release Notes.

---

## Tests

**Backend** (`scripts/run_tests_chunked.sh`, wie CI; lokal Python 3.13): **24.597 bestanden, 20 fehlgeschlagen.** `tests/sample` stürzt unter Python 3.13 mit dem numba-Segfault ab (siehe M11); unter 3.11 nicht ausgeführt.

| Test | Bewertung |
|---|---|
| `tests/library2/test_artist_search_query.py` (9), `test_artist_alpha_sort.py` (2), `test_artist_rollup.py` (1) | **Veraltet** durch den letzten Commit `3827b022` (`list_artists` zeigt nur noch Artists mit Datei oder Monitor-Intent); die Tests legen nackte Artists an. Verhalten passt zu H1-Wunsch, Tests anpassen. |
| `tests/library2/test_two_libraries.py::…reorganize_plans…` (2) | Veraltet: Fixture mit `track_number=0`, der neue Planer verweigert das bewusst. |
| `tests/downloads/test_file_finder.py::test_extensionless_encoded_title_prefers_closest_stem` | **Echter Bug → H11** |
| `tests/downloads/test_redownload_search_sources_stream.py::…` | Flaky (einzeln grün) |
| `tests/test_enrichment_tag_preservation.py::test_core_tags_written_on_happy_path_artless_file` | Verhaltensänderung: `enhance_file_metadata` gibt bei fehlendem Cover jetzt `False` zurück (`core/metadata/enrichment.py`, `art_ok`). Folge: Jeder Download ohne Cover gilt als „Metadaten nicht angereichert“. Entscheiden: Test oder Code anpassen. |
| `tests/repair_jobs/test_section27_repair_findings.py::test_a_single_edition_album_is_unchanged` | Veraltet (neues Feld `disc_number` im Ergebnis) |
| `tests/blocklist/test_blocklist_api.py::test_search_proxies_active_source` | Veraltet: patcht `web_server._search_service`, die Route liegt jetzt in `api/discover_routes.py` |
| `tests/database/test_discovery_pool_playlist_filter.py::test_filter_keys_match_write_path_keys` | Reihenfolgeabhängig, identisch mit `dev` |

**Frontend:** `tsc --noEmit` ✅, `npm run build` ✅, `vitest` 10.006 / 10.010. Der Fehlschlag `src/routes/library/-route.test.tsx` „browses the library by album and opens one (A07)“ ist **deterministisch** (Button „Open Selected Ambient Works“ fehlt; bitte prüfen, ob die Album-Ansicht betroffen ist). Drei weitere Fehlschläge waren nur unter Last und verschwanden beim Einzellauf.

→ **CI für diesen PR ist rot** (Backend und Frontend).
