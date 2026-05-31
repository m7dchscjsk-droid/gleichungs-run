# Gleichungs-Run — Projekt-Handover

Single-Page Mathe-Trainer für **LOOIS** (Hauptschulabschluss Bayern, ohne Taschenrechner – außer Geometrie). Monochromer Look mit goldenem Akzent. Komplett in **einer** HTML-Datei (`Gleichungs-Trainer.html`, ~1550 Zeilen, HTML + CSS + JS inline, keine Dependencies).

Das Spiel hat inzwischen **fünf Modi**, die alle auf derselben Spiellogik laufen: Start „Gleichungs Run" mit **Modus-Auswahl**, dann pro Aufgabe Countdown → Aufgabe → Antwort prüfen → Ergebnis-Overlay → weiter, bis **1000 Punkte** erreicht sind (= eine Runde). Punkte, Tempo-Multiplikator, AURA-Serie, Pixel-Dino, Endgegner (Serie 10) und Scoreboard sind in allen Modi gleich.

---

## Datei-Struktur

```
Gleichungs-Run/
├── Gleichungs-Trainer.html       ← die ganze App (alles inline)
├── HANDOVER.md                   ← dieses Dokument
└── .claude/
    ├── launch.json               ← Preview-Config (Claude Code preview_start "static")
    └── serve.js                  ← Mini-Static-Server (Node, port 8123, rootet auf den Projektordner)
```

