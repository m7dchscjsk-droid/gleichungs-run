# Englisch-Trainer — Konzept (Quali / qualifizierender Hauptschulabschluss Bayern)

Erweiterung des bestehenden **Gleichungs-Run** um ein **Englisch-Training** auf Quali-Niveau (GER **A2+**), gebaut auf **derselben Spiel-Engine** (eine HTML-Datei, alles inline, monochromer Look + Gold, AURA-Serie, Endgegner, Klassen-Board/Cloud). Spieler weiterhin frei benannt (LOOIS = Default).

> Leitidee: Gleiches Spielgefühl wie Mathe — Countdown → Aufgabe → prüfen → Ergebnis-Overlay → weiter, bis **1000 Punkte**. Pro Bereich ein **eigener Modus** + ein **Mix**. Selbstkorrektur ohne Lehrkraft.

---

## 0. Quali-Englisch — woran wir uns halten (Kurzfassung)

Schriftliche Prüfung (120 Min, 6 Teile): **A** Hör-/Hörsehverstehen · **B** Sprachgebrauch (Grammatik/Wortschatz) · **C** Leseverstehen · **D** Sprachmittlung · **E** Text-/Medienkompetenz · **F** Schreiben. Mündlich ~15 Min. Niveau **A2+**. Wortschatz bis Kl. 9 ca. **1.850** Wörter/Wendungen, Fokus **Alltag & Beruf**. (Details + Quellen siehe Chat/HANDOVER.)

**Was ein Selbstkorrektur-Trainer sinnvoll abbildet** (eindeutig prüfbar):
- ✅ **Sprachgebrauch/Grammatik** (Teil B) — Lücken/Formen
- ✅ **Wortschatz** — Vokabeln/Wendungen
- ✅ **Leseverstehen** (Teil C) — MC, richtig/falsch, Zuordnen
- ✅ **Sprachmittlung** (Teil D) — in vereinfachter, prüfbarer Form (Fakten/MC)
- ◻️ **Hörverstehen** (Teil A) — via **Text-to-Speech** des Browsers (mit Vorbehalten, s. u.)
- ⚠️ **Freies Schreiben** (Teil F) & freie mündliche Produktion — **nicht** automatisch bewertbar → nur als geführte Mini-Varianten (Satzbau/Reihenfolge), keine offene Korrektur.

---

## 1. Was von der Mathe-Engine 1:1 bleibt

- **Spiel-Flow**, Punkte (leicht 10 / mittel 20 / schwer 35 / sehr schwer 50, Boss 80), Tempo-Multiplikator, **AURA** ab 3er-Serie, Endgegner bei Serie 10, Goal-Bar bis 1000, Win-Overlay.
- **Scoreboard** pro Modus (eigener `localStorage`-Key via `scoreKey()`) + **Cloud/Klassen-Board** (Supabase). Die `mode`-Spalte ist bereits frei und nimmt neue Modus-Keys ohne Schema-Änderung auf.
- **Sound, Pixel-Figur (Sahur), Sprüche (`MSG`)**, Namens-System.

## 2. Zentraler Unterschied zu Mathe: Inhalte

Mathe ist prozedural mit exaktem Zahl-Ergebnis. Englisch braucht **garantiert korrekte Sprache** → **Hybrid**:

- **Generierbar (regelbasiert, sicher prüfbar)** — aus Tabellen/Templates:
  - Vokabel-Abfragen aus einer **Wortliste** `{en, de, topic, level, alt[]}`
  - Verbformen aus einer **Verb-Tabelle** (regelmäßig + unregelmäßig): Simple Past, Present Perfect (Partizip), -ing
  - Steigerung von Adjektiven, Pluralformen, Artikel (a/an), einfache Präpositionen, Mengenwörter (some/any), Pronomen
- **Kuratiert (Inhaltsbank im Code)** — wo Generierung fehleranfällig wäre:
  - Lesetexte + Fragen, Sprachmittlungs-Texte, komplexe Grammatik in Kontext (if-clauses, passive, reported speech), idiomatische Wendungen, Hörtexte (= kuratierte Sätze für TTS)

> Konsequenz: Englisch ist **inhaltsgetrieben**. Qualität = Größe & Pflege der Bänke. Das ist der eigentliche Aufwand (s. Roadmap §10).

