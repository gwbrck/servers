# servers

Ansible-basierte Infrastruktur zum Provisionieren und Verwalten selbstgehosteter Services.

## Voraussetzungen

- [Ansible](https://docs.ansible.com/) >= 2.15
- [SOPS](https://github.com/getsops/sops) (Secrets-Management)
- Secrets-Datei unter `~/OpenCloud/Dots/server_infra.yaml`

## Setup

```bash
# Ansible-Dependencies installieren
ansible-galaxy install -r requirements.yaml

# Alle Plays ausfuehren
ansible-playbook main.yaml

# Einzelnen Service deployen (via Tags)
ansible-playbook main.yaml --tags vaultwarden

# Mehrere Services
ansible-playbook main.yaml --tags traefik,ntfy

# Alles ausser Backups
ansible-playbook main.yaml --skip-tags restic_backup

# Verfuegbare Tags anzeigen
ansible-playbook main.yaml --list-tags
```

## Hosts

| Host | Funktion |
|------|----------|
| `pivpn` | VPN-Gateway (Tailscale Exit-Node) |
| `homeeins-1` | Haupt-Server (Docker-Services) |

## Services

| Service | Tag | Beschreibung |
|---------|-----|-------------|
| Traefik | `traefik` | Reverse-Proxy, SSL via Let's Encrypt (Hetzner DNS) |
| Vaultwarden | `vaultwarden` | Passwort-Manager |
| Audiobookshelf | `audiobookshelf` | Audiobook-/Podcast-Server |
| Ntfy | `ntfy` | Push-Benachrichtigungen |
| OpenCloud | `opencloud` | Cloud-Speicher |
| Actual Budget | `actual_budget` | Finanz-Tracking |
| FreshRSS | `freshrss` | RSS-Reader |
| Immich | `immich` | Foto-Verwaltung |
| Uptime Kuma | `uptime_kuma` | Monitoring |
| KiwiFS | `kiwifs` | Markdown-Wissensbasis mit Vektorsuche und WebDAV |
| Zotero MCP | `zotero_mcp` | MCP-Zugriff auf die Zotero-Bibliothek mit semantischer Suche |
| Hermes Git | `hermes_git` | Eingeschränkter SSH-Zugriff auf Bare-Git-Repositories |
| Restic Backup | `restic_backup` | Backups auf Hetzner S3 |
| Tailscale | `tailscale` | VPN-Mesh-Netzwerk |
| Security | `security` | SSH-Haertung, Fail2ban, Sudo und automatische Updates |

Die lokale Rolle `security` unterstuetzt Debian und Ubuntu. SSH, Sudo, Fail2ban
und automatische Sicherheitsupdates werden ueber `security_*`-Variablen konfiguriert.
Weitere OS-Familien (z. B. openSUSE) benoetigen insbesondere eine eigene
Update-Konfiguration; die Rolle bricht dort vor Aenderungen ab.

## Struktur

```
main.yaml                    # Haupt-Playbook
inventory.yaml               # Hosts
requirements.yaml            # Ansible-Dependencies
ansible.cfg                  # Ansible-Konfiguration
roles/
  {service}/
    defaults/main.yaml       # Variablen-Defaults
    tasks/main.yaml           # Deployment-Tasks
    templates/                # Jinja2-Templates (Compose, .env)
    handlers/main.yaml        # Event-Handler (optional)
```

## Secrets

Alle Secrets werden via SOPS verwaltet und liegen **nicht** im Repository.
Die verschluesselte Datei wird beim Start von `main.yaml` entschluesselt und als `sops`-Variable an alle Rollen weitergegeben.

Fuer KiwiFS wird folgende Sektion benoetigt. `embedding.provider` kann `mistral` oder `openai` sein; Modell, Basis-URL und Dimensionen werden passend vorbelegt.

```yaml
kiwifs:
  domain: "kiwifs.example.com"
  webdav_domain: "dav.kiwifs.example.com"
  webdav_password: "..."
  embedding:
    provider: "mistral"
    api_key: "..."
    # model: "mistral-embed"
    # base_url: "https://api.mistral.ai"
    # dimensions: 1024
```

Das KiwiFS-WebDAV-Laufwerk ist innerhalb des Tailnets im Finder ueber `Cmd+K` erreichbar:

```text
https://dav.kiwifs.example.com/
```

Als Benutzername kann `kiwifs` verwendet werden; das Passwort ist `kiwifs.webdav_password` aus SOPS.

Fuer Zotero MCP werden Domain und Zotero-Web-API-Zugangsdaten benoetigt. Die Mistral-Einstellungen aus `kiwifs.embedding` werden fuer die OpenAI-kompatible Embedding-Schnittstelle wiederverwendet.

```yaml
zotero_mcp:
  domain: "zotero.example.com"
  api_key: "..."
  library_id: "..."
```

Der Streamable-HTTP-Endpunkt ist unter `https://zotero.example.com/mcp` erreichbar.

## Konventionen

Variablen folgen dem Schema `{rolle}_{zweck}`:

```yaml
{rolle}_compose_dir         # /opt/{rolle}
{rolle}_data_dir            # /srv/service_data/{rolle}
{rolle}_public_url          # Domain
{rolle}_traefik_network     # Traefik-Netzwerk
{rolle}_traefik_enable      # Traefik aktiviert
{rolle}_traefik_router_name # Router-Name
{rolle}_traefik_cert_resolver # Zertifikats-Resolver
```
