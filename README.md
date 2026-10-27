# IT-Service-Desk-Automatisierung – Ticket-Workflow mit Wissensdatenbank (n8n)

## Projektziel

Dieser n8n-Workflow automatisiert die Bearbeitung von IT-Support-Anfragen. Mitarbeitende öffnen über ein Formular ein Ticket. Der Workflow sucht in einer Wissensdatenbank nach einer passenden Lösung, lässt die Antwort vom Support freigeben und eskaliert ungelöste Fälle automatisch über First Level und Second Level Support. Jede Bearbeitung wird protokolliert, und neue, wiederverwendbare Lösungen fließen zurück in die Wissensdatenbank.

**Problem:** IT-Anfragen kommen unstrukturiert per E-Mail oder Telefon, ähnliche Probleme werden immer wieder neu gelöst, und es gibt keine einheitliche Dokumentation. Der Workflow sorgt für strukturierte Tickets, schnellere Antworten, klare Eskalationswege und eine lernende Wissensdatenbank.

## Verwendete Tools

- **n8n** – Workflow-Automatisierung (Form Trigger, Edit Fields, If, Gmail Trigger)
- **Notion** – Wissensdatenbank („n8n Use Case“) sowie Protokolle („Support-Protokoll“, „Protokoll Second Level“)
- **Gmail** – Versand von Kundenantworten, Freigabe- und Eskalationsmails (inkl. *Send and Wait* mit Rückmeldeformular) sowie Fallback-Benachrichtigung

> Hinweis: Google Sheets, OpenAI und Telegram sind in der Projektvorlage vorgesehen, werden in diesem Workflow aber nicht verwendet. Eine KI kommt nicht zum Einsatz; die „Auswertung“ erfolgt durch den Abgleich mit der Wissensdatenbank (siehe Abschnitt 3).

## Workflow

### 1. Daten empfangen
- Der **Form Trigger** „Open an IT Service Ticket“ erfasst:
  - Name und E-Mail-Adresse
  - Datum des Vorfalls
  - betroffenes System (*Microsoft Outlook, Crucial system, Hardware, Other*)
  - Dringlichkeit (*High, Medium, Low*)
  - Beschreibung des Problems
  - Geschäftsbereich und Organisation (z. B. Sales, Logistics, Procurement)
- **Edit Fields** bereitet die Eingaben auf und vereinheitlicht die Feldnamen: `System`, `Priority`, `Information`, `Organisation`, `Email`, `Name`.
- Ein zusätzlicher **Gmail Trigger** (prüft jede Minute ein bestimmtes Label) ist als Eingangskanal für E-Mail-Antworten vorbereitet und kann später ausgebaut werden.

### 2. Verarbeitung
- **Get many database pages (Notion):** Sucht in der Wissensdatenbank „n8n Use Case“ nach einem Eintrag, dessen *System* dem gemeldeten System entspricht (Limit: 1 Treffer).
- **Fehlerausgang → Fallback manueller Prozess:** Tritt ein technischer Fehler auf, geht eine Gmail-Nachricht mit allen Ticketdaten und der Fehlermeldung an den Support, damit die Anfrage manuell bearbeitet wird.
- **If:** Prüft, ob ein passender Eintrag mit einer hinterlegten Lösung existiert und das System nicht „Other“ ist.
- **Antwortvorschlag:** Erstellt Betreff und Antworttext für den Kunden (Anrede, Problembeschreibung, Lösung aus der Wissensdatenbank, Hinweis auf Rückantwort).

### 3. KI-Auswertung / Freigabe
- Statt einer KI wird die **Wissensdatenbank** ausgewertet (regelbasierter Abgleich nach System).
- **Send message and wait for response (Gmail):** Der Antwortvorschlag geht zur Freigabe an den Support (Human-in-the-Loop mit *Approve / Disapprove*).
- **Freigegeben?** Je nach Entscheidung geht der Workflow weiter:
  - **Freigegeben** → Antwort an den Kunden
  - **Abgelehnt** → Eskalation als Ticket an den First Level Support

### 4. Speicherung
Alle Vorgänge werden in Notion dokumentiert:
- **Protokoll:** Automatisch beantwortete Anfragen (Status „Automatisch beantwortet“) inkl. Link zum verwendeten Wissenseintrag.
- **Protokoll First Level:** Durch First Level gelöste Tickets inkl. Bearbeitungsschritten und Lösung.
- **Protokoll Second Level:** Eskalierte Tickets inkl. Schritten des First Levels und der finalen Lösung.
- **Neuer Wissenseintrag:** Hat Second Level die Lösung als „Reusable“ markiert, wird sie mit Kurztitel, System, Priorität, Organisation, Beschreibung und Lösung in der Wissensdatenbank angelegt. So wächst die Datenbank mit jedem gelösten Fall.

### 5. Benachrichtigung
- **Kundenantwort (Gmail):** Freigegebene Lösung geht per E-Mail an den Kunden.
- **First Level Support (Gmail, Send and Wait):** Der Support erhält alle Ticketdaten (Kunde, Abteilung, System, Priorität, Datum, Beschreibung, Grund der Eskalation) und meldet per Formular zurück: *Solved (Yes/No)*, *Steps taken*, *Solution*.
- **Gelöst?** Bei *Yes* wird protokolliert, bei *No* wird zum Second Level eskaliert.
- **Second Level Support (Gmail, Send and Wait):** Enthält zusätzlich die bisherigen Schritte; Rückmeldeformular mit *Solution*, *Short title* und *Reusable*.
- **Fallback manueller Prozess:** Benachrichtigung bei technischen Fehlern.

