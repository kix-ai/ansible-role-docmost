# ansible-role-docmost

Ansible-Rolle, die **Docmost** (Open-Source-Wiki/Dokumentationsplattform)
idempotent per **Docker-Compose** aufsetzt:

- **App** – `docmost/docmost`
- **Datenbank** – `postgres:15-alpine`
- **Cache/Queue** – `redis:7-alpine`
- **Optional** – Nginx als Reverse-Proxy mit TLS (`none` | `selfsigned` | `letsencrypt`)

Die Rolle ist **generisch und secret-frei**: Es stehen keine Hosts, Domains, IPs
oder Passwoerter im Repository. Zugangsdaten kommen aus der Controller-Umgebung
(Vault/Env) oder werden beim ersten Lauf zufaellig erzeugt und ausschliesslich
auf dem Zielhost abgelegt (`/opt/docmost/.secrets.yml`, chmod 600).

> Ausfuehrliche Aufsetz- und Administrations-Anleitung:
> `docs/DOCMOST.md` in diesem Repo.

## Anforderungen

- Ansible >= 2.14
- Collections: `community.docker`, `community.general`
- Zielhost: Debian/Ubuntu mit erreichbarem APT-Repo (Docker muss vorhanden sein;
  die Rolle `common` des Begleit-Repos installiert Docker CE)

## Verwendung

```yaml
# playbook.yml
- hosts: wiki
  become: true
  roles:
    - role: docmost
      vars:
        docmost_app_url: "https://docs.example.com"
        docmost_enable_nginx: true
        docmost_server_name: "docs.example.com"
        docmost_tls_mode: "letsencrypt"
        docmost_letsencrypt_email: "admin@example.com"
        docmost_manage_ufw: true
```

Zugangsdaten vor dem Lauf setzen (nicht ins Repo):

```bash
export DOCMOST_APP_SECRET="$(openssl rand -hex 32)"
export DOCMOST_DB_PASSWORD="$(openssl rand -base64 24)"
ansible-playbook -i inventory/hosts.yml playbook.yml
```

Beim ersten Lauf werden fehlende Werte zufaellig erzeugt und in
`/opt/docmost/.secrets.yml` gespeichert. Weitere Laeufe sind idempotent.

### Wichtige Variablen (Auszug)

| Variable | Default | Zweck |
| --- | --- | --- |
| `docmost_dir` | `/opt/docmost` | Zielverzeichnis des Compose-Stacks |
| `docmost_app_url` | `https://<domain>` | Oeffentliche Basis-URL (**muss gesetzt werden**) |
| `docmost_http_bind` | `127.0.0.1` | Nur Loopback; Zugriff ueber Reverse-Proxy |
| `docmost_http_port` | `3000` | Interner App-Port |
| `docmost_image` | `docmost/docmost:latest` | App-Image (Version pinnen empfohlen) |
| `docmost_enable_nginx` | `false` | Reverse-Proxy aktivieren |
| `docmost_tls_mode` | `none` | `none` / `selfsigned` / `letsencrypt` |
| `docmost_smtp_enabled` | `false` | E-Mail-Versand (Einladungen/Reset) |
| `docmost_manage_ufw` | `false` | Firewall-Freigaben fuer 80/443 |

Alle Variablen: `defaults/main.yml`.

## Was die Rolle tut

1. Legt `/opt/docmost` an.
2. Erzeugt einmalig `/opt/docmost/.secrets.yml` (chmod 600, Zufallswerte) und
   laedt die Werte wieder ein.
3. Rendert `/opt/docmost/.env` (chmod 600) und `docker-compose.yml`.
4. Faehrt den Stack idempotent hoch (`community.docker.docker_compose_v2`).
5. Optional: Nginx + TLS + UFW.

## Sicherheit

- App-Port wird **nur an Loopback** gebunden (kein direkter Zugriff an der
  Firewall/Proxy vorbei).
- Secrets stehen weder im Compose-File noch im Repo, sondern nur in der
  `.env`/`.secrets.yml` auf dem Host (chmod 600).
- Der Stack laeuft in einem eigenen Compose-Netz; PostgreSQL/Redis sind nicht
  nach aussen veroeffentlicht.

## Lizenz

MIT (siehe `LICENSE`).

## Weiterfuehrend

- `docs/DOCMOST.md` – Installation, Konfiguration, Benutzer/Spaces/Rechte,
  Backup/Restore, Administration, Troubleshooting, Best Practices.
- Team-Wiki (Space *Internal Services*), Seite
  *"Anleitung: Docmost aufsetzen & administrieren"*.
