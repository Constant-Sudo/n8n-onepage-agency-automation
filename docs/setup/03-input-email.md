# Setup-Anleitung: "Input: E-Mail"

Für Kollegen, die diesen Workflow bereits in n8n importiert haben. Geht nur um die Gmail-Anbindung.

## Kurzüberblick

Pollt das Gmail-Postfach jede Minute nach ungelesenen Mails mit dem Label **"briefing"** → extrahiert Absender und Inhalt → speichert über "Sub: extract_and_store".

---

## Schritt 1 – Label "briefing" in Gmail anlegen

1. In Gmail einloggen (das Postfach, das die Agentur für Briefing-Mails nutzt).
2. Links auf **"Weitere" → "Neues Label erstellen"** (Zahnrad-Einstellungen → Labels geht ebenfalls).
3. Label exakt **"briefing"** benennen (Kleinschreibung, da der Node-Filter `label:briefing` lautet).

> - Hier bitte Screenshot von Gmail → Labels → "briefing" erstellen einfügen

4. Optional, aber empfohlen: einen **Filter** anlegen (Einstellungen → Filter und blockierte Adressen → Neuen Filter erstellen), der eingehende Briefing-Mails automatisch mit "briefing" labelt, z. B. nach Absender-Domain oder Betreff-Stichwort.

---

## Schritt 2 – Google-Cloud-Projekt & Gmail-API

1. [console.cloud.google.com](https://console.cloud.google.com) öffnen, neues Projekt anlegen (oder bestehendes Agentur-Projekt nutzen).
2. **APIs & Services → Bibliothek** → nach "Gmail API" suchen → aktivieren.

> - Hier bitte Screenshot von Google Cloud Console → APIs & Services → Gmail API aktiviert einfügen

3. **APIs & Services → OAuth-Zustimmungsbildschirm** einrichten (externe oder interne Nutzer je nach Workspace-Setup). Solange die App im Status "Testing" ist, unter "Audience" die genutzte Gmail-Adresse als Testnutzer hinzufügen.
4. **APIs & Services → Anmeldedaten → "+ Anmeldedaten erstellen" → OAuth-Client-ID**, Anwendungstyp **"Webanwendung"**.

---

## Schritt 3 – Redirect-URI verbinden

1. In n8n: **Credentials → New Credential → "Gmail OAuth2 API"** (oder "Gmail Trigger OAuth2 API", je nach n8n-Version).
2. Die im Credential-Dialog angezeigte **OAuth Redirect URL** kopieren.
3. In der Google Cloud Console beim erstellten OAuth-Client unter **"Autorisierte Weiterleitungs-URIs"** genau diese URL einfügen, speichern.
4. Aus Google die **Client-ID** und das **Client-Secret** kopieren und in die n8n-Credential einfügen.

> - Hier bitte Screenshot vom n8n-Credential-Dialog "Gmail OAuth2 API" mit Redirect-URL einfügen

5. In n8n auf **"Sign in with Google"/"Connect"** klicken, mit dem Agentur-Google-Account einloggen und Zugriff erlauben.
6. Diese Credential im Node **"Gmail Briefing Trigger"** hinterlegen.

---

## Schritt 4 – Workflow aktivieren

Da es sich um einen Polling-Trigger handelt (jede Minute), muss der Workflow in n8n **aktiviert** sein, damit er überhaupt läuft.

---

## Schritt 5 – Testen

1. Eine Test-Mail ins Postfach senden bzw. eine bestehende Mail manuell mit dem Label "briefing" versehen und als ungelesen markieren.
2. Bis zu eine Minute warten, dann in n8n unter **Executions** prüfen, ob die Mail abgeholt und an "Sub: extract_and_store" übergeben wurde.
