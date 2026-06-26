# Setup: Builder — Website aus Briefing (OnePage MCP)

**Voraussetzung:** Workflow bereits in n8n importiert. Dieser Workflow ist der aufwändigste der Reihe, da er den Onepage-MCP-Server anspricht — eine Beta-Funktion mit noch eingeschränkter Dokumentation. Diese Anleitung behandelt ausschließlich Credentials, Endpoint-Konfiguration und OAuth2-Setup — nicht den Workflow-Aufbau selbst.

## Kurzüberblick

Formular-Trigger nimmt eine Projekt-ID entgegen → lädt das konsolidierte Briefing aus der Data Table "projects" → ein Claude-Opus-Agent baut darauf basierend per Onepage-MCP-Tools direkt eine Website (Seiten, Sections, Branding) im Onepage-Account.

**Wichtiger Hinweis vorab:** Der Onepage-MCP-Zugang befindet sich aktuell in **BETA** und erfordert eine **aktive Onepage-AI-Mitgliedschaft** sowie eine **Einladung** durch den Onepage-Support. Ohne das geht in diesem Workflow nichts.

---

## Schritt 1 – Anthropic-Credential (Claude Opus)

Der Node **"Claude Opus (Build)"** nutzt das leistungsfähigste Claude-Modell, da er komplexe, mehrstufige Bau-Entscheidungen trifft.

1. Falls bereits für "Sub: extract_and_store" oder "Sub: Webscraper" eine Anthropic-Credential existiert, **diese wiederverwenden**.
2. Andernfalls auf [console.anthropic.com](https://console.anthropic.com) → **Settings → API Keys → Create Key**.

> - Hier bitte Screenshot vom n8n-Credential-Dialog "Anthropic" (zugeordnet zu "Claude Opus (Build)") einfügen

**Hinweis zu Kosten:** Opus-Modelle sind deutlich teurer als Sonnet/Haiku und das Anthropic-Konto braucht ausreichendes Guthaben/Billing-Setup, da ein kompletter Website-Bau viele Tool-Aufrufe mit großem Kontext erzeugt.

---

## Schritt 2 – Onepage-AI-Mitgliedschaft & Beta-Zugang prüfen

1. Sicherstellen, dass der genutzte Onepage-Account eine **aktive Onepage-AI-Mitgliedschaft** hat.
2. Falls der MCP-Beta-Zugang noch nicht aktiviert ist: eine E-Mail an **support@onepage.io** mit der Bitte um Freischaltung des MCP-Beta-Zugangs senden.

> - Hier bitte Screenshot von der Onepage-Account-Übersicht / Mitgliedschaftsstatus einfügen

**Wichtig:** Von der KI erstellte/bearbeitete Inhalte sind aktuell nur unter **beta.onepage.io** sichtbar und editierbar – nicht im normalen **app.onepage.io**. Das ist kein Konfigurationsfehler, sondern der aktuelle Stand der Beta.

---

## Schritt 3 – Endpoint-URL im Node "OnePage MCP" eintragen

Im importierten Workflow steht im Node **"OnePage MCP"** beim Feld **Endpoint-URL** noch ein Platzhaltertext (z. B. `<Onepage MCP Endpoint-URL (z.B. https://mcp.onepage.io/mcp)>`). Dieser **muss vor dem ersten Lauf ersetzt werden**, sonst schlägt der Node fehl.

1. Node **"OnePage MCP"** öffnen.
2. Als Endpoint-URL `https://mcp.onepage.io/` eintragen (offiziell für Claude Desktop dokumentierte Adresse).
3. Falls das nicht funktioniert: alternativ `https://mcp.onepage.io/mcp` testen (Variante aus dem Platzhaltertext des Nodes selbst).

> - Hier bitte Screenshot vom geöffneten "OnePage MCP"-Node mit der Endpoint-URL einfügen

**Hinweis:** Onepage dokumentiert den MCP-Zugang offiziell nur für Claude Desktop, Cursor und Codex – nicht explizit für n8n oder andere generische MCP-Clients. Falls die Verbindung nicht zustande kommt, beim Onepage-Support (support@onepage.io) gezielt nach dem korrekten Endpoint und OAuth2-Setup für Drittanbieter-/Automatisierungstools fragen.

---

## Schritt 4 – OAuth2-Credential für den MCP-Zugang anlegen

1. In n8n: **Credentials → New Credential** → Typ passend zum Node wählen (generische **OAuth2 API**, vom Node als "mcpOAuth2Api" referenziert).
2. Die in n8n angezeigte **Redirect-/Callback-URL** kopieren (falls Onepage dafür ein Feld zur Registrierung verlangt).
3. Authorization-/Token-URL sowie Client-ID/Secret werden von Onepage im Rahmen der Beta-Freischaltung bereitgestellt – falls diese Werte nicht selbst im Onepage-Account auffindbar sind, beim Support nachfragen.
4. Nach Eingabe der Werte auf **"Connect"/"Sign in"** klicken – das führt zum Onepage-Login + Berechtigungs-Bestätigung.

> - Hier bitte Screenshot vom n8n-OAuth2-Credential-Dialog für den "OnePage MCP"-Node einfügen

> - Hier bitte Screenshot vom Onepage-Login-/Consent-Bildschirm nach Klick auf "Connect" einfügen

---

## Schritt 5 – Data Table "projects" prüfen

Der Node, der das Briefing lädt, greift auf die Data Table **"projects"** zu (Spalte `consolidated_brief` u. a.). Wie in [05-sub-extract-and-store.md](05-sub-extract-and-store.md) beschrieben: nach einem Import in eine andere n8n-Instanz muss diese Tabelle ggf. neu angelegt und im entsprechenden Node neu zugeordnet werden.

---

## Schritt 6 – Testen

1. Workflow in n8n aktivieren.
2. Den Formular-Trigger **"Onepage Builder Form"** öffnen, die **Test-URL** aufrufen und im Feld **"Projekt-ID"** eine echte, bereits über die anderen Workflows befüllte Projekt-ID eingeben.
3. In n8n unter **Executions** den Verlauf prüfen – insbesondere, ob der Node "OnePage MCP" erfolgreich verbindet und der Claude-Opus-Agent Tool-Aufrufe (Seiten/Sections erstellen) ausführt.
4. Ergebnis unter **beta.onepage.io** im entsprechenden Projekt kontrollieren.

---

## Quellen

- [Onepage MCP – offizielle Dokumentation (Claude Desktop, Cursor, Codex)](https://docs.onepage.io)
- Onepage Support: support@onepage.io (für Beta-Freischaltung und OAuth2-Details bei Drittanbieter-Clients)
