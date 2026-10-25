# Release und Update über ghcr.io

Die Jagd-App wird als drei fertige Images in der GitHub Container Registry veröffentlicht: `jagd-app-api`, `jagd-app-web`, `jagd-app-geodaten`. Datenbank (`postgis/postgis`), Mailpit und Alpine kommen unverändert von Docker Hub. Alle drei Images eines Releases tragen dieselbe Versionsnummer.

Der Quellcode liegt in einem **privaten** Repository (`jagd-app`). Öffentlich sind nur:

- die drei **Container-Images** (Pakete auf ghcr.io, einzeln auf „public“ gestellt),
- das **Repository `jagd-app-release`** mit Releases (Betriebspaket), dieser Anleitung, der Landingpage und dem README.

Das API-Image enthält kein TypeScript, sondern nur übersetztes JavaScript (ohne Quelltext und Tests). Lesbar bleibt es trotzdem, denn JavaScript ist keine Verschlüsselung; es ist nur nicht der Originalquelltext.

## Überblick: die Kette

1. Arbeit läuft auf `prototyp` (Feature-Branches werden dorthin gemergt).
2. Pull Request `prototyp` → `main` im privaten Repository, prüfen, mergen. `main` ist der freigegebene Stand.
3. Auf `main` einen Tag `v1.3.0` setzen und pushen. Der Workflow `.github/workflows/release.yml`
   - baut die drei Images und schiebt sie nach `ghcr.io/<konto>/jagd-app-<dienst>` mit den Tags `1.3.0`, `1.3` und `latest`,
   - gleicht Landingpage, Anleitung und README mit dem öffentlichen Repository ab,
   - legt dort das Release „Jagd-App 1.3.0“ mit Änderungsliste und dem Betriebspaket `jagd-app-1.3.0.tar.gz` (plus `jagd-app.tar.gz` als feste Adresse für „neuestes Release“) an.
4. Auf dem Server wird nur die Version in `.env` geändert, dann `pull` und `up`.

Zwischen zwei Releases hält `.github/workflows/oeffentlich.yml` Landingpage und Anleitung im öffentlichen Repository aktuell (bei Änderungen an `site/`, `release-repo/` und den beiden Anleitungen auf `main`).

Die Versionsnummer steht allein im Tag. `GET /api/health` meldet sie (`"version": "1.3.0"`, lokal gebaut `"dev"`).

## Einmalig auf GitHub einrichten

1. **Öffentliches Repository anlegen:** `jagd-app-release` (aktuell unter `BroetzensIT-CattlesCows`, die Adresse steht in `release.yml` und `oeffentlich.yml`), öffentlich, mit Initialcommit (README genügt). *Issues* aktivieren. Der Inhalt wird später automatisch überschrieben.
2. **GitHub Pages im öffentlichen Repository:** Settings → Pages → Source „GitHub Actions“. Die Landingpage erscheint unter `https://<konto>.github.io/jagd-app-release/`.
3. **Zugriffstoken für das Veröffentlichen:** GitHub → Settings → Developer settings → *Fine-grained personal access tokens* → neu:
   - Repository access: nur `jagd-app-release`,
   - Permissions: *Contents* = Read and write, *Workflows* = Read and write, *Metadata* = Read,
   - Ablaufdatum setzen (z. B. 1 Jahr) und im Kalender vermerken.

   Den Wert im **privaten** Repository speichern: Settings → Secrets and variables → Actions → *New repository secret* mit dem Namen `RELEASE_REPO_TOKEN`.
4. **Actions im privaten Repository erlauben:** Settings → Actions → General → *Workflow permissions* auf „Read and write permissions“ (der Workflow braucht `packages: write` für den Push der Images).
5. **Tag-Schutz (empfohlen):** Settings → Rules → Rulesets → neue Tag-Regel für `v*` im privaten Repository: Löschen und Verschieben verbieten. Ein veröffentlichter Tag soll nie ein anderes Image bedeuten.
6. **Pakete öffentlich stellen:** Nach dem ersten Release erscheinen die Pakete unter dem Konto → *Packages*. Sie sind zunächst privat. Für jedes der drei: Paket öffnen → *Package settings* → *Change visibility* → Public. Das ist unabhängig davon, ob das Repository privat ist, und nur einmal nötig. Unter *Manage Actions access* muss das private Repository mit Schreibrecht eingetragen bleiben.
7. **Privates Repository:** Erst nach den Schritten 1 bis 6 auf privat stellen (Settings → General → Danger Zone → Change visibility). Ab dann zeigen die alten Links auf `github.com/<konto>/jagd-app` ins Leere.

