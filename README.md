# Bili Learning Platform

> Bilinguale KI-gestützte Lernplattform für Primarschulen
> Live: learnbili.com | GitHub: github.com/glennmorrison/bili-learning

---

## Projekt-Kontext

**Projekt:** Bili Learning Platform (interner Pilot)
**Pilot:** Mathematik 1. Klasse Primarstufe, Zyklus 1
**Sprachen:** DE / EN / FR / IT / ES / PT
**Lehrplan:** LP21 (Schweiz), KMK (Deutschland), AHS (Österreich)

Kern-Idee: KI generiert individualisierte Aufgaben basierend auf Milestone-Fortschritt. Lehrperson druckt aus (analog), scannt ein, Claude wertet aus, Lehrperson validiert Milestones.

Bilinguales Modell: Schüler wechseln Unterrichtssprache je nach Wochentag/Lehrperson. Pro Lehrperson-Klassen-Zuweisung gibt es eine unterrichtssprache (DE/EN/FR etc.).

---

## Tech Stack

- Frontend: Vanilla HTML/CSS/JS — eine einzige index.html
- Datenbank: Supabase PostgreSQL (Zürich, eu-central-2)
- Auth: Supabase Auth (E-Mail + Passwort)
- KI: Claude API (claude-sonnet-4-20250514) via Edge Functions
- Hosting: Netlify Personal ($9/Mt) → learnbili.com

## Supabase

- URL: https://kcvfsmugfsfupzccynoi.supabase.co
- Anon Key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImtjdmZzbXVnZnNmdXB6Y2N5bm9pIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzQwMzkwNzgsImV4cCI6MjA4OTYxNTA3OH0.aUfTFVBpgYTCna57M6kUH7VgUMcfgyPKjRqpm6OvNs4
- Functions URL: https://kcvfsmugfsfupzccynoi.supabase.co/functions/v1

## Edge Functions (deployed)

1. anthropic-proxy — proxies Claude API (secret: ANTHROPIC_API_KEY)
2. create-user — creates Auth + profiles (secret: SERVICE_ROLE_KEY)
   BUG: muss user_id im Response zurückgeben: { user_id: data.user.id, ... }

---

## Rollen

super_admin > laender_admin > schulleitung > lehrperson > schueler > eltern

Aktuelle User:
- Glenn Morrison — inbox.morris@gmail.com — super_admin
- peter meier — securebox.morris@gmail.com — lehrperson
- lena müller — lena.mller@bili.local — schueler
- timon ritschi — timon.ritschi@bili.local — schueler
- xx yy — securebox.morris@gmail.com — schueler (Test, löschen)

---

## Datenbank-Schema

### Kern-Tabellen
- organisations
- laender (iso_code, sprachen[], lehrplan, notensystem, timezone)
- schulen (land_id, name, city)
- klassen (schule_id, name, stufe, schuljahr, aktiv)
- klasse_faecher (klasse_id, fach_id)
- faecher (name)
- profiles (id=auth.uid, vorname, nachname, email, rolle, sprache, aktiv)
- lehrperson_klassen (lehrperson_id, klasse_id, unterrichtssprache)
- schueler_klassen (schueler_id, klasse_id)

### Lern-Tabellen
- curricula (land_id, fach_id, name, version, status, stufe, zyklus)
- milestones (curriculum_id, code, title_de/en/fr, lp21_ref, bloom_stufe, niveau, unit, sort_order)
- milestone_prerequisites (milestone_id, prerequisite_id)
- schueler_milestones (schueler_id, milestone_id, uebungs_status, validiert_status)
- aufgaben, antworten, pruefungen, pruefung_items, pruefung_ergebnisse

### Wichtige Design-Entscheidungen
- uebungs_status (KI) und validiert_status (Lehrperson) sind GETRENNT
- Curricula an stufe gebunden, nicht an Klassen (Jahreswechsel-sicher)
- Beim Jahreswechsel: neue Klassen anlegen, Schüler neu zuweisen, Milestone-Status bleibt

