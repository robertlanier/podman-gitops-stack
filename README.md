# Podman GitOps Stack — Bootstrap Component (Ansible OS Provisioning)

Welcome to the **Podman GitOps Stack Bootstrap** — a fully automated, reproducible, GitOps-ready foundation for deploying your homelab media stack using **CentOS Stream 9**, **Ansible**, and (later) **rootless Podman**.

This README guides you through the entire process **starting from a fresh workstation**, all the way to having your CentOS Stream server configured exactly as your automation expects.

The bootstrap component handles the **operating system**, **users**, **NAS mounts**, and **base packages** as part of the overall Podman GitOps Stack.  
The *next* phase will automate **container orchestration** using the Podman Python SDK.

---

## 1. What This Bootstrap Actually Does

Running this Ansible playbook configures your CentOS server to be GitOps-ready by:

### 👤 Creating the system automation user (`stack`)

- UID/GID = 2000  
- Passwordless sudo  
- SSH-key only authentication  
- Prepared for **rootless Podman** (subuid/subgid configured)

> Note: The `solwyn` user (or installer user) is used only for initial SSH access and bootstrap execution.

### 🗄 Mounting NAS storage

- Creates `/mnt/data`
- Securely stores SMB credentials at `/etc/secure/cifs_creds`
- Ensures persistent mount via `/etc/fstab`
- Guarantees the share is always mounted via Ansible

### 🧰 Installing base OS packages

Including:

- podman  
- git  
- python3 / pip  
- cifs-utils  
- cockpit + cockpit-podman  
- nano / neovim  
- unzip  
- epel-release  
- btop  

### Enabling Cockpit Web Console

Access at:

```text
https://<server-hostname>:9090
```

---

## 2. Prepare Your Workstation

### Step 2.1 — Install Homebrew (macOS) or your package manager (Linux)

For macOS, install Homebrew if not already installed:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Step 2.2 — Set Up Python Virtual Environment & Install Dependencies

Create a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install Python dependencies:

```bash
pip install -e ".[dev]"
```

Install Ansible Galaxy collections:

```bash
ansible-galaxy collection install -r bootstrap/ansible/collections/requirements.yml
```

Confirm installation:

```bash
ansible --version
```

---

### Why Ansible Galaxy Is Still Required

Python package managers (pip / pyproject.toml) and Ansible Galaxy solve **different problems**:

- **pip / pyproject.toml** installs *Python libraries* (Ansible itself, ansible-lint, SDKs, CLI tools).
- **Ansible Galaxy** installs *Ansible content* (collections, modules, plugins, roles).

Even when Ansible is installed via `pip`, Galaxy collections **are not Python packages** and must be installed separately.

Examples of Galaxy-managed content:
- `ansible.posix`
- `community.general`
- Custom modules and filters

That is why this project uses **both**:
- `pip install -e ".[dev]"` for Python tooling
- `ansible-galaxy collection install -r bootstrap/ansible/collections/requirements.yml` for Ansible content

This separation is intentional and follows current Ansible best practices.

---

### Step 2.3 — Configure VS Code (Optional)

If using VS Code with the Ansible extension:

1. Open VS Code settings (Cmd+, or Ctrl+,)
2. Search for "Python: Default Interpreter Path"
3. Set it to: `/your/path/to/podman-gitops-stack/.venv/bin/python`
4. Reload the window (Cmd+Shift+P → "Reload Window")

### Step 2.4 — Create your SSH key for the automation user

```bash
ssh-keygen -t ed25519 -f ~/.ssh/stack_server
```

This generates:

- `~/.ssh/stack_server` (private key)
- `~/.ssh/stack_server.pub` (public key)

Copy the public key for later:

```bash
pbcopy < ~/.ssh/stack_server.pub
```

---

## 3. Prepare Your CentOS Stream Server

### During OS installation

- Set hostname accordingly
- Create an installer/admin user (ex: `solwyn`)
- Ensure it has sudo access
- Boot into the OS

### After install

1. SSH into the server **once** to store its fingerprint:

    ```bash
    ssh solwyn@<server-ip>
    ```

2. Log out.

You're now ready to automate the entire OS.

---

## 4. Repository Structure (What You Should See)

The following tree shows only the files relevant to the **bootstrap Ansible component**.

```text
podman-gitops-stack/
  ├── bootstrap/
  │     └── ansible/
  │            ├── inventory/
  │            │     └── hosts.yml
  │            ├── group_vars/
  │            │     └── stack/
  │            │            └── vault.yml              # encrypted secrets
  │            ├── roles/
  │            │     ├── users/
  │            │     │     ├── tasks/main.yml
  │            │     │     └── defaults/main.yml       # user/group defaults
  │            │     ├── base/
  │            │     │     ├── tasks/main.yml
  │            │     │     ├── handlers/main.yml       # cockpit restart handler
  │            │     │     └── defaults/main.yml       # base service defaults
  │            │     └── nas/
  │            │           ├── tasks/main.yml
  │            │           └── defaults/main.yml       # NAS mount defaults
  │            ├── ansible.cfg
  │            ├── site.yml
  │            ├── collections/
  │            ├── scripts/
  │            │     └── vault-pass/
  │            │           └── vault_pass.sh            # vault password helper script
  │            ├── .ansible/                             # runtime cache and Galaxy metadata (gitignored)
  │            ├── requirements.txt                      # Ansible/Python deps for bootstrap
  │            └── README.md
  ├── scripts/
  │     └── vault-pass/
  │           └── vault_pass.sh                        # vault password helper script
  ├── .gitignore
  ├── .venv/                                             # root Python venv (recommended)
  └── README.md
```

