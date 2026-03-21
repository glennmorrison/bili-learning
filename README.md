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

- Frontend: Vanilla HTML/CSS/JS — EINE einzige index.html (ca. 2600 Zeilen)
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

### Netlify injiziert Cloudflare-Scripts
Lösung: netlify.toml mit skip_processing=true. Admin-Nav-Items mit data-view Attributen + Event-Delegation.

### Admin-Nav erst nach switchToAdmin() klickbar
Lösung: Event-Listener wird erst in switchToAdmin() registriert (einmalig via adminNavReady Flag).

### Nach Login landet Browser auf alter URL
Lösung: launch() ruft window.history.replaceState({}, '', '/') auf.

### renderSchueler() Namenskonflikt
Lösung: Admin-Version heisst renderAdminSchueler().

### RLS blockiert INSERT auf laender/schulen/klassen
Lösung: RLS Policies für Admin-Rollen erstellt.

### Jahreswechsel Revert löscht Klassen nicht
Ursache: RLS Policy für DELETE auf klassen fehlte.
Lösung: CREATE POLICY klassen_delete ON klassen FOR DELETE USING (rolle IN ('super_admin','laender_admin','schulleitung'));

### schueler_klassen Insert schlug fehl
Ursache: Edge Function create-user gab user_id nicht zurück (Response: {success:true, user: authData.user}).
Lösung: Frontend liest jetzt d.user?.id || d.user_id.

### renderAdminSchueler lud nicht automatisch
Lösung: showView() auf async umgestellt, await renders[v]().

### saveEditKlasse speicherte ohne unterrichtssprache
Lösung: Modal hat jetzt Checkboxen mit Sprach-Dropdown pro Lehrperson.

### lehrperson_klassen INSERT 400 Bad Request
Ursache: fach_id war NOT NULL und Teil des Primary Key.
Lösung: PK auf (lehrperson_id, klasse_id) geändert, fach_id nullable.

### Milestones luden nicht (SB_MS leer)
Ursache: RLS Policy fehlte auf milestones + curricula Tabellen.
Lösung: CREATE POLICY ... FOR SELECT USING (true) auf beiden Tabellen.

### curricula unique constraint auf (land_id, fach_id)
Lösung: ALTER TABLE curricula DROP CONSTRAINT curricula_land_id_fach_id_key.

### lehrperson_klassen 409 Conflict bei bilingualem Fach
Ursache: PK (lehrperson_id, klasse_id) erlaubte keinen zweiten Eintrag pro Fach.
Lösung: PK auf (lehrperson_id, klasse_id, fach_id) erweitert.

---

## Datenbank-Schema

### Kern-Tabellen
```sql
organisations
laender (id, name, iso_code, sprachen[], lehrplan, notensystem, timezone)
schulen (id, land_id, name, city, active)
klassen (id, schule_id, name, stufe, schuljahr, klasse_buchstabe, aktiv, curriculum_id)
klasse_faecher (klasse_id, fach_id, curriculum_id)
faecher (id, name, is_bilingual)  -- is_bilingual: mehrere LP pro Fach möglich
profiles (id uuid=auth.uid, vorname, nachname, email, rolle, sprache, aktiv, geburtsjahr)
lehrperson_klassen (lehrperson_id, klasse_id, fach_id, unterrichtssprache)
  -- PK: (lehrperson_id, klasse_id, fach_id)
  -- Bilingual: zwei Einträge pro Fach möglich (DE + EN)
schueler_klassen (schueler_id, klasse_id)
```

### Lern-Tabellen
```sql
curricula (id, land_id, fach_id, name, version, status, stufe, zyklus)
milestones (id, curriculum_id, code, title_de, title_en, title_fr, lp21_ref, bloom_stufe, niveau, unit, sort_order)
milestone_prerequisites (milestone_id, prerequisite_id)
schueler_milestones (schueler_id, milestone_id, uebungs_status, validiert_status)
aufgaben, antworten, pruefungen, pruefung_items, pruefung_ergebnisse
```

