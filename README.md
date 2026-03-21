# Bili Learning Platform

> Bilinguale KI-gestützte Lernplattform für Primarschulen
> Live: [learnbili.com](https://learnbili.com)
> GitHub: github.com/glennmorrison/bili-learning

---

## 🎯 Projekt-Kontext

**Auftraggeber:** SIS Swiss International School (intern: "Bili")
**Pilot:** Mathematik Klasse 5b, Zyklus 2 (Klassen 5–6)
**Sprachen:** Deutsch / Englisch (bilingual), später FR + IT
**Lehrplan:** LP21 (Schweiz), KMK (Deutschland), AHS (Österreich)

**Kern-Idee:** KI generiert individualisierte Aufgaben basierend auf dem Milestone-Fortschritt jedes Schülers. Lehrpersonen drucken Aufgabenblätter aus (analog), scannen ausgefüllte Blätter ein, Claude wertet aus und schlägt Milestone-Updates vor. Nur die Lehrperson kann Milestones auf "Erreicht" setzen.

---

## 🏗 Architektur

### Tech Stack
- **Frontend:** Vanilla HTML/CSS/JS – eine einzige `index.html`
- **Datenbank:** Supabase PostgreSQL (Region: Zürich 🇨🇭 eu-central-2)
- **Auth:** Supabase Auth (E-Mail + Passwort)
- **KI:** Claude API (claude-sonnet-4-20250514) via Supabase Edge Functions
- **Hosting:** Netlify (Personal Plan) → learnbili.com
- **Domain:** learnbili.com (bei Netlify registriert)

### Datei-Struktur GitHub (glennmorrison/bili-learning)
```
index.html          ← Komplette App (Login + Admin + Lehrer + Schüler)
sis_admin.html      ← Alte separate Admin-Version (deprecated)
sis_komplett.html   ← Alte separate App-Version (deprecated)
netlify.toml        ← Netlify Konfiguration (skip_processing=true)
README.md           ← Diese Datei
```

### Supabase Projekt
- **Projekt:** bili-prod
- **URL:** https://kcvfsmugfsfupzccynoi.supabase.co
- **Region:** Central Europe (Zurich) eu-central-2
- **Edge Functions:** anthropic-proxy, create-user

---

## 👥 Rollen-Hierarchie

```
Super-Admin (Glenn Morrison)
  └── Länder-Admin (pro Land)
        └── Schulleitung (pro Schule)
              └── Lehrperson (pro Klasse/Fach)
                    └── Schüler
                          └── Eltern (readonly, noch nicht implementiert)
```

**Wichtig:** Niemand kann sich selbst registrieren. Jede Rolle wird von der übergeordneten Rolle angelegt. Super-Admin wird einmalig manuell in Supabase angelegt.

**Aktuell in Supabase:**
- Glenn Morrison → super_admin → inbox.morris@gmail.com
- peter meier → lehrperson → securebox.morris@gmail.com (Test)
- xx yy → schueler → securebox.morris@gmail.com (Test, Name muss korrigiert werden)

---

## 🗄 Datenbank-Schema

### Kern-Tabellen
```
organisations → laender → schulen → klassen → klasse_faecher
profiles (alle User: admin/lehrer/schüler)
lehrperson_klassen (m:n)
schueler_klassen (m:n)
```

### Lern-Tabellen
```
curricula (pro Land & Fach)
milestones (mit title_de, title_en, title_fr)
milestone_prerequisites
schueler_milestones (uebungs_status + validiert_status GETRENNT!)
aufgaben (KI-generiert)
antworten
pruefungen + pruefung_items + pruefung_ergebnisse
```

### Wichtige Design-Entscheidung: Zwei Status-Felder
`schueler_milestones` hat ZWEI getrennte Status:
- **uebungs_status** (0-3): Wird von KI/Schüler gesetzt. Zeigt Übungsfortschritt.
- **validiert_status** (0-3): Wird NUR von Lehrperson gesetzt (nach Prüfung). Einzige Wahrheit ob ein Milestone "erreicht" ist.

→ KI kann nie einen Milestone als "erreicht" markieren. Nur die Lehrperson nach einer echten Prüfung.

---

## 📚 Curriculum: Mathematik Klasse 5/6 (LP21)

### 16 Milestones in 5 Einheiten

**Zahl & Variable**
- MS1.1 Grosse Zahlen → MS1.2 Runden & Schätzen
- MS2.1 Addition/Subtraktion → MS2.2 Multiplikation → MS2.3 Division

**Brüche**
- MS3.1 Bruchbegriff → MS3.2 Vergleichen → MS3.3 Kürzen → MS3.4 Addition gl.N. → MS3.5 Addition versch.N.

**Dezimalzahlen**
- MS4.1 Dezimalzahlen → MS4.2 Rechnen Dezimal

**Geometrie**
- MS6.1 Umfang → MS6.2 Flächeninhalt

**Daten**
- MS12.1 Daten erheben → MS12.2 Diagramme

### Bloom-Niveaus
erinnern → verstehen → anwenden → analysieren → evaluieren → kreieren

### Differenzierung
- ★ Standard (Klassenziel)
- ★★ Erweiterung
- ★★★ Vertiefung

---

## 🌍 i18n (Mehrsprachigkeit)

Zentrales Dictionary `T` in `index.html`:
```javascript
const T = { DE: {...}, EN: {...}, FR: {...}, IT: {...} }
```

**Neue Sprache hinzufügen:** Einfach neuen Block ins `T{}` Dictionary eintragen. Kein weiterer Code nötig.

Sprache pro User in `profiles.sprache` gespeichert und in Supabase persistiert.

---

## 🔧 Supabase Edge Functions

### anthropic-proxy
- Leitet Anfragen an Claude API weiter
- Hält den API-Key serverseitig (nie im Browser)
- URL: `https://kcvfsmugfsfupzccynoi.supabase.co/functions/v1/anthropic-proxy`

### create-user
- Legt neue User in Supabase Auth + profiles Tabelle an
- Wird vom Admin-Panel aufgerufen
- Nutzt SERVICE_ROLE_KEY (serverseitig)

---

## 🎓 Pädagogisches Modell

### Digital → Analog → Digital Flow
1. **KI generiert** personalisiertes Aufgabenblatt (PDF/Print)
2. **Schüler löst** auf Papier (Primarschule bleibt analog!)
3. **Lehrperson scannt** ein → Claude wertet aus
4. **Lehrperson bestätigt** Milestone-Updates

### Differenzierungs-Routen
- **Förder-Route:** Prerequisite-Lücken → KI übt Vorläufer-Milestones
- **Standard-Route:** Klassenziel → KI übt auf ★–★★
- **Voraus-Route:** Schon erreicht → KI pusht zu nächstem Milestone oder ★★★

### Wichtige Regel
Prüfung ist gleich für alle. Differenzierung nur im Üben, nicht im Beurteilen.

---

## 📋 Offene Todos (Stand: März 2026)

### Dringend (Pilot-Start)
- [x] Demo-Daten aus DB-Objekt entfernen (Länder/Schulen/Klassen/Lehrpersonen)
- [x] Admin-Panel: Länder/Schulen/Klassen mit echter Supabase-Anbindung
- [ ] Lehrperson-Dashboard: Schüler aus Supabase laden (nicht Demo-Daten)
- [ ] Test-Schüler "xx yy" umbenennen / löschen
- [ ] Ersten echten Schüler-Flow testen (Login → Aufgabe → Feedback)

### Mittelfristig
- [ ] i18n komplett durchziehen (alle hardcodierten Texte auf t() umstellen)
- [ ] Curriculum-Parametrisierung über Webapp (Admin kann Milestones anlegen/bearbeiten)
- [ ] Prüfungs-Matrix Interface (Lehrperson definiert: Aufgabe X = Milestone Y)
- [ ] PDF-Export für Aufgabenblätter (jsPDF oder Puppeteer)
- [ ] Eltern-Login (readonly auf Kind-Profil)
- [ ] E-Mail Benachrichtigungen (Supabase Auth Email Templates)

### Langfristig
- [ ] Weitere Fächer parametrisieren (Deutsch, Englisch, NMG)
- [ ] Weitere Länder/Lehrpläne (KMK, AHS, IB)
- [ ] Mobile App (PWA)
- [ ] Eigene Domain für Supabase
- [ ] DSGVO/revDSG Datenschutzkonzept finalisieren
- [ ] DPA mit Supabase unterzeichnen
- [ ] Eltern-Einwilligung Prozess implementieren

---

## 🔑 Wichtige Keys & Zugänge

> ⚠️ Diese Datei ist PUBLIC auf GitHub. Keine echten Keys hier eintragen!

Alle Keys sind in:
- **Netlify:** Environment Variables (Settings → Environment Variables)
- **Supabase:** Edge Function Secrets (CLI: `supabase secrets set`)
- **Lokal:** Mac Notizen (Glenn)

Keys die existieren:
- `SUPABASE_URL` → Netlify + App hardcoded
- `SUPABASE_ANON_KEY` (Legacy) → hardcoded in index.html (ok, da RLS aktiv)
- `ANTHROPIC_API_KEY` → Supabase Edge Function Secret
- `SERVICE_ROLE_KEY` → Supabase Edge Function Secret

---

## 🚀 Deployment

```
GitHub (glennmorrison/bili-learning)
  → Netlify Auto-Deploy (bei jedem Push auf main)
  → learnbili.com
```

**Lokales Testen:** Einfach `index.html` im Browser öffnen (funktioniert mit hardcodiertem Supabase Key).

---

## 💬 Für neuen Claude-Chat

Wenn du dieses README einem neuen Claude-Chat gibst, weiss Claude sofort:
- Was Bili ist und für wen
- Die komplette technische Architektur
- Alle Entscheidungen die getroffen wurden
- Was noch offen ist

Zusätzlich die aktuelle `index.html` von GitHub hochladen für den Code-Kontext.

---

*Erstellt: März 2026 | Autor: Glenn Morrison & Claude*