## Ein Release erstellen

Voraussetzung: Tests grün, `prototyp` ist über den Pull Request in `main`.

```sh
git checkout main && git pull
git tag -a v1.3.0 -m "Jagd-App 1.3.0"
git push origin v1.3.0
```

- Der Workflow startet automatisch (privates Repository, Reiter *Actions* → „Release“). Dauer: einige Minuten, das Geodaten-Image kompiliert tippecanoe.
- Fertig ist es, wenn im öffentlichen Repository unter *Releases* „Jagd-App 1.3.0“ steht und die drei Pakete den Tag `1.3.0` haben.
- Die Änderungsliste im Release besteht aus den Commit-Titeln, die mit `feat` oder `fix` beginnen. Sprechende Titel zahlen sich hier aus.
- Fehlerhaftes Release: Den Tag **nicht** wiederverwenden. Release und Tag löschen ist nur sinnvoll, wenn noch kein Server es gezogen hat; sonst die Korrektur als `v1.2.1` veröffentlichen.
- Nummernschema: Patch (`1.2.1`) für Fehlerkorrekturen, Minor (`1.3.0`) für neue Funktionen, Major (`2.0.0`) für Änderungen, die Betreiber per Hand begleiten müssen.

## Server vorbereiten (einmalig je Server)

Keine Anmeldung bei GitHub nötig, weil Images und Releases öffentlich sind.

1. Betriebspaket laden und entpacken:
   ```sh
   curl -LO https://github.com/BroetzensIT-CattlesCows/jagd-app-release/releases/latest/download/jagd-app.tar.gz
   tar xzf jagd-app.tar.gz && cd jagd-app-*
   cp .env.example .env
   ```
   Das Paket enthält `compose.yml`, `compose.release.yml`, `.env.example` (mit eingetragener `JAGD_VERSION` und `COMPOSE_FILE`), `db/init/` und diese Anleitung.
2. In `.env` die Pflichtwerte setzen (`POSTGRES_PASSWORD`, `JAGD_OWNER_PASSWORD`, `JAGD_APP_PASSWORD`, `SESSION_SECRET`, `PUBLIC_URL`, Mailserver, `MAIL_FROM`). `DATENQUELLE_SCHLUESSEL` (mindestens 32 Zeichen) wird für die Adressbuch-Anbindung gebraucht. Die Datenbankpasswörter gelten nur beim ersten Anlegen des Datenbank-Volumes.
3. Das externe Netz des Proxys muss existieren: `docker network create docker-apps_proxy` (siehe README → Betrieb hinter dem Nginx Proxy Manager).
4. Starten:
   ```sh
   docker compose up -d
   ```
   Das funktioniert, weil `COMPOSE_FILE=compose.yml:compose.release.yml` in der `.env` steht. Ohne diese Zeile jeweils `docker compose -f compose.yml -f compose.release.yml …` verwenden.
5. Optional für Tests: Demo-Betrieb einspielen mit `docker compose exec api npm run seed:demo`.

## Update einspielen

Die Datenbank wird beim Start der API automatisch migriert. Migrationen laufen nur vorwärts.

1. **Release-Hinweise lesen** (öffentliches Repository → Releases), besonders neue Pflichtvariablen für `.env`. Neue Vorlagen stehen in `.env.example` des neuen Betriebspakets.
2. **Sicherung der Datenbank:**
   ```sh
   docker compose exec -T db pg_dump -U postgres -d jagd | gzip > ~/backups/jagd/db-vor-1.3.0-$(date +%Y%m%d-%H%M).sql.gz
   ```
   Dazu eine Kopie der `.env` und, wenn nötig, des Datenvolumes `jagd_daten` (PDFs, Fotos, Geodaten).
3. **Version eintragen:** in `.env` `JAGD_VERSION=1.3.0`.
4. **Ziehen und starten:**
   ```sh
   docker compose pull
   docker compose up -d
   ```
   Compose ersetzt nur Container, deren Image sich geändert hat. Die kurze Unterbrechung dauert, bis die API gesund ist (etwa eine Minute).
