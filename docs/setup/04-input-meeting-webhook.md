# Setup: Input — Meeting (Fireflies + Granola)

**Voraussetzung:** Workflow bereits in n8n importiert. Diese Anleitung behandelt ausschließlich die Einrichtung der externen Tools (Fireflies, Granola) und der n8n-Credentials — nicht den Workflow-Aufbau selbst.

## Kurzüberblick des Workflows

Der Workflow hat zwei unabhängige Eingänge:

- **Fireflies-Zweig:** Webhook empfängt nur eine `meetingId` → Node "Fetch Fireflies Transcript" holt sich per Fireflies-API das volle Transkript → wird normalisiert und gespeichert.
- **Granola-Zweig:** Webhook erwartet, dass Titel, Teilnehmer und Transkript **direkt im Webhook-Payload** mitgeschickt werden (kein Nachladen über eine API) → wird normalisiert und gespeichert.

Wichtig vorab: **Fireflies kann das automatisch von sich aus**, **Granola nicht ohne Umweg**. Details dazu unten.

---

## Schritt 1 – Fireflies einrichten

### 1.1 Webhook-URL aus n8n kopieren
Im n8n-Workflow den Node **"Fireflies Webhook"** öffnen und die **Production-Webhook-URL** kopieren (Pfad endet auf `/meeting-fireflies`).

> - Hier bitte Screenshot vom geöffneten "Fireflies Webhook"-Node mit sichtbarer Production-URL einfügen

### 1.2 Webhook in Fireflies eintragen
Fireflies unterstützt native Webhooks, die automatisch ausgelöst werden, sobald ein Meeting fertig transkribiert ist (Event "Transcription completed"). Das mitgeschickte Feld heißt `meetingId` – passt exakt zu dem, was der n8n-Node erwartet.

1. In fireflies.ai einloggen.
2. Zu **Settings → Developer Settings → Webhooks** navigieren.
3. Die in 1.1 kopierte n8n-URL eintragen und speichern.
4. Optional: Webhook-Auth aktivieren, falls vorhanden (zusätzliche Absicherung).

> - Hier bitte Screenshot von fireflies.ai Settings → Developer Settings → Webhooks einfügen

### 1.3 Fireflies API-Key holen
Dieser Key wird gebraucht, weil der n8n-Node "Fetch Fireflies Transcript" nach Eingang der `meetingId` per GraphQL-Request das volle Transkript bei Fireflies abruft.

1. In fireflies.ai: **Settings → Integrations → Developer** (Bereich "Fireflies API").
2. API-Key kopieren und sicher zwischenspeichern (z. B. Passwortmanager).

> - Hier bitte Screenshot von fireflies.ai Settings → Integrations → Developer (API-Key-Bereich) einfügen

### 1.4 Credential in n8n anlegen
1. In n8n: **Credentials → New Credential → "Bearer Auth"** (Generic Auth Type: HTTP Bearer Auth).
2. Als Token den in 1.3 kopierten Fireflies-API-Key einfügen, Credential z. B. "Fireflies API" benennen, speichern.
3. Im Node **"Fetch Fireflies Transcript"** unter Authentication diese Credential auswählen.

> - Hier bitte Screenshot vom n8n-Credential-Dialog "Bearer Auth" einfügen

---

## Schritt 2 – Granola einrichten

**Wichtiger Unterschied zu Fireflies:** Granola bietet **keinen nativen Webhook**, der nach jedem Meeting automatisch alle Daten (Titel, Teilnehmer, Transkript) an eine beliebige URL schickt. Der Workflow-Zweig "Granola Webhook" funktioniert daher nur, wenn die Daten über einen der folgenden Umwege dorthin gelangen:

**Option A – Zapier (empfohlen, offizieller Weg):**
1. Voraussetzung: Granola Business-Plan oder höher.
2. In Granola: **Settings → Integrations → Zapier** verbinden.
3. In Zapier ein neues Zap anlegen: Trigger **"Note Added to Granola Folder"** (löst aus, wenn eine Notiz in einen bestimmten Ordner abgelegt wird – das ist der einzige automatische Trigger).
4. Als Action **"Webhooks by Zapier" → POST** wählen und die in n8n kopierte Granola-Webhook-URL (Node "Granola Webhook", Pfad `/meeting-granola`) eintragen.
5. Im Zapier-Webhook-Body die Felder so benennen/mappen, dass sie zu dem passen, was der n8n-Node "Normalize Granola" erwartet: `title`, `attendees` (Liste), `transcript` bzw. `notes`, `id`.

> - Hier bitte Screenshot vom Zapier-Zap-Editor (Trigger + Webhook-Action + Feld-Mapping) einfügen

**Option B – inoffizielles Community-Tool ("granola-webhook" auf GitHub):**
Ein Drittanbieter-Tool, das lokal auf einem Rechner mit installiertem Granola läuft, die lokale Cache-Datei überwacht und bei neuen Notizen selbst einen Webhook an n8n schickt. Nicht offiziell von Granola unterstützt – vor Einsatz mit der Agentur-IT/Datenschutz abklären, da es lokal auf einem Mitarbeiter-Rechner laufen muss.

Egal welche Option gewählt wird: Nach dem ersten echten Testaufruf unbedingt im n8n-Node **"Normalize Granola"** prüfen, ob die Feldnamen im eingehenden Payload (`body.title`, `body.attendees`, `body.transcript`/`body.notes`, `body.id`) tatsächlich so heißen – sonst die Set-Node-Expressions entsprechend anpassen.

> - Hier bitte Screenshot vom n8n-Node "Normalize Granola" mit den Feld-Zuordnungen einfügen

---

## Schritt 3 – Sub-Workflow prüfen

Beide Zweige übergeben die Daten am Ende an den Sub-Workflow **"Sub: extract_and_store"**. Sicherstellen, dass dieser Sub-Workflow ebenfalls importiert, vorhanden und aktiv mit derselben ID verknüpft ist – sonst schlägt der letzte Schritt in beiden Zweigen fehl.

## Schritt 4 – Testen

1. Workflow in n8n aktivieren.
2. Fireflies-Zweig: Ein Testmeeting durchführen oder über das Fireflies-Dashboard ein Testaudio hochladen, das einen Webhook auslöst.
3. Granola-Zweig: Je nach gewählter Option (Zap manuell auslösen bzw. Community-Tool testen) eine Testnotiz durchlaufen lassen.
4. In n8n unter **Executions** prüfen, ob beide Zweige fehlerfrei durchlaufen und die Daten beim Sub-Workflow ankommen.

---

### Quellen
- [Fireflies.ai – Webhooks](https://docs.fireflies.ai/graphql-api/webhooks)
- [Fireflies.ai – Authorization](https://docs.fireflies.ai/fundamentals/authorization)
- [Granola – Zapier Integration Docs](https://docs.granola.ai/help-center/sharing/integrations/zapier)
- [Granola – Integrations Übersicht](https://www.granola.ai/blog/granola-integrations-complete-guide-connecting-meeting-tools)
- [granola-webhook (Community-Tool, GitHub)](https://github.com/owengretzinger/granola-webhook)
