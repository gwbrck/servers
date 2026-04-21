# AGENTS.md

Ansible-basierte Infrastruktur zum Provisionieren selbstgehosteter Services.

## Voraussetzungen

- Ansible >= 2.15
- SOPS (Secrets-Management)
- Secrets-Datei: `~/OpenCloud/Dots/server_infra.yaml`

## Hosts

| Host | Funktion |
|------|----------|
| `pivpn` | VPN-Gateway (Tailscale Exit-Node) |
| `homeeins-1` | Haupt-Server (alle Docker-Services) |

## Struktur

```
main.yaml                    # Haupt-Playbook (SOPS-Loading + alle Plays)
inventory.yaml               # Hosts
requirements.yaml            # Ansible-Galaxy Dependencies
ansible.cfg                  # Ansible-Konfiguration
group_vars/all.yaml          # Globale Variablen (Basispfade, Traefik-Defaults)
roles/
  {service}/
    defaults/main.yaml       # Variablen-Defaults
    tasks/main.yaml          # Deployment-Tasks
    templates/               # Jinja2-Templates (docker-compose.yaml.j2, ggf. .env.j2)
    handlers/main.yaml       # Event-Handler (optional)
```

## Konventionen

### Basispfade (aus `group_vars/all.yaml`)

```
services_compose_base: "/opt"              # Compose-Files: /opt/{service}
services_data_base: "/srv/service_data"    # Daten: /srv/service_data/{service}
system_bin_dir: "/usr/local/bin"           # Binaries
```

### Variablen-Schema

Alle Variablen folgen dem Schema `{rolle}_{zweck}`:

```yaml
{rolle}_compose_dir: "{{ services_compose_base }}/{rolle}"
{rolle}_compose_file: "{{ default_compose_file }}"
{rolle}_data_dir: "{{ services_data_base }}/{rolle}"
{rolle}_public_url: "{{ sops.{rolle}.domain }}"
{rolle}_traefik_enable: true
{rolle}_traefik_network: "{{ traefik_network }}"
{rolle}_traefik_router_name: "{rolle}"
{rolle}_traefik_cert_resolver: "{{ traefik_cert_resolver }}"
```

### Secrets (SOPS)

- Alle Secrets liegen in `~/OpenCloud/Dots/server_infra.yaml` (SOPS-verschluesselt)
- Secrets werden **nie** ins Repository committet
- Zugriff in Templates/Rollen: `{{ sops.{rolle}.key }}`

### Traefik

- Alle Services nutzen das externe Traefik-Netzwerk (`traefik_network` aus `group_vars/all.yaml`)
- Keine `ports:`-Sektion in Compose-Files (alles laeuft ueber Traefik)
- Netzwerk als `external: true` deklarieren
- Cert-Resolver und Netzwerkname kommen aus SOPS

### Environment-Variablen

Beides ist erlaubt:
- **Inline** in `docker-compose.yaml.j2` unter `environment:` (bei wenigen Variablen)
- **Separate `.env.j2`** (bei vielen Variablen, z.B. immich, mealie, opencloud, traefik, vaultwarden)

### Backup (restic + resticprofile)

- Backup-Profile in `roles/restic_backup/templates/profiles.yaml.j2`
- Jedes Profile `inherit: default` (uebernimmt S3-Repo, Retention, ntfy-Notifications)
- Bei DB-basierten Services: Container stoppen, backup, starten (ausser es gibt einen speziellen DB-Backup-Befehl wie `/vaultwarden backup` oder `pg_dumpall`)
- ntfy-Notifications und Uptime-Kuma Push werden vererbt

## Neuen Service hinzufuegen

### 1. SOPS-Eintrag

In `~/OpenCloud/Dots/server_infra.yaml` eine neue Sektion:

```yaml
{rolle}:
  domain: "{service}.meinedomain.de"
  # Weitere Secrets je nach Bedarf:
  # api_key: "..."
  # db_password: "..."
```

### 2. SOPS-Mapping in main.yaml

In der `Map SOPS secrets`-Task (ca. Zeile 31) eine Zeile hinzufuegen:

```yaml
{rolle}: "{{ sops_raw.{rolle} | default({}, true) }}"
```

### 3. Rolle erstellen

Verzeichnisstruktur:

```
roles/{rolle}/
├── defaults/main.yaml
├── tasks/main.yaml
└── templates/
    └── docker-compose.yaml.j2
    # ggf. .env.j2 bei vielen Env-Vars
```

#### defaults/main.yaml

