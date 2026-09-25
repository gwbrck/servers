# servers

Ansible-basierte Infrastruktur zum Provisionieren und Verwalten selbstgehosteter Services.

## Voraussetzungen

- [Ansible](https://docs.ansible.com/) >= 2.15
- [SOPS](https://github.com/getsops/sops) (Secrets-Management)
- Secrets-Datei unter `~/OpenCloud/Dots/server_infra.yaml`

## Setup

```bash
# Ansible-Dependencies installieren (Rollen ausserhalb des Repositories)
ansible-galaxy role install -r requirements.yaml -p ~/.ansible/roles
ansible-galaxy collection install -r requirements.yaml

# Alle Plays ausfuehren (Einstiegspunkt bleibt main.yaml)
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

| Host         | Funktion                                   |
|--------------|--------------------------------------------|
| `pivpn`      | VPN-Gateway (Tailscale Exit-Node)          |
| `homeeins-1` | Haupt-Server (Docker-Services)             |
| `hermes`     | Hermes-Server (System Defaults, Tailscale) |

## Erstzugang per SSH

Vor den Rollen probiert der Bootstrap fuer jeden Host den Inventory-Benutzer und
`root` jeweils auf dem konfigurierten SSH-Port (`security_ssh_port`, derzeit 6861)
und auf Port 22. Der erste funktionierende Zugang wird genutzt. Danach legt
`roles/host_setup/tasks/account.yaml` den Inventory-Benutzer an bzw. aktualisiert
ihn, nimmt ihn in die Gruppe `sudo` auf, installiert seinen SSH-Schluessel und
richtet passwortloses sudo ein. Erst nach erfolgreichem Login samt sudo wird SSH
gehaertet und auf den Zielport umgestellt.

In `~/OpenCloud/Dots/server_infra.yaml` werden dazu bestehende Eintraege verwendet:

```yaml
host_setup:
  password_hashed: "$6$..."  # bereits gehashter Passwortwert
  public_ssh_key: "ssh-ed25519 ..."
```

Bei frischen Rechnern muss der Controller zunaechst als `root` (per SSH-Key oder
mit `ansible-playbook main.yaml -k`) einloggen koennen. Bei Schluesselzugang muss
der private Schluessel zu `public_ssh_key` auf dem Controller verfuegbar sein.
Fuer Hermes zeigt der Inventory-Name `hermes` direkt auf den SSH-Hostnamen;
`ansible-playbook main.yaml --limit 'hermes,localhost'` fuehrt SOPS-Laden,
Bootstrap, System Defaults, Tailscale und die Hermes-Umgebungsdatei aus.

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
SSH wird als dauerhaft aktivierter Dienst betrieben; eine vorhandene
SSH-Socket-Aktivierung wird gestoppt und deaktiviert. Der SSH-Port wird ueber
`security_ssh_port` in `sshd_config` festgelegt.
Weitere OS-Familien (z. B. openSUSE) benoetigen insbesondere eine eigene
Update-Konfiguration; die Rolle bricht dort vor Aenderungen ab.

### Fish-Konfiguration

`host_setup` installiert fuer den Administrator den portablen Teil der
Fish-Konfiguration: Prompt, Farben und allgemeine Abbreviations. Die Quelldateien
liegen im Dotfiles-Repository unter `dot_config/private_fish`; Environment- und
Desktop-Einstellungen werden nicht auf die Server uebernommen.

Die Rolle laedt diese Dateien direkt von GitHub. Repository-Basis und Git-Ref
koennen mit `host_setup_fish_raw_base_url` und `host_setup_fish_ref` angepasst
werden. Fuer reproduzierbare Deployments kann statt `main` ein Commit-Hash
gesetzt werden.

## Struktur

```
main.yaml                    # Reihenfolge der importierten Playbooks
playbooks/
  secrets.yaml               # SOPS entschluesseln
  bootstrap.yaml             # SSH-Zugang erkennen
  system.yaml                # System Defaults und Docker
  network.yaml               # Tailscale
  services.yaml              # Anwendungen und Monitoring
  backups.yaml               # Restic
  hosts.yaml                 # Host-spezifische Dienste
inventory.yaml               # Hosts und Funktionsgruppen
requirements.yaml            # Ansible-Dependencies
ansible.cfg                  # Ansible-Konfiguration
group_vars/all.yaml          # Gemeinsame Variablen
tasks/bootstrap_probe.yaml   # SSH-Verbindungsprobe
roles/
  {service}/                 # Auch tailscale und hermes
    defaults/main.yaml       # Variablen-Defaults
    tasks/main.yaml           # Deployment-Tasks
    templates/                # Jinja2-Templates (Compose, .env)
    handlers/main.yaml        # Event-Handler (optional)
```

`inventory.yaml` definiert `managed_servers`, `docker_hosts`,
`application_hosts` und `sync_hosts`. Neue Hosts werden ihren Funktionen
zugeordnet; neue Anwendungen werden in `playbooks/services.yaml` eingetragen.
`--tags <service>` funktioniert weiterhin ueber `main.yaml`. Die Services mit
Restart-Handlern (Immich, KiwiFS, Zotero MCP) behalten eigene Plays, damit die
Handler vor dem naechsten Service ausgefuehrt werden.

Die Backup-Profilliste fuer beide Zeitplaene liegt in
`roles/restic_backup/defaults/main.yaml` (`restic_backup_profile_names`);
die zugehoerigen Backup-Befehle stehen im resticprofile-Template.

## Secrets

Alle Secrets werden via SOPS verwaltet und liegen **nicht** im Repository.
Die verschluesselte Datei wird in `playbooks/secrets.yaml` entschluesselt und als `sops`-Variable an alle Rollen weitergegeben. Neue Sektionen sind ohne zusaetzliches Mapping verfuegbar.

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
