# AI Agent Instructions — Podman GitOps Stack

> **Source of Truth**: See [project_todo.md](../project_todo.md) for the complete architectural vision, module catalog, phase roadmap, and identity/permissions model.

## Project Overview

**Podman GitOps Stack** is a multi-phase homelab automation project for multi-host Podman container deployments. The stack uses Ansible for OS provisioning and Quadlet for declarative container management via systemd. The architecture targets three distinct hosts (netapp, appserver, controlplane) with clear separation of concerns.

1. **Bootstrap** — Ansible roles/playbooks for OS provisioning (Fedora IoT, CentOS Stream 9) in `bootstrap/ansible/`
2. **Deployment** — Quadlet + systemd for declarative container lifecycle (Phase 4, future implementation)

## Current Development Phase

**🚧 Phase P0: Repo Baseline** — Making repo public-ready with proper documentation, examples, and lint configs. Focus areas: removing personal data, adding templates, standardizing configurations.

## Critical Architecture Decisions

### Config-Driven Deployment Pattern

- **Design**: `stack.yaml` (per-host config) → Ansible (Jinja2 templates) → Quadlet .container files → systemd units → Podman
- **Goal**: Single YAML file per host declares enabled modules and parameters; Ansible generates Quadlet files; systemd manages lifecycle
- **Status**: Config schema (P1), Ansible bootstrap (P3), and Quadlet deployment (P4) are planned; currently in P0 (repo baseline)

### Module Organization by Host

Containers are organized into **modules** (functional groupings) and deployed via **Quadlet pods** to specific hosts:

**netapp (Network Appliance):**

- `network_foundation` — pihole, nextdns (rootful)
- `home` — homebridge (optional, rootful)
- `timesync` — chrony NTP (host service)

**appserver (Application Platform):**

- `ingress` — traefik (rootless)
- `media` — plex, sonarr, radarr, prowlarr, bazarr, jellyseerr, autoscan, qbitmanage, solvearr, autobrr (rootless)
- `download` — qbittorrent (rootless)
- `documents` — paperless-ngx (optional, rootless)
- `utilities` — dozzle (optional, rootless)

**controlplane (Control Plane):**

- `observability` — grafana, prometheus, loki, promtail, alertmanager (rootless)
- `monitoring` — uptime-kuma, blackbox-exporter, scrutiny-web (rootless)
- `dashboard` — homepage (rootless)
- `notifications` — ntfy (rootless)
- `pki` — step-ca (rootless)
- `backup` — restic (rootless)
- `secrets` — vaultwarden (rootless)
- `chat` — thelounge, znc (rootless)
- `sync` — syncthing (rootless, all hosts)

Full module catalog in [project_todo.md](../project_todo.md) under `Phase 5: Modules by Host`.

### Multi-Host Architecture

The stack targets three distinct hosts with clear separation of concerns:

1. **netapp (Network Appliance)** — Fedora IoT, rootful Podman, foundational services (DNS, NTP, HomeKit)
2. **appserver (Application Platform)** — CentOS Stream 9, rootless Podman, user-facing services (media, ingress, downloads)
3. **controlplane (Control Plane)** — CentOS Stream 9, rootless Podman, monitoring/observability/automation

**Deployment Pattern:**

- Ansible provisions OS, users, directories, NAS mounts (Phase 3)
- Ansible generates Quadlet .container files from stack.yaml templates (Phase 4)
- systemd manages container lifecycle (start, stop, restart, dependencies)
- Quadlet integrates Podman containers natively with systemd (boot-safe, no long-running controller)

### UID/GID Policy (Non-Negotiable)

All Ansible roles enforce a strict UID/GID policy to avoid collisions with NAS exports and container UIDs:

- System users: `< 1000`
- Human/admin users: `1000–9999`
- **Stack service users: `13000+`** (default: `stack` user = UID 13000)
- **Function-based service accounts**:
  - `svc_media` (13100) — runs Plex + *arr apps + automation
  - `svc_download` (13110) — runs qBittorrent
  - `svc_ingress` (13120) — runs Traefik
  - `svc_infra` (13130) — runs Pi-hole, NextDNS, dashboards
  - `svc_home` (13140) — runs Homebridge
- **Shared groups**:
  - `media` (13000) — shared read/write for media stack
  - `infra` (13010) — DNS/dashboard separation
  - `home` (13020) — home automation
- Enforced in `bootstrap/ansible/roles/users/tasks/main.yml` via `ansible.builtin.assert`
- **Never suggest UIDs below 13000 for service accounts**

Complete identity model in [project_todo.yaml](../project_todo.md) under `identity_and_permissions`.

### Secrets Management

- Ansible Vault encrypts secrets in `bootstrap/ansible/group_vars/stack/vault.yml`
- Vault password scripts in `bootstrap/ansible/scripts/vault-pass/` support file/env/1Password integration (currently using 1Password)
- `.gitignore` aggressively blocks `vault.yml`, `hosts.yml`, and credential files
- **Never suggest committing unencrypted secrets**

