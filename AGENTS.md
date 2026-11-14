# FoE Tools – Kurzanleitung für Modelle (zuerst lesen)

Forge-of-Empires-PWA, drei Seiten: `index.html` (LG-Rechner + Powerlevel + Tab-Leiste), `foeff.html` (Effizienz), `city.html` (Planer).
Nutzer: Bach, schreibt Deutsch, will kurze Antworten und kleine gezielte Änderungen statt Umschreiben. Ausführlich: `docs/HANDOFF.md`.

## Regeln
- Klassische Skripte, **keine Module, kein Bundler**. Alle Dateien einer Seite teilen den globalen Bereich; die Reihenfolge steht in den `<script src>`-Tags der HTML-Seite.
- Auf Funktionsebene ändern, nie ganze Dateien neu schreiben. Prettier Breite 120 (`npm run format`).
- Neue Datei = in HTML eintragen; der Service Worker bekommt sie über `python3 tools/release.py` automatisch.
- `data/*.js` nie von Hand ändern (Prüfsummen). Nach absichtlicher Änderung: `python3 tools/data_checksums.py --update` **und** `python3 tests/calc_test.py --update`.
- `tests/calc_test.py` läuft parallel; `--changed` (so ruft es `release.py` auf) überspringt LG-Rechner/Powerlevel, wenn sich die Dateien aus `tests/calc_deps.json` nicht geändert haben (neue Skripte in `index.html` dort eintragen, sonst Fehler); `--quick` nur Stichprobe; `release.py --full` erzwingt alles. `tests/flow_test.py` (ca. 15 s) prüft die Bedienabläufe.
- Rechnung (LG-Beiträge, Effizienz, Powerlevel, Planer-Werte und Straßennetz) ändern: `tests/calc_test.py` muss rot werden; nur wenn die Änderung gewollt ist, danach `--update`.

## Wo ist was?
- LG-Rechner: `js/index-lg-{data,state,calc,render,profiles,boot}.js`; Powerlevel: `js/index-powerlevel.js` (Shadow-DOM); Kopier-Gesten: `index-copy.js`.
- Effizienz: `js/foeff-{data,core,tab-buildings,tab-weights,shared,tab-import,csv,tab-allies,tab-trials}.js` (je Tab eine Datei).
- Planer: `js/city-{core,layouts,calc,values,draw,plan,ui,init}.js` (früher eine Datei `city.js`; **`city-init.js` muss zuletzt geladen werden**, Reihenfolge = `<script>`-Tags in `city.html`), Katalog `city-data.js`. Gemeinsam: `js/i18n-t.js` (`createT`), `i18n-items.js`.
- Zeitplan-Erinnerungen (Tab „Zeitplan“, `notify.html`): `js/notify.js`, `i18n/notify.js`, `css/notify.css`; Gegenstück ist der Cloudflare-Worker in `worker/` (nicht Teil des Uploads, eigene `README.md`).
- Stile: `css/*.css`. Daten: `data/` (siehe `data/README.md`).

## Texte und Sprachen (DE/EN)
- Alle Oberflächentexte stehen in `i18n/<seite>.js` unter `texts`: `"schlüssel": {"en": "…", "de": "…"}`, Platzhalter `{n}`.
- Im Code: `tl("schlüssel", {n: 5})`; im HTML: `data-i18n`, `data-i18n-title`, `data-i18n-placeholder`. Ein fehlender Schlüssel zeigt den Schlüssel selbst an.
- **Text ändern = nur die i18n-Datei.** Neuer Text = Schlüssel dort anlegen + `tl(...)` einsetzen. Dann `python3 tests/i18n_check.py`.
- Gespeicherte Daten (Profil-/Gruppennamen wie „Profil 2“) zeigt `profName()`/`groupName()` je Sprache.
- Ausnahme: Spielnamen (Items, Events, Stats) übersetzt in foeff ein DOM-Übersetzer (`js/foeff-i18n.js`, Tabellen `stats`/`events`/`labels` in `i18n/foeff.js`, Item-Namen in `i18n/items.js`).

