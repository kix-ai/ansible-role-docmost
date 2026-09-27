# Docmost aufsetzen & administrieren

Diese Anleitung beschreibt das **Aufsetzen und Administrieren von Docmost** –
einer Open-Source-Wiki-/Dokumentationsplattform (App + PostgreSQL + Redis, per
Docker-Compose) – so generisch, dass ein anderes Team/ein anderer Agent sie
1:1 nachvollziehen kann.

> **Keine echten Werte:** Alle konkreten Angaben sind Platzhalter
> (`<domain>`, `<pw>`, `<host>`, ...). Hosts, IPs, Domains und Passwörter
> gehören **nicht** in Dokumentation oder Repository, sondern in eine
> nicht versionierte `.env`/Vault.

## 1. Überblick & Architektur

Docmost besteht aus drei Containern:

| Dienst | Image (Beispiel) | Zweck | Veröffentlicht |
| --- | --- | --- | --- |
| `docmost` | `docmost/docmost:latest` | Web-App + API | nur `127.0.0.1:<port>` |
| `db` | `postgres:15-alpine` | Datenbank | nein (intern) |
| `redis` | `redis:7-alpine` | Cache/Queue (Echtzeit-Collab) | nein (intern) |

Zugriff erfolgt **ausschließlich über einen Reverse-Proxy** (Nginx/Traefik/Caddy)
mit HTTPS. Der App-Port wird nur an Loopback gebunden, damit er nicht an der
Firewall vorbei öffentlich erreichbar ist.

Datenhaltung:
- PostgreSQL-Volume (`postgres_data`) – Inhalte, Benutzer, Spaces, Rechte.
- App-Volume (`docmost_data`, im Container `/app/data`) – hochgeladene Dateien/Anhänge.
- Redis-Volume ist **cacheflüchtig** und für Backups nicht relevant.

## 2. Voraussetzungen

- Linux-Host (Debian/Ubuntu empfohlen) mit Docker Engine + `docker compose` (v2).
- Ein DNS-Name, der auf den Host zeigt (für TLS empfohlen).
- Offene Ports: `80`/`443` am Reverse-Proxy; der App-Port bleibt intern.
- Optional: `certbot` für Let's-Encrypt-Zertifikate.

Docker prüfen:

```bash
docker --version
docker compose version
```

## 3. Installation (Docker-Compose)

### 3.1 Verzeichnis und Konfiguration

```bash
sudo mkdir -p /opt/docmost
cd /opt/docmost
```

**`/opt/docmost/.env`** (chmod 600, nicht ins Repo):

```dotenv
# Öffentliche Basis-URL (muss exakt der aufgerufenen URL entsprechen)
APP_URL=https://<domain>

# Zufälliges Signatur-Secret (lang, z. B. 64 Hex-Zeichen) – NIE ändern, sonst
# werden alle bestehenden Sessions ungültig.
# erzeugen mit: openssl rand -hex 32
APP_SECRET=<app-secret>

# Datenbank- und Cache-Verbindung (Host = Compose-Servicename)
DATABASE_URL=postgresql://<db-user>:<db-pw>@db:5432/<db-name>?schema=public
REDIS_URL=redis://redis:6379

PORT=3000
FILE_UPLOAD_SIZE_LIMIT=50mb
DISABLE_TELEMETRY=true

# Nur für den db-Container (Compose-Variablen-Substitution):
POSTGRES_DB=<db-name>
POSTGRES_USER=<db-user>
POSTGRES_PASSWORD=<db-pw>

# Optional: E-Mail (Einladungen, Passwort-Reset)
# SMTP_HOST=<smtp-host>
# SMTP_PORT=587
# SMTP_USERNAME=<smtp-user>
# SMTP_PASSWORD=<smtp-pw>
# SMTP_SECURE=true
# MAIL_FROM_ADDRESS=docmost@<domain>
# MAIL_FROM_NAME=Docmost
```

```bash
chmod 600 /opt/docmost/.env
```

### 3.2 `docker-compose.yml`

**`/opt/docmost/docker-compose.yml`**:

```yaml
services:
  docmost:
    image: docmost/docmost:latest
    container_name: docmost
    restart: unless-stopped
    env_file: [./.env]
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    ports:
      # NUR Loopback – öffentlicher Zugriff ausschließlich über Reverse-Proxy!
      - "127.0.0.1:3000:3000"
    volumes:
      - docmost_data:/app/data

  db:
    image: postgres:15-alpine
    container_name: docmost_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?muss in .env gesetzt sein}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: docmost_redis
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis_data:/data

volumes:
  docmost_data:
  postgres_data:
  redis_data:
```