### Project Conventions

- **Timezone**: `America/Chicago` (IANA) — canonical for hosts, containers, logs, scheduled jobs
- **Domain structure**: `solwyn.dev` (apex) with LAN zone `lan.solwyn.dev` (e.g., `sonarr.lan.solwyn.dev`)
- **No personal branding**: No hardcoded personal hostnames/paths; use examples/templates only

## Development Workflows

### Environment Setup (Required First Step)

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev,orchestrator]"
ansible-galaxy collection install -r bootstrap/ansible/collections/requirements.yml
```

**Why both pip AND Galaxy?**

- `pip` installs Python packages (ansible-core, ansible-lint, ruff, pytest)
- `ansible-galaxy` installs Ansible content (modules, plugins, roles from `ansible.posix`, `community.general`)
- This is intentional and documented in the README

### Running Checks Locally

```bash
# Install pre-commit hooks (one-time setup)
pre-commit install

# Run all checks manually
pre-commit run --all-files

# Run specific check
pre-commit run ruff --all-files
pre-commit run ansible-lint --all-files
```

This project uses **industry-standard pre-commit hooks** for all code quality checks:

- ruff (Python linting + formatting)
- yamllint, yamlfmt (YAML validation and formatting)
- markdownlint-cli2 (Markdown validation)
- ansible-lint (Ansible best practices)
- Basic hygiene checks (trailing whitespace, end-of-file, etc.)

Pre-commit hooks run automatically on `git commit` and can be run manually on all files. This ensures CI/local parity.

### CI Architecture

- CI uses a containerized toolchain (`.github/ci/Dockerfile`) published to GHCR
- Workflow: `.github/workflows/ci.yml`
- **CI is the source of truth for lint results** (local tooling versions may vary)
- Commit message check enforces Conventional Commits (non-blocking)

## Project-Specific Conventions

### Commit Message Format

**Strictly enforced via git-cliff** for automated changelog generation:

```text
feat: add NAS role validation
fix: correct subuid range calculation
docs: update bootstrap instructions
chore: bump ansible-lint version
```

Types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`

### Ansible Role Structure

Roles in `bootstrap/ansible/roles/` follow this pattern:

1. **Input validation first** — extensive `ansible.builtin.assert` blocks with descriptive error messages
2. **Idempotent operations** — all tasks must be safely re-runnable
3. **No handler chains** — prefer explicit task ordering

Example: `bootstrap/ansible/roles/users/tasks/main.yml` validates 15+ preconditions before creating the `stack` user.

### Python Module Organization

- **`orchestration/`** — Minimal Python tooling for development/testing workflows (currently stub in `orchestration/stackctl/cli.py`)
- Entry point: `stackctl` → `orchestration.stackctl.cli:main`
- **Note:** Container deployment uses Quadlet + systemd, not Python SDK. Python tools are for repo management only.

**Logging Best Practices**:

- Use standard Python logging module (`import logging`)
- Use standard methods: `logger.info()`, `logger.warning()`, `logger.error()`, `logger.debug()`
- Avoid `print()` statements in production code

### Version Management

- Uses `setuptools-scm` to derive versions from Git tags
- Configuration in `pyproject.toml`:
  - `version_scheme = "guess-next-dev"` → tags generate dev versions (e.g., `0.1.2.devN`)
  - `local_scheme = "no-local-version"` → no `+g<hash>` suffixes
- **Never manually edit version strings** — tag releases instead

## Key Files to Reference

| File | Purpose |
|------|---------|
| `project_todo.md` | Architectural vision, phase roadmap, module catalog, identity model |
| `bootstrap/ansible/site.yml` | Main Ansible playbook entry point |
| `bootstrap/ansible/group_vars/stack/main.yml` | Default configuration (UID/GID policy, NAS settings) |
| `.pre-commit-config.yaml` | Pre-commit hooks configuration (code quality checks) |
| `pyproject.toml` | Package metadata, dependencies, tool configs |
| `CONTRIBUTING.md` | Full development workflow documentation |

## What NOT to Suggest

- ❌ Changing UID/GID defaults below 13000 (breaks NAS ACL assumptions)
- ❌ Committing secrets, vaults, or inventory files
- ❌ Adding Makefiles (project intentionally uses Python entry points)
- ❌ Generic error handling advice (focus on ansible.builtin.assert validation patterns instead)
- ❌ Custom check orchestration (use pre-commit hooks)

## When in Doubt

1. **Ansible changes?** Check if similar validation exists in other roles (e.g., users, base, nas)
2. **Python changes?** Keep orchestration code in `orchestration/` module
3. **Commit message unclear?** Use `feat:` for new functionality, `fix:` for bugs, `docs:` for documentation-only changes
4. **CI failure?** Run `pre-commit run --all-files` locally first — CI uses the same checks
