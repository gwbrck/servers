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
| Restic Backup | `restic_backup` | Backups auf Hetzner S3 |
| Tailscale | `tailscale` | VPN-Mesh-Netzwerk |

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
