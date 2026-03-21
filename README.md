# Bili Learning Platform — Entwicklungs-README

> Live: learnbili.com | GitHub: github.com/glennmorrison/bili-learning
> Dieser Chat: Glenn Morrison + Claude (Sonnet 4.6)
> Stand: 21. März 2026

---

## WICHTIG FÜR NEUEN CLAUDE-CHAT

Wenn du dieses README liest, bitte folgendes beachten:

1. Antworte auf Deutsch
2. Sei direkt und pragmatisch — kein Overhead
3. Baue immer weiter auf dem bestehenden Code auf, nie neu starten
4. Fixes immer als Python-Script via bash_tool (str_replace oder direktes Ersetzen)
5. Nach jedem Fix: Datei nach /mnt/user-data/outputs/ kopieren und present_files aufrufen
6. Wenn Glenn eine Datei hochlädt: sofort nach /home/claude/ kopieren und dort weiterarbeiten
7. Nie fragen ob du weitermachen sollst — einfach machen
8. Immer beide Dateien ausgeben wenn sich was ändert: index.html + README.md
9. Screenshots werden für Bug-Reports hochgeladen — direkt analysieren und fixen
10. Stil: kurze, direkte Antworten. Kein "Ich werde nun..." — einfach tun.

---

## Projekt-Kontext

Bili = bilinguale KI-Lernplattform für Primarschulen. Schüler wechseln Unterrichtssprache je nach Wochentag/Lehrperson (z.B. Mathe Montag auf Deutsch, Dienstag auf Englisch).

Kern-Flow: KI generiert Aufgabenblatt → Schüler löst auf Papier → Lehrperson scannt ein → Claude wertet aus → Lehrperson validiert Milestones.

Nur Lehrperson kann Milestones als "erreicht" markieren. KI setzt nur uebungs_status, nie validiert_status.

---

## Tech Stack

- Frontend: Vanilla HTML/CSS/JS — EINE einzige index.html (ca. 2400 Zeilen)
- Datenbank: Supabase PostgreSQL (Zürich 🇨🇭 eu-central-2)
- Auth: Supabase Auth (E-Mail + Passwort, keine Selbstregistrierung)
- KI: Claude API via Supabase Edge Functions (API-Key nie im Browser)
- Hosting: Netlify Personal ($9/Mt) → learnbili.com
- netlify.toml: skip_processing=true (verhindert Cloudflare Script-Injection!)

## Supabase Credentials (hardcoded in index.html — ok wegen RLS)

```
SB_URL = 'https://kcvfsmugfsfupzccynoi.supabase.co'
SB_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImtjdmZzbXVnZnNmdXB6Y2N5bm9pIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzQwMzkwNzgsImV4cCI6MjA4OTYxNTA3OH0.aUfTFVBpgYTCna57M6kUH7VgUMcfgyPKjRqpm6OvNs4'
SB_FN = 'https://kcvfsmugfsfupzccynoi.supabase.co/functions/v1'
```

WICHTIG: Legacy Anon Key (eyJ...) verwenden, NICHT den neuen sb_publishable_ Key — der funktioniert nicht mit signInWithPassword!

---

## Gelöste Probleme (Referenz für neue Chats)

### Problem: Netlify injiziert Cloudflare-Scripts die onclick-Handler brechen
Lösung: netlify.toml mit skip_processing=true. Alle Admin-Nav-Items bekommen data-view Attribute statt onclick. Event-Delegation Script am Ende der Seite.

### Problem: Admin-Nav klickbar erst wenn switchToAdmin() aufgerufen wird
Ursache: Event-Delegation Script war im DOM bevor Admin-Screen existierte.
Lösung: Event-Listener wird erst in switchToAdmin() registriert (einmalig via adminNavReady Flag).

### Problem: Nach Login landet Browser auf alter URL (sis_komplett.html)
Lösung: In launch() wird window.history.replaceState({}, '', '/') aufgerufen.

### Problem: renderSchueler() Namenskonflikt (Admin + Lehrer hatten gleiche Funktion)
Lösung: Admin-Version heisst renderAdminSchueler().

### Problem: Netlify deploy limit erreicht (Free Plan)
Lösung: Upgrade auf Personal Plan ($9/Mt).

### Problem: RLS blockiert INSERT auf laender/schulen/klassen
Lösung: RLS Policies für Admin-Rollen erstellt (siehe SQL unten).

### Problem: Demo-Daten (SIS Basel, SIS Zürich etc.) waren hardcodiert
Lösung: DB-Objekt geleert, loadDBFromSupabase() lädt echte Daten aus Supabase.