> **Version pinnen:** Für reproduzierbare Deployments `docmost/docmost:latest`
> durch einen festen Tag (z. B. `docmost/docmost:<version>`) ersetzen.

### 3.3 Starten

```bash
cd /opt/docmost
docker compose up -d
docker compose ps
docker compose logs -f docmost
```

Danach ist die App unter der internen Adresse `http://127.0.0.1:3000` erreichbar
(über den Reverse-Proxy dann unter `https://<domain>`). Beim **ersten Aufruf**
führt Docmost durch die Einrichtung des Workspace und des ersten Admin-Kontos.

### 3.4 Alternative: Ansible-Rolle

Statt manueller Schritte kann die idempotente Rolle `docmost` verwendet werden
(Repo `ansible-role-docmost`, siehe `README.md`): Sie rendert `.env` und
`docker-compose.yml`, erzeugt Secrets einmalig auf dem Host und startet den
Stack. Optional richtet sie Nginx + TLS + UFW ein.

## 4. Konfiguration

### 4.1 Wichtige Umgebungsvariablen

| Variable | Pflicht | Beschreibung |
| --- | --- | --- |
| `APP_URL` | ja | Öffentliche Basis-URL (Schema + Host). Muss zur aufgerufenen URL passen. |
| `APP_SECRET` | ja | Signatur-Secret für Sessions/JWT. Lang und zufällig; niemals ändern. |
| `DATABASE_URL` | ja | PostgreSQL-DSN (`postgresql://user:pw@host:5432/db?schema=public`). |
| `REDIS_URL` | ja | Redis-URL (`redis://redis:6379`). |
| `PORT` | nein | App-Port im Container (Default `3000`). |
| `FILE_UPLOAD_SIZE_LIMIT` | nein | Max. Uploadgröße (z. B. `50mb`). |
| `DISABLE_TELEMETRY` | nein | `true` deaktiviert Telemetrie. |
| `SMTP_HOST`/`SMTP_PORT`/`SMTP_USERNAME`/`SMTP_PASSWORD`/`SMTP_SECURE` | nein | Mailversand (Einladungen/Reset). |
| `MAIL_FROM_ADDRESS`/`MAIL_FROM_NAME` | nein | Absender der System-Mails. |

> Je nach Docmost-Version können weitere Variablen existieren – vor dem Rollout
> die offizielle Doku der eingesetzten Version prüfen.

### 4.2 HTTPS / Reverse-Proxy (Nginx)

Nginx-Serverblock (WebSocket-Upgrade ist für die **Echtzeit-Zusammenarbeit**
erforderlich):

```nginx
server {
    listen 80;
    server_name <domain>;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name <domain>;

    ssl_certificate     /etc/letsencrypt/live/<domain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;

    client_max_body_size 50m;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
    }
}
```

Aktivieren und prüfen:

```bash
sudo ln -s /etc/nginx/sites-available/docmost.conf /etc/nginx/sites-enabled/docmost.conf
sudo nginx -t
sudo systemctl reload nginx
```

### 4.3 TLS

**Let's Encrypt (empfohlen, benötigt DNS + erreichbaren Port 80):**

```bash
sudo certbot certonly --standalone --agree-tos --non-interactive \
  --email <admin-mail> -d <domain>
```

Danach im Nginx-Block `ssl_certificate`/`ssl_certificate_key` auf
`/etc/letsencrypt/live/<domain>/...` setzen und Nginx neu laden. Erneuerung
übernimmt der `certbot`-Timer (oder ein Cronjob mit `certbot renew`).

**Selbst-signiert (nur intern/Test):**

```bash
sudo mkdir -p /etc/ssl/docmost
sudo openssl req -x509 -newkey rsa:2048 -sha256 -nodes -days 3650 \
  -keyout /etc/ssl/docmost/privkey.pem \
  -out /etc/ssl/docmost/fullchain.pem \
  -subj "/CN=<domain>" -addext "subjectAltName=DNS:<domain>"
sudo chmod 600 /etc/ssl/docmost/privkey.pem
```

> Wichtig: Nach einem Wechsel von HTTP auf HTTPS muss `APP_URL` auf die
> `https://`-URL zeigen, sonst brechen Login/Redirects.

## 5. Benutzer, Spaces, Rechte & Verwaltung

### 5.1 Rollen

