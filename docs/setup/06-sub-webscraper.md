# Setup-Anleitung: "Sub: Webscraper (Deep, agentisch)"

Für Kollegen, die diesen Sub-Workflow bereits in n8n importiert haben. Kein eigener Trigger – wird von "Input: Onepage Formular" (und ggf. manuell) per "Execute Workflow" aufgerufen, wenn eine `existing_url` vorhanden ist.

## Kurzüberblick

Lädt die Sitemap der Kunden-Website, lässt einen Discovery-Agent fehlende Seiten per Probing finden, kategorisiert die relevanten URLs und lässt zwei parallele Claude-Agenten Corporate-Design (Farben/Fonts/Navigation) und Content-Struktur extrahieren. Ergebnis wird wie bei den anderen Input-Workflows über die Data Tables "projects"/"briefing_items" gespeichert.

---

## Schritt 1 – Anthropic-Credential (Claude API)

Vier Nodes nutzen Claude-Modelle: "Claude (Categorize)", "Claude (CI)", "Claude (Content)", "Claude Haiku (Discovery)".

Falls bereits im Rahmen von "Sub: extract_and_store" eine Anthropic-Credential angelegt wurde, **dieselbe wiederverwenden** – sonst wie folgt neu anlegen:

1. [console.anthropic.com](https://console.anthropic.com) → **Settings → API Keys → Create Key** → Key kopieren.
2. In n8n: **Credentials → New Credential → "Anthropic"** → Key einfügen, speichern.
3. Credential in **allen vier** oben genannten Claude-Nodes hinterlegen.

> - Hier bitte Screenshot vom n8n-Credential-Dialog "Anthropic" einfügen

---

## Schritt 2 – Jina AI Credential (für die drei Jina-Tool-Nodes)

Die Nodes **"Jina Markdown Reader"**, **"Jina Content Scraper"** und **"Jina Discovery Reader"** rufen die Jina AI Reader-API auf, um Webseiten als saubere Markdown-Inhalte zu laden. Ohne API-Key funktioniert das nur sehr eingeschränkt (niedriges Rate-Limit), ein Key wird daher empfohlen.

1. Auf [jina.ai](https://jina.ai/) → Bereich **API** öffnen.
2. Dort **API Key & Billing** auswählen – ein kostenloser Key mit 10 Millionen Frei-Tokens kann ohne separate Kontoanlage erzeugt werden.
3. Key kopieren.

> - Hier bitte Screenshot von jina.ai → API → API Key & Billing einfügen

4. In n8n: **Credentials → New Credential → "Jina AI"**.
5. API-Key einfügen, speichern.
6. Credential in allen drei Jina-Nodes ("Jina Markdown Reader", "Jina Content Scraper", "Jina Discovery Reader") hinterlegen.

> - Hier bitte Screenshot vom n8n-Credential-Dialog "Jina AI" einfügen

---

## Schritt 3 – Reine HTTP-Request-Nodes (kein Credential nötig)

Die Nodes **"Download Sitemap XML"**, **"Raw HTML Source Extractor"** und **"Discovery HTTP Probe"** rufen die Kunden-Website direkt per HTTP auf (kein Login, kein Key). Hier nur sicherstellen, dass die n8n-Instanz ausgehende Verbindungen ins offene Internet zulässt (kein Firewall-Block) – sonst schlagen Sitemap-Download und URL-Probing fehl.

---

## Schritt 4 – Data Tables prüfen

Wie in [[setup-sub-extract-and-store]] beschrieben: "projects" und "briefing_items" müssen in der n8n-Instanz existieren und in den Nodes **"Upsert Project"** und **"Insert Briefing Item"** korrekt zugeordnet sein.

---

## Schritt 5 – Testen

1. Workflow manuell mit Testdaten ausführen: `project_id`, `client_name`, `contact_email`, `existing_url` (echte, online erreichbare Domain).
2. Prüfen, ob in "briefing_items" ein Eintrag mit `source = scrape` und befülltem `extracted_json` (Design + Sektionen) erscheint.