### Problem: Fächer waren hardcodiert im faecher_pool Array
Lösung: Admin-Panel → Fächer Tab, lädt/speichert in Supabase faecher Tabelle.

### Problem: Lehrpersonen-Modal hatte "Einladung senden" Button ohne Funktion
Lösung: Modal hat jetzt E-Mail + Passwort Felder, ruft create-user Edge Function auf.

### Problem: "Person anlegen" Button im Schüler-Modal rief nur closeModal() auf
Ursache: saveNewSchueler() Funktion fehlte komplett, Button hatte falschen onclick.
Lösung: saveNewSchueler() Funktion hinzugefügt, ruft create-user Edge Function auf.

### Problem: schueler_klassen Insert schlägt fehl
Ursache: Edge Function create-user gibt user_id nicht im Response zurück.
Status: NOCH NICHT GELÖST — nächste Aufgabe!
Fix: Edge Function muss return new Response(JSON.stringify({ user_id: data.user.id })) zurückgeben.

### Problem: saveEditKlasse speichert nur ins lokale DB-Objekt, nicht Supabase
Lösung: saveEditKlasse() ist jetzt async, löscht und re-insertet lehrperson_klassen in Supabase.

---

## Datenbank-Schema

### Kern-Tabellen
```sql
organisations
laender (id, name, iso_code, sprachen[], lehrplan, notensystem, timezone)
schulen (id, land_id, name, city, active)
klassen (id, schule_id, name, stufe, schuljahr, aktiv)
klasse_faecher (klasse_id, fach_id)
faecher (id, name)
profiles (id uuid=auth.uid, vorname, nachname, email, rolle, sprache, aktiv, geburtsjahr)
lehrperson_klassen (lehrperson_id, klasse_id, unterrichtssprache)  -- unterrichtssprache NEU
schueler_klassen (schueler_id, klasse_id)
```

### Lern-Tabellen
```sql
curricula (id, land_id, fach_id, name, version, status, stufe, zyklus)  -- stufe+zyklus NEU
milestones (id, curriculum_id, code, title_de, title_en, title_fr, lp21_ref, bloom_stufe, niveau, unit, sort_order, ai_generatable)
milestone_prerequisites (milestone_id, prerequisite_id)
schueler_milestones (schueler_id, milestone_id, uebungs_status, validiert_status)
aufgaben (id, milestone_id, frage_text, loesung_text, kontext, bloom_stufe, niveau, sprache, ki_generiert)
antworten (id, aufgabe_id, schueler_id, antwort_text, ist_korrekt, ki_feedback)
pruefungen, pruefung_items, pruefung_ergebnisse
```

### RLS Policies (alle ausgeführt in Supabase SQL Editor)
```sql
-- profiles: alle können lesen
CREATE POLICY profiles_open ON profiles FOR SELECT USING (true);

-- laender: admin insert
CREATE POLICY laender_insert ON laender FOR INSERT WITH CHECK (
  EXISTS (SELECT 1 FROM profiles WHERE id=auth.uid() AND rolle IN ('super_admin','laender_admin')));
CREATE POLICY laender_select ON laender FOR SELECT USING (true);
CREATE POLICY laender_update ON laender FOR UPDATE USING (
  EXISTS (SELECT 1 FROM profiles WHERE id=auth.uid() AND rolle IN ('super_admin','laender_admin')));

-- schulen
CREATE POLICY schulen_insert ON schulen FOR INSERT WITH CHECK (
  EXISTS (SELECT 1 FROM profiles WHERE id=auth.uid() AND rolle IN ('super_admin','laender_admin','schulleitung')));
CREATE POLICY schulen_select ON schulen FOR SELECT USING (true);

-- klassen
CREATE POLICY klassen_insert ON klassen FOR INSERT WITH CHECK (
  EXISTS (SELECT 1 FROM profiles WHERE id=auth.uid() AND rolle IN ('super_admin','laender_admin','schulleitung')));
CREATE POLICY klassen_select ON klassen FOR SELECT USING (true);

-- faecher, klasse_faecher, lehrperson_klassen, schueler_klassen: alle erlaubt (Pilot)
CREATE POLICY faecher_all ON faecher FOR ALL USING (true) WITH CHECK (true);
CREATE POLICY klasse_faecher_all ON klasse_faecher FOR ALL USING (true) WITH CHECK (true);
CREATE POLICY lehrperson_klassen_insert ON lehrperson_klassen FOR ALL USING (true) WITH CHECK (true);
CREATE POLICY schueler_klassen_insert ON schueler_klassen FOR ALL USING (true) WITH CHECK (true);
```

