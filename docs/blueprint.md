# Blueprint: n8n-Workflows – Multi-Source Agentur-Website-Automation

**Projekt:** Onepage MCP × Adcon — Agentur-Workflow
**Stand:** 24. Juni 2026
**Autor:** Constantin Cartellieri
**Basis:** Projektnotizen & Besprechung vom 16.06.2026

---

## 1. Zielbild

Agenturen sammeln Kunden-Briefings über viele Kanäle parallel: ein Onboarding-Meeting (Fireflies/Granola), eine Nachfass-E-Mail, eine Slack-Abstimmung und ein ausgefülltes Onepage-Formular. Diese Daten liegen heute verstreut. Der Workflow führt sie **automatisch zusammen, speichert sie persistent** und erlaubt einem Projektverantwortlichen, **per Knopfdruck** eine fertige Onepage-Website daraus bauen zu lassen.

Die Architektur trennt bewusst **Sammeln** (laufend, automatisch) von **Bauen** (manuell, kontrolliert). So bleibt der teure und qualitätskritische Bau-Schritt unter menschlicher Kontrolle, während das Sammeln im Hintergrund passiert.

```
INPUTS (automatisch)                  SPEICHER              BUILD (manuell)
─────────────────────                 ────────              ───────────────
Onepage-Formular ─┐
Slack            ─┤                 n8n Data Tables          Projektleiter
E-Mail           ─┼──► Normalisieren ──► (projects +    ──► drückt Trigger ──► OnePage MCP ──► fertige Website
Meeting-Transkript┤      + KI-Extraktion   briefing_items)        │              (Ping-Pong mit LLM)
(optional) Scraper┘                              ▲                 │
                                                 └─────────────────┘
                                                  liest alle Daten zum Projekt
```

Fünf Sammel-Workflows (vier Input-Quellen + optionaler Scraper) schreiben in eine gemeinsame Datenbank. Ein sechster Workflow (Builder) liest sie aus und steuert OnePage MCP.

---

## 2. Datenmodell (n8n Data Tables)

Zwei Tabellen. `projects` ist der Anker pro Kunde/Projekt, `briefing_items` sammelt alle Roh- und veredelten Inputs dazu. Diese Trennung erlaubt beliebig viele Inputs pro Projekt aus beliebigen Quellen.

### Tabelle `projects`

| Spalte | Typ | Beschreibung |
|---|---|---|
| `project_id` | string | Eindeutiger Schlüssel (z. B. `acme-2026-06`). Manuell oder aus Formular. |
| `client_name` | string | Kundenname |
| `contact_email` | string | Haupt-Kontakt — dient auch als Matching-Schlüssel für E-Mail/Meeting-Inputs |
| `slack_channel_id` | string | Zugeordneter Slack-Channel (Matching für Slack-Input) |
| `existing_url` | string | Optionale bestehende Website (Trigger für Scraper) |
| `status` | string | `collecting` → `ready_to_build` → `building` → `built` → `published` |
| `onepage_site_id` | string | Wird vom Builder nach Site-Erstellung zurückgeschrieben |
| `onepage_page_url` | string | Live-URL nach Publish |
| `created_at` | datetime | Anlagezeitpunkt |
| `updated_at` | datetime | Letzte Änderung |

### Tabelle `briefing_items`

| Spalte | Typ | Beschreibung |
|---|---|---|
| `item_id` | string | Eindeutige ID des Einzel-Inputs |
| `project_id` | string | Fremdschlüssel auf `projects` |
| `source` | string | `form` \| `slack` \| `email` \| `meeting` \| `scrape` |
| `source_ref` | string | Original-Referenz (Message-ID, Thread-URL, Meeting-ID) für Nachvollziehbarkeit |
| `raw_content` | string | Roh-Inhalt (Transkript, E-Mail-Body, Slack-Text, HTML-Auszug) |
| `extracted_json` | string | KI-strukturierte Felder als JSON (siehe unten) |
| `received_at` | datetime | Eingangszeitpunkt |

### Einheitliches Extraktions-Schema (`extracted_json`)

Jeder Input wird per LLM in dasselbe Schema überführt — das ist der Kern, der die Multi-Source-Zusammenführung erst möglich macht:

```json
{
  "goals":        ["Was will der Kunde erreichen?"],
  "target_audience": "Zielgruppe",
  "tone_of_voice": "Markenton (z.B. seriös, verspielt)",
  "key_messages": ["Kernbotschaften / USPs"],
  "sections_wanted": ["Hero", "Leistungen", "Über uns", "Kontakt"],
  "brand_colors": ["#hex", "..."],
  "content_blocks": [{"section": "...", "text": "..."}],
  "cta": "Gewünschte Handlungsaufforderung",
  "constraints": ["Muss/Darf-nicht-Vorgaben"],
  "confidence": "high | medium | low"
}
```

---

## 3. Input-Workflows (Sammeln)

