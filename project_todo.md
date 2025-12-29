# Podman GitOps Stack - Project Roadmap

## Table of Contents

- [Project Overview](#project-overview)
  - [Key Principles](#key-principles)
  - [Constraints](#constraints)
  - [Hardware Topology](#hardware-topology)
  - [Domain](#domain)
  - [Timezone](#timezone)
  - [Project Directory Structure](#project-directory-structure)
  - [Storage](#storage)
  - [Backup Layout](#backup-layout)
  - [DNS Plan](#dns-plan)
- [Identity and Permissions](#identity-and-permissions)
  - [Groups](#groups)
  - [Service Accounts](#service-accounts)
  - [Container Account Mapping](#container-account-mapping)
- [Required Outputs](#required-outputs)
- [Workflow Management](#workflow-management)
- [Reference Guides](#reference-guides)
- [Architecture](#architecture)
  - [Overview](#overview)
  - [Config-Driven Contract](#config-driven-contract)
  - [Host Module Assignments](#host-module-assignments)
- [Development Phases](#development-phases)
  - [Phase 0: Repo Baseline + Public-Ready Standards](#phase-0-repo-baseline--public-ready-standards)
  - [Phase 1: Single Config File + Module System](#phase-1-single-config-file--module-system)
  - [Phase 2: CI Containerized Toolchain (GHCR)](#phase-2-ci-containerized-toolchain-ghcr)
  - [Phase 3: Ansible Bootstrap (OS/Users/Dirs/Perms)](#phase-3-ansible-bootstrap-osusersdirsperms)
  - [Phase 4: Quadlet + Ansible Deployment](#phase-4-quadlet--ansible-deployment)
  - [Phase 5: Modules by Host](#phase-5-modules-by-host)
  - [Phase 6: Local Developer UX (pre-commit hooks)](#phase-6-local-developer-ux-pre-commit-hooks)
- [Execution Order](#execution-order)

## Project Overview

**Name:** podman-gitops-stack

**Vision:** Public, distro-agnostic, modular GitOps-friendly stack for running Podman containers across multiple Linux hosts. Desired state lives in Git; Ansible + Quadlet converge server state repeatedly without drift.

**Primary Goal:** Multi-host homelab platform using Podman + Quadlet + systemd. Replace Plex on Synology NAS by running it on dedicated hardware, while keeping media stored on the NAS. Separate concerns: network foundation (netapp), applications (appserver), and control plane (controlplane).

### Key Principles

- **Modularity** — everything is a module; users enable/disable modules via a single config file
- **GitOps** — git repo is source-of-truth; reconcile is idempotent; config changes converge on rerun
- **Portability** — distro-agnostic repo structure; OS specifics isolated to Ansible roles
- **Automation First** — low-maintenance; automate install, updates, backups/restore, and health checks
- **No Personal Branding** — no hardcoded personal hostnames/paths; ship examples + templates

### Constraints

- **External Exposure:** LAN-only for now; no public ingress
- **Naming Convention:** `<service>.lan.solwyn.dev` for hostnames/routes
- **Orchestration:**
  - Engine: Podman (rootful on netapp, rootless on appserver/controlplane)
  - Deployment: Quadlet (.container files → systemd units)
  - Management: Ansible (apply configs, enable services)
- **Config-Driven Pattern:** `stack.yaml → Ansible → Quadlet → systemd`
- **Multi-Host Architecture:** 3 hosts with clear separation of concerns

### Hardware Topology

#### netapp — Network Appliance

- **OS:** Fedora IoT (or Fedora Server)
- **Role:** Foundational network services (DNS, HomeKit, NTP)
- **Philosophy:** Appliance, not a server; rarely rebooted; house works even if other servers are down
- **Podman:** Rootful + Quadlet
- **Hardware Example:** Raspberry Pi 3 B+, Intel NUC, Odroid, or similar SBC/mini-PC

#### appserver — Application Platform

- **OS:** CentOS Stream 9
- **Role:** User-facing services (media, downloads, proxy)
- **Philosophy:** Flexible, rebuildable; if down, apps stop but network/house still works
- **Podman:** Rootless + Quadlet
- **Hardware Example:** HP Elitebook 840 G4, Dell XPS, Lenovo ThinkPad, or similar x86 laptop/desktop

#### controlplane — Control Plane

- **OS:** CentOS Stream 9
- **Role:** Trust & automation control; GitOps apply engine
- **Philosophy:** Source of truth; if down, nothing breaks, you just can't apply changes
- **Podman:** Rootless + Quadlet
- **Host Tools:** Ansible, Terraform, Git (not containerized)
- **Hardware Example:** Dell Latitude 3330, HP ProBook, Lenovo ThinkCentre, or similar x86 system

#### Synology DS718+ — Data Plane

- **Role:** Durable storage (media, backups, configs, logs)
- **Access:** NFS/SMB mounts to all hosts

### Domain

- **Apex:** solwyn.dev
- **LAN Zone:** lan.solwyn.dev
- **Reverse Proxy:** Traefik (appserver)
- **Example Routes:**
  - sonarr.lan.solwyn.dev
  - radarr.lan.solwyn.dev
  - prowlarr.lan.solwyn.dev
  - grafana.lan.solwyn.dev

### Timezone

- **Name:** America/Chicago
- **Database:** IANA
- **Note:** Canonical timezone for hosts, containers, logs, and scheduled jobs. All automation should default to this unless explicitly overridden.

### Project Directory Structure

```text
podman-gitops-stack/
├── .github/                    # CI/CD workflows and tooling
│   ├── ci/
│   │   └── Dockerfile          # Containerized CI toolchain
│   ├── copilot-instructions.md
│   └── workflows/
│       └── ci.yml
│
├── bootstrap/                  # Phase 3: OS provisioning
│   └── ansible/
│       ├── ansible.cfg
│       ├── site.yml            # Main playbook entry point
│       ├── collections/        # Galaxy collections (gitignored)
│       ├── group_vars/
│       │   └── stack/
│       │       ├── main.yml
│       │       ├── main.example.yml
│       │       ├── vault.yml   # Encrypted secrets (gitignored)
│       │       └── vault.example.yml
│       ├── inventory/
│       │   ├── hosts.yml       # Host inventory (gitignored)
│       │   └── hosts.example.yml
│       ├── roles/              # OS configuration roles
│       │   ├── base/           # Podman install, directories
│       │   ├── users/          # Service accounts, groups
│       │   └── nas/            # NFS/SMB mounts
│       └── scripts/
│           └── vault-pass/     # Vault password helpers
│
├── config/                     # Phase 1 & 4: Config-driven deployment
│   ├── hosts/                  # Per-host stack configs
│   │   ├── netapp.yml          # Network appliance config
│   │   ├── appserver.yml       # Application platform config
│   │   └── controlplane.yml    # Control plane config
│   ├── modules/                # Module definitions
│   │   ├── ingress/            # Traefik module
│   │   ├── media/              # Plex + *arr module
│   │   ├── download/           # qBittorrent module
│   │   ├── observability/      # Grafana/Prometheus/Loki
│   │   ├── monitoring/         # Uptime Kuma/Scrutiny
│   │   ├── network_foundation/ # Pi-hole/NextDNS
│   │   └── ...                 # Other modules (15 total)
│   └── examples/               # Example configs
│       ├── stack.minimal.yml
│       └── stack.full.yml
│
├── deployment/                 # Phase 4: Quadlet deployment
│   └── ansible/
│       ├── deploy.yml          # Main deployment playbook
│       ├── roles/
│       │   ├── quadlet/        # Generate .container files
│       │   └── systemd/        # Enable/start systemd units
│       └── templates/          # Jinja2 templates for Quadlet
│
├── docs/                       # Documentation
│   ├── architecture/           # Architecture decision records
│   ├── modules/                # Per-module documentation
│   └── guides/                 # Setup/troubleshooting guides
│
├── devtools/                   # Development utilities
│   ├── __init__.py
│   ├── cli.py                  # stackctl entry point (validate, dry-run, list-modules)
│   ├── checks.py               # Config validation, pre-commit helpers
│   ├── logs.py                 # Log tailing wrapper (stackctl logs <service>)
│   ├── models.py               # Pydantic schemas for stack.yaml validation
│   └── runner.py               # Test runner wrapper (molecule, ansible dry-run)
│
├── scripts/                    # Helper scripts
│   ├── backup.sh               # Restic backup wrapper
│   ├── restore.sh              # Restic restore wrapper
│   └── validate-config.sh      # Config validation
│
├── tests/                      # Tests
│   ├── ansible/                # Ansible role tests
│   ├── config/                 # Config validation tests
│   └── integration/            # End-to-end tests
│
├── .pre-commit-config.yaml     # Pre-commit hooks
├── pyproject.toml              # Python package metadata
├── ruff.toml                   # Ruff configuration
├── .yamllint                   # YAML linting
├── .markdownlint.yaml          # Markdown linting
├── cliff.toml                  # Changelog configuration
├── project_todo.md             # Source of truth roadmap
├── CONTRIBUTING.md             # Contributor guide
├── README.md                   # Main documentation
└── CHANGELOG.md                # Generated changelog
```

**Key Directories:**

- **bootstrap/ansible/** — OS provisioning (users, packages, mounts)
- **config/** — Per-host stack.yaml configs + module definitions
- **deployment/ansible/** — Quadlet .container file generation + systemd management
- **devtools/** — Development utilities and stackctl CLI

**Gitignored Files:**

- `bootstrap/ansible/collections/` (installed via ansible-galaxy)
- `bootstrap/ansible/group_vars/stack/vault.yml` (encrypted secrets)
- `bootstrap/ansible/inventory/hosts.yml` (host inventory)
- `.venv/` (Python virtual environment)

### Storage

- **Media Source:** Synology NAS (existing)
- **Migration Note:** Plex currently runs on the NAS; Plex will move to this server, but media remains on the NAS. The stack must mount NAS shares for media and (optionally) downloads.

### Backup Layout

**Strategy:** Backup strategy should prioritize host-mounted config/state directories (not container layers). Keep backups on local disk for fast restores, and optionally replicate to NAS. This layout is designed for rootless Podman (per-service UIDs) and simple shared-group permissions.

**Host Paths:**

- `/srv/podman-gitops-stack` — Base stack directory for configs/state (local storage)
- `/srv/podman-gitops-stack/appdata` — Application configs/state (bind mounts, isolated per service)
- `/srv/podman-gitops-stack/appdata/restic` — Restic config and cache
- `/srv/podman-gitops-stack/backups` — Local backup root
- `/srv/podman-gitops-stack/backups/restic` — Restic local backup repository
- `/mnt/nas/backups` — Optional: mount point to replicate backups to NAS (off-host)

**Backup Scope:**

- ✅ **Include:** appdata_root (all service configs/state), orchestrator config (stack.yaml), ansible inventory + group_vars (non-secret), critical host configs (e.g., traefik dynamic config)
- ❌ **Exclude:** media libraries (source of truth is NAS), download temp/incomplete directories, container images/layers (re-pullable)

**Permissions Model:**

- **Owner:** root (directory creation via Ansible)
- **Group Strategy:**
  - Backup directories owned by group 'infra' (gid 13010) by default
  - Restic runs as svc_infra (uid 13130) and writes to backup_root
  - App configs are owned by each service account; backups read them via group-readable perms

**Recommended Modes:**

| Path | Mode |
|------|------|
| stack_root | 0755 |
| appdata_root | 0750 |
| backup_root | 0770 |
| restic_local_repo | 0770 |

**Nice to Haves:**

- Optionally add a separate svc_backup user later for stricter separation
- Optionally replicate backups to NAS or another host using a second Duplicati destination
- Document restore drill steps as part of P4_T3_backup_restore_update

### DNS Plan

- **Internal DNS Flow:** Pi-hole → NextDNS → fallback (Cloudflare)
- **Note:** DNS is part of infra module; LAN-only routing is the default

## Identity and Permissions

### Groups

> Keep group names generic/public-friendly. Users can override names/ids in stack.yaml.
> Recommended strategy: media GID 13000 for all media/download apps (shared read/write to media + downloads); infra/home have separate groups only if you need stricter separation.

| Group | GID | Purpose |
|-------|-----|---------|
| **media** | 13000 | Shared group for media + download stack. Prefer this group for all *arr/qbit/plex apps so permissions are simple. |
| **infra** | 13010 | Optional: use if you want DNS/dashboard configs separated from media. |
| **home** | 13020 | Optional: use if Homebridge needs different device access patterns. |

### Service Accounts

> Service accounts are function-based and reused across containers. UID values are recommendations; users can override in stack.yaml. Rootless Podman runs as these accounts (system users, no interactive shell).

| Account | UID | Primary Group | Purpose |
|---------|-----|---------------|---------|
| **svc_media** | 13100 | media | Runs Plex + *arr apps + Solvearr + Autoscan + qbitmanage. Owns app configs; group-writable to shared media/download paths. |
| **svc_download** | 13110 | media | Runs qBittorrent. Shares media group so it can write to downloads and media staging paths. |
| **svc_ingress** | 13120 | infra | Runs Traefik. Typically only needs read access to its own config and cert storage (LAN-only). |
| **svc_infra** | 13130 | infra | Runs Pi-hole and NextDNS (and optionally dashboards). |
| **svc_home** | 13140 | home | Runs Homebridge. May need additional device/group access depending on host integrations. |

### Container Account Mapping

> Map each container to a service account + primary shared group. Rootless containers on appserver/controlplane use these UIDs/GIDs via userns mapping. Rootful containers on netapp run as root (no UID mapping).

#### netapp (Network Appliance) - Rootful

| Container | Rootful | Notes |
|-----------|---------|-------|
| pihole | root | Needs port 53 (DNS) |
| nextdns | root | DoH proxy |
| homebridge | root | May need device access |

#### appserver (Application Platform) - Rootless

**ingress-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| traefik | svc_ingress | 13120 | infra | 13010 |

**media-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| plex | svc_media | 13100 | media | 13000 |
| sonarr | svc_media | 13100 | media | 13000 |
| radarr | svc_media | 13100 | media | 13000 |
| prowlarr | svc_media | 13100 | media | 13000 |
| bazarr | svc_media | 13100 | media | 13000 |
| jellyseerr | svc_media | 13100 | media | 13000 |
| autoscan | svc_media | 13100 | media | 13000 |
| qbitmanage | svc_media | 13100 | media | 13000 |
| solvearr | svc_media | 13100 | media | 13000 |
| autobrr | svc_media | 13100 | media | 13000 |

**download-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| qbittorrent | svc_download | 13110 | media | 13000 |

**documents-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| paperless-ngx | svc_media | 13100 | media | 13000 |

**utilities-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| dozzle | svc_infra | 13130 | infra | 13010 |

#### controlplane (Control Plane) - Rootless

**observability-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| loki | svc_infra | 13130 | infra | 13010 |
| promtail | svc_infra | 13130 | infra | 13010 |
| grafana | svc_infra | 13130 | infra | 13010 |
| prometheus | svc_infra | 13130 | infra | 13010 |
| alertmanager | svc_infra | 13130 | infra | 13010 |

**monitoring-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| uptime-kuma | svc_infra | 13130 | infra | 13010 |
| blackbox-exporter | svc_infra | 13130 | infra | 13010 |
| scrutiny-web | svc_infra | 13130 | infra | 13010 |

**dashboard-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| homepage | svc_infra | 13130 | infra | 13010 |

**notifications-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| ntfy | svc_infra | 13130 | infra | 13010 |

**pki-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| step-ca | svc_infra | 13130 | infra | 13010 |

**backup-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| restic | svc_infra | 13130 | infra | 13010 |

**secrets-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| vaultwarden | svc_infra | 13130 | infra | 13010 |

**chat-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| thelounge | svc_infra | 13130 | infra | 13010 |
| znc | svc_infra | 13130 | infra | 13010 |

**sync-pod:**

| Container | User | UID | Group | GID |
|-----------|------|-----|-------|-----|
| syncthing | svc_infra | 13130 | infra | 13010 |

## Required Outputs

- ✅ Ansible roles configure OS/users/dirs/perms cleanly and idempotently
- ✅ Python orchestrator reconciles pods/containers from YAML config repeatedly without drift
- ✅ Modules/bundles allow deploying media-only or adding extras (DNS/dashboard/home)
- ✅ Backup/restore/update tooling exists (script or containerized service)
- ✅ CI is green; pre-commit hooks provide local/CI parity

## Workflow Management

- **Tracking:** TODOs → GitHub Issues → Milestones (phases)
- **Rule:** Each meaningful TODO becomes an Issue; phases become Milestones

## Reference Guides

- [TRaSH Guides - File/Folder Structure](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Docker/)
- [Servarr Docker Guide](https://wiki.servarr.com/docker-guide)
- [ezarr (reference repo)](https://github.com/Luctia/ezarr)

## Architecture

### Overview

| Layer | Responsibility |
|-------|----------------|
| **Ansible** | OS config (packages, firewalld/selinux, sysctls); users, groups, permissions, directories; deploy Quadlet .container files; enable systemd units; configure monitoring agents (node-exporter, cAdvisor, promtail) |
| **Quadlet** | Declarative container definitions (.container files); systemd integration; boot-safe; no long-running controller |
| **Systemd** | Container lifecycle (start, stop, restart); dependency ordering; auto-restart on failure |
| **Runtime** | Podman containers (rootful on Pi, rootless on HP/Dell); Traefik ingress (LAN only); DNS services (Pi-hole/NextDNS) |

### Config-Driven Contract

- **Config File:** `stack.yaml` (per-host config declaring enabled modules)
- **Pattern:** `stack.yaml → Ansible (Jinja2) → Quadlet .container files → systemd enable → podman run`
- **Module Selection:** Each host has its own `stack.yaml` with enabled modules

### Host Module Assignments

**netapp (Network Appliance):**

- `network_foundation` (pihole, nextdns)
- `home` (homebridge)
- `timesync` (chrony)

**appserver (Application Platform):**

- `ingress` (traefik)
- `media` (plex, *arr apps, autobrr)
- `download` (qbittorrent)
- `documents` (paperless-ngx)
- `chat` (optional: thelounge, znc)

**controlplane (Control Plane):**

- `observability` (grafana, prometheus, loki, alertmanager)
- `monitoring` (uptime-kuma, blackbox-exporter, scrutiny)
- `dashboard` (homepage)
- `notifications` (ntfy)
- `pki` (step-ca)
- `backup` (restic)
- `secrets` (vaultwarden)
- `chat` (thelounge, znc)
- `sync` (syncthing)

**Bundle Examples:**

- **minimal:** netapp (network + home), appserver (ingress + media + download)
- **media_plus_monitoring:** Add controlplane (observability + monitoring + dashboard)
- **full_homelab:** All modules enabled across all hosts

## Development Phases

### Phase 0: Repo Baseline + Public-Ready Standards

**Objective:** Make repo shareable, distro-agnostic, lintable, releasable, and contributor-friendly.

**Deliverables:**

- No hardcoded personal hostnames/users/paths
- Examples/templates for config + secrets
- Issues + Milestones workflow

#### Tasks

##### Scrub private data + add templates**

- Replace internal/private hostnames with examples and placeholders
- Add example stack.yaml (module selection + minimal config)
- Move sensitive values to *.example.yml and document expected secrets
- ✅ **Validation:** Repo can be cloned and run with example config without referencing private paths

##### Dynamic versioning via setuptools-scm**

- Ensure pyproject.toml has [tool.setuptools_scm] configured
- Ensure CI checkout uses fetch-depth: 0 so tags are available
- Define expected behavior: tag vX.Y.Z => version X.Y.Z; main => X.Y.(Z+1).devN
- ✅ **Validation:** `python -c "import importlib.metadata as m; print(m.version('podman-gitops-stack'))"`, `python -m setuptools_scm`

##### Changelog via git-cliff**

- Tune cliff.toml ordering/sections and formatting
- Ensure CHANGELOG.md is generated and matches git-cliff output
- Release CI job compares generated changelog to committed CHANGELOG.md
- ✅ **Validation:** `git-cliff --unreleased`, `git-cliff -o CHANGELOG.md`

##### Repo-wide lint configs**

- Maintain .yamllint with standard excludes
- Maintain .markdownlint.yaml with standard excludes and line-length rules
- Maintain ruff config (ruff.toml or [tool.ruff] in pyproject)
- Add .ansible-lint config and exclude collections/
- ✅ **Validation:** `yamllint -c .yamllint .`, `markdownlint -c .markdownlint.yaml .`, `ruff check .`

##### Pre-commit framework**

- Add .pre-commit-config.yaml
- Hooks: ruff check/fix, ruff format, yamllint, markdownlint, ansible-lint (optional)
- Ensure hooks skip collections/ and other generated/vendor directories
- ✅ **Validation:** `pre-commit install`, `pre-commit run --all-files`

---

### Phase 1: Single Config File + Module System

**Objective:** Define a single YAML config that declares enabled modules and their parameters.

#### Tasks

##### Define stack.yaml schema (v1)**

- Schema sections: enabled_modules, domain + LAN naming, users/groups (gid strategy), storage (NAS mounts), traefik routing, module-specific settings (media/download/infra/home)
- ✅ **Validation:** orchestrator can parse config and produce a planned reconcile summary

##### Module contract**

- Each module declares: pods, containers, volumes, env, labels, ports
- Module can be enabled/disabled without breaking others
- Bundle presets live as examples (not hardcoded)
- ✅ **Validation:** enable media-only works; enable infra adds dns without changing media

---

### Phase 2: CI Containerized Toolchain (GHCR)

**Objective:** Fast, consistent CI toolchain using a published GHCR CI image.

**Note:** This is the chosen CI approach (containerized) vs running tools directly on GitHub runners.

#### Tasks

##### CI Dockerfile**

- File: `.github/ci/Dockerfile`
- Requirements: linux/amd64 only, install core tooling for ansible + python checks, keep yamlfmt optional (do not require pip install if it conflicts)

##### CI workflow aligns with local check tooling**

- Use GHCR CI image; fallback to build locally if not published
- Skip heavy jobs when paths/tests not present
- Release-only workflow validates changelog

---

### Phase 3: Ansible Bootstrap (OS/Users/Dirs/Perms)

**Objective:** Use Ansible for all OS configuration and prerequisites.

**Inputs:** `bootstrap/ansible/site.yml`

#### Tasks

##### base role**

- Install Podman + dependencies
- Configure rootless prerequisites (lingering, subuid/subgid if needed)
- Create base directories for stack

##### users/groups role**

- Create shared media group (gid=13000)
- Create function-based service accounts for modules (media/infra/home)
- Set permissions/ownership on directories for rootless Podman

##### NAS mounts**

- Mount NAS exports (media and optional downloads)
- Ensure consistent mount points for orchestrator

---

### Phase 4: Quadlet + Ansible Deployment

**Objective:** Use Quadlet (.container files) + Ansible for declarative container deployment via systemd.

#### Tasks

##### Quadlet template system**

- Features: Jinja2 templates for .container files, per-host stack.yaml config parsing, Ansible role to deploy Quadlet files to `/etc/containers/systemd/`, systemd daemon-reload + enable/start units
- **Why Quadlet:** systemd-native, boot-safe, no long-running controller, RHEL/Fedora first-class citizen
- ✅ **Validation:** `systemctl --user status container-plex` (rootless), `systemctl status container-pihole` (rootful on netapp)

##### Pod/container organization**

- Rule: Logical grouping by function (media, observability, monitoring, etc.)
- Use Quadlet pods where inter-container communication needed
- Ansible generates all .container and .pod files from stack.yaml

##### Backup/restore/update automation**

- Goal: Restic-based backup for appdata, Ansible playbook for restore, systemd timers for scheduled backups, update strategy (watchtower alternative: systemd timer + podman auto-update)
- Deliverables: restic backup command (daily timer), restore playbook, update command (podman auto-update)

---

### Phase 5: Modules by Host

**Objective:** Implement module definitions and map containers to hosts using Quadlet.

#### netapp Modules (Network Appliance)

##### network_foundation

- **Enabled by Default:** ✅ Yes
- **Host:** netapp
- **Podman:** Rootful + Quadlet
- **Containers:**
  - pihole (pihole/pihole)
  - nextdns (nextdns/nextdns)
- **Notes:** First-line DNS blocking + DoH encryption; LAN-wide DNS service

##### home

- **Enabled by Default:** ❌ No
- **Host:** netapp
- **Podman:** Rootful + Quadlet
- **Containers:**
  - homebridge (homebridge/homebridge)
- **Notes:** HomeKit bridge for IoT devices

##### timesync

- **Enabled by Default:** ✅ Yes
- **Host:** netapp (also installed on all hosts)
- **Type:** Host service (systemd)
- **Service:** chrony (NTP)
- **Notes:** Accurate time synchronization; critical for log correlation and scheduled jobs

#### appserver Modules (Application Platform)

##### ingress

- **Enabled by Default:** ✅ Yes
- **Host:** appserver
- **Podman:** Rootless + Quadlet
- **Containers:**
  - traefik (traefik:latest)
- **Notes:** LAN-only reverse proxy; service.<lan_zone> host rules; Prometheus metrics on :8082/metrics

##### media

- **Enabled by Default:** ✅ Yes
- **Host:** appserver
- **Podman:** Rootless + Quadlet
- **Containers:**
  - plex (hotio/plex) — **Anime support enabled** (ASS scanner + HAMA agent)
  - sonarr (hotio/sonarr)
  - radarr (hotio/radarr)
  - prowlarr (hotio/prowlarr)
  - bazarr (hotio/bazarr)
  - jellyseerr (fallenbagel/jellyseerr)
  - autoscan (hotio/autoscan)
  - qbitmanage (hotio/qbitmanage)
  - solvearr (nabilak/solvearr)
  - autobrr (ghcr.io/autobrr/autobrr)
- **Notes:** Complete *arr stack for media automation

##### download

- **Enabled by Default:** ✅ Yes
- **Host:** appserver
- **Podman:** Rootless + Quadlet
- **Containers:**
  - qbittorrent (hotio/qbittorrent)
- **Notes:** Download client with media group permissions

##### documents

- **Enabled by Default:** ❌ No
- **Host:** appserver
- **Podman:** Rootless + Quadlet
- **Containers:**
  - paperless-ngx (ghcr.io/paperless-ngx/paperless-ngx)
- **Notes:** Document management with OCR; upload PDFs/images via web UI or email

##### utilities

- **Enabled by Default:** ❌ No
- **Host:** appserver
- **Podman:** Rootless + Quadlet
- **Containers:**
  - dozzle (amir20/dozzle)
- **Notes:** Real-time container log viewer (quick troubleshooting)

#### controlplane Modules (Control Plane)

##### observability

- **Enabled by Default:** ❌ No
- **Host:** controlplane
- **Podman:** Rootless + Quadlet
- **Containers:**
  - loki (grafana/loki)
  - promtail (grafana/promtail) — **Installed on all hosts**
  - grafana (grafana/grafana)
  - prometheus (prom/prometheus)
  - alertmanager (prom/alertmanager) — **Routes alerts to Discord/ntfy**
- **Notes:**
  - Loki aggregates logs from all hosts (via promtail agents)
  - Prometheus scrapes metrics from all hosts (node-exporter, cAdvisor, Traefik)
  - Grafana provides unified dashboards for logs + metrics
  - Alertmanager sends alerts to Discord webhooks
  - Expose Grafana via Traefik on LAN only

##### monitoring

- **Enabled by Default:** ❌ No
- **Host:** controlplane
- **Podman:** Rootless + Quadlet
- **Containers:**
  - uptime-kuma (louislam/uptime-kuma)
  - blackbox-exporter (prom/blackbox-exporter)
  - scrutiny-web (ghcr.io/analogj/scrutiny)
- **Notes:**
  - Uptime Kuma: Status page and uptime monitoring
  - Blackbox Exporter: Synthetic endpoint monitoring (HTTP/DNS probes)
  - Scrutiny: Drive health monitoring (SMART data); **collectors on all hosts**

##### dashboard

- **Enabled by Default:** ❌ No
- **Host:** controlplane
- **Podman:** Rootless + Quadlet
- **Containers:**
  - homepage (ghcr.io/gethomepage/homepage)
- **Notes:** Single unified dashboard for service discovery and status

##### notifications

- **Enabled by Default:** ❌ No
- **Host:** controlplane
- **Podman:** Rootless + Quadlet
- **Containers:**
  - ntfy (binwiederhier/ntfy)
- **Notes:** Self-hosted push notification service; **single instance** (not on all hosts); services POST to ntfy → push to devices

##### pki

- **Enabled by Default:** ❌ No
- **Host:** controlplane
- **Podman:** Rootless + Quadlet
- **Containers:**
  - step-ca (smallstep/step-ca)
- **Notes:** Private CA for internal TLS certificates; integrates with Traefik for automatic cert provisioning

##### backup

- **Enabled by Default:** ❌ No
- **Host:** controlplane
- **Podman:** Rootless + Quadlet
- **Containers:**
  - restic (restic/restic)
- **Notes:** Encrypted, deduplicated backups; daily systemd timer; backs up appdata to local + NAS repos

##### secrets

- **Enabled by Default:** ❌ No
- **Host:** controlplane (trust boundary)
- **Podman:** Rootless + Quadlet
- **Containers:**
  - vaultwarden (vaultwarden/server)
- **Notes:** Self-hosted Bitwarden; stores service credentials and API keys; backed up via Restic + Syncthing

##### chat

- **Enabled by Default:** ❌ No
- **Host:** controlplane (always-on for message buffering)
- **Podman:** Rootless + Quadlet
- **Containers:**
  - thelounge (thelounge/thelounge)
  - znc (linuxserver/znc)
- **Notes:**
  - ZNC: Always-on IRC bouncer + message buffering
  - The Lounge: Web UI for IRC (connects to ZNC)
  - Expose The Lounge via Traefik on LAN only

##### sync

- **Enabled by Default:** ❌ No
- **Host:** All hosts (controlplane primary, appserver/netapp replicas)
- **Podman:** Rootless + Quadlet
- **Containers:**
  - syncthing (syncthing/syncthing)
- **Notes:** File sync for config replication and backup distribution; syncs Ansible configs (controlplane → netapp/appserver) and Restic backups (controlplane → appserver)

#### Host Services (All Hosts)

**Monitoring Agents (systemd services, not containers):**

- **node-exporter** — Host metrics (CPU, memory, disk, network)
- **cAdvisor** — Container metrics (per-container CPU/memory)
- **promtail** — Log shipper (sends logs to Loki on Dell)
- **scrutiny-collector** — Drive health collector (sends to scrutiny-web on Dell)

**Installation:** Via package manager (dnf) or static binary

#### Routing Examples

#### Service Routes by Host

**netapp (Network Appliance):**

- pihole.lan.solwyn.dev (DNS admin)
- homebridge.lan.solwyn.dev (HomeKit setup)

**appserver (Application Platform):**

- traefik.lan.solwyn.dev (dashboard)
- plex.lan.solwyn.dev
- sonarr.lan.solwyn.dev
- radarr.lan.solwyn.dev
- prowlarr.lan.solwyn.dev
- bazarr.lan.solwyn.dev
- jellyseerr.lan.solwyn.dev
- qbittorrent.lan.solwyn.dev
- paperless.lan.solwyn.dev
- logs.lan.solwyn.dev (dozzle)

**controlplane (Control Plane):**

- grafana.lan.solwyn.dev
- prometheus.lan.solwyn.dev
- uptime.lan.solwyn.dev (uptime-kuma)
- ntfy.lan.solwyn.dev
- homepage.lan.solwyn.dev
- vault.lan.solwyn.dev (vaultwarden)
- ca.lan.solwyn.dev (step-ca)
- irc.lan.solwyn.dev (thelounge)
- drives.lan.solwyn.dev (scrutiny)
- sync.lan.solwyn.dev (syncthing)

---

### Phase 6: Local Developer UX (pre-commit hooks)

**Objective:** Industry-standard pre-commit hooks provide local/CI parity with auto-fixing.

#### Tasks

##### Pre-commit framework integration**

- Uses industry-standard pre-commit hooks for all checks
- Mirrors CI checks (ruff/format/ansible-lint/yamllint/markdownlint)
- Auto-fixes safe issues (formatting, EOF, trailing whitespace)
- CI runs same checks via 'pre-commit run --all-files'
- Documentation in CONTRIBUTING.md guides contributors

---

## Execution Order

1. P0_repo_baseline
2. P1_config_schema
3. P2_ci_option_b_ghcr
4. P3_ansible_bootstrap
5. P4_python_orchestrator
6. P5_modules_and_pods
7. P6_local_dev_ux