5. **Prüfen:**
   ```sh
   curl -s https://<ihre-domain>/api/health    # {"ok":true,"db":true,"version":"1.3.0"}
   docker compose ps                            # alle Dienste „healthy“ bzw. „Up“
   docker compose logs --tail 50 api            # keine Migrationsfehler
   ```
6. **Aufräumen** (später, optional): `docker image prune`.

### Zurückrollen

- **Ohne neue Migration** (die Release-Hinweise nennen keine): `JAGD_VERSION` auf die alte Version setzen, `docker compose pull`, `docker compose up -d`.
- **Mit Migration:** Die Datenbank ist nicht rückwärts migrierbar. Dann die alte Version eintragen **und** die Sicherung von vor dem Update einspielen. Alles, was seitdem eingegeben wurde, geht verloren, daher Updates in ruhiger Zeit und nur nach Sicherung durchführen.
  ```sh
  docker compose stop api web
  docker compose exec db psql -U postgres -c 'drop database jagd with (force)' -c 'create database jagd owner jagd_owner'
  gunzip -c ~/backups/jagd/db-vor-1.3.0-….sql.gz | docker compose exec -T db psql -U postgres -d jagd
  ```
  (Wiederherstellen vorher an einer Kopie üben.)

## Fragen, die sich beim Update stellen

- **Brauche ich nach dem Update den Demo-Seed?** Nein. Migrationen bringen Struktur und Standardwerte mit, echte Betriebe bleiben unberührt. Der Seed (`npm run seed:demo`) ist nur für Demo- und Entwicklungsumgebungen und setzt dort den Demo-Mandanten neu auf (löscht und erzeugt ihn, schaltet dabei den Adressbuch-Abgleich für ihn ein). Nie gegen einen Betrieb mit echten Daten starten, ohne zu wissen, dass nur der Demo-Mandant betroffen ist.
- **Neue Funktionen sichtbar machen:** Manche Funktionen sind je Betrieb ein Betreiberschalter (z. B. Adressbuch-Abgleich im Systemadmin-Bereich) und nach dem Update zunächst aus.
- **Watchtower:** Läuft auf dem Server bereits Watchtower im Modus `MONITOR_ONLY`, meldet es neue Images per Mail, aktualisiert aber nichts. Das ist hier gewollt: Updates nur bewusst per Anleitung. Wer Watchtower mit Aktualisierung betreibt, sollte `latest` nie verwenden und den Dienst auf Labels beschränken, damit die Datenbank nicht mit hochgezogen wird.
- **`latest` vs. feste Version:** Im Betrieb immer die feste Version (`1.3.0`) in `.env`. `1.2` folgt Fehlerkorrekturen automatisch, `latest` springt auch über Minor-Versionen und ist nur zum Ausprobieren gedacht.
- **Unterschied zum Bauen aus dem Quellcode:** `docker compose up -d --build` (README → Starten) bleibt für Entwicklung und für Server, die direkt aus dem privaten Repository bauen. Beide Wege gleichzeitig auf einem Server zu mischen vermeiden.

## Technische Notizen

- Images sind nur für `linux/amd64` gebaut. Für ARM-Server in `release.yml` `platforms: linux/amd64,linux/arm64` ergänzen (Build dauert dann deutlich länger).
- Das API-Image hat drei Stufen (`api/Dockerfile`): `dev` (Quellcode und Entwicklungswerkzeuge, der Dev-Stack baut mit `target: dev`), `bauen` (übersetzt `api/src` und `shared/src` mit `scripts/js-bauen.mjs` Datei für Datei nach JavaScript, ohne Bündeln) und `release` (Standard: Laufzeit-Abhängigkeiten, `dist/`, Migrationen). Im Image zeigen `shared/package.json` und die npm-Skripte auf das übersetzte JavaScript, im Repository bleiben sie auf TypeScript. `npm run seed:demo`, `systemadmin` usw. funktionieren im Image weiter.
- Die API-Version kommt aus dem Build-Argument `JAGD_VERSION` (Umgebungsvariable im Image).
- `compose.release.yml` ersetzt in `compose.yml` nur die Bau-Angaben der drei eigenen Dienste durch `image:`; alles andere (Volumes, Healthchecks, Netze) bleibt gleich.
- Das Token `RELEASE_REPO_TOKEN` läuft ab. Läuft es ab, scheitert der Job „veröffentlichen“ mit „Secret fehlt“ oder 403. Neues Token erzeugen, Secret ersetzen, Lauf wiederholen (die Images sind dann schon da).