---

## Edge Functions (deployed via Supabase CLI)

### anthropic-proxy
Leitet Claude API Anfragen weiter. Secret: ANTHROPIC_API_KEY

### create-user
Legt User in Supabase Auth + profiles Tabelle an. Secret: SERVICE_ROLE_KEY

BUG: Gibt user_id nicht zurück!
Aktueller Response (falsch): { message: 'ok' } oder ähnliches
Korrekter Response (fix nötig): { user_id: data.user.id }

Deploy-Befehl (Mac Terminal):
```bash
eval "$(/opt/homebrew/bin/brew shellenv zsh)"
supabase link --project-ref kcvfsmugfsfupzccynoi
supabase functions deploy create-user
```

---

## Curriculum: Mathematik 1. Klasse (live in Supabase)

Curriculum ID: 2c33c72a-2c1e-4d83-bef9-4015af83f7a9

14 Milestones:
- Zahl & Variable: MS1.1 Zahlen 1-10, MS1.2 Mengen, MS1.3 Zahlen 11-20, MS1.4 Addition bis 10, MS1.5 Subtraktion bis 10, MS1.6 Addition/Subtraktion bis 20, MS1.7 Verdopplungen
- Form & Raum: MS2.1 Formen erkennen, MS2.2 Formen zeichnen, MS2.3 Raumorientierung
- Grössen & Masse: MS3.1 Längen messen, MS3.2 Zeit/Uhr
- Daten & Zufall: MS4.1 Daten sammeln, MS4.2 sicher/möglich/unmöglich

Prerequisites eingetragen. Status: draft (noch nicht aktiv für Schüler).

Hardcodiertes Fallback-Curriculum (MS[]) für Mathe 5./6. Klasse ist noch im Code als Fallback.
getMS() gibt SB_MS (Supabase) zurück wenn geladen, sonst MS (hardcoded).

---

## i18n System

6 Sprachen: DE / EN / FR / IT / ES / PT
Dictionary T{} in index.html mit allen Interface-Texten + Bloom-Stufen + Einheiten-Namen.

Helper-Funktionen:
- getLang() — aktuelle Sprache aus CU.sprache oder localStorage
- t('key') — übersetzt Key in aktuelle Sprache
- msTitle(ms) — Milestone-Titel in aktueller Sprache (ms.title_de/en/fr etc.)
- msBloom(bloom) — Bloom-Stufe übersetzt
- msUnit(unit) — Einheiten-Name übersetzt
- setLang(lang) — wechselt Sprache, speichert in Supabase + localStorage

---

## Rollen & Aktuelle User

Hierarchie: super_admin > laender_admin > schulleitung > lehrperson > schueler > eltern

Niemand kann sich selbst registrieren. Jede Rolle wird von der übergeordneten angelegt.

Aktuelle User in Supabase:
- Glenn Morrison — inbox.morris@gmail.com — super_admin (Auth UID: 72815f7f-572e-4862-804e-562b9962bc75)
- peter meier — securebox.morris@gmail.com — lehrperson
- lena müller — lena.mller@bili.local — schueler (angelegt via Admin-Panel)
- timon ritschi — timon.ritschi@bili.local — schueler (angelegt via Admin-Panel)
- xx yy — securebox.morris@gmail.com — schueler (Test, sollte gelöscht werden)

---

## Echte Daten in Supabase

- Land: Schweiz (iso_code=CH, LP21, sprachen=[DE,EN,FR])
- Schule: SIS Männedorf
- Klasse: 1a (stufe=1, schuljahr=2025/26, Fach=Mathematik)
- Lehrperson zugewiesen: peter meier (lehrperson_klassen Eintrag vorhanden)
- Fächer: Mathematik (+ weitere via Admin-Panel)

---

## Code-Architektur index.html (~2400 Zeilen)

### Screens
- login-screen: Login-Formular
- admin-screen: Admin-Panel (nur super_admin via "Admin-Panel" Button in Sidebar)
- app: Lehrer + Schüler Interface

### Wichtige globale Variablen
```javascript
CU          // current user (aus Supabase profiles)
sb          // Supabase client
DEMO        // false in Produktion
SB_MS       // Milestones aus Supabase (leer bis loadCurriculumForTeacher() aufgerufen)
MS          // Hardcoded Fallback-Milestones (Mathe 5/6)
GS          // Generator State {selMs, lang, niveau}
```

### Wichtige Funktionen

Data Loading:
```javascript
loadDBFromSupabase()       // laender, schulen, klassen, lehrpersonen (profiles), faecher
loadCurriculumForTeacher() // Milestones → SB_MS[]
getMS()                    // SB_MS || MS (Fallback)
```