### RLS Policies
```sql
-- Alle SELECT offen:
profiles, laender, schulen, klassen, faecher, klasse_faecher,
lehrperson_klassen, schueler_klassen, milestones, curricula

-- INSERT/UPDATE nur für Admins:
laender, schulen, klassen (super_admin, laender_admin, schulleitung)

-- Pilot: alles erlaubt:
faecher, klasse_faecher, lehrperson_klassen, schueler_klassen
```

---

## Klassen-Logik

Klasse = Schuljahr + Stufe + Buchstabe → Name wird auto-generiert (z.B. "1a")
- schuljahr: "2025/26"
- stufe: 1–9
- klasse_buchstabe: "a", "b" etc. (Doppelzügigkeit)
- name: stufe + buchstabe (auto)

Jahreswechsel (noch nicht implementiert):
- Neue Klasse anlegen (schuljahr +1, stufe +1, gleicher buchstabe)
- Schüler zuweisen
- schueler_milestones bleibt erhalten (Fortschritt kumuliert)

---

## Curricula

4 Curricula live in Supabase:
- Mathematik 1. Klasse LP21 (ID: 2c33c72a-...) — 14 Milestones
- Mathematik 2. Klasse LP21 (ID: 3ed4b164-...) — 11 Milestones
- Deutsch 1. Klasse LP21 (ID: f3053c7d-...) — 10 Milestones
- Deutsch 2. Klasse LP21 (ID: 7980589c-...) — 9 Milestones

Curriculum-Zuweisung: pro Fach in klasse_faecher.curriculum_id
Laden: loadCurriculumForTeacher() → holt curricula aus klasse_faecher der eigenen Klassen

---

## Bilinguales Modell

faecher.is_bilingual = true → Fach kann zwei Lehrpersonen haben (eine pro Sprache)
Aktuell: Mathematik = bilingual, Deutsch = monolingual

lehrperson_klassen Einträge pro bilingualem Fach:
- peter meier → Klasse 1a → Mathematik → DE
- glenn morrison → Klasse 1a → Mathematik → EN

Lehrperson sieht nur Schüler aus eigenen Klassen (via lehrperson_klassen → schueler_klassen).

---

## i18n System

6 Sprachen: DE / EN / FR / IT / ES / PT
Dictionary T{} in index.html. Helper: getLang(), t('key'), msTitle(ms), msBloom(), msUnit(), setLang()

---

## Rollen & Aktuelle User

Hierarchie: super_admin > laender_admin > schulleitung > lehrperson > schueler > eltern

- Glenn Morrison — inbox.morris@gmail.com — super_admin
- peter meier — atomatsound@gmail.com — lehrperson (Klasse 1a, Mathematik DE)
- lena müller — lena.mller@bili.local — schueler
- timon ritschi — timon.ritschi@bili.local — schueler
- lisa simens — lisa.simens@bili.local — schueler
- sandra tiler — sandra.tiler@bili.local — schueler
- ramon ramstein — ramon.ramstein@bili.local — schueler (Klasse 1a zugewiesen ✓)
- test.debug999@bili.local — schueler (Test, löschen)
- xx yy — securebox.morris@gmail.com — schueler (Test, löschen)

---

## Echte Daten in Supabase

- Land: Schweiz (iso_code=CH, LP21, sprachen=[DE,EN,FR])
- Schule: SIS Männedorf
- Klasse: 1a (stufe=1, schuljahr=2025/26, buchstabe=a)
  - Fach Mathematik → Curriculum Mathe 1. Klasse, bilingual (peter meier DE, glenn EN)
  - Fach Deutsch → noch kein Curriculum zugewiesen

---

## Edge Functions (deployed via Supabase CLI)

### anthropic-proxy
Leitet Claude API Anfragen weiter. Secret: ANTHROPIC_API_KEY

### create-user
Legt User in Auth + profiles an. Secret: SERVICE_ROLE_KEY (als SUPABASE_SERVICE_ROLE_KEY)
Response: { success: true, user: authData.user } → Frontend liest d.user?.id

