# Onepage Multi-Source Agency Automation

n8n-Workflows, die Kunden-Briefings aus mehreren Kanälen automatisch sammeln und
auf Knopfdruck über **OnePage MCP** in eine fertige Website verwandeln.

Agenturen sammeln Briefings parallel über viele Kanäle: Onboarding-Meeting
(Fireflies/Granola), Nachfass-E-Mail, Slack-Abstimmung, ausgefülltes
Onepage-Formular. Diese Sammlung führt alle Quellen **automatisch zusammen,
speichert sie persistent** und erlaubt einem Projektverantwortlichen, **manuell**
eine fertige Onepage-Website daraus bauen zu lassen.

Die Architektur trennt bewusst **Sammeln** (laufend, automatisch) von **Bauen**
(manuell, kontrolliert): Der teure, qualitätskritische Bau-Schritt bleibt unter
menschlicher Kontrolle, während das Sammeln im Hintergrund passiert.

## Architektur

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

Vier Input-Workflows + ein optionaler Scraper schreiben über den gemeinsamen
Sub-Workflow `extract_and_store` in eine geteilte Datenbank. Der Builder-Workflow
liest sie aus und steuert OnePage MCP.

## Datenmodell (n8n Data Tables)

Zwei Tabellen. `projects` ist der Anker pro Kunde/Projekt, `briefing_items`
sammelt alle Roh- und veredelten Inputs dazu — beliebig viele Inputs pro Projekt
aus beliebigen Quellen.

**`projects`** — `project_id`, `client_name`, `contact_email`,
`slack_channel_id`, `existing_url`, `status`
(`collecting` → `ready_to_build` → `building` → `built` → `published`),
`onepage_site_id`, `onepage_page_url`, `created_at`, `updated_at`.

**`briefing_items`** — `item_id`, `project_id` (FK), `source`
(`form` | `slack` | `email` | `meeting` | `scrape`), `source_ref`, `raw_content`,
`extracted_json`, `received_at`.

Jeder Input wird per LLM in **dasselbe** `extracted_json`-Schema überführt — das
ist der Kern, der die Multi-Source-Zusammenführung möglich macht:

```json
{
  "goals":           ["Was will der Kunde erreichen?"],
  "target_audience": "Zielgruppe",
  "tone_of_voice":   "Markenton (z.B. seriös, verspielt)",
  "key_messages":    ["Kernbotschaften / USPs"],
  "sections_wanted": ["Hero", "Leistungen", "Über uns", "Kontakt"],
  "brand_colors":    ["#hex", "..."],
  "content_blocks":  [{"section": "...", "text": "..."}],
  "cta":             "Gewünschte Handlungsaufforderung",
  "constraints":     ["Muss/Darf-nicht-Vorgaben"],
  "confidence":      "high | medium | low"
}
```

Details: [docs/blueprint.md](docs/blueprint.md) ·
CRM-Formular: [docs/crm-form.md](docs/crm-form.md).

## Workflows

Jeder Workflow trägt dieselbe Nummer in `workflows/` und `docs/setup/` —
Reihenfolge und Zuordnung sind ohne Nachschlagen klar.

| # | n8n-Workflow | Template | Setup-Anleitung |
|---|---|---|---|
| 1 | Input: Onepage Formular | [`01-input-onepage-formular.json`](workflows/01-input-onepage-formular.json) | [01-input-onepage-formular.md](docs/setup/01-input-onepage-formular.md) |
| 2 | Input: Slack | [`02-input-slack.json`](workflows/02-input-slack.json) | [02-input-slack.md](docs/setup/02-input-slack.md) |
| 3 | Input: E-Mail | [`03-input-email.json`](workflows/03-input-email.json) | [03-input-email.md](docs/setup/03-input-email.md) |
| 4 | Input: Meeting (Fireflies + Granola) | [`04-input-meeting-webhook.json`](workflows/04-input-meeting-webhook.json) | [04-input-meeting-webhook.md](docs/setup/04-input-meeting-webhook.md) |
| 5 | Sub: extract_and_store | [`05-sub-extract-and-store.json`](workflows/05-sub-extract-and-store.json) | [05-sub-extract-and-store.md](docs/setup/05-sub-extract-and-store.md) |
| 6 | Sub: Webscraper (Deep, agentisch) | [`06-sub-webscraper.json`](workflows/06-sub-webscraper.json) | [06-sub-webscraper.md](docs/setup/06-sub-webscraper.md) |
| 7 | Builder: Website aus Briefing (OnePage MCP) | [`07-builder-website-aus-briefing.json`](workflows/07-builder-website-aus-briefing.json) | [07-builder-website-aus-briefing.md](docs/setup/07-builder-website-aus-briefing.md) |

Mockdaten zum Testen liegen unter [`schemas/`](schemas/)
(`crm-form-schema.json`, `mockdata_normalize_form.json`, `mockdata_scrape_input.json`).

## Voraussetzungen / Credentials

- **Anthropic API** (Claude — Sonnet & Opus) für Extraktion und Bau
- **OnePage MCP** (MCP Client Node) — schreibende Tools + Skills
- **Slack** (Bot Token, Event-/Reaction-Scopes)
- **Gmail** (OAuth)
- **Fireflies / Granola** (API Key + Webhook)
- **Jina AI Reader** o. ä. für robustes Scraping
- **n8n Data Tables** — Tabellen `projects` und `briefing_items` vorab anlegen

## Installationsreihenfolge

1. Data Tables `projects` + `briefing_items` anlegen.
2. Sub-Workflow `extract_and_store` (#5) bauen und mit Beispiel-Text testen.
3. Formular-Input (#1, höchster Hebel) verdrahten.
4. Übrige Inputs (#2 Slack, #3 E-Mail, #4 Meeting) nach demselben Muster ergänzen.
5. Scraper (#6) als optionalen Sub-Workflow.
6. Builder (#7) mit OnePage MCP; zuerst an einem Testprojekt durchspielen.
7. Auf Konsistenz prüfen, dann Status-Flow und Benachrichtigungen schärfen.

## Lizenz & Kontakt

[MIT](LICENSE) · Constantin Cartellieri — Onepage MCP × Adcon.