- **Workspace-Rollen:** `owner`, `admin`, `member`.
  - `owner` – volle Kontrolle inkl. Abrechnung/Workspace-Einstellungen.
  - `admin` – Benutzer-, Space- und Einstellungsverwaltung.
  - `member` – normale Nutzung (wie Space-Mitgliedschaften es erlauben).
- **Space-Rollen:** `admin`, `writer`, `reader`.
- **Workspace > Spaces > Pages** (Seiten über `parentPageId` verschachtelbar).

### 5.2 Benutzer einladen

1. **Mit SMTP:** Einstellungen → Members → *Invite*; Docmost versendet die
   Einladungs-Mail.
2. **Ohne SMTP (Invite-Link):** Über die API eine Einladung erzeugen und den
   Einladungslink manuell weitergeben (kein Mailversand nötig):
   - `invites/create` – Einladung anlegen,
   - `invites/link` – Einladungslink abrufen,
   - `invites/accept` – Einladung annehmen.

   Den Link dem Nutzer über einen sicheren Kanal geben (nicht öffentlich posten).

### 5.3 Spaces & Berechtigungen

- Zugriff ist **mitgliedschaftsbasiert**: Wer Mitglied eines Space ist, sieht ihn;
  Nicht-Mitglieder erhalten `404` bzw. "Space permissions not found".
- Ein Space gilt praktisch als *privat*, wenn nur die gewünschten Nutzer als
  Mitglieder eingetragen sind; ein *offener* Space enthält breitere
  Mitgliedschaft.
- **Hinweis (Version 0.96):** Das Feld `spaces.visibility` (`open`/`private`)
  steuert die Zugriffskontrolle **nicht** – maßgeblich ist allein die
  Mitgliedschaft (Tabelle `space_members`). Privatsphäre daher über exklusive
  Mitgliedschaft abbilden, nicht über `visibility`.
- Der Default-Space des Workspace (i. d. R. `general`) erscheint bei allen
  Mitgliedern.

### 5.4 API-Hinweise (für Automatisierung)

- Globaler API-Prefix ist `/api`; **fast alle Routen sind POST**.
- Login: `POST /api/auth/login` (E-Mail + Passwort) setzt ein `httpOnly`-Cookie
  (`authToken`, JWT, signiert mit `APP_SECRET`).
- Antworten sind in `{"data": {...}}` gewrappt; Listen unter `data.items`.
- Space-Mitglieder hinzufügen: `POST /api/spaces/members/add` verlangt **beide**
  Arrays `userIds` **und** `groupIds` (sonst HTTP 400) – `groupIds` ggf. leer
  (`[]`) mitschicken.
- Seiten anlegen/ändern: `/api/pages/create`, `/api/pages/update`
  (`format: markdown|json|html`, `operation: replace|append|prepend`).

## 6. Backup & Restore

### 6.1 Was sichern?

1. **Datenbank** (Inhalte, Benutzer, Rechte) – `pg_dump`.
2. **App-Daten** (Uploads/Anhänge) – Volume `docmost_data` (`/app/data`).
3. Redis **nicht** nötig (nur Cache/Queue).

### 6.2 Datenbank-Backup (PostgreSQL / pg_dump)

Custom-Format (komprimiert, für `pg_restore`):

```bash
docker exec -t docmost_db pg_dump -U <db-user> -d <db-name> -Fc \
  > /backup/docmost-$(date +%F).dump
```

Alternativ als Klartext-SQL:

```bash
docker exec -t docmost_db pg_dump -U <db-user> -d <db-name> \
  > /backup/docmost-$(date +%F).sql
```

### 6.3 App-Daten-Backup (Uploads)

```bash
docker run --rm \
  -v docmost_docmost_data:/data:ro \
  -v /backup:/backup \
  alpine tar czf /backup/docmost-data-$(date +%F).tgz -C /data .
```

(Volume-Namen ggf. mit `docker volume ls` ermitteln, z. B.
`<projekt>_docmost_data`.)

### 6.4 Restore

```bash
# 1. Stack stoppen (App, DB, Redis)
cd /opt/docmost && docker compose down

# 2. Datenbank-Volume leeren bzw. DB neu anlegen und Dump einspielen
docker compose up -d db
# Custom-Dump:
docker exec -i docmost_db pg_restore -U <db-user> -d <db-name> --clean --if-exists \
  < /backup/docmost-<datum>.dump
# ODER Klartext-SQL:
# docker exec -i docmost_db psql -U <db-user> -d <db-name> < /backup/docmost-<datum>.sql

# 3. Uploads zurückspielen
docker run --rm \
  -v docmost_docmost_data:/data \
  -v /backup:/backup \
  alpine tar xzf /backup/docmost-data-<datum>.tgz -C /data

# 4. Stack starten
docker compose up -d
```

