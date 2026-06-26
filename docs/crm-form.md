# D2 — Optimiertes Onepage CRM-Formular für Agentur-Onboarding

**Kontext:** Deliverable D2 aus dem Adcon-Projekt "Agency Workflow: Multi-Source-Briefing zu Website" für Onepage.io. Dieses Formular ist der zentrale Sammelpunkt und Trigger des n8n-Workflows "Input: Onepage Formular" — sobald ein Onepage-Agenturkunde (z. B. eine Agentur, die selbst Kundenanfragen für Website-Projekte über Onepage abwickelt) es auf seiner Website einbindet, läuft bei jeder Einreichung automatisch die Multi-Source-Aggregation, KI-Extraktion und (optional) der Website-Scraper an.

> Hinweis zum aktuellen Onepage-Workspace: Das Konto hat sein Site-Limit erreicht, daher liegt dieses Deliverable als **Spezifikation** vor (Markdown + fertiges `create_crm_form`-JSON in `crm-form-schema.json`), statt als live gebaute Onepage-Section. Sobald ein Site-Slot frei ist, lässt sich das Formular 1:1 über die Onepage-MCP-Tools `create_vibe_section` + `create_crm_form` umsetzen — das JSON ist bereits in der vom Tool erwarteten Struktur.

---

## 1. Warum dieses Formular "optimiert" ist

- **Progressive Disclosure statt Wall-of-Fields:** 4 kurze Schritte (Kontakt → Projekt → Briefing → Abschluss) statt eines langen Einzelformulars. Erhöht Abschlussquote, weil der Erstaufwand klein wirkt.
- **Minimale Pflichtfelder:** Nur 4 von 19 Feldern sind `required` (Firma, Ansprechpartner, E-Mail, Projektart, Briefing-Ziel, DSGVO-Consent). Alles, was die KI-Extraktion auch aus Folgegesprächen (Meeting, Slack, E-Mail) nachträglich ergänzen kann, ist optional.
- **Feldnamen 1:1 auf den n8n-Normalize-Node gemappt:** `client_name`, `contact_email`, `existing_url` greifen direkt in die bestehende Feldauflösung des Webhook-Workflows (siehe Abschnitt 3) — kein Anpassungsbedarf am n8n-Workflow.
- **Direkt mit dem Briefing-Extraktionsschema kompatibel:** Felder wie `goals`, `target_audience`, `tone_of_voice`, `key_messages`, `brand_colors`, `cta`, `constraints` entsprechen exakt den Spalten des einheitlichen `extracted_json`-Schemas aus dem n8n-Blueprint — die KI-Extraktion im Sub-Workflow `extract_and_store` muss diese Werte dann nicht erst aus Fließtext raten, sondern bekommt sie bereits strukturiert.
- **DSGVO-konform:** `gdpr_consent` als nativer `form-consent`-Block (Pflichtfeld), `newsletter_optin` separat und freiwillig (Trennung von Vertragszweck und Marketing-Einwilligung).
- **Conversion-Pfad für Leads ohne fertiges Briefing:** Wer nur Firma, Name, E-Mail und Projektart angibt, kann trotzdem absenden — die übrigen Kanäle (Meeting, Slack, E-Mail) liefern den Rest nach, der Sub-Workflow `extract_and_store` reichert das Projekt im Hintergrund weiter an.

---

## 2. Formularstruktur

| Step | Feld (Key) | Typ | Pflicht | Optionen / Platzhalter |
|---|---|---|---|---|
| 1. Kontakt & Unternehmen | `client_name` | form-input | ✅ | "z. B. Musterfirma GmbH" |
| | `contact_name` | form-name | ✅ | — |
| | `contact_email` | form-email | ✅ | "name@firma.de" |
| | `phone` | form-phone-number | – | — |
| | `existing_url` | form-input | – | "https://" |
| 2. Projekt-Eckdaten | `project_type` | form-select | ✅ | Neue Website / Relaunch / Landingpage / Onlineshop / Sonstiges |
| | `budget` | form-select | – | < 5k / 5–15k / 15–50k / > 50k / Noch unklar |
| | `deadline` | form-datepicker | – | — |
| | `desired_pages` | form-select (Mehrfachauswahl) | – | Startseite, Leistungen, Über uns, Team, Referenzen, Blog, Kontakt, Karriere, Onlineshop |
| 3. Briefing & Markenwelt | `goals` | form-textarea | ✅ | "Was soll die neue Website erreichen?" |
| | `target_audience` | form-input | – | "Wer soll angesprochen werden?" |
| | `tone_of_voice` | form-select | – | seriös / freundlich / verspielt / luxuriös / minimalistisch / technisch |
| | `key_messages` | form-textarea | – | "Eine Botschaft pro Zeile" |
| | `brand_colors` | form-input | – | "#1A73E8, Blau & Weiß ..." |
| | `cta` | form-input | – | "Jetzt Termin buchen" |
| | `constraints` | form-textarea | – | "Muss/Darf-nicht-Vorgaben" |
| 4. Abschluss | `attachments` | form-uploader | – | Logo / Markenmaterial |
| | `newsletter_optin` | form-checkbox | – | — |
| | `gdpr_consent` | form-consent | ✅ | Datenschutz-Einwilligung |