Alle vier Input-Workflows folgen demselben Muster: **Trigger → Projekt zuordnen → KI-Extraktion → in `briefing_items` speichern → `projects.updated_at` aktualisieren.** Nur Trigger und Vorverarbeitung unterscheiden sich. Ein wiederverwendbarer Sub-Workflow `extract_and_store` kapselt die gemeinsamen letzten drei Schritte.

### Gemeinsamer Sub-Workflow: `extract_and_store`

Aufgerufen per *Execute Sub-workflow* mit `{project_id?, contact_email?, slack_channel_id?, source, source_ref, raw_content}`.

1. **Resolve Project** — *Data Table (Get Row)*: Projekt über `project_id` ODER `contact_email` ODER `slack_channel_id` finden. Kein Treffer → optional neues `projects`-Row anlegen (Status `collecting`) oder in „Unmatched"-Ablage parken.
2. **AI Extract** — *AI Agent / Basic LLM Chain* (Claude). System-Prompt: „Extrahiere aus folgendem Agentur-Input die Briefing-Felder im vorgegebenen JSON-Schema. Erfinde nichts; fehlende Felder leer lassen." → Output Parser auf das Schema oben.
3. **Insert Item** — *Data Table (Insert Row)* in `briefing_items`.
4. **Touch Project** — *Data Table (Update Row)*: `updated_at = now()`.

### 3.1 Workflow „Input: Onepage-Formular" (primärer Fokus)

Laut Besprechung der Hauptkanal: Onepage-Formulare sind direkt ins CRM integriert und liefern Webhooks.

- **Trigger:** *Webhook* (POST) — Onepage CRM-Formular ruft die n8n-Webhook-URL beim Absenden auf.
- **Normalize:** *Set / Edit Fields* — Formularfelder mappen. `contact_email` und ggf. `existing_url` aus dem Payload ziehen.
- **Upsert Project:** *Data Table* — bei Formular oft der erste Kontakt → Projekt anlegen, falls nicht vorhanden, mit `existing_url` füllen.
- **→ `extract_and_store`** mit `source = form`.
- **Optional:** Wenn `existing_url` gesetzt → den Scraper-Workflow (3.5) per *Execute Sub-workflow* anstoßen.

### 3.2 Workflow „Input: Slack"

- **Trigger:** *Slack Trigger* (Event `message`) auf den relevanten Channels, ODER ein Slash-Command/Emoji-Reaction (z. B. 📋) zum gezielten Markieren von Briefing-Nachrichten — empfohlen, um Rauschen zu vermeiden.
- **Filter:** *If* — nur Nachrichten aus zugeordneten Channels / mit Marker-Reaction.
- **Map channel → project:** über `slack_channel_id`.
- **→ `extract_and_store`** mit `source = slack`, `source_ref = permalink`.

### 3.3 Workflow „Input: E-Mail"

- **Trigger:** *Gmail Trigger* (neue Mail) — gefiltert per Label (z. B. `Briefing`) oder Absender.
- **Extract:** Body (Plain/HTML bereinigt) + Anhänge optional.
- **Map sender → project:** über `contact_email`.
- **→ `extract_and_store`** mit `source = email`, `source_ref = message-id`.

### 3.4 Workflow „Input: Meeting-Transkript"

- **Trigger:** *Fireflies/Granola Webhook* bei „Transkript fertig" (alternativ *Schedule* + API-Poll).
- **Fetch transcript:** *HTTP Request* an Fireflies/Granola API mit `meeting_id`.
- **Map participants → project:** Teilnehmer-E-Mails gegen `contact_email` abgleichen.
- **→ `extract_and_store`** mit `source = meeting`, `source_ref = meeting_id`. Da Transkripte lang sind: ggf. vor der Extraktion zusammenfassen (Map-Reduce über Abschnitte).

### 3.5 Workflow „Input: Webscraper" (optional)

Scraped eine bestehende Website auf Inhalte UND Designstruktur — als Referenz für Ton, Farben und Sektionsaufbau.

- **Trigger:** *Execute Sub-workflow* (von Formular/Builder aufgerufen) oder manuell mit `existing_url`.
- **Fetch:** *HTTP Request* auf die URL. Für JS-gerenderte Seiten: Jina AI Reader (`r.jina.ai/<url>` — laut Notizen großzügiges Freikontingent) oder ein Scraping-Dienst.
- **Extract Content:** *HTML Extract* — Texte, Headings, Meta-Tags, Bild-URLs.
- **Extract Design:** LLM-Auswertung der HTML/CSS → Farbpalette, Schriftfamilien, Sektionsreihenfolge, Layout-Muster → in `brand_colors` / `sections_wanted`.
- **→ `extract_and_store`** mit `source = scrape`, `source_ref = url`.

> Hinweis: Der in der Besprechung erwähnte „Website-Klon als Messe-Feature" (Domain eintragen → QR-Code → geklonte Seite) ist im Kern derselbe Scraper-Pfad, direkt gefolgt vom Builder.

---

