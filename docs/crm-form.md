# Onepage CRM-Formular: Agentur-Website-Briefing

**Beschreibung:** Ein öffentlich verfügbares, mehrstufiges CRM-Formular für Agenturen, die Website-Projekte über Onepage abwickeln. Es dient als zentraler Sammelpunkt für Kundenanfragen und ist direkt als Trigger für n8n-Automations-Workflows konzipiert — sobald ein Interessent das Formular auf der Agentur-Website einreicht, startet automatisch die Multi-Source-Aggregation, KI-Extraktion und (optional) der Website-Scraper.

Das Formular ist als fertige Vorlage nutzbar: Das vollständige `create_crm_form`-Payload liegt in `crm-form-schema.json` in der exakt vom Tool erwarteten Struktur vor und lässt sich über die Onepage-MCP-Tools `create_vibe_section` + `create_crm_form` direkt einrichten.

---

## 1. Warum dieses Formular "optimiert" ist

- **Progressive Disclosure statt Wall-of-Fields:** 4 kurze Schritte (Kontakt → Projekt → Briefing → Abschluss) statt eines langen Einzelformulars. Erhöht die Abschlussquote, weil der Erstaufwand klein wirkt.
- **Minimale Pflichtfelder:** Nur wenige Felder sind `required` (Firma, Ansprechpartner, E-Mail, Projektart, Briefing-Ziel, DSGVO-Consent). Alles, was die KI-Extraktion auch aus Folgegesprächen (Meeting, Slack, E-Mail) nachträglich ergänzen kann, ist optional.
- **Feldnamen 1:1 auf den n8n-Normalize-Node gemappt:** `client_name`, `contact_email`, `existing_url` greifen direkt in die bestehende Feldauflösung des Webhook-Workflows — kein Anpassungsbedarf am n8n-Workflow.
- **Direkt mit dem Briefing-Extraktionsschema kompatibel:** Felder wie `goals`, `target_audience`, `tone_of_voice`, `key_messages`, `brand_colors`, `cta`, `constraints` entsprechen exakt den Spalten des einheitlichen `extracted_json`-Schemas — die KI muss diese Werte nicht erst aus Fließtext raten, sondern bekommt sie bereits strukturiert.
- **DSGVO-konform:** `gdpr_consent` als nativer `form-consent`-Block (Pflichtfeld), `newsletter_optin` separat und freiwillig (Trennung von Vertragszweck und Marketing-Einwilligung).
- **Conversion-Pfad für Leads ohne fertiges Briefing:** Wer nur Firma, Name, E-Mail und Projektart angibt, kann trotzdem absenden — die übrigen Kanäle (Meeting, Slack, E-Mail) liefern den Rest nach.

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

## 3. Anbindung an einen n8n-Webhook

Das Formular ist für die Anbindung an einen n8n-Webhook ausgelegt. Der **"Normalize Form"**-Node löst Felder mit Fallback-Kette auf:

| n8n-Zielfeld | Fallback-Kette im Code-Node | Formularfeld liefert |
|---|---|---|
| `project_id` | `body.project_id ?? body.contact_email ?? body.email ?? ...` | wird automatisch aus `contact_email` generiert — **kein eigenes Formularfeld nötig** |
| `client_name` | `body.client_name ?? body.company ?? body.name` | `client_name` (exakter Treffer, höchste Priorität) |
| `contact_email` | `body.contact_email ?? body.email` | `contact_email` (exakter Treffer) |
| `existing_url` | `body.existing_url ?? body.website` | `existing_url` (exakter Treffer) |
| `source` | fix `"form"` | — |
| `raw_content` | `JSON.stringify(body)` | **gesamtes Payload**, inkl. aller zusätzlichen Felder |

**Wichtig:** Alle Felder, die nicht explizit gemappt werden, gehen nicht verloren — sie landen vollständig in `raw_content` und werden vom nachgeschalteten Sub-Workflow per Claude in das einheitliche `extracted_json`-Schema überführt. Da das Formular diese Felder bereits strukturiert liefert, arbeitet die KI-Extraktion deutlich präziser als bei freiem Fließtext.

Ist eine `existing_url` vorhanden, kann optional automatisch ein Website-Scraper angestoßen werden, um Ton, Farben und Sektionsstruktur der Bestandsseite einzulesen.

---

## 4. Einrichtung

1. **Vibe-Section bauen:** Im gewünschten Onepage-Site/-Page eine React-Section über `create_vibe_section` anlegen (mehrstufiges Formular mit Step-Indikator, `@siteui`-Primitives: Title, Text, Input, Select, Textarea, Checkbox, Button).
2. **CRM-Form registrieren:** `create_crm_form({ react_app_id, name: "Agentur-Website-Briefing", steps: [...] })` mit dem Inhalt aus `crm-form-schema.json` aufrufen → liefert `form_id`.
3. **Submit-Code verdrahten:** Im Formular-Code `import { crm } from 'onepage'` und beim Absenden `await crm.submitForm({ formId, data })` mit den flachen Keys (Uploads vorher per `crm.uploadFormFile(file)`).
4. **Webhook im Onepage CRM eintragen:**
   - Onepage → Projekt → **CRM** → **+ → Add Integration → Webhook**
   - Webhook-URL eintragen, **Submit**.
5. **Workflow aktivieren** und mit einem Testformular prüfen, ob `client_name`, `contact_email`, `existing_url` korrekt durchkommen und `raw_content` das vollständige Payload enthält.

---

## 5. Einsatzmöglichkeiten

Dieses Formular eignet sich als **Vorlage für Agenturpartner**: Jede Agentur, die Website-Projekte über Onepage abwickelt, kann es auf ihrer eigenen Site einbinden und an ihren n8n-Workflow anschließen. Es zeigt, wie nativ Onepage-CRM-Formulare mit Automatisierungs-Stacks wie n8n zusammenspielen — und ist ein konkretes Beispiel für einen vollständigen, KI-gestützten Briefing-Prozess von der Anfrage bis zur strukturierten Projektdaten-Übergabe.