Lokales Öffnen: Doppelklick auf `Gleichungs-Trainer.html` (file://). Sauberer über den Preview-Server (`preview_start("static")` → http://localhost:8123).

> **Hinweis für Claude Code:** Liegt das Session-Verzeichnis nicht im Projektordner, findet `preview_start` die `.claude/launch.json` nicht. Workaround: temporär eine `launch.json` im Session-`.claude` anlegen, deren `runtimeArgs` mit **absolutem Pfad** auf `…/Gleichungs-Run/.claude/serve.js` zeigt (ROOT in serve.js wird relativ zur serve.js aufgelöst, serviert also immer den Projektordner). Nach dem Test wieder löschen.

---

## Die fünf Modi (Startseite: Modus-Buttons)

| Modus | `state.mode` | localStorage-Key | Rechner? | Inhalt |
|---|---|---|---|---|
| **Gleichungen** | `gleich` | `gleichungsrun_scores` | nein | Lineare Gleichungen nach x auflösen (Brüche, Dezimal, Hauptnenner) |
| **Prozent** | `prozent` | `prozentrun_scores` | nein | Prozentwert/Grundwert/Prozentsatz + Textaufgaben (MwSt, Rabatt, Veränderung, Verhältnis) |
| **Größen** | `groessen` | `groessenrun_scores` | nein | Einheiten umrechnen (Länge/Masse/Fläche/Volumen/Zeit/Daten) + zusammengesetzte Einheiten |
| **Geometrie** | `geo` | `georun_scores` | **ja** (π≈3,14) | Pythagoras, Flächen, Kreis, Volumen & Oberfläche — **Toleranz-Prüfung** |
| **Mix** | `mix` | `mixrun_scores` | gemischt | Alles gemischt, **überwiegend sehr schwer**, jedes Mal neu zufällig |

- Modus-Auswahl auf der Startseite (`#ovStart`), Umschalten über `setMode(m)`. **Header-Button „↩ Auswahl"** (`#btnMenu` → `showStart()`) bringt jederzeit zurück zur Auswahl (laufende Runde wird verworfen).
- Jeder Modus hat **eigenes Scoreboard** (eigener localStorage-Key via `scoreKey()`); beim Moduswechsel wird neu geladen.
- Mode-abhängige Texte (Brand-Subtitle, Intro, Placeholder, Hinweis) stehen in `MODE_TXT`.

---

## Spiellogik (für alle Modi gleich)

- **Aufgabe = `prob`-Objekt**: `{difficulty, displayHTML, steps, answer, estSeconds, …}`. Der gesamte Flow hängt nur an dieser Form — neue Aufgabentypen müssen nur ein `prob` liefern.
  - `answer` ist ein Bruch `F{n,d}` (exakter Vergleich). **Geometrie** trägt zusätzlich `av` (exakter Float-Wert) + `tol` (Toleranz) und `isGeo:true` → `checkAnswer` prüft `|Eingabe − av| ≤ tol` statt exakt.
  - `isBoss:true` markiert den Endgegner (Express-Rundenende bei Sieg).
- **Punkte** pro Aufgabe: `base × Tempo-Multiplikator`. Base nach Schwierigkeit: leicht 10 / mittel 20 / schwer 35 / sehr schwer 50, Endgegner 80. Multiplikator: Blitz (≤50 % Soll) ×2, schnell (≤80 %) ×1.5, normal ×1.2.
- **Über SOLL** → Minuspunkte (wachsend), **Serie reset**.
- **AURA-Punkte** ab 3er-Serie, linear wachsend: `(streak−2)·5`. Goldener AURA-Chip + goldener Dino ab Serie ≥3.
- **Spickzettel-Strafe**: „Lösung zeigen" vor dem Lösen → 0 Punkte, Serie weg. Nach dem Lösen frei.
- **Endgegner bei Serie 10** (`nextIsBoss`): super lange Aufgabe; **in SOLL besiegt → Runde endet sofort** (+100 Master-Bonus, Express-Ende); **über SOLL → Minus + Serie weg, Runde läuft weiter**.
- **Runde = bis 1000 Punkte** (`GOAL=1000`) **oder** Endgegner besiegt. Goal-Bar mit goldenem AURA-Anteil. Danach Win-Overlay + Scoreboard-Eintrag.
- **Scoreboard** (`renderBoard`): Mini-Balkendiagramm (Balken = Aufgaben pro Runde, kürzer = besser) + Liste der letzten Runden. Persistenz in localStorage pro Modus. Cloud (Supabase) **vorbereitet, nicht aktiv** (siehe unten).
- **Sounds** (WebAudio, offline): Trillern (30-s-Warnung), Wah-Wahh (falsch), Ding (richtig), Sieges-Jingle (Boss/Rundenende), Countdown-Beeps. Toggle ♪/✕ im Header.
- **Sprüche** (`MSG`-Buckets): blitz (enthält „GOOD BOY"), gut (enthält „DU MUSST AUCH MAL NON CHALANT BLEIBEN."), drueber, peek, falsch, boss. Werden über alle Modi geteilt.

---

## Aufgaben-Generatoren je Modus

Jeder Generator gibt ein `prob` zurück (oder `null` bei Reject → Retry-Loop in `makeProblem`). Dispatch über `state.mode` in `makeProblem()` / `makeBoss()`.

### Gleichungen — `GENS` (Z. ~616)
`genVarBoth`, `genBracketConst`, `genBracketBoth`, `genMinusBracket`, `genFracSimple`, `genFracDen`, `genFracCross`, `genFracBoth`, `genHardFrac` (dominant, Hauptnenner), `genDecimal`, `genMixedDecFrac`. Endgegner `genBoss` (3 Brüche + Dezimal). Niceness-Garantie: Lösungs-Nenner ∈ {1..6}, |Zähler| ≤ 30, Konstanten ≤ 200. Hauptnenner via `lcdOk()` entzerrt (12 gecappt).

### Prozent — `PGENS` (Z. ~825)
Abstrakte Grundtypen (Aufwärmer): `genProzentwert`, `genProzentsatz`, `genGrundwert` — Dreisatz mit 1-%-Anker. Textaufgaben: `genRabatt` (verminderter/vermehrter Grundwert), `genMwSt` (rein/raus, 19 %/7 %), `genVeraenderung` (prozentuale Änderung, zeigt absolut + in %), `genVerhaeltnis` (Verhältnis → Prozent, ISB-Nüsse-Stil). Endgegner `genProzentBoss` (Grundwert → davon Prozentwert). Alle Antworten glatt (Nenner 1 oder 2).

### Größen — `GGENS` (Z. ~931)
`genConvert(catKey)` — einfache Umrechnung A→B per Umrechnungszahl/Komma-Schieben, abstrakt oder als Spaß-Textaufgabe. `genMixedUnit` — zusammengesetzte Einheit („2 h 30 min = ? min"). Endgegner `genGroessenBoss` — dreifach gemischt. Kategorien-Faktoren in `G_CATS`; nur ganzzahlige Verhältnisse, Werte bleiben kopfrechenbar. Einheiten beim Eingeben optional (Parser strippt „kg", „m²" …).

### Geometrie — `GEOGENS` (Z. ~1080)  *(Teil B: Taschenrechner + Formelsammlung)*
`genRechteck` (Fläche/Umfang), `genDreieck`, `genParallelogramm`, `genTrapez`, `genPythagoras` (Hypotenuse/Kathete), `genKreisflaeche`, `genKreisumfang`, `genKreisring`, `genQuaderV`, `genQuaderO`, `genWuerfel`, `genZylinderV`, `genZylinderO`, `genPyramideV`, `genKegelV`. Endgegner `genGeoBoss` (zusammengesetzte „Fensterfläche" = Rechteck + Halbkreis).
- **π = 3,14** (Konstante `PI`). Ergebnisse auf 2 Dezimalen gerundet, **Toleranz** `tol = max(0.05, 0.015·av)` akzeptiert sowohl 3,14- als auch Taschenrechner-π-Antworten.
- **Schematische SVG-Figuren** pro Form (`figRect`, `figRTri`, `figTri`, `figPara`, `figTrap`, `figCirc`, `figRing`, `figCuboid`, `figCyl`, `figPyr`, `figCone`, `figWindow`), monochrom, nicht maßstäblich.
- Lösungsweg-Schema: Formel → einsetzen → Ergebnis.

### Mix — `MIXGENS` (Z. ~1088)
Gewichteter Pool aus den Generatoren **aller** Sektionen, **Übergewicht sehr schwer** (~50 % sehr schwer, ~41 % schwer, ~8 % mittel). Endgegner = zufälliger Sektions-Boss (`genBoss` / `genProzentBoss` / `genGroessenBoss` / `genGeoBoss`). Der Hinweistext schaltet **pro Aufgabe** um (Geometrie: „mit Taschenrechner", sonst „ohne"). **Keine** Boss-Generatoren im regulären Pool (sonst versehentliches Express-Ende).

---

## Code-Struktur (innerhalb `<script>`)

1. **Rationalzahlen**: `gcd`, `lcm`, `class F`, `decimalSuffix`, `numDE` (deutsche Dezimaldarstellung).
2. **Render-Helfer**: `fracHTML`, `xc`, `withSign`, `polyHTML`, `iHTML`, `renderTerms`, `row`, `eqRow`, … Mathematisches Minus `−` (U+2212) überall, nie ASCII `-`.
3. **Gleichungen**: `standardSolve`, Term-Bausteine, `solveFractionEquation`, alle `gen*`-Gleichungs-Generatoren, `GENS`, `makeProblem`, `makeBoss`.
4. **Prozent-Block**: Helfer (`ptype`), Generatoren, `PGENS`.
5. **Größen-Block**: `G_CATS`, `G_V`, `G_CTX`, `toF`, `niceVal`, Generatoren, `GGENS`.
6. **Geometrie-Block**: `PI`, `gnum`, `geoProb`, `gT`, `fin`, `fig*`-SVGs, Generatoren, `GEOGENS`.
7. **Mix-Block**: `MIXGENS`.
8. **`MSG`** (Sprüche), **Pixel-Dino**, **WebAudio**.
9. **State + Konstanten**: `state` (inkl. `mode`), `GOAL=1000`, `PLAYER='LOOIS'`, Supabase-Block.
10. **Persistenz**: `scoreKey`, `loadScoresLocal`, `saveScoresLocal`, `cloudFetch`, `cloudPush`, `initScores`.
11. **UI/Flow**: `parseAnswer` (strippt %, €, Einheiten), `loadProblem` (inkl. Mix-Hint-Logik), `startTimer`, `tick`, `checkAnswer` (exakt **oder** Toleranz bei `current.tol`), `onCorrect`, `updateStats`, `updateDino`, `renderBoard`, `finishRound`.
12. **Overlay/Flow**: `runCountdown`, `startGame`, `goNext`, `showStart` (zurück zur Auswahl), `showTaskResult`.
13. **Modus-Umschalter**: `MODE_TXT`, `setMode(m)`, Button-Handler (`btnModeEq/P/G/Geo/Mix`).
14. **Init**: Event-Handler + `initScores()` + Dino-Render. Startseite initial sichtbar.

---

## Verifikations-Stand (jeweils der echte Code in Node, mit DOM-Stub, kein Browser nötig)

- **Gleichungen**: 33.000+ Bruch-Checks (unabhängiger Rationalzahlen-Rechner) → 0 Fehler; 250/250 Auto-Solve im Browser.
- **Prozent**: 16.000 Aufgaben → 0 Fehler (Antwort eingebbar, unabhängig nachgerechnet, Lösungsweg-Endwert == Antwort, Parser-Round-Trip).
- **Größen**: 22.000 Aufgaben → 0 Fehler; Umrechnungen gegen die **physikalische Invariante** geprüft (Antwort × Zieleinheit == Quelle × Quelleinheit).
- **Geometrie**: 19.000 Aufgaben → 0 Fehler; jede Formel unabhängig nachgerechnet, Toleranz akzeptiert 3,14- **und** Taschenrechner-π, weist ×2-Fehler ab.
- **Mix**: 24.000 Aufgaben → sehr schwer 50,3 % (Mehrheit), alle 4 Sektionen vertreten, 0 Boss-Aufgaben im regulären Pool.
- **Browser** (Preview): Modus-Wechsel, echte Eingaben (auch mit Einheit/€/% bzw. gerundet), Lösungswege, Figuren und Endgegner gerendert; „↩ Auswahl"-Button getestet.

> **Verifikations-Pattern:** Den `<script>`-Block aus der HTML extrahieren und mit `vm.runInContext` in Node laufen lassen, dabei `document`/`localStorage` als Proxy stubben. So lassen sich Generatoren/Parser massenhaft prüfen, ohne Browser. (Wegwerf-Skripte; nicht eingecheckt.)

---

## Design-Konventionen

- **Monochrom** (Schwarz/Weiß/Grau), einzige Akzentfarbe `--gold: #ffcf4a` (AURA, Boss, Gold-Dino, Geometrie-Figuren-Labels neutral grau).
- **Mathematisches Minus** `−` (U+2212) überall.
- **Brüche** rein in CSS (`.frac`). Kein MathJax.
- **Spieler** = **LOOIS** (Single-Player; Sprüche personalisiert). Cloud-Schema hat `player`-Spalte für Multiplayer.
- **Lösungsweg** einheitlich: `.srow` mit `lhs = rhs` + Operations-Notiz rechts; Wort-Operationen kursiv-grau, Rechen-Operationen mono. Finale Zeile weiß; deren Operations-Notiz dunkel (`.srow.final .sop`).
- **Caption** pro Aufgabe: `.ptype` (kleine Großbuchstaben, grau). Textaufgaben in `.wp` (lesbarer Fließtext). Geometrie-Figuren in `.geofig`.

---

## Cloud / Multiplayer — vorbereitet, nicht aktiviert

Im Script-Kopf:
```js
const SUPABASE_URL = '';   // z.B. 'https://abcd1234.supabase.co'
const SUPABASE_KEY = '';   // anon public key
const PLAYER = 'LOOIS';
```
Ohne Konfiguration speichert das Spiel **lokal pro Modus** (localStorage). Mit Supabase (gratis) wird geräteübergreifend gespeichert. SQL-Schema steht als Kommentar in der HTML; Tabelle `scores` hat u. a. `player, round, tasks, time, aura, peak, pts, boss, ts`.

**Geplant/besprochen (Multiplayer für mehrere Kinder, async):** Jedes Kind spielt seine eigene Lernstrecke online, gemeinsames **Klassen-Board** (wer wie viele Runden, Zeit, AURA). Technisch machbar; offene To-dos: (1) Supabase aktivieren, (2) `mode`-Spalte ergänzen (5 Modi), (3) Namens-Eingabe pro Gerät, (4) aggregiertes Multi-Spieler-Board, (5) Datei online hosten (z. B. Netlify Drop). Echtzeit-Duell wäre möglich, aber deutlich aufwändiger (Realtime + Räume).

---

## Hosting — ERLEDIGT (GitHub Pages)

- **Live-Link für die Kinder:** https://m7dchscjsk-droid.github.io/gleichungs-run/
- Repo: https://github.com/m7dchscjsk-droid/gleichungs-run (public). GitHub-Account-Login: `m7dchscjsk-droid` (Anzeigename „sigmaligma").
- `index.html` ist eine **Meta-Refresh-Weiterleitung** auf `Gleichungs-Trainer.html` (damit die Root-URL direkt das Spiel öffnet). Single source of truth bleibt `Gleichungs-Trainer.html`.
- `.gitignore` schließt `.claude/` (Preview-Tooling) aus dem Repo aus.
- **Update-Workflow:** Datei ändern → `git commit` → `git push` (origin/main). Pages baut automatisch (~1 Min), gleicher Link. Git-Auth läuft über `gh` (credential helper via `gh auth setup-git`).

## Offene Punkte / Ideen für den nächsten Chat

1. **Cloud aktivieren + Multiplayer-Board** (siehe oben) — Hauptwunsch des Users für „mehrere Kinder online".
2. (Optional) Geometrie: mehr zusammengesetzte Figuren / Sachkontexte, Maßstabs-/Diagonalaufgaben.
3. (Optional) Größen: l↔dm³ / ml↔cm³ Äquivalenzen als eigener Mini-Typ; Tonnen/Hektar-Sachaufgaben.
4. (Optional) Prozent: Zinsrechnung (Teil B), gestaffelte Rabatte.
5. (Optional) Mix: Schwierigkeits-Schieberegler (wie stark „sehr schwer" überwiegt).
6. (Optional) Eigene Sprüche-Buckets pro Modus statt geteiltem `MSG`.

---

## Quick-Start für einen neuen Chat

> „Wir machen am Gleichungs-Run weiter. Briefing in `HANDOVER.md`, App in `Gleichungs-Trainer.html`. Lies erst die HANDOVER.md, dann starten wir bei Punkt X."

Der neue Chat sollte die HANDOVER.md zuerst lesen, dann gezielt am offenen Punkt arbeiten, den der User angibt. Für Verifikation den Node-vm-Ansatz nutzen; für visuelle Checks den Preview-Server (ggf. temporäre `launch.json` mit absolutem serve.js-Pfad).
