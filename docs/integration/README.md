# Integrations-Schnittstelle (Version 1, lesend)

Mit dieser Schnittstelle lesen externe Systeme (z. B. n8n, Berichtswerkzeuge, GIS-Skripte) Daten eines Betriebs, ohne Anmeldung
im Browser. Sie kann **nichts ändern**: Es gibt ausschließlich lesende Aufrufe. Die vollständige Beschreibung steht in
[`api/openapi/integration-v1.yaml`](../../api/openapi/integration-v1.yaml).

Basisadresse: `https://<Ihre-Adresse>/api/integration/v1`

## 1. Zugang einrichten

1. **Betreiber:** Der Systemadmin erlaubt in der Betriebsverwaltung unter „Schnittstelle (Integrationen)“ die Integrationen für den
   Betrieb. Neue Betriebe starten ohne Zugang.
2. **Zentrale:** In der App unter *Verwaltung › Betrieb › Integrationen* auf „Token anlegen“. Name, Zugriffsbereiche, Bezirke und
   Gültigkeit (30 bis 365 Tage) wählen. Das Token wird **nur einmal** angezeigt; kopieren und wie ein Passwort aufbewahren.
3. **Zielsystem:** Das Token als `Authorization: Bearer <token>` senden. Prüfen Sie es mit `GET /ping`.

```bash
curl -H "Authorization: Bearer $TOKEN" https://<Ihre-Adresse>/api/integration/v1/ping
```

Das Token steht **nur im Header**. Im Query-String, in Formularen oder als Cookie wird es nicht angenommen.

## 2. Was ein Token darf

| Zugriffsbereich | liefert |
|---|---|
| `strecke` | Erlegungen (`/erlegungen`, `/erlegungen/{id}`) |
| `jagden` | Jagden mit Datum, Status, Treiben und Teilnehmerzahlen (`/jagden`, `/jagden/{id}`) |
| `abschussplan` | Soll-Abschussplan je Bezirk (`/abschussplan?jagdjahr=…`) |
| `auswertung` | Streckenliste mit Soll und Ist (`/auswertung/strecke?jagdjahr=…`) |
| `stammdaten` | Wildarten (`/katalog`), Bezirke, Jagdeinrichtungen |
| Zusatz `positionen` | zusätzlich genaue Koordinaten (`punkt`) |
| Zusatz `personen_namen` | zusätzlich Namen der Erleger (`erleger`) |

Ein Token gilt für **einen Betrieb** und für „alle Bezirke“ oder eine Auswahl. Objekte außerhalb der Reichweite antworten mit 404.

## 3. Beispiel: n8n

1. Neue Zugangsdaten vom Typ **Header Auth**: Name `Authorization`, Wert `Bearer <token>`.
2. Knoten **HTTP Request**: Methode `GET`, URL `https://<Ihre-Adresse>/api/integration/v1/erlegungen`, Query-Parameter
   `jagdjahr=2026` und `limit=500`, Authentifizierung: die Header-Zugangsdaten.
3. **Seitenweise lesen:** Die Antwort hat die Form `{ "daten": [...], "weiter": "<wert>" }`. Solange `weiter` nicht `null` ist, den
   Aufruf mit `nach=<wert>` wiederholen. In n8n: Option *Pagination* des HTTP-Request-Knotens, Modus „Update a Parameter in Each
   Request“, Parameter `nach`, Wert `{{ $response.body.weiter }}`, Abbruch, wenn `{{ $response.body.weiter }}` leer ist.

## 4. Änderungen nachziehen und Löschungen

- **Neu und geändert:** Merken Sie sich den höchsten `geaendertAm` der zuletzt gelesenen Erlegungen und rufen Sie beim nächsten Lauf
  `/erlegungen?geaendertSeit=<dieser Wert>` ab. Überlappung ist gewollt; führen Sie die Datensätze über `id` zusammen (Upsert).
- **Löschungen werden nicht gemeldet.** Lesen Sie deshalb regelmäßig (z. B. nachts) ein Jagdjahr vollständig
  (`jagdjahr=<Jahr>`) und gleichen Sie ab.

## 5. Grenzen und Fehler

- 120 Anfragen je Minute und Token, höchstens 2 gleichzeitig; danach `429` mit `Retry-After` (Sekunden). Die Kopfzeilen
  `RateLimit-Limit`, `RateLimit-Remaining` und `RateLimit-Reset` zeigen den Stand.
- Seitengröße `limit`: 1 bis 1000, Standard 200. Unbekannte Parameter ergeben `400`.
- Fehler haben die Form `{ "code": "...", "meldung": "..." }`:
  `400 ungueltige_eingabe`, `401 ungueltiges_token`, `403 scope_fehlt`, `404 nicht_gefunden`, `429 zu_viele_anfragen`.
- `401` heißt: Das Token fehlt, ist falsch, abgelaufen oder widerrufen, oder der Betrieb ist nicht freigeschaltet. Aus Sicherheitsgründen
  wird das nicht unterschieden.
- Zeiten sind ISO 8601 mit Zeitzone, Gewichte in Kilogramm, Koordinaten als `[Länge, Breite]`.

## 6. Datenschutz und Sicherheit

- **Datensparsam:** Ohne die Zusätze liefert die Schnittstelle weder Koordinaten noch Namen. `erlegerId` ist eine UUID (ein Pseudonym,
  keine Anonymisierung). Vergeben Sie `personen_namen` nur, wenn Sie dafür eine Rechtsgrundlage haben und die Betroffenen informiert sind.
- **Freitext:** Bemerkungen werden nie ausgeliefert. Der Jagdtitel ist vom Betrieb eingegebener Text.
- **Aufbewahrung:** Jeder Aufruf wird 90 Tage protokolliert (Zeit, Pfad ohne Parameterwerte, Status, Anzahl, IP). In der App sehen Sie
  die letzten 50 Aufrufe je Token.
- **Token schützen:** Nicht in Workflow-Exporte, Skripte oder Chats kopieren. Bei Verdacht sofort unter „Integrationen“ **widerrufen**; das wirkt
  beim nächsten Aufruf. Im Notfall kann der Betreiber alle Tokens eines Betriebs widerrufen oder den Zugang sperren.
- **Sprachmodelle:** Hängen Sie ein Sprachmodell (z. B. einen KI-Knoten in n8n) an diese Daten, kann Text in den Daten (etwa ein
  Jagdtitel) Anweisungen enthalten, die das Modell fälschlich befolgt (Prompt-Injection). Geben Sie dem Modell keine Werkzeuge mit
  Schreibrechten und behandeln Sie die Ausgaben als unvertrauenswürdig.