Deploy:
```bash
eval "$(/opt/homebrew/bin/brew shellenv zsh)"
cd ~/supabase/functions  (oder wo index.ts liegt)
supabase functions deploy create-user --project-ref kcvfsmugfsfupzccynoi
```

---

## Code-Architektur index.html (~2600 Zeilen)

### Wichtige globale Variablen
```javascript
CU          // current user
sb          // Supabase client
DEMO        // false in Produktion
SB_MS       // Milestones aus Supabase
DB          // {laender, schulen, klassen, lehrpersonen, faecher_pool, faecher_list, rollen}
GS          // Generator State
```

### DB.klassen Objekt
```javascript
{
  id, schuleId, name, stufe, schuljahr, buchstabe,
  faecher: [{name, fachId, curriculumId}],
  lehrpersonIds: [uuid],
  lehrpersonSprachen: {lpId: lang},
  lehrpersonIds_byFach: {fachId: [{lpId, lang}]},
  schuelerCount
}
```

### Wichtige Funktionen
```javascript
// Data
loadDBFromSupabase()        // laender/schulen/klassen/lehrpersonen/faecher/lehrperson_klassen
loadCurriculumForTeacher()  // Milestones via klasse_faecher.curriculum_id
getMS()                     // SB_MS (kein Fallback mehr)

// Navigation
boot() → launch() → tNav()/showView()/switchToAdmin()/switchToApp()

// Teacher
renderDash()     // nur eigene Schüler (via lehrperson_klassen → schueler_klassen)
renderSchueler() // nur eigene Schüler
renderCurr()     // gruppiert nach Curriculum
renderGen()      // Aufgaben-Generator
renderUpload()   // Scan + Claude-Auswertung

// Admin
renderKlassen()         // Klassen mit LP pro Fach
openEditKlasseModal()   // async, lädt Curricula, bilingual-aware LP-Auswahl
saveEditKlasse()        // speichert klasse_faecher + lehrperson_klassen pro Fach
openEditSchuelerModal() // Schüler bearbeiten inkl. Klassenzuweisung
saveEditSchueler()      // profiles + schueler_klassen
openEditLpModal()       // Lehrperson Klassen zuweisen
saveEditLp()            // lehrperson_klassen
renderAdminSchueler()   // async, lädt live aus Supabase
```

---

## Bekannte Bugs / Nächste Schritte

### Offen
- Jahreswechsel-Assistent (neue Klassen anlegen, Schüler verschieben)
- Auto-Vorschlag Curriculum beim Klasse bearbeiten (Stufe + Land + Fach → passendes Curriculum)
- is_bilingual Toggle im Fächer-Admin UI
- Admin-Übersicht zeigt noch Demo-Zahlen (renderOverview)
- Hardcodiertes MS[] Fallback-Array noch im Code (kann entfernt werden)
- Test-Schüler löschen: xx yy + test.debug999

### Erledigt heute
- Jahreswechsel-Assistent mit Revert ✓
- RLS DELETE Policy auf klassen ✓
- Bug #1: schueler_klassen Insert (d.user?.id Fix) ✓
- Bug #2: renderAdminSchueler auto-load (showView async) ✓
- Bug #3: saveEditKlasse mit unterrichtssprache ✓
- Schüler bearbeiten Modal (Klassenzuweisung) ✓
- Lehrperson bearbeiten Modal (Klassen zuweisen) ✓
- LP pro Fach mit Unterrichtssprache ✓
- Bilinguales Fach: mehrere LP pro Fach (is_bilingual) ✓
- Curricula pro Fach in klasse_faecher ✓
- 4 Curricula + Milestones in Supabase ✓
- Klassen-Schema: stufe + buchstabe + schuljahr ✓
- Nur eigene Schüler im Lehrer-Portal ✓

---

## Deployment

GitHub push auf main → Netlify Auto-Deploy → learnbili.com (~30 Sekunden)

---

Erstellt: März 2026 | Glenn Morrison & Claude Sonnet 4.6
Zuletzt aktualisiert: 21. März 2026
