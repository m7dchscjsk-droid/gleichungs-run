# Gleichungs-Run / „Quali Run" — Projekt-Handover (aktueller Stand)

Single-Page-Lern-Trainer für den **Quali / qualifizierenden Hauptschulabschluss Bayern**, gebaut für **LOOIS** (und weitere Kinder online). **Eine** HTML-Datei, alles inline (HTML+CSS+JS), **keine Build-Tools**, **offline lauffähig** (Schriften eingebettet). Inzwischen **zwei Fächer** (Mathe + Englisch), **Fortnite-Look**, **Online-Klassen-Board** (Supabase).

- **App:** `Gleichungs-Trainer.html` (~270 KB inkl. eingebetteter Fonts)
- **Live-Link (für die Kinder):** https://m7dchscjsk-droid.github.io/gleichungs-run/
- **Repo:** https://github.com/m7dchscjsk-droid/gleichungs-run (public) · GitHub-Login `m7dchscjsk-droid`

---

## ⚠️ WICHTIGSTE REGEL: Punktestände nie verlieren
Die Punktestände der Kinder liegen **NICHT im Code**, sondern in der **Supabase-Cloud** (Tabelle `scores`). Ein `git push` deployt nur die HTML und fasst die DB nicht an. **Bei JEDEM Push beachten:**
1. `const SUPABASE_URL` / `const SUPABASE_KEY` **niemals ändern** (sonst zeigt die App auf eine leere DB).
2. **Kein destruktives SQL** (`delete`/`drop`) ohne ausdrückliche Ansage.
3. Ablauf: Keys-Diff prüfen → Scores zählen (vorher) → push → Scores zählen (nachher) → muss gleich sein.

Zähl-Befehl:
```bash
URL="https://xmihfeelhmrqaddmohqr.supabase.co"; KEY="sb_publishable_FaZMAa5l-cgKNULOmN6JSQ_goOaEql7"
curl -s -I "$URL/rest/v1/scores?select=id" -H "apikey: $KEY" -H "Authorization: Bearer $KEY" -H "Prefer: count=exact" -H "Range: 0-0" | grep -i content-range
```

---

## Hosting & Update-Workflow (GitHub Pages)
- Statisches Hosting über GitHub Pages (Branch `main`, root). `index.html` ist eine Meta-Refresh-Weiterleitung auf `Gleichungs-Trainer.html`.
- Git-Auth läuft über `gh` (credential helper). Update = Datei ändern → `git commit` → `git push origin main` → ~1 Min Build → gleicher Link.
- `.gitignore` schließt `.claude/` (Preview-Tooling) aus.
- Nach Updates: User soll **hart neu laden** (Cmd+Shift+R), sonst Browser-Cache.

## Cloud / Multiplayer (Supabase, AKTIV)
- Im Script-Kopf: `SUPABASE_URL` + `SUPABASE_KEY` (**publishable**/öffentlich, beginnt mit `sb_publishable_`). Der geheime `sb_secret_`-Key gehört **niemals** ins File.
- Tabelle `scores` mit Spalten u. a. `player, mode, round, tasks, time, aura, peak, pts, boss, ts`. **RLS:** anon darf nur `select` + `insert` (kein `delete`/`update`) → Aufräumen/Reset nur über den **Supabase SQL-Editor** (`delete from scores;`).
- Name pro Gerät in `localStorage['gleichungsrun_player']` (Default LOOIS), beim 1. Start `#nameModal`, änderbar über Header-Namens-Chip. `nm(str)` ersetzt Platzhalter „LOOIS" in Sprüchen durch echten Namen.

---

