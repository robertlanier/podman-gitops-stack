# Contributing

This document applies to all contributors to this project. Please note that the continuous integration (CI) environment is the source of truth for build, test, and linting results. Following these guidelines helps ensure a smooth contribution process.

## Table of Contents

- [Project Goals](#project-goals)
- [Development Setup](#development-setup)
  - [Prerequisites](#prerequisites)
  - [Tooling Used](#tooling-used)
  - [Virtual Environment](#virtual-environment)
  - [Install Dev Dependencies](#install-dev-dependencies)
- [Project Layout (High Level)](#project-layout-high-level)
- [Commit Style](#commit-style)
- [Running Checks Locally](#running-checks-locally)
- [CI Expectations](#ci-expectations)
- [Secrets Policy](#secrets-policy)
- [Pull Requests](#pull-requests)
- [Releases and Changelog Notes](#releases-and-changelog-notes)

## Project Goals

Thank you for contributing! This project aims to provide a clean, portable, and GitOps-friendly way to manage multi-host Podman-based homelabs using Ansible and Quadlet.

This guide is intended for contributors working on Ansible roles, bootstrap logic, CI tooling, or development utilities within this repository.

Contributions are welcome in the form of documentation improvements, bug fixes, lint cleanup, small features, and refinements to existing roles, playbooks, or orchestration logic.

This project targets Linux hosts where Podman is supported (e.g., RHEL / CentOS Stream / Fedora).

This repository is designed to be:

- **Portable** (works across Linux distros where Podman is supported)
- **GitOps-friendly** (configuration stored in git, converged via Ansible + Quadlet)
- **Safe** (no secrets committed)

## Development Setup

### Prerequisites

- Python **3.11+**
- Git

### Tooling Used

This project uses the following tools during development and release:

- **git-cliff** for changelog generation and release workflows (requires Conventional Commits). It is used in CI to generate release notes and changelogs.
- **ruff** for Python linting
- **markdownlint** for Markdown linting (configured via `.markdownlint.yaml`)
- **yamllint** for YAML formatting checks
- **yamlfmt** for optional local YAML auto-formatting
- **ansible-lint** for Ansible best practices
- **pytest** for Python tests (where applicable)

Note that the CI environment runs these tools inside containerized toolchains, which are authoritative and ensure consistent results across contributors. Local tooling versions may vary slightly.

### Virtual Environment

```bash
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install -U pip
```

### Install Dev Dependencies

```python
python -m pip install -e ".[dev,orchestrator]"
```

This installs development dependencies. The `dev` group includes general development and testing tools. The `orchestrator` group adds optional Python tooling required only if you are working on orchestration features like `stackctl`.

Verify:

```python
stackctl --help
python -c "import yaml; print('PyYAML ok')"
```

## Project Layout (High Level)

- `bootstrap/` - Ansible roles & playbooks for OS/bootstrap setup
- `orchestration/` - Minimal Python utilities for repo management and development
- `config/` - declarative stack configuration (examples only; secrets excluded)
- `docs/` - documentation (architecture, setup, troubleshooting)

## Commit Style

Changelog entries are generated automatically using git-cliff, so commit messages must follow this format.

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

## Running Checks Locally

This project uses **pre-commit hooks** for all code quality checks. This is the industry-standard approach and ensures CI/local parity.

### One-Time Setup

```bash
# Install pre-commit hooks
pre-commit install
```

Now checks run automatically on every `git commit`.

### Manual Check Execution

```bash
# Run all checks on all files
pre-commit run --all-files

# Run specific check
pre-commit run ruff --all-files
pre-commit run ansible-lint --all-files

# Skip hooks for a specific commit (not recommended)
git commit --no-verify
```

### What Gets Checked

The `.pre-commit-config.yaml` file defines all checks:

- **Python**: ruff (linting + formatting)
- **YAML**: yamllint + yamlfmt
- **Ansible**: ansible-lint
- **Markdown**: markdownlint-cli2
- **General**: trailing whitespace, end-of-file, merge conflicts

### Keeping Hooks Updated

Update pre-commit hooks to their latest versions:

```bash
pre-commit autoupdate
```

This updates the hook versions in `.pre-commit-config.yaml`. Review and commit changes.

### CI Behavior

CI runs the same checks using `pre-commit run --all-files` inside a containerized environment. This ensures local and CI results match exactly.

### Excluded Directories

The following are automatically excluded: `collections/`, `vendor/`, `dist/`, `build/`, `node_modules/`, `.venv/`

Optionally, you may use `yamlfmt` to auto-format YAML files locally before running `yamllint`.

If your change affects Ansible roles or playbooks, also run:

```bash
ansible-playbook --syntax-check bootstrap/site.yml
```

Fix any warnings or errors before opening a PR.

## CI Expectations

- CI runs `yamllint`, `ansible-lint`, and `ansible-playbook` syntax checks
- CI builds a reusable GitHub Container Registry (GHCR) image for consistency
- Contributors should ensure local checks pass before opening a PR
- CI does not auto-fix or auto-commit changes
- CI may skip jobs if no relevant files are changed

## Secrets Policy

**Do not commit secrets. Ever.**

Examples of things that must never be committed:

- CIFS/SMB credentials
- Vault passwords
- Private inventories with IPs/usernames/domains
- `.env` files with tokens

Use one of:

- Ansible Vault
- 1Password CLI - see README

## Pull Requests

- Keep PRs small and focused
- Include documentation updates when behavior/config changes
- Prefer adding a test when fixing a bug
- Do not include generated files or directories such as `collections/`, `dist/`, `build/`, or `vendor/` in PRs

## Releases and Changelog Notes

This project uses **git-cliff** to generate `CHANGELOG.md` from commit history.

Contributors do not need to update the changelog manually. However, clear and well-scoped commit messages are essential, as they directly affect release notes.