---

## Curriculum: Mathematik 1. Klasse (live in Supabase)

Curriculum ID: 2c33c72a-2c1e-4d83-bef9-4015af83f7a9

14 Milestones in 4 Einheiten:
- Zahl & Variable: MS1.1–MS1.7
- Form & Raum: MS2.1–MS2.3
- Grössen & Masse: MS3.1–MS3.2
- Daten & Zufall: MS4.1–MS4.2

Prerequisites ebenfalls eingetragen.

---

## i18n

Dictionary T{} in index.html: DE / EN / FR / IT / ES / PT
Helper: getLang(), t('key'), msTitle(ms), msBloom(bloom), msUnit(unit)
Neue Sprache: neuer Block in T{} + title_XX Felder in DB

---

## Echte Daten in Supabase

- Land: Schweiz (CH, LP21, DE/EN/FR)
- Schule: SIS Männedorf
- Klasse: 1a (Stufe 1, 2025/26, Mathematik, peter meier zugewiesen)
- Fächer: Mathematik (+ weitere via Admin-Panel angelegt)

---

## BEKANNTE BUGS — NÄCHSTER CHAT

### 1. DRINGEND: schueler_klassen Insert
Edge Function create-user gibt user_id nicht zurück.
Fix in supabase/functions/create-user/index.ts:
Response muss sein: return new Response(JSON.stringify({ user_id: data.user.id }))
Dann funktioniert schueler_klassen automatisch beim Schüler anlegen.

### 2. lehrperson_klassen beim Bearbeiten
saveEditKlasse() speichert ohne unterrichtssprache.
Fix: beim Bearbeiten pro LP ein Sprach-Dropdown zeigen (wie beim Anlegen).

### 3. Admin-Übersicht Kennzahlen
Zeigen noch Demo-Zahlen. Auf echte Supabase-Counts umstellen.

### 4. renderAdminSchueler()
Lädt Daten nur manuell via Konsole. Soll beim Öffnen automatisch laden.

---

## Nächste Schritte (Priorität)

1. Edge Function create-user fixen (user_id im Response)
2. Schüler-Login testen (lena.mller@bili.local + generiertes Passwort)
3. Lehrer-Nav erweitern: neue Reiter links (Klassen-Ansicht, Milestone-Validierung)
4. Schüler-Interface: Milestone-Status aus Supabase laden
5. Admin-Übersicht auf echte Daten umstellen
6. Jahreswechsel-Assistent bauen

---

## Code-Architektur (index.html)

Wichtige Funktionen:
- boot() — Auth check, startet App
- loadDBFromSupabase() — lädt laender/schulen/klassen/lehrpersonen/faecher
- loadCurriculumForTeacher() — lädt Milestones → SB_MS[]
- getMS() — gibt SB_MS oder MS (hardcoded fallback) zurück
- tNav(view) — Teacher nav (dashboard/schueler/curriculum/generator/upload/setup)
- showView(view) — Admin nav
- switchToAdmin() / switchToApp() — wechselt zwischen Admin und App
- renderDash() — lädt eigene Klassen, Schüler, Fortschritt aus Supabase
- saveNewKlasse() — → klassen + klasse_faecher + lehrperson_klassen
- saveEditKlasse() — → klassen + lehrperson_klassen (delete+reinsert)
- saveNewSchueler() — → Edge Function create-user + schueler_klassen
- saveLehrperson() — → Edge Function create-user

---

## Deployment

GitHub push → Netlify Auto-Deploy → learnbili.com
netlify.toml: skip_processing = true

## Lokales Setup (Mac)

eval "$(/opt/homebrew/bin/brew shellenv zsh)"
supabase link --project-ref kcvfsmugfsfupzccynoi
supabase functions deploy create-user
supabase secrets set SERVICE_ROLE_KEY=...

---

Erstellt: März 2026 | Glenn Morrison & Claude | Zuletzt: 21. März 2026