**Wichtig:** `APP_SECRET` muss beim Restore identisch bleiben (gleiche `.env`),
sonst werden bestehende Sessions ungültig.

### 6.5 Automatisierung

- Täglichen `pg_dump` + Volume-Tarball per Cron/systemd-Timer erstellen.
- Backups **außerhalb** des Hosts replizieren (z. B. Pull-Backup auf einen
  zweiten Host, `borg`/`restic`).
- Regelmäßig **Restore testen** – ein ungetestetes Backup ist kein Backup.

## 7. Administration & Troubleshooting

### 7.1 Nützliche Befehle

```bash
cd /opt/docmost
docker compose ps                 # Status
docker compose logs -f docmost    # App-Logs
docker compose logs -f db         # DB-Logs
docker compose restart docmost    # App neu starten
docker compose pull && docker compose up -d   # Update
```

### 7.2 Häufige Probleme

| Symptom | Ursache | Lösung |
| --- | --- | --- |
| Web-App von außen direkt erreichbar (Proxy umgangen) | Port ohne Host-Bindung (`3000:3000`) veröffentlicht an alle Interfaces | Auf `127.0.0.1:3000:3000` binden; Zugriff nur über Reverse-Proxy. |
| Login/Redirects brechen nach HTTPS-Umstellung | `APP_URL` noch `http://` | `APP_URL` auf `https://<domain>` setzen und Stack neu starten. |
| Echtzeit-Zusammenarbeit funktioniert nicht | WebSocket-Header fehlen im Proxy | `Upgrade`/`Connection "upgrade"` und `proxy_http_version 1.1` setzen. |
| Einladungen kommen nicht an | Kein SMTP konfiguriert | SMTP-Grundeinstellungen setzen **oder** Invite-Link über die API nutzen. |
| `404 Space permissions not found` (vereinzelt) | transienter Permission-/Cache-Effekt | Kurz warten und identischen Aufruf wiederholen. |
| `POST /spaces/members/add` → 400 "each value in groupIds must be a UUID" | `groupIds` fehlt | `groupIds: []` mitschicken. |
| DB-Container startet nicht | `POSTGRES_PASSWORD`/`.env` fehlt oder inkonsistent | `.env` prüfen (chmod 600), `docker compose config` validieren. |
| Uploads schlagen ab einer Größe fehl | Limit in App **und** Proxy | `FILE_UPLOAD_SIZE_LIMIT` und `client_max_body_size` erhöhen. |

### 7.3 Updates

```bash
# Vorher Backup erstellen!
cd /opt/docmost
docker compose pull
docker compose up -d
docker compose logs -f docmost
```

Migrations führt die App beim Start selbst aus. Bei gepinnten Versionen gezielt
das Image-Tag anheben.

## 8. Best Practices

- **Secrets extern halten:** `.env`/`Secrets` nur auf dem Host (chmod 600) bzw.
  im Vault; niemals in Repos oder Doku. Platzhalter verwenden.
- **Loopback-Bindung:** Den App-Port nie direkt veröffentlichen – immer hinter
  den Reverse-Proxy mit TLS.
- **Versionen pinnen** und Updates kontrolliert (nach Backup) einspielen.
- **Backups automatisieren + Restore testen** (DB **und** Uploads).
- **APP_SECRET sichern** und nie ändern.
- **Least Privilege:** Workspace-Rollen sparsam vergeben; Space-Zugriff über
  Mitgliedschaft statt über globale Rollen steuern.
- **Monitoring:** Healthcheck/Logs überwachen (Container-Status, Fehlerrate).
- **Firewall:** Nur 80/443 öffentlich; DB/Redis/App-Port intern.

## 9. Anhang: Schnellstart (Kurzfassung)

```bash
sudo mkdir -p /opt/docmost && cd /opt/docmost
# .env und docker-compose.yml wie oben anlegen (.env: chmod 600)
docker compose up -d
docker compose ps
# Reverse-Proxy mit TLS konfigurieren, APP_URL auf https://<domain>, Nginx reload
```

Weiterführend: das Repository `ansible-role-docmost` (idempotente Rolle,
`README.md`) für eine automatisierte, secret-freie Installation.
