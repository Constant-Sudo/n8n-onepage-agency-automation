# Setup-Anleitung: "Sub: extract_and_store"

Für Kollegen, die diesen Sub-Workflow bereits in n8n importiert haben. Dieser Workflow hat **keinen eigenen Trigger** – er wird von allen Input-Workflows (Meeting, Slack, E-Mail, Formular, Webscraper) per "Execute Workflow" aufgerufen. Es geht hier nur um die externen Zugänge/Credentials, nicht um den Aufbau selbst.

## Kurzüberblick

Nimmt rohen Briefing-Text entgegen → lässt Claude die Kernfelder extrahieren → ordnet den Input per KI-Agent einem bestehenden oder neuen Projekt zu → speichert beides in zwei n8n Data Tables ("projects", "briefing_items").

---

## Schritt 1 – Anthropic-Credential (Claude API)

Der Workflow nutzt zwei KI-Modelle: **Claude Sonnet 4.6** (Node "Claude Sonnet (Extract)") und **Claude Haiku 4.5** (Node "Claude Haiku (Resolve)"). Beide brauchen denselben Anthropic-API-Key.

1. Auf [console.anthropic.com](https://console.anthropic.com) einloggen (ggf. Account für die Agentur anlegen).
2. Links **Settings → API Keys → Create Key**.
3. Key benennen (z. B. "n8n-Agency-Automation") und kopieren – wird nur einmal angezeigt.

> - Hier bitte Screenshot von console.anthropic.com → Settings → API Keys einfügen

4. In n8n: **Credentials → New Credential → "Anthropic"**.
5. API-Key einfügen, speichern (z. B. als "Anthropic API – Agentur").
6. Diese Credential in **beiden** Nodes "Claude Sonnet (Extract)" und "Claude Haiku (Resolve)" auswählen.

> - Hier bitte Screenshot vom n8n-Credential-Dialog "Anthropic" einfügen

**Hinweis:** Dieselbe Anthropic-Credential wird auch in den Workflows "Sub: Webscraper" und "Builder: Website aus Briefing" benötigt – einmal anlegen, überall wiederverwenden.

---

## Schritt 2 – Data Tables prüfen ("projects" & "briefing_items")

Der Workflow schreibt in zwei n8n **Data Tables**. Data Tables sind instanzspezifisch und werden beim Import eines Workflows **nicht automatisch mitkopiert** – nur die Verknüpfung (ID) im Node. Falls der Workflow in eine andere n8n-Instanz/ein anderes Projekt importiert wurde, müssen die Tabellen ggf. neu angelegt und in den Nodes neu zugeordnet werden.

1. In n8n links im Menü auf **Data Tables** gehen und prüfen, ob "projects" und "briefing_items" existieren.
2. Falls nicht vorhanden, neu anlegen mit folgenden Spalten:
   - **projects**: `project_id`, `client_name`, `contact_email`, `slack_channel_id`, `existing_url`, `status`, `onepage_site_id`, `onepage_page_url`, `consolidated_brief`
   - **briefing_items**: `item_id`, `project_id`, `source`, `source_ref`, `raw_content`, `extracted_json`
3. Falls neu angelegt: in den Nodes **"Upsert Project"** und **"Insert Briefing Item"** das Feld "Data Table" öffnen und die neu erstellte Tabelle auswählen (alte ID zeigt sonst ins Leere).

> - Hier bitte Screenshot der n8n Data-Table-Übersicht einfügen

---

## Schritt 3 – Testen

1. Workflow-Liste öffnen → "Sub: extract_and_store" → **Execute Workflow** (manuell, mit Testdaten für `raw_content`, `source` etc.).
2. Prüfen, ob in der Data Table "projects" ein Eintrag erscheint und in "briefing_items" der Rohtext + die extrahierten Felder gespeichert wurden.

Da dieser Workflow nur über andere Workflows aufgerufen wird, lohnt sich ein echter Test meist erst zusammen mit einem der Input-Workflows (z. B. "Input: Slack").
