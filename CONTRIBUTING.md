# Contributing

Good first contributions include documentation improvements, lint fixes, and small enhancements to existing roles or commands.

This project targets Linux hosts where Podman is supported (e.g. RHEL / CentOS Stream / Fedora ).

Thanks for contributing! This repo is designed to be:

- **Portable** (works across Linux distros where Podman is supported)
- **GitOps-friendly** (config in git, reconciliation via `stackctl`; contributors do not need to understand internals to get started)
- **Safe** (no secrets committed)

## Development setup

### Prereqs

- Python **3.11+**
- Git

### Create a virtual environment

```bash
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install -U pip
```

### Install dev dependencies

```python
python -m pip install -e ".[dev,orchestrator]"
```

Verify:

```python
stackctl --help
python -c "import yaml; print('PyYAML ok')"
```

## Project layout (high level)

- `bootstrap/` - Ansible roles & playbooks for OS/bootstrap setup
- `orchestration/` - Python code for reading desired state and reconciling Podman
- `config/` - declariative stack configuration (examples only; secrets excluded)
- `docs/` - documentation (architecture, setup, troubleshooting)

## Commit style

We use Conventional Commits:

- `feat:` new functionality
- `fix:` bug fix
- `docs:` documentation only
- `chore:` tooling / packaging / repo maintenance
- `refactor:` internal refactor (no behavior change)
- `test:` test only

Examples:

- `feat: add stackctl status command`
- `chore: update pyproject dependency groups`
- `docs: clarify NAS mount configuration`

## Running checks

As the project evolves, the following checks may be available:

```bash
ruff check .
pytest
```

## Secrets policy

**Do not commit secrets. Ever.**

Examples of things that must never be committed:

- CIFS/SMB credentials
- vault passwords
- private inventories with IPs/usernames/domains
- .env files with tokens

Use one of:

- Ansible Vault
- 1Password CLI - see README

## Pull request

- Keep PRs small and focused
- Include documentation updates when behavior/config changes
- Prefer adding a test when fixing a bug
