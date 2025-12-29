# Release Checklist

This checklist ensures consistent, safe releases for the Podman GitOps Stack project.

## Pre-Release Preparation

### 1. Code Quality & Testing

- [ ] Run local checks: `pre-commit run --all-files`
  - [ ] All ruff checks pass (lint + format)
  - [ ] All yamllint checks pass
  - [ ] All ansible-lint checks pass
  - [ ] All markdownlint checks pass
  - [ ] pytest passes (or reports no tests if tests/ is empty)

- [ ] Review CI status on main branch
  - [ ] GitHub Actions CI workflow is green
  - [ ] No pending errors or warnings

### 2. Documentation Review

- [ ] README.md is up to date
  - [ ] Installation instructions are current
  - [ ] Prerequisites match pyproject.toml dependencies
  - [ ] Examples work with current code

- [ ] CONTRIBUTING.md reflects current workflow
  - [ ] Dev setup instructions are accurate
  - [ ] Commit style guidelines are clear
  - [ ] Tooling versions match actual usage

- [ ] `.github/copilot-instructions.md` is current
  - [ ] Architecture decisions are documented
  - [ ] Current phase is accurate
  - [ ] Module catalog matches project_todo.md

- [ ] `project_todo.md` is organized
  - [ ] No duplicate sections
  - [ ] Phase roadmap is clear
  - [ ] Module catalog is complete

### 3. Dependency & Configuration Review

- [ ] `pyproject.toml` dependencies are current
  - [ ] Ansible version range is appropriate
  - [ ] Python version requirement is correct
  - [ ] Dev dependencies are minimal and necessary

- [ ] Configuration files are valid
  - [ ] `.ansible-lint` excludes are correct
  - [ ] `.yamllint` rules are appropriate
  - [ ] `ruff.toml` settings match project style
  - [ ] `.gitignore` protects secrets

- [ ] Ansible collections are up to date
  - [ ] `bootstrap/ansible/collections/requirements.yml` versions are pinned
  - [ ] Collections install without errors

### 4. Secrets & Security

- [ ] No secrets are committed
  - [ ] `vault.yml` is ignored
  - [ ] `hosts.yml` is ignored
  - [ ] `.vault_pass` is ignored
  - [ ] All example files have `.example` suffix

- [ ] `.gitignore` is comprehensive
  - [ ] Python artifacts excluded
  - [ ] Ansible runtime files excluded
  - [ ] Editor files excluded

## Version & Changelog

### 5. Version Tagging

- [ ] Determine version number
  - [ ] Follow semantic versioning (MAJOR.MINOR.PATCH)
  - [ ] Previous version: check `git tag --list`
  - [ ] New version: `v0.1.2` (example)

- [ ] Verify setuptools-scm configuration
  - [ ] `pyproject.toml` has `[tool.setuptools_scm]` section
  - [ ] `version_scheme = "guess-next-dev"` is set
  - [ ] `local_scheme = "no-local-version"` is set

### 6. Changelog Generation

- [ ] Generate changelog with git-cliff

  ```bash
  git cliff --unreleased --tag v0.1.2 -o CHANGELOG.md
  ```

- [ ] Review generated CHANGELOG.md
  - [ ] Sections are organized (Added, Changed, Fixed, etc.)
  - [ ] Commit messages are meaningful
  - [ ] Breaking changes are highlighted
  - [ ] No private/sensitive information

- [ ] Commit changelog

  ```bash
  git add CHANGELOG.md
  git commit -m "chore(release): update CHANGELOG for v0.1.2"
  ```

## Release Execution

### 7. Create & Push Tag

- [ ] Create annotated tag

  ```bash
  git tag -a v0.1.2 -m "Release v0.1.2"
  ```

- [ ] Verify tag locally

  ```bash
  git tag --list
  git show v0.1.2
  ```

- [ ] Push commits and tags

  ```bash
  git push origin main
  git push origin v0.1.2
  ```

### 8. Verify CI Build

- [ ] GitHub Actions CI workflow triggered
- [ ] All jobs pass (ruff, yamllint, ansible-lint, etc.)
- [ ] Version detection works correctly
- [ ] CI image builds successfully (if applicable)

### 9. GitHub Release

- [ ] Create GitHub Release
  - [ ] Go to: <https://github.com/USERNAME/podman-gitops-stack/releases/new>
  - [ ] Select tag: `v0.1.2`
  - [ ] Release title: `v0.1.2`
  - [ ] Description: Copy from CHANGELOG.md
  - [ ] Check "Set as the latest release" if appropriate
  - [ ] Publish release

## Post-Release Verification

### 10. Validation

- [ ] Verify version detection

  ```bash
  python -m setuptools_scm
  ```

- [ ] Test fresh installation

  ```bash
  python3 -m venv test-venv
  source test-venv/bin/activate
  pip install git+https://github.com/USERNAME/podman-gitops-stack@v0.1.2
  stackctl --help
  ```

- [ ] Verify package metadata

  ```bash
  python -c "import importlib.metadata as m; print(m.version('podman-gitops-stack'))"
  ```

### 11. Communication

- [ ] Update any external documentation
- [ ] Announce release (if applicable)
  - [ ] Project README
  - [ ] Release notes
  - [ ] Community channels

## Rollback Procedure (If Needed)

If issues are discovered post-release:

### 12. Emergency Rollback

- [ ] Delete GitHub release
- [ ] Delete tag remotely

  ```bash
  git push origin :refs/tags/v0.1.2
  ```

- [ ] Delete tag locally

  ```bash
  git tag -d v0.1.2
  ```

- [ ] Investigate and fix issue
- [ ] Repeat release process with patch version

## Notes

### Conventional Commits Reminder

All commits must follow Conventional Commits format:

- `feat:` - New functionality
- `fix:` - Bug fix
- `docs:` - Documentation only
- `chore:` - Tooling/repo maintenance
- `refactor:` - Internal refactor
- `test:` - Test only

### Version Scheme

- **MAJOR**: Breaking changes (e.g., 1.0.0 → 2.0.0)
- **MINOR**: New features, backward compatible (e.g., 0.1.0 → 0.2.0)
- **PATCH**: Bug fixes, backward compatible (e.g., 0.1.1 → 0.1.2)

### Git Cliff Configuration

The `cliff.toml` file controls changelog generation:

- Conventional commits are required
- Sections: Security, Added, Changed, Fixed, Performance, Documentation, Maintenance, Revert, Other
- Commits without conventional format are filtered out

### CI/CD Pipeline

- CI runs on every push to main and on pull requests
- Release workflow only runs on version tags
- Containerized CI toolchain ensures reproducibility
- CI is the source of truth for lint/test results