## Fächer & Modi
Fach-Umschalter auf der Startseite (`setSubject('math'|'eng')`). `ENG_MODES=['vocab','grammar','reading','engmix']` (steuert u. a. Brand „English Run").

| Fach | `state.mode` | Inhalt |
|---|---|---|
| Mathe | `gleich` | Lineare Gleichungen |
| | `prozent` | Prozentrechnung |
| | `groessen` | Größen umrechnen |
| | `geo` | Geometrie (Taschenrechner, Toleranz) |
| | `mix` | alles gemischt, überwiegend sehr schwer |
| Englisch | `vocab` | Vokabeln (EN↔DE, MC, Lücken) |
| | `grammar` | Sprachgebrauch/Grammatik (Teil B) + **Satzbau** |
| | `reading` | Leseverstehen (Teil C) — Text + Fragen |
| | `engmix` | Vokabeln + Grammatik gemischt |

`scoreKey()` → eigener localStorage-Key + Cloud-`mode` pro Modus (`engvocab_scores`, `enggrammar_scores`, `engreading_scores`, `engmix_scores`, …).

## Spiellogik (für alle Modi gleich)
- **Aufgabe = `prob`-Objekt.** Mathe: `{difficulty, displayHTML, steps, answer(F), estSeconds, …}` (Geo: `av`+`tol`+`isGeo`). Englisch (`engProb`): `{eng:true, kind:'text'|'choice'|'order', displayHTML, answer/accept[] | choices[]+answerIndex | tokens[]+order[], explainHTML, estSeconds, vocabRef?}`.
- **Antwort-Prüfung:** Mathe exakt (Bruch) bzw. Toleranz (Geo). Englisch: `normalizeEN()` + Vergleich gegen `accept` (Text), Index (Choice), Reihenfolge (Order).
- Punkte base (leicht 10/mittel 20/schwer 35/sehr schwer 50, Boss 80) × Tempo-Multiplikator; AURA ab 3er-Serie; Endgegner bei Serie 10; Runde = 1000 Punkte.
- **Countdown:** voller `3·2·1` nur am **Rundenstart**; zwischen Aufgaben kurzes „LOS!" (`runCountdown(then, short)`).
- **Return/Enter** löst **immer Prüfen** aus (globaler keydown-Handler; deckt Text, Satzbau und SHITLIST ab; MC ignoriert Enter; springt nie „weiter").

## Englisch-Inhalte & Generatoren
- **`EN_VOCAB`** (~**509** Einträge) über alle Quali-Themen (school, jobs, home, food, clothes, body, family, feelings, freetime, travel, nature, tech, town, time, verbs, adjectives, phrases). Felder `en, de, topic, level, enAlt?, deAlt?, sentence?`. **Keine Dubletten innerhalb eines Themas** (wichtig für MC) — per Verifier geprüft.
- **`EN_VERBS`** (40, mit `past/pp/ing/s3`), `EN_ADJ` (comp/sup), `EN_NOUNS` (Plural), `EN_ARTNOUNS` (a/an).
- **Grammatik-Generatoren** (`GRAMMARGENS`): Zeiten (Simple Past/Present, 3rd person, Present Perfect, Past/Present Progressive, will- & going-to-Future, **since/for**, **used to**, Verneinungen), Steigerung, Plural, a/an, **if-clauses**, **Passiv**, **reported speech**, **Relativsätze**, **Modalverben**, **question tags**, **Satzbau** (`genOrder`, Bank `EN_ORDER` = 22 Sätze; Gewicht bewusst hoch, damit Satzbau oft kommt).
- **Vokabel-Generatoren** (`VOCABGENS`): `genVocEN2DE/DE2EN/MC/Gap` + Boss. Alle setzen `vocabRef` (für SHITLIST).
- **Leseverstehen** (`READINGGENS`): `EN_TEXTS` = **7** kurze A2+-Texte (E-Mail, Aushang, Blog, Sachtext, London, Tagesablauf, Handys) mit Fragen (MC, Richtig/Falsch/Nicht-im-Text, Detail-Tippantwort). `genReading` + `genReadingBoss`. Längere SOLL-Zeit je Textlänge.
- **`ENGMIXGENS`** mischt Vokabeln + Grammatik.

## SHITLIST (Vokabel-Wiederholung)
- Falsch beantwortete **Vokabeln** werden während der Runde gesammelt (`addShit`, dedupe per `en`, nur wo `vocabRef` gesetzt).
- Am **Rundenende** (in `finishRound`, vor der Sieg-Anzeige) erscheint `#ovShit` („💩 SHITLIST — Prüfen wir, ob du es dir gemerkt hast!") und fragt jedes Wort DE→EN ab; falsch → ans Ende der Schlange (Korrektur ~2,8 s sichtbar), richtig → raus; Schleife bis alle sitzen → dann `showWinOverlay()`. `shitLock` verhindert Doppel-Enter. Liste wird in `resetRound`/`showStart`/`finishShit` geleert.

## Scoreboard / Klassen-Board
- Pro Modus eigenes lokales Board (`renderBoard`, Mini-Balken). Cloud: `cloudPush`/`cloudFetch`(eigene Runden, mode+player) / `cloudFetchAll(modeOrAll)`.
- **Klassen-Board auf der Startseite sichtbar** (Default `setBoardView('class')`), mit **Themen-Filtern** (`BOARD_FILTERS`, inkl. „Alle"). **Aufgeteilt** in zwei Spalten: **🏆 Meiste Übungen** (Summe `totalTasks`) und **⏱ Beste Zeiten** (min `bestTime`). Aggregation: `aggregateClass` (pro Spieler `rounds/totalTasks/bestTasks/bestTime`), Render: `classSplitHTML`. Eigener Eintrag hervorgehoben (`.me`), Platz 1 in Gold (`.top1`).

---

## Design / Look (LIVE = Fortnite-Style)
- **Theme:** dunkles Navy + Akzente Blau/Lila/Gold, primärer Action-Akzent **Grün** (`--hi`), AURA/Boss **Gold**. Palette in `:root` (`--bg,--panel,--panel2,--line,--ink,--muted,--dim,--hi,--gold,--blue,--purple,--green`).
- **Schriften eingebettet (offline, base64 @font-face):** **Anton** (`--disp`, Display/Titel/Buttons) + **Rubik** (Fließtext, `body`). Mathe-Brüche bleiben Serif (`#equation` Georgia).
- **Aufbau:** `.wrap` max 1080px. **Top-Bar** (`.topbar`): Level-Badge (= Runde, `#stRound`), Namens-Chip, **XP-Leiste** (`#xpFill` = Fortschritt zum Ziel, in `updateStats` gesetzt), **Coins** (CASH=`#stPoints`, AURA=`#stAura`), Icon-Buttons (↩ `#btnMenu`, ♪ `#btnSound`). **Hero** (`.hero2`, zwei Spalten): links `.charcard` (Sahur-Figur `#ovDino` + `#ovTitle` + Rang), rechts `.heroR` (kicker `#brandSub`, Titel `#brandTitle`, `#ovSub`, **Rarity-Modi-Karten** `.modebtn .mc-no/.mc-nm/.mc-ds` mit Klassen `rare/epic/legend`, `.cta` mit `#btnStart`). Fach-Tabs (`.subjectpick`) als eigene Reihe **über** dem Hero.
- **Lobby passt sich an:** `fitLobby()` setzt `#card` min-height auf Hero-Höhe (bei `showStart`/`setMode`/Init/Resize); `loadProblem` setzt min-height zurück. Overlay `overflow:auto; align-items:safe center`.
- Chunky 3D-Buttons (grüner START/PRÜFEN mit Hover-Glanz), runde Karten, Hover-Effekte.
- **Pixel-Figur** = „Tung Tung Tung Sahur" (Holzmann mit Knüppel, `SAHUR`+`KNUEPPEL` → `buildDino()`; Knüppel schwingt per CSS `@keyframes swing`; Gold bei Serie ≥3). Funktions-/ID-Namen `buildDino/dino/updateDino` aus Kompatibilität beibehalten.

### Entwürfe (lokal, NICHT deployt, untracked)
- `entwurf-fortnite.html` — der gewählte Stil (Referenz/Vergleich).
- `entwurf-gaming.html` — Vice-City-Variante (Magenta/Pink), nicht gewählt.
> Diese zwei Dateien sind reine Design-Referenzen im Projektordner; sie sind nicht im Repo und nicht online.

---

## Code-Struktur (innerhalb `<script>`, Funktionsnamen statt Zeilen — Datei wächst)
Rationalzahlen (`F`, `gcd`, `numDE`) · Render-Helfer (`fracHTML`, math `−` U+2212) · Mathe-Generatoren (`GENS`,`PGENS`,`GGENS`,`GEOGENS`,`MIXGENS`, `makeProblem`/`makeBoss`-Dispatch) · **Englisch**: `engProb/normalizeEN/shuffle/esc`, `EN_VOCAB`, `VOCABGENS`, `EN_VERBS/ADJ/NOUNS`, `GRAMMARGENS` (+ Phase-2-/Zeiten-Generatoren), `EN_ORDER`/`genOrder`, `EN_TEXTS`/`READINGGENS`, `ENGMIXGENS` · `MSG` (Sprüche) · Pixel-Figur · WebAudio · State + Supabase-Block + `scoreKey/loadScoresLocal/cloudFetch(All)/cloudPush/initScores` · UI/Flow (`parseAnswer`, `loadProblem`, `checkAnswer/checkChoice/checkOrder`, `onCorrect/wrongFeedback`, `updateStats/updateDino/renderBoard`, SHITLIST `addShit/startShitlist/shitNext/checkShit/finishShit/showWinOverlay`, `finishRound`) · Overlay/Flow (`runCountdown/startGame/goNext/showStart/showTaskResult/fitLobby`) · Board-Umschalter (`setBoardView/setClassFilter/aggregateClass/classSplitHTML/renderSharedBoard`) · Modus/Fach (`MODE_TXT/setMode/setSubject` + Handler) · Namens-System · Init.

## Verifikation (Node, ohne Browser)
`<script>`-Block extrahieren und mit `vm.runInContext` laufen lassen; **stubben:** `document`, `localStorage`, `fetch` (resolved `{ok,json:[]}`), **`window.addEventListener`**, `requestAnimationFrame`. Tests (Wegwerf-Skript `/tmp/verify_gr.js`): Script lädt fehlerfrei; EN_VOCAB sauber + **keine Dubletten (topic+en / topic+de)**; alle Generatoren liefern konsistente Antworten (Text-Antwort ∈ accept, Choice-Index gültig, Order: tokens==order & answer==order.join); `aggregateClass` (totalTasks/bestTime); alle 9 Modi liefern Aufgaben. Für visuelle Checks: temporäre `launch.json` mit absolutem Pfad auf `…/Gleichungs-Run/.claude/serve.js`, Preview `preview_start("static")`, danach löschen.

---

## Offene Punkte / Ideen für den nächsten Chat
1. **Phase 4 — Sprachmittlung** (Teil D): aus EN-Text deutsche Faktfragen (MC/Detail).
2. **Phase 5 — Hörverstehen** (Teil A) via Browser-TTS (`speechSynthesis`) + Fallback.
3. **Mehr Inhalte:** Vokabeln Richtung 1.850, mehr Lesetexte (aktuell 7), mehr `EN_ORDER`-Sätze.
4. **Look-Feinschliff:** Rarity-Farben auch im Gameplay, Mobil-Abstände, optionaler Hintergrund-Grid/Animationen; ggf. echtes Charakterbild statt Sahur in der Charakter-Karte.
5. **HUD:** Top-Bar-Text „Level/XP" evtl. echtes Level-System; SERIE/OK ggf. zusätzlich anzeigen.
6. Optional: eigene Sprüche-Buckets pro Modus; Mix-Schwierigkeits-Schieberegler.

## Quick-Start für den neuen Chat
> „Wir machen am Quali-Run weiter. Lies `HANDOVER.md` (aktueller Stand). App = `Gleichungs-Trainer.html`, live auf GitHub Pages, Klassen-Board in Supabase. **Wichtig: bei jedem Push die Punktestände schützen** (Keys nie ändern, vorher/nachher zählen). Dann starten wir bei Punkt X."
