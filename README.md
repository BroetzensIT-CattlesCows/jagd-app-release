# Hunts – Releases

Hunts plant Bewegungsjagden über die ganze Saison: Revierdaten, Einladungen, Zusagen und Strecke an einem Ort. Dieses Repository enthält die **fertigen Releases** und die Betriebsanleitung. Der Quellcode wird in einem privaten Repository entwickelt.

- Landingpage: <https://broetzensit-cattlescows.github.io/jagd-app-release/>
- **Aktuelles Release:** [Releases](../../releases/latest) (Betriebspaket `jagd-app.tar.gz`)
- **Anleitung für Einrichtung, Update und Zurückrollen:** [docs/release-und-update.md](docs/release-und-update.md)
- Schnittstelle für n8n & Co.: [docs/integration/README.md](docs/integration/README.md)
- Fehler melden oder Fragen stellen: [Issues](../../issues)

## Schnellstart

Voraussetzung: Docker mit Compose und ein Reverse-Proxy (Beispiel: Nginx Proxy Manager).

```sh
curl -LO https://github.com/BroetzensIT-CattlesCows/jagd-app-release/releases/latest/download/jagd-app.tar.gz
tar xzf jagd-app.tar.gz && cd jagd-app-*
cp .env.example .env          # Werte setzen, siehe Anleitung
docker network create docker-apps_proxy
docker compose up -d
```

Die Images (`ghcr.io/foe05/jagd-app-api`, `-web`, `-geodaten`) sind öffentlich und tragen dieselbe Versionsnummer wie das Release. Im Betrieb immer eine feste Version verwenden (`JAGD_VERSION` in `.env`), nicht `latest`.

> Dieses Repository wird automatisch befüllt. Änderungen hier gehen beim nächsten Release verloren; Rückmeldungen bitte als Issue.