Navigation:
```javascript
boot()                     // Auth-Check, startet App
launch()                   // Nach Login: Teacher/Student/Admin UI + URL fix
tNav(view)                 // Teacher: dashboard/schueler/curriculum/generator/upload/setup
showView(view)             // Admin: overview/laender/schulen/klassen/faecher/lehrpersonen/...
switchToAdmin()            // Wechsel zu Admin-Panel, registriert Event-Delegation einmalig
switchToApp()              // Zurück zur App
```

Teacher Dashboard:
```javascript
renderDash()               // Eigene Klassen (lehrperson_klassen), Schüler, Fortschritt, Schnellaktionen
renderSchueler()           // Schüler-Karten aus Supabase profiles
renderCurr()               // Curriculum aus getMS()
renderGen()                // Aufgaben-Generator mit Supabase-Milestones
renderUpload()             // Bild-Upload + Claude-Auswertung via anthropic-proxy
renderSetup()              // Einstellungen inkl. Sprachauswahl, Schüler anlegen
```

Admin:
```javascript
renderOverview()           // Kennzahlen (noch Demo-Daten!)
renderLaender()            // Aus DB.laender (Supabase)
renderSchulen()            // Aus DB.schulen (Supabase)
renderKlassen()            // Aus DB.klassen (Supabase)
renderFaecher()            // Live aus Supabase faecher Tabelle
renderAdminSchueler()      // Live aus Supabase (muss manuell via renderAdminSchueler() getriggert werden — Bug)
renderLehrpersonen()       // Aus DB.lehrpersonen (Supabase)
renderAdmins()             // Noch Demo-Daten
renderCurriculum()         // Übersicht der Curricula
renderSchema()             // SQL-Schema Ansicht
renderDatenschutz()        // Datenschutz-Infos
```

Save-Funktionen (alle async, Supabase):
```javascript
saveNewLand()              // → laender
saveNewSchule()            // → schulen
saveNewKlasse()            // → klassen + klasse_faecher + lehrperson_klassen (mit unterrichtssprache)
saveEditKlasse(id)         // → klassen update + lehrperson_klassen delete+reinsert (BUG: ohne unterrichtssprache)
saveNewFach()              // → faecher
deleteFach(id, name)       // → faecher delete
saveNewSchueler()          // → Edge Function create-user → schueler_klassen (BUG: user_id fehlt im Response)
saveLehrperson()           // → Edge Function create-user
setLang(lang)              // → profiles update + localStorage
```

---

## Bekannte Bugs (Priorität)

### ~~BUG #1 — ERLEDIGT: schueler_klassen Insert~~
Fix: Edge Function create-user gibt jetzt user_id: data.user.id zurück. Via Terminal deployed.

### ~~BUG #2 — ERLEDIGT: renderAdminSchueler lädt nicht automatisch~~
Fix: showView() auf async umgestellt, await renders[v]() statt renders[v]().

### BUG #3: saveEditKlasse ohne unterrichtssprache
Problem: Beim Bearbeiten einer Klasse werden Lehrpersonen ohne unterrichtssprache gespeichert (Default DE).
Fix: Im openEditKlasseModal pro Lehrperson ein Sprach-Dropdown anzeigen (wie beim Anlegen).

### BUG #4: Admin-Übersicht zeigt Demo-Zahlen
Problem: renderOverview() zeigt noch hardcodierte Zahlen (4 Länder, 7 Schulen, 98 Klassen, 2290 Schüler).
Fix: Echte COUNT(*) Abfragen aus Supabase.

---

## Nächste Schritte (nach Bug-Fixes)

1. Schüler-Login testen: lena.mller@bili.local einloggen, Passwort aus dem Modal-Feedback
2. Lehrer-Dashboard Nav erweitern: neue Reiter links (Klassen-Ansicht, Milestone-Validierung, Schüler-Dossier)
3. Schüler-Interface: Milestone-Status aus Supabase laden statt hardcodiert
4. Jahreswechsel-Assistent: neue Klassen anlegen, Schüler verschieben
5. Weitere Curricula importieren (Mathe 2-6, andere Fächer)
6. CSV-Import für Schüler (bulk)

---

## Deployment

GitHub push auf main → Netlify Auto-Deploy → learnbili.com (~30 Sekunden)

Releases auf GitHub setzen vor grossen Änderungen (v0.1-pilot bereits gesetzt).

---

Erstellt: März 2026 | Glenn Morrison & Claude Sonnet 4.6