## 3. Erweitertes `prob`-Objekt + Antwort-Prüfung

Das `prob`-Objekt wird um Antworttyp & Akzeptanz erweitert (Mathe bleibt unverändert kompatibel):

```js
prob = {
  difficulty,            // 'leicht' | 'mittel' | 'schwer' | 'sehr schwer'
  kind,                  // 'text' | 'choice' | 'order'  (Interaktionstyp)
  promptHTML,            // Aufgabenstellung (Satz mit Lücke, Frage, Text+Frage …)
  // -- bei kind:'text' --
  answer,                // kanonische Lösung (String)
  accept,                // [normalisierte akzeptierte Varianten]
  // -- bei kind:'choice' --
  choices,               // [Optionen]  (answerIndex = richtige)
  answerIndex,
  // -- bei kind:'order' --
  tokens, order,         // Wörter/Bausteine + richtige Reihenfolge
  // gemeinsam:
  explainHTML,           // Auflösung: richtige Lösung + kurze Regel/Übersetzung
  estSeconds, isBoss
}
```

**Antwort-Normalisierung** (für `kind:'text'`): trim → klein → Mehrfach-Leerzeichen kollabieren → typografische Apostrophe/Anführungen vereinheitlichen → optionaler Punkt am Ende ignorieren. Akzeptanz-Logik:
- **Alternativenliste** `accept` (z. B. `["colour","color"]` BE/AE; `["do not","don't"]`).
- Optionale **Tippfehler-Toleranz** (Levenshtein ≤1) nur bei langen Wörtern — vorsichtig, abschaltbar pro Aufgabe.
- Feedback bei knappem Treffer: „Fast! Achte auf die Schreibweise."

**`choice`/`order`** werden exakt gegen Index bzw. Reihenfolge geprüft (wie Geo-Toleranz nur eben diskret).

## 4. UI-Erweiterungen (über das Text-Eingabefeld hinaus)

- **Multiple-Choice-Buttons** (für Lese-/Hörverstehen, schnelle Grammatik) — 2–4 Antwort-Buttons statt/ neben Eingabe.
- **Wort-Reihenfolge** (Satzbau): antippbare Bausteine, die in Reihenfolge gebracht werden (mobile-freundlich, Tap statt Drag).
- **Audio-Button** (Hörverstehen): ▶︎ „Hören" nutzt `speechSynthesis` (Web Speech API), inkl. „nochmal" und Tempo. Fallback: Text einblenden, wenn keine Stimme verfügbar.
- **Lesetext-Panel**: kurzer Text oben, Frage(n) darunter (eigenes Layout `.engtext`).
- Bestehendes Eingabefeld bleibt für `kind:'text'`.

---

## 5. Die Trainings-Modi

Fünf Einzel-Modi + Mix. Modus-Keys (Cloud/`scoreKey`): `vocab`, `grammar`, `reading`, `mediation`, `listening`, `engmix`.