## 4. Builder-Workflow (manuell, OnePage MCP)

Wird **vom Projektverantwortlichen bewusst ausgelöst**, wenn genug Briefing-Daten vorliegen.

- **Trigger:** *Form Trigger* (n8n-internes Formular: Eingabe `project_id`) oder *Manual Trigger*. Optional vorgeschaltet eine Freigabe (`projects.status = ready_to_build`).

**Ablauf:**

1. **Load Project** — *Data Table (Get Row)* aus `projects` per `project_id`.
2. **Load all Items** — *Data Table (Get Rows)* aus `briefing_items` mit `project_id`.
3. **Merge & Deduplicate** — *AI Agent* (Claude — laut Besprechung **Claude Opus** für hochwertige/komplexe Seiten, Sonnet reicht für kleine). Alle `extracted_json`-Objekte aller Quellen werden zu **einem konsolidierten Briefing** verschmolzen: Widersprüche auflösen, Dubletten entfernen, finale Sektionsliste, Farben, Texte, Tonalität festlegen. Output: strukturierter Website-Bauplan.
4. **Set Status** — `projects.status = building`.
5. **Build via OnePage MCP (Ping-Pong)** — iterativer Prozess zwischen LLM und OnePage MCP-Tools (in der Besprechung als „Ping-Pong" beschrieben):
   - `create_site` → Site anlegen
   - `create_color_schema` / `create_font_kit` → Branding aus konsolidiertem Briefing
   - `create_page` → Seite
   - pro Sektion `create_vibe_section` + `write_files`/`edit_files` → React-Sektionen befüllen
   - `build_react_app` → Build/Validierung; bei Fehlern Auto-Correction-Loop (LLM liest Build-Output, korrigiert, baut neu)
   - `publish_page` → Veröffentlichen

   > **Wichtig zur Umsetzung:** OnePage MCP verlangt, dass vor jedem schreibenden Tool das passende **Skill via `onepage_skill_get` geladen** wird (server-seitig erzwungen). In n8n wird OnePage MCP über den *MCP Client*-Node angebunden; der steuernde AI-Agent ruft erst das Skill, dann die Aktion. Alternativ läuft dieser Bau-Schritt außerhalb n8n in einer Claude-Session, die per n8n angestoßen wird.
6. **Write back** — `onepage_site_id`, `onepage_page_url`, `status = built/published` zurück in `projects`.
7. **Notify** — Slack/E-Mail an den Projektleiter mit Vorschau-Link.

---

## 5. Trigger- & Matching-Logik (Zusammenfassung)

| Quelle | Trigger | Projekt-Matching |
|---|---|---|
| Onepage-Formular | Webhook (CRM) | legt Projekt an / `contact_email` |
| Slack | Slack Trigger (Reaction/Channel) | `slack_channel_id` |
| E-Mail | Gmail Trigger (Label) | `contact_email` |
| Meeting | Fireflies/Granola Webhook | Teilnehmer-E-Mail → `contact_email` |
| Scraper | Sub-workflow / manuell | `existing_url` aus Projekt |
| **Builder** | **Manual / Form Trigger** | **`project_id` (explizit)** |

---

## 6. Benötigte Credentials & Setup

- **Anthropic API** (Claude — Sonnet & Opus) für Extraktion und Bau
- **OnePage MCP** (MCP Client Node) — schreibende Tools + Skills
- **Slack** (Bot Token, Event/Reaction-Scopes)
- **Gmail** (OAuth)
- **Fireflies/Granola** (API Key + Webhook)
- **Jina AI Reader** o. ä. für robustes Scraping (großzügiges Freikontingent)
- **n8n Data Tables** — Tabellen `projects` und `briefing_items` vorab anlegen

---

## 7. Empfohlene Bau-Reihenfolge

1. Data Tables `projects` + `briefing_items` anlegen.
2. Sub-Workflow `extract_and_store` bauen und mit Beispiel-Text testen.
3. Formular-Input-Workflow (höchster Hebel laut Besprechung) verdrahten.
4. Die übrigen drei Inputs (Slack, E-Mail, Meeting) nach demselben Muster ergänzen.
5. Scraper als optionalen Sub-Workflow.
6. Builder-Workflow mit OnePage MCP; zuerst an einem Testprojekt durchspielen.
7. Auf Konsistenz prüfen, dann Status-Flow und Benachrichtigungen schärfen.

---

## 8. Designentscheidungen aus der Besprechung (verankert)

- **Fokus auf „Formular → Website"** statt überkomplexer, individualisierter Kundenkommunikation — bewusst praktikabel gehalten.
- **Claude Opus** für komplexe Seiten mit vielen Unterseiten, **Sonnet** für kleine.
- **„Ping-Pong"** zwischen MCP-Tool und LLM ist das erwartete Bau-Muster.
- Lösung soll als **n8n-Repository** bereitgestellt werden (begleitend: Artikel + Tutorial-Video).
- Budget-/Projektrahmen: 1.500 Einheiten.