> Note: The `.ansible/` directory under `bootstrap/ansible` is used for runtime cache and Galaxy metadata.  
> It is intentionally ignored by git to avoid committing transient files.

---

## 5. Set Up Your Ansible Vault (Required Before First Run)

Create vault directory:

```bash
mkdir -p bootstrap/ansible/group_vars/stack
```

Create an encrypted vault file:

```bash
ansible-vault create bootstrap/ansible/group_vars/stack/vault.yml
```

Paste inside (replace with your actual values):

```yaml
# SMB credentials for NAS access
nas_username: automation
nas_password: YOUR_REAL_SMB_PASSWORD

# SSH public key for stack automation user
# Generate with: ssh-keygen -t ed25519 -f ~/.ssh/stack_server
# Copy with: pbcopy < ~/.ssh/stack_server.pub
stack_user_ssh_pubkey: "ssh-ed25519 AAAA...."
```

Save and exit.  
This file is now AES-encrypted and **safe to commit**.

**Vault Variables Explained:**

- `nas_username` — SMB username for NAS share authentication  
- `nas_password` — SMB password (use a strong password; consider rotating periodically)  
- `stack_user_ssh_pubkey` — Your public SSH key for automation user authentication (key-only auth, no passwords)  

---

## 6. Secrets Management UX

This project standardizes vault password handling via a single entrypoint script located at:

```text
bootstrap/ansible/scripts/vault-pass/vault_pass.sh
```

This script integrates with the 1Password CLI by default to securely fetch the vault password, simplifying automation and improving security.

You can customize or extend this script to use other secrets managers or vault password sources.

---

## 7. Configure the Inventory

Edit `bootstrap/ansible/inventory/hosts.yml`:

```yaml
all:
  children:
    stack:
      hosts:
        server:
          ansible_host: 192.168.x.x
          ansible_user: stack
          ansible_ssh_private_key_file: ~/.ssh/stack_server
          ansible_python_interpreter: /usr/bin/python3
```

> Note: Initial runs typically use the installer/admin user (e.g., `solwyn`) for SSH access and bootstrap execution.  
> After bootstrapping completes, you may update `ansible_user` to `stack` for subsequent automation runs.

---

## 8. Test Connectivity

```bash
ansible -i bootstrap/ansible/inventory/hosts.yml stack -m ping --vault-password-file bootstrap/ansible/scripts/vault-pass/vault_pass.sh
```

Expected output:

```bash
server | SUCCESS => {"changed": false, "ping": "pong"}
```

---

## 9. Run the Bootstrap

```bash
ansible-playbook -i bootstrap/ansible/inventory/hosts.yml bootstrap/ansible/site.yml --vault-password-file bootstrap/ansible/scripts/vault-pass/vault_pass.sh
```

A successful run ends with:

```bash
PLAY RECAP **********************************************************
server : ok=16  changed=0  unreachable=0  failed=0  skipped=0
```

If `changed=0`, that means your system is **fully converged** and exactly matches the desired state.

---

## 10. Verify the System

### Check NAS mount

```bash
mount | grep /mnt/data
ls /mnt/data
```

### Check automation user

```bash
id stack
```

### Check sudo (should not require a password)

```bash
sudo whoami
```

### Check Cockpit

Visit:

```text
https://<server-hostname>:9090
```

---

## 11. Recommended .gitignore

```text
# macOS
.DS_Store

# Ansible artifacts
*.retry
.ansible_vault_pass

# Python cache
__pycache__/
*.pyc

# Python virtual environment (repo root)
.venv/

# Editor folders
.vscode/
.idea/
```

---

## 12. Tag Your Bootstrap Release

Once everything looks correct:

```bash
git add .
git commit -m "Bootstrap: OS, NAS, users, and base packages"
git tag -a bootstrap -m "Podman GitOps Stack bootstrap"
git push
git push --tags
```

---

## 13. What Comes Next (Phase 2)

This bootstrap component prepares the OS.  
Next we will build:

### 🐳 A declarative container stack using

- Rootless Podman  
- Traefik routing  
- Jellyfin / Jellyseerr  
- Sonarr / Radarr / Prowlarr  
- qBittorrent + management utilities  
- Auto-updating & health checks

### 🐍 Python Podman SDK (GitOps)

- Generate systemd units automatically  
- Reconcile container state on demand  
- Push config updates through Git  

### Optional: PXE / Kickstart auto-installer

Your NAS can host a PXE boot environment that automatically:

1. Installs CentOS Stream  
2. Pulls this repo  
3. Runs the bootstrap  
4. Registers itself into your fleet  

---

## Done

Your OS layer is now fully automated, reproducible, and GitOps-friendly.  
You're ready for Phase 2: **container orchestration automation**.