### Ablauf im Überblick
```
Formular → Aufbereitung → Notion-Suche ─(Fehler)→ Fallback-Mail
                              │
                       Antwortvorschlag → Freigabe durch Support
                                              │
                          ┌───────────────────┴──────────────────┐
                    freigegeben                              abgelehnt
                          │                                       │
              Kundenantwort + Protokoll                  First Level Support
                                                                  │
                                                  ┌───────────────┴──────────────┐
                                               gelöst                        nicht gelöst
                                                  │                               │
                                      Protokoll First Level              Second Level Support
                                                                                  │
                                                                      Protokoll Second Level
                                                                                  │
                                                                     wiederverwendbar? → neuer Wissenseintrag
```

## Screenshots

> Bilder direkt in GitHub in diese README ziehen (Drag & Drop) – GitHub erzeugt automatisch den Link. Platzhalter unten ersetzen.

### Gesamter Workflow
<img width="1524" height="744" alt="Bildschirmfoto 2026-10-27 um 11 29 35" src="https://github.com/user-attachments/assets/40cfc887-678b-40bc-ae8f-cdf72043798f" />

### Notion-Abfrage der Wissensdatenbank („Get many database pages“)
<img width="1629" height="942" alt="Bildschirmfoto 2026-10-27 um 11 29 43" src="https://github.com/user-attachments/assets/7aaf060a-e004-4b8f-8d94-7d892137d8c0" />

### Protokollierung in Notion („Protokoll First Level“)
<img width="1625" height="964" alt="Bildschirmfoto 2026-10-27 um 11 29 51" src="https://github.com/user-attachments/assets/62ca6034-f988-46d7-b9d8-e949953f2665" />

### Ticket-Formular
<img width="414" height="907" alt="Bildschirmfoto 2026-10-27 um 11 35 37" src="https://github.com/user-attachments/assets/b5a5e3da-ff76-4b95-aacc-a66a254060d3" />


## Herausforderungen

- **Mehrstufiger Prozess:** Automatische Antwort, First Level und Second Level mussten sauber mit Verzweigungen (If-Nodes) abgebildet werden.
- **Human-in-the-Loop:** Mit *Send and Wait* pausiert der Workflow, bis der Support per E-Mail-Button oder Formular antwortet; die Rückgabewerte (`data.approved`, `data.Solved`, `data.Reusable`) mussten korrekt ausgewertet werden.
- **Zugriff auf Daten früherer Nodes:** Nach mehreren Schritten sind Werte nur noch über Expressions wie `$('Edit Fields').item.json.System` erreichbar.
- **Checkbox-Feld im Formular:** Mehrfachauswahl liefert ein Array. Verwendet wird der erste Wert (`[0]`), die Auswahl ist auf genau eine Option begrenzt.
- **Notion-Anbindung:** Die Property-Typen (Select, Rich Text, E-Mail, Datum, URL) mussten exakt zur jeweiligen Datenbank passen, und die Verbindung muss in Notion den Seiten freigegeben werden.
- **Fehlerbehandlung:** Über den Fehlerausgang des Notion-Nodes und eine Fallback-Mail geht keine Anfrage verloren, wenn die Wissensdatenbank nicht erreichbar ist.
- **Selbstlernende Wissensdatenbank:** Nur als „Reusable“ markierte Lösungen sollen neue Einträge erzeugen.

## Lernerfahrungen

- Aufbau eines vollständigen Support-Prozesses mit **Formular, Datenbank und E-Mail** in n8n.
- Einsatz von **Gmail Send and Wait** für Freigaben und Rückmeldeformulare.
- Anbindung von **Notion** als Wissensdatenbank und Protokollsystem (Lesen, Filtern, Anlegen).
- Verzweigungen mit **If-Nodes**, Expressions und Fehlerausgängen (*Continue on Error*).
- Wie wichtig klare Namen der Nodes und Felder sind, wenn Expressions auf frühere Nodes verweisen.
- Eine Wissensdatenbank, die durch den laufenden Betrieb wächst, spart langfristig Bearbeitungszeit.

## Mögliche Erweiterungen

- Antwortvorschlag per **KI** (z. B. Gemini oder OpenAI) bei fehlendem Wissenseintrag
- Auswertung eingehender E-Mail-Antworten über den vorbereiteten **Gmail Trigger**
- Priorisierung nach Dringlichkeit (z. B. schnellere Eskalation bei „High“)
- Benachrichtigung des Supports über **Telegram**
- Auswertung der Tickets in **Google Sheets** (Dashboard, Bearbeitungszeiten)
- Kundeninformation, wenn First Level das Ticket gelöst hat

## Nutzung

1. Workflow-JSON in n8n importieren (*Workflow → Import from File*).
2. Eigene Zugangsdaten für **Gmail** und **Notion** hinterlegen und die Notion-Verbindung den Datenbanken freigeben.
3. Notion-Datenbanken anlegen bzw. anpassen: *n8n Use Case* (Wissensdatenbank), *Support-Protokoll* und *Protokoll Second Level*.
4. Empfänger-E-Mail-Adressen der Support-Nodes auf das eigene Postfach setzen.
5. Formular-URL öffnen, Ticket absenden und den Ablauf über die E-Mail-Freigaben durchlaufen.