```yaml
---
{rolle}_compose_dir: "{{ services_compose_base }}/{rolle}"
{rolle}_compose_file: "{{ default_compose_file }}"
{rolle}_data_dir: "{{ services_data_base }}/{rolle}"

{rolle}_public_url: "{{ sops.{rolle}.domain }}"
{rolle}_traefik_enable: true
{rolle}_traefik_network: "{{ traefik_network }}"
{rolle}_traefik_router_name: "{rolle}"
{rolle}_traefik_cert_resolver: "{{ traefik_cert_resolver }}"
```

#### tasks/main.yaml

```yaml
---
- name: Ensure {rolle} directories exist
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: "0750"
  become: true
  loop:
    - "{{ {rolle}_compose_dir }}"
    - "{{ {rolle}_data_dir }}"

- name: Deploy {rolle} Docker Compose file
  ansible.builtin.template:
    src: docker-compose.yaml.j2
    dest: "{{ {rolle}_compose_dir }}/{{ {rolle}_compose_file }}"
    owner: root
    group: root
    mode: "0644"

- name: Pull images
  community.docker.docker_compose_v2_pull:
    project_src: "{{ {rolle}_compose_dir }}"

- name: Create and start services
  community.docker.docker_compose_v2:
    project_src: "{{ {rolle}_compose_dir }}"
  register: {rolle}_output

- name: Show results
  ansible.builtin.debug:
    var: {rolle}_output
    verbosity: 1
```

#### templates/docker-compose.yaml.j2 (mit Traefik)

```yaml
services:
  {rolle}:
    image: {image}:{tag}
    container_name: "{rolle}"
    restart: unless-stopped
    environment:
      - TZ={{ default_timezone }}
      # Weitere Env-Vars
    volumes:
      - "{{ {rolle}_data_dir }}:/data"
    labels:
      - "traefik.enable={{ {rolle}_traefik_enable | default(true) | ternary('true', 'false') }}"
      - "traefik.docker.network={{ {rolle}_traefik_network }}"
      - "traefik.http.routers.{{ {rolle}_traefik_router_name }}.rule=Host(`{{ {rolle}_public_url }}`)"
      - "traefik.http.routers.{{ {rolle}_traefik_router_name }}.entrypoints=websecure"
      - "traefik.http.routers.{{ {rolle}_traefik_router_name }}.tls.certresolver={{ {rolle}_traefik_cert_resolver }}"
      - "traefik.http.services.{{ {rolle}_traefik_router_name }}.loadbalancer.server.port=80"
    networks:
      - "{{ {rolle}_traefik_network }}"

networks:
  {{ {rolle}_traefik_network }}:
    name: {{ {rolle}_traefik_network }}
    external: true
```

### 4. Playbook-Eintrag in main.yaml

Neuen Play vor `restic_backup` einfuegen:

```yaml
- name: Setup {rolle}
  hosts: homeeins-1
  become: true
  tags: [{rolle}]
  roles:
    - {rolle}
```

### 5. Backup-Integration

In `roles/restic_backup/templates/profiles.yaml.j2`:

**Neues Profile** (am Ende der `profiles:`-Sektion):

```yaml
  {rolle}:
    inherit: default
    run-before:
      - "docker compose -f {{ services_compose_base }}/{rolle}/{{ default_compose_file }} stop {container_name}"
    run-after:
      - "docker compose -f {{ services_compose_base }}/{rolle}/{{ default_compose_file }} start {container_name}"
    backup:
      source:
        - "{{ services_data_base }}/{rolle}"
```

Ohne DB (statische Daten) kann `run-before`/`run-after` entfallen.

**Profile in Backup-Gruppen aufnehmen** (`nightly-backup` und `weekly-check`):

```yaml
    profiles:
      # ... bestehende ...
      - {rolle}
```

Optional: Uptime-Kuma Push am letzten Profil der Gruppe:

```yaml
      send-after:
        - method: GET
          url: {{ restic_backup_uptime_kuma_push_url }}
```

### 6. README.md

Service in der Tabelle ergaenzen.

## Deployment

```bash
# Alle Services
ansible-playbook main.yaml

# Einzelner Service
ansible-playbook main.yaml --tags {rolle}

# Mehrere Services
ansible-playbook main.yaml --tags traefik,{rolle}

# Ohne Backups
ansible-playbook main.yaml --skip-tags restic_backup

# Verfuegbare Tags
ansible-playbook main.yaml --list-tags
```

## Ansible-Dependencies

```bash
ansible-galaxy install -r requirements.yaml
```