### 5.1 Vokabeln — `vocab`  (Wortschatz)
- **Quali-Bezug:** Grundlage für alle Teile; Themen **Alltag & Beruf**.
- **Aufgabentypen:** EN→DE und DE→EN abfragen; Wendung vervollständigen; „odd one out" (MC); Wort in Lücke (Kontextsatz).
- **Antwort:** `text` (mit `accept`, inkl. Artikel-Toleranz „the/–") oder `choice`.
- **Datenquelle:** kuratierte **Wortliste** nach Themenfeldern (school/jobs, daily life, shopping, travel, health, environment, technology, feelings …), je Eintrag `level` (leicht/mittel/schwer) + `alt[]`.
- **Generator:** wählt Eintrag nach Schwierigkeit, Richtung zufällig, baut Prompt; Distraktoren für MC aus gleichem Thema.
- **Schwierigkeit:** leicht = Grundwortschatz EN→DE; mittel = DE→EN; schwer = Wendungen/false friends; sehr schwer = Lücke im Kontextsatz.
- **Endgegner:** „Vokabel-Sprint" — 8–10 Wörter einer Berufs-/Alltagsszene am Stück.
- **estSeconds:** 8–18.
- **Beispiel:** „**apply for** a job = sich ___ eine Stelle ___" → „bewerben"; bzw. EN→DE „reliable = ___" → „zuverlässig".

### 5.2 Grammatik — `grammar`  (Sprachgebrauch / Teil B)
- **Quali-Bezug:** Teil B (Lückentexte, richtige Form). Kernmodus.
- **Sub-Themen** (gewichtet, Schwierigkeit steuert Auswahl):
  1. **Zeitformen** (simple present/progressive, simple past, present perfect, past progressive, will/going-to-future) — *generierbar* aus Verb-Tabelle + Signalwörtern (yesterday, already, next week …).
  2. **If-clauses** Typ I & II (III als „sehr schwer") — Template-basiert.
  3. **Passive voice** (present/past simple) — Template aus Aktivsatz-Bausteinen.
  4. **Reported speech** (Aussagen, einfache Fragen) — Template.
  5. **Relativsätze** (who/which/that), **Steigerung**, **Modalverben**, **gerund/infinitive**, **question tags**, **Präpositionen**, **a/an/some/any** — Tabellen/Templates.
- **Aufgabentypen:** Lücke mit Vorgabe (Infinitiv/Signalwort) → richtige Form; richtige von 4 Formen wählen (MC); Satz umformen (Aktiv→Passiv / direkt→indirekt) als `text`.
- **Antwort:** meist `text` (genau eine korrekte Form, `accept` für Kurzformen), teils `choice`.
- **Schwierigkeit:** leicht = eine bekannte Zeitform mit klarem Signalwort; mittel = Zeitform-Kontrast; schwer = if II/passive/reported; sehr schwer = Umformungen / Typ III.
- **Endgegner:** „Grammar Gauntlet" — kurzer Lückentext mit 4–6 Lücken gemischter Themen.
- **estSeconds:** 12–30.
- **Beispiel:** „Yesterday she ___ (go) to the doctor." → „went"; „If it ___ (rain), we will stay home." → „rains"; Umformung „They build houses." → Passiv: „Houses ___ ___ ___." → „are built".

### 5.3 Leseverstehen — `reading`  (Teil C)
- **Quali-Bezug:** Teil C (MC, Fragen zum Text, Zuordnen).
- **Aufgabentypen:** kurzer Text (60–150 Wörter) + 1–3 Fragen: **MC**, **true/false/not given**, Detail-Frage (`text`, kurze Antwort), Überschrift/Absatz zuordnen.
- **Antwort:** überwiegend `choice`; Detailfragen `text` mit großzügigem `accept`.
- **Datenquelle:** **kuratierte Text-Bank** (E-Mail, Anzeige, Blog, Infotext; Themen Alltag/Beruf/Landeskunde), jeweils mit Fragen + Lösungen + kurzer Begründung.
- **Schwierigkeit:** über Textlänge/Wortschatz + Fragetyp (explizit → leicht, Schlussfolgerung → schwer).
- **Endgegner:** längerer Text mit 4–5 Fragen (Express-Ende bei voller Punktzahl in SOLL).
- **estSeconds:** 45–120 (höhere SOLL-Zeit als Mathe).

### 5.4 Sprachmittlung — `mediation`  (Teil D)
- **Quali-Bezug:** Teil D (aus EN-Text Infos auf Deutsch wiedergeben).
- **Prüfbare Vereinfachung:** statt freier Übertragung → **gezielte Faktfragen auf Deutsch** zum EN-Text (Was kostet …? Wann …? Wo …?) als `text`/`choice`; oder „welche 3 der 6 deutschen Aussagen stimmen laut Text" (MC-Mehrfach).
- **Antwort:** `text` (Zahl/Datum/Stichwort, robust normalisiert) oder `choice`.
- **Datenquelle:** kuratierte EN-Kurztexte (Aushang, Mail, Hinweis) mit deutschen Faktfragen + Lösungen.
- **Schwierigkeit:** Anzahl/Verstecktheit der Infos.
- **Endgegner:** ein Text, aus dem 4–5 Fakten korrekt „gemittelt" werden müssen.
- **estSeconds:** 40–100.

### 5.5 Hörverstehen — `listening`  (Teil A, via TTS)
- **Quali-Bezug:** Teil A. **Technik:** `speechSynthesis` liest einen kuratierten Hörtext/Dialog vor; danach MC/Lücke/Zuordnen.
- **Aufgabentypen:** „Hör zu und beantworte" (MC), Lücke im Transkript, richtig/falsch, Informationen zuordnen.
- **Antwort:** `choice` oder `text`.
- **Wichtige Vorbehalte:**
  - Stimmen-Qualität/-Verfügbarkeit hängt vom Gerät/OS ab; manche Mobilgeräte brauchen online. → **Fallback**: Text einblenden + „nochmal hören"; Hinweis, wenn keine en-Stimme vorhanden.
  - Tempo/Stimme einstellbar; mehrfaches Abspielen erlaubt (kein „echtes" Prüfungstempo nötig fürs Training).
- **Datenquelle:** kuratierte Sätze/Dialoge (kann teils Lese-Bank wiederverwenden).
- **estSeconds:** 30–90 (inkl. Hören).

### 5.6 Mix — `engmix`
- **Gewichteter Pool** aus allen fünf Bereichen, **ausgewogen** (Standard: je ~20 %, leicht hard-bias über Schwierigkeitsverteilung), pro Aufgabe neu. Hinweistext schaltet pro Aufgabe um (z. B. „Hörverstehen" / „Grammatik").
- **Endgegner:** zufälliger Bereichs-Boss.
- Optional später: **Schieberegler**, der Bereichs-Gewichte/Schwierigkeit steuert (vgl. Mathe-Mix-Idee).

---

## 6. Inhalts-/Datenmodell (im Code, inline)

```js
// Wortliste (Vokabeln + Distraktoren + Lückensätze)
const EN_VOCAB = [
  {en:"reliable", de:"zuverlässig", topic:"jobs", level:"mittel",
   alt:[], sentence:"A good worker is ___."},
  ...
];
// Verb-Tabelle (für generierte Zeitformen)
const EN_VERBS = [
  {inf:"go", past:"went", pp:"gone", reg:false},
  {inf:"work", past:"worked", pp:"worked", reg:true}, ...
];
// Grammatik-Templates (Slots aus kontrolliertem Vokabular)
const EN_GRAMMAR_TPL = { tenses:[...], ifclauses:[...], passive:[...], reported:[...] };
// Lese-/Mittlungs-/Hör-Bank
const EN_TEXTS = [
  {id, mode:"reading", level, text:"...", items:[
     {kind:"choice", q:"...", choices:[...], answerIndex:1, explain:"..."},
     {kind:"text",   q:"...", answer:"...", accept:[...], explain:"..."}
  ]}, ...
];
```

Vorteile: klar trennbar, **massenhaft per Node lintbar** (s. §9), leicht erweiterbar, ohne Build-Schritt.

## 7. Schwierigkeit, Punkte, AURA, Boss
Identisch zu Mathe (Wiederverwendung). Nur die **SOLL-Zeiten** (`estSeconds`) sind je Modus höher kalibriert (Lesen/Hören brauchen länger). Endgegner je Modus = lange Mehrteil-Aufgabe; Sieg in SOLL → Express-Rundenende + Master-Bonus.

## 8. Integration in die bestehende App

- **Fächer-Umschalter (empfohlen):** Auf der Startseite zuerst **Fach** wählen — **Mathe** ↔ **Englisch** — dann die Modus-Buttons des Fachs. Verhindert 12 Buttons nebeneinander. Brand-/Intro-Texte passen sich dem Fach an. (App könnte gesamt als **„Quali Run"** firmieren; „Gleichungs Run" bleibt als Mathe-Brand möglich.)
- **`setMode(m)`** kennt die neuen Keys; **`MODE_TXT`** bekommt Einträge je Englisch-Modus (Brand-Sub, Intro, Placeholder, Hinweis).
- **`scoreKey()`** erweitert um `vocab/grammar/reading/mediation/listening/engmix` (eigene Boards je Modus).
- **Cloud:** `mode`-Spalte nimmt die neuen Keys ohne Migration; Klassen-Board funktioniert je Modus automatisch. (Optional: Fach in der Board-Anzeige gruppieren.)
- **`makeProblem()/makeBoss()`** dispatchen je Modus auf die Englisch-Generatoren/Bänke (analog `GENS`, `PGENS` …): `VOCABGENS`, `GRAMMARGENS`, Bank-Reader für `reading/mediation/listening`, `ENGMIXGENS`.
- **Antwort-Pipeline:** `checkAnswer` erhält Zweige für `kind:'choice'`/`'order'`; `parseAnswer`/Normalisierung für Text-Englisch.
- **Sprüche:** vorerst geteilter `MSG`-Pool (LOOIS-Personalisierung bleibt); optional eigene EN-Buckets später.

## 9. Verifikation (Node-vm, wie bei Mathe)
- **Bank-Lint (massenhaft):** jede Aufgabe valide? `answer` ∈ `accept`; bei `choice` ist `answerIndex` gültig & Distraktoren ≠ Lösung & eindeutig; keine leeren Felder; Normalisierung von `answer` ist „round-trip"-stabil; Längen-/HTML-Checks.
- **Generatoren:** Verbformen gegen `EN_VERBS` gegenprüfen; Template-Slots erzeugen nur Werte aus kontrollierten Listen; 10.000+ generierte Items → 0 Inkonsistenzen.
- **Parser-Tests:** Normalisierung (Apostrophe, BE/AE, Groß/Klein, Satzzeichen) deterministisch.
- **Browser:** Antworttypen (MC/Order/Audio), TTS-Fallback, Layout Lesetext, Endgegner, Klassen-Board je Modus.

## 10. Umfang & Roadmap (Phasen)
Inhalt ist der Aufwandstreiber. Vorschlag in Stufen, jede für sich spielbar:

- **Phase 1 — Fundament + Vokabeln + Grammatik (generiert).** Fächer-Umschalter, MC-/Text-UI, `vocab` (Startliste ~300–500 Wörter, ausbaubar Richtung 1.850) + `grammar` (Zeitformen/Steigerung/Artikel aus Tabellen). Bereits voll spielbar inkl. Board.
- **Phase 2 — Grammatik-Templates** (if-clauses, passive, reported, relative, modals, question tags) + Satzbau-`order`.
- **Phase 3 — Leseverstehen** (Text-Bank, MC/true-false) + Mix.
- **Phase 4 — Sprachmittlung** (Faktfragen) + **Hörverstehen** (TTS) inkl. Fallbacks.
- **Phase 5 — Feinschliff:** mehr Inhalte Richtung 1.850-Wortschatz, eigene EN-Sprüche, Mix-Schieberegler, ggf. Fehler-Wiederholung (falsch beantwortete Items häufiger).

## 11. Entscheidungen (festgelegt)
1. **Fächer-Umschalter Mathe ↔ Englisch** auf der Startseite → **JA**. App-Dach „Quali Run" (Mathe-Brand „Gleichungs Run" bleibt erhalten).
2. **Hörverstehen (TTS):** **später** — `listening` rückt ans Ende der Roadmap; Phasen 1–3 ohne Audio voll spielbar.
3. **Inhalts-Herkunft:** **Ich (Assistent) kuratiere** Wortlisten/Texte auf A2+/Quali-Niveau, **User prüft** gegen. (Eigenes Material des Users kann jederzeit ergänzt werden.)

### Offen / noch zu klären
- **Vokabel-Umfang** zum Start (Vorschlag ~400 Wörter, ausbaubar Richtung 1.850) und **Themen-Priorität** (Alltag vs. Beruf zuerst).

### Roadmap nach Entscheidungen (aktualisiert)
- **Phase 1 ✅:** Fächer-Umschalter + Antwort-UI (MC/Text) + **Vokabeln** (306, ausbaubar) + **Grammatik (generiert: Zeitformen/Steigerung/Plural/Artikel)** — spielbar inkl. Board.
  - UX: Countdown 3·2·1 nur am Rundenstart, dazwischen kurzes „LOS!"; Return/Enter löst Prüfen aus (nicht „weiter").
- **Phase 2 ✅:** Grammatik-Templates (if-clauses I/II, passive, reported speech, relative clauses, modals, question tags) + **Satzbau** (Antworttyp `order`, Antipp-Bausteine).
- **Phase 3:** **Leseverstehen** (Text-Bank) + **Englisch-Mix**.
- **Phase 4:** **Sprachmittlung** (Faktfragen).
- **Phase 5 (zuletzt):** **Hörverstehen** (TTS + Fallbacks).
- **Laufend:** Inhalte Richtung 1.850 Wörter ausbauen, eigene EN-Sprüche, Mix-Schieberegler, Fehler-Wiederholung.