Vollständiges, direkt einsetzbares Tool-Payload: siehe `crm-form-schema.json` (Struktur exakt passend zu `create_crm_form`/`update_crm_form` der Onepage-MCP).

---

## 3. Mapping auf den bestehenden n8n-Workflow

Workflow **"Input: Onepage Formular"** (n8n, ID `evQHmYTPf6MLlEKk`), Webhook-Node **"Onepage Form Webhook"**:

- Production-URL: `https://n8n.adcon-tech.de/webhook/onepage-form`
- Methode: `POST`, kein Credential nötig (natives Onepage-Webhook-Format)

Der Node **"Normalize Form"** löst Felder mit Fallback-Kette auf:

| n8n-Zielfeld | Fallback-Kette im Code-Node | Formularfeld liefert |
|---|---|---|
| `project_id` | `body.project_id ?? body.contact_email ?? body.email ?? ...` | wird automatisch aus `contact_email` generiert — **kein eigenes Formularfeld nötig** |
| `client_name` | `body.client_name ?? body.company ?? body.name` | `client_name` (exakter Treffer, höchste Priorität) |
| `contact_email` | `body.contact_email ?? body.email` | `contact_email` (exakter Treffer) |
| `existing_url` | `body.existing_url ?? body.website` | `existing_url` (exakter Treffer) |
| `source` | fix `"form"` | — |
| `raw_content` | `JSON.stringify(body)` | **gesamtes Payload**, inkl. aller zusätzlichen Felder (`project_type`, `budget`, `goals`, `tone_of_voice`, `key_messages`, …) |

**Wichtig:** Alle Felder, die nicht explizit von "Normalize Form" gemappt werden (also alles außer den drei oben), gehen nicht verloren — sie landen vollständig in `raw_content` und werden vom nachgeschalteten Sub-Workflow `extract_and_store` per Claude in das einheitliche `extracted_json`-Schema überführt (`goals`, `target_audience`, `tone_of_voice`, `key_messages`, `sections_wanted`, `brand_colors`, `content_blocks`, `cta`, `constraints`, `confidence`). Da das Formular diese Felder bereits strukturiert liefert, kann die KI-Extraktion deutlich präziser arbeiten als bei freiem Fließtext.

Anschließend greift wie gewohnt:
- `Has Website?` prüft `existing_url` → bei vorhandenem Wert wird automatisch `Sub: Webscraper (Deep, agentisch)` angestoßen, um Ton, Farben und Sektionsstruktur der Bestandsseite einzulesen.

---

## 4. Setup, sobald ein Onepage-Site-Slot frei ist

1. **Vibe-Section bauen:** Im gewünschten Onepage-Site/-Page eine React-Section "Agentur-Briefing Formular" über `create_vibe_section` anlegen (Mehrstufiges Formular mit Step-Indikator, `@siteui`-Primitives: Title, Text, Input, Select, Textarea, Checkbox, Button).
2. **CRM-Form registrieren:** `create_crm_form({ react_app_id, name: "Agentur-Website-Briefing", steps: [...] })` mit dem Inhalt aus `crm-form-schema.json` aufrufen → liefert `form_id`.
3. **Submit-Code verdrahten:** Im Formular-Code `import { crm } from 'onepage'` und beim Absenden `await crm.submitForm({ formId, data })` mit den oben genannten flachen Keys (Uploads vorher per `crm.uploadFormFile(file)`).
4. **Webhook in Onepage CRM eintragen** (laut bestehender Anleitung `Setup-Anleitung_Input-Onepage-Formular.md`):
   - Onepage → Projekt → **CRM** → **+ → Add Integration → Webhook**
   - Production-URL `https://n8n.adcon-tech.de/webhook/onepage-form` eintragen, **Submit**.
5. **Workflow aktivieren** und mit einem Testformular durchspielen; in n8n unter **Executions** prüfen, ob `client_name`, `contact_email`, `existing_url` korrekt durchkommen und `raw_content` das vollständige Payload enthält.

---

## 5. Empfehlung an Onepage.io

Dieses Formular eignet sich als **Vorlage/Showcase für Agenturpartner**: Jede Agentur, die selbst Kundenanfragen für Website-Projekte sammelt, kann es 1:1 auf ihrer eigenen Onepage-Site einbinden und an ihren bestehenden (oder diesen) n8n-Workflow anschließen. Es demonstriert gleichzeitig, wie nativ Onepage-CRM-Formulare mit Automatisierungs-Stacks wie n8n zusammenspielen — relevanter Bestandteil für Blogartikel, Screenshot-Guide und Video-Tutorial (D3–D5).