## Ablauf: Daten-Update (neue Spielwerte, neue Gebäude/LGs/Events)
Die Daten stammen aus Bachs eigenen Quellen und kommen als fertige Datei(en); Erzeugungsskripte gibt es nicht (`tools/legacy/` läuft nicht). Nie Datenzeilen von Hand umbauen.
1. **Alten Stand sichern:** letzten gelieferten Ordner entpacken (für `--compare`).
2. **Datei ersetzen:** `data/jX.js` (Tabelle siehe `data/README.md`) komplett durch die gelieferte Datei ersetzen. Kopfzeile muss `(window.DATA=window.DATA||{}).jX=` bleiben. Neue Gebäude für den Planer stehen in `city-data.js` (Katalog, Wurzelordner).
3. **Prüfsummen:** `python3 tools/data_checksums.py` muss rot werden (zeigt, welche Datei sich geändert hat); danach `python3 tools/data_checksums.py --update`.
4. **Rechentest:** `python3 tests/calc_test.py` – rot bei `lg_all` (LG-Kosten), `eff_all` (Effizienz), `powerlevel_all`, `city_val` (Planer-Werte) heißt: die neuen Daten ändern Ergebnisse (gewollt). Fehler bei „Invarianten“ heißt: Daten kaputt (Summe, endliche Zahlen) – dann Datei prüfen, nicht Test ändern. Passt es, `python3 tests/calc_test.py --update`.
5. **Item-Namen:** neue Items/Kits übersetzt man mit `tools/build_item_names.py` (Wiki-Dumps nötig, Nutzer fragen) → `python3 tools/apply_itm.py --merge`; Details `docs/HANDOFF.md` Abschnitt 4. Fehlt ein deutscher Name, zeigt die App den englischen.
6. **Neue Texte:** neue Stat-/Event-/Item-Namen stehen in `i18n/foeff.js` (`stats`, `events`) bzw. `i18n/items.js`; fehlender Eintrag = englischer Name bleibt stehen (kein Fehler).
7. **Liefern:** `python3 tools/release.py OUTDIR --update-data --compare ALTER_ORDNER`. Unterschiede in `snapshot`/`explore`/`city_flow` sind jetzt erwartet; prüfen, dass sie nur die geänderten Gebäude/LGs betreffen. Bricht `release.py` beim Vergleich ab, mit `--no-bump` und ohne `--compare` erneut bauen.
8. Lauf dauert mit `--compare` einige Minuten (calc_test parallel ca. 40 s, `explore` ca. 3 Min.): im Hintergrund starten (`setsid nohup … &`) und nachsehen.

## Neue Item-Namen/Übersetzungen (foeff)
- `python3 tools/untranslated.py` listet Items, die in DE noch Englisch sind. Eintragen: Items in `i18n/items.js` (`ITX`: "English": "Deutsch"; Kits/Einheiten/Boosts regelt `itd()` in `js/i18n-items.js`), Stats/Events in `i18n/foeff.js`.
- Der Item-Filter (Stadt/Inventar) zeigt Vorschläge übersetzt und findet deutsche **und** englische Namen (`trn()`), sobald ein Name übersetzt ist.

## Prüfen und liefern (im Projekt-Root)
- Schnell: `npm test` = Funktionstest + Rechentest + Schlüsselprüfung + Datenprüfsummen. Dazu `python3 tools/lint.py` (toter Code, ESLint) und `npm run format:check`.
- Liefern: `python3 tools/release.py OUTDIR --compare ALTER_ORDNER` (hochzählen, alle Tests, Vergleich Text-Snapshots alter/neuer Stand, ZIPs). Erwartung: 0 Unterschiede, außer bei gewollten Textänderungen.
- Auf GitHub Pages kommt **nur** `FoE_Tools-vNN-upload.zip` (10 flache Dateien, Handy-Upload hat keine Ordner). `…vNN.zip` ist das ganze Projekt.
- Nicht lauffähig/historisch: `tools/legacy/`.

## Offen: LG-Daten nicht vollständig/100 % korrekt (Stand v194)
Die LG-Daten (Kosten, Platzbelohnungen, Zusatzkosten, Effekte, Stufen > 300, Siphon-Kosten) sind aus Linnuns Tabelle/dem Update-Excel abgeleitet und noch nicht 100 % richtig (u. a. Linnun-Kosten > 300 bekannt falsch, Siphon L102-Ausreißer, Effekte ohne Label, keine Zusatzkosten für Siphon). Der Nutzer lädt später vollständige/korrigierte Daten hoch; dann Kosten (LGN), Zusatzkosten (LGX), Effekte (jLGV/jLGI/jLGL, Generator genfx.py-Logik) neu einspielen.
