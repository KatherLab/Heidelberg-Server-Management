# Heidelberg Server Management (Ansible)

This repository manages the Heidelberg Lab GPU servers (and related nodes) using **Ansible** from a macOS/Linux control
machine. The intent is to keep configuration **consistent, auditable, and repeatable**.

> **Golden rule:** avoid one-off manual changes on a single host. Fix the playbook and re-apply.

---

## Infrastructure Overview

### Production GPU Servers (Heidelberg)

| Hostname | IP Address      | GPU Type       |
|----------|-----------------|----------------|
| pixel    | 129.206.138.101 | RTX 5070 12GB  |
| cosmos   | 129.206.138.69  | RTX 5070 12GB  |
| photon   | 129.206.138.91  | RTX 4090 24GB  |

### Additional DL Nodes (Dresden / VPN)

| Hostname | IP Address   | GPU Type      |
|----------|--------------|---------------|
| dl0      | 172.24.4.103 | RTX 6000 24GB |
| dl2      | 172.24.4.97  | RTX 6000 24GB |
| dl3      | 172.24.4.101 | RTX 8000 48GB |

---

## Governance & Security Model

### Admin Policy

- `jeff` is the only sudo-capable administrator.
- Lab members are **non-sudo users**.
- All system-level changes are performed via Ansible.
- Manual per-host changes are strongly discouraged (prevents configuration drift).

### Package Management Policy

**System-level packages (APT)** (libraries, docker, drivers, kernel modules, OS utilities)

Rules:
1. No lab member receives sudo.
2. All system packages must be installed via Ansible.
3. Never `apt install` on a single server only.
4. All changes must be committed to git.

**User-space software**
- Conda / Mamba (preferred)
- `pip --user`
- local builds under `$HOME`

Never use `sudo` for personal experiments.

---

## Repository Structure

```
Heidelberg-Server-Management/
├── README.md
├── ADMIN_SOP.md
├── .ansible-lint
├── .yamllint
├── .pre-commit-config.yaml
├── .github/workflows/ci.yml
└── ansible/
    ├── ansible.cfg
    ├── inventories/
    │   └── production/
    │       ├── hosts.ini
    │       └── group_vars/
    │           ├── all.yml
    │           ├── heidelberg.yml
    │           └── dresden.yml
    ├── playbooks/
    │   ├── site.yml
    │   ├── base.yml
    │   ├── users.yml
    │   ├── docker.yml
    │   ├── tailscale.yml
    │   ├── gpu_check.yml
    │   ├── hosts.yml
    │   ├── audit_packages.yml
    │   └── clear_pw_expiry.yml
    └── vault/
        └── lab_users.vault.yml (encrypted)
```

---

## Control Machine Setup (macOS / Linux)

### Install Ansible

macOS:
```bash
brew install ansible
```

Linux (example):
```bash
python3 -m pip install --user ansible
```

Verify:
```bash
ansible --version
```

### SSH prerequisites

1. Ensure SSH access to all servers (typically as `jeff` or the group-specific `ansible_user`).
2. Add host keys to `~/.ssh/known_hosts` (recommended since `host_key_checking=True` in `ansible.cfg`):
   ```bash
   ssh-keyscan -H 129.206.138.101 129.206.138.69 129.206.138.91 >> ~/.ssh/known_hosts
   ```

### Working directory convention

Run Ansible **from inside** the `ansible/` directory so relative paths resolve:

```bash
cd ansible
```

---

## Vault Setup (user management)

User accounts are managed from the encrypted vault file:

- `ansible/vault/lab_users.vault.yml`

**Do not commit the vault password.**

Recommended local setup:
```bash
mkdir -p .secrets
chmod 700 .secrets
# Put your vault password into:
#   .secrets/vault_pass.txt
chmod 600 .secrets/vault_pass.txt
```

Editing the vault:
```bash
ansible-vault edit vault/lab_users.vault.yml --vault-password-file .secrets/vault_pass.txt
```

---

## Core Operational Commands

All commands below assume you are in the `ansible/` directory.

### Test connectivity

```bash
ansible all -m ping
```

### Apply baseline system state

```bash
ansible-playbook playbooks/base.yml --ask-become-pass
```

### Manage users

```bash
ansible-playbook playbooks/users.yml   --vault-password-file .secrets/vault_pass.txt   --ask-become-pass
```

### Docker + NVIDIA runtime

```bash
ansible-playbook playbooks/docker.yml --ask-become-pass
```

Smoke test on a target host:
```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

### Tailscale install (does not auto-authenticate)

```bash
ansible-playbook playbooks/tailscale.yml --ask-become-pass
```

### GPU verification

```bash
ansible-playbook playbooks/gpu_check.yml
```

### Name resolution management

`playbooks/hosts.yml` manages a dedicated block in `/etc/hosts` and keeps cluster names consistent.

```bash
ansible-playbook playbooks/hosts.yml --ask-become-pass
```

### Run the whole “site”

`playbooks/site.yml` composes the common playbooks:

```bash
ansible-playbook playbooks/site.yml   --vault-password-file .secrets/vault_pass.txt   --ask-become-pass
```

### Limit to a single host (safe for bootstrapping)

```bash
ansible-playbook playbooks/base.yml --limit pixel --ask-become-pass
```

---

## Drift Detection

Audit manually installed apt packages:

```bash
ansible-playbook playbooks/audit_packages.yml --ask-become-pass
```

Results are fetched to:
- `ansible/audits/<host>-apt-manual.txt`

Never fix drift manually — always update Ansible and reapply.

---

## Common Admin Workflows

### Add a new lab member

See `ADMIN_SOP.md` for the full procedure. In short:
1. Generate a password hash (SHA-512 crypt).
2. Add the user entry to the vault.
3. Run `playbooks/users.yml`.

### Add a new server

1. Install Ubuntu.
2. Create the bootstrap admin user with sudo (e.g. `jeff`) and enable SSH.
3. Add the host to `inventories/production/hosts.ini`.
4. Ensure it belongs to the correct group (`heidelberg` or `dresden`).
5. Apply baseline + users playbooks (limit to the new host).

---

## CI/CD, Linting, and Local Quality Gates

### GitHub Actions

A workflow at `.github/workflows/ci.yml` runs:
- `yamllint`
- `ansible-lint`

### Local pre-commit (recommended)

Install pre-commit:
```bash
python3 -m pip install --user pre-commit
pre-commit install
```

Run checks:
```bash
pre-commit run --all-files
```

---

## Notes on Idempotence and Safety

- Prefer `apt`, `copy`, `template`, `blockinfile`, and service modules over `shell`.
- When `shell/command` is necessary (vendor bootstrap), document why and guard it with `creates:` or content checks.
- Consider converting playbooks into roles if the repository grows (e.g. `roles/base`, `roles/users`, `roles/docker`).

---

## Troubleshooting

- **Host key verification failed:** update `~/.ssh/known_hosts` (or remove the offending line and re-scan).
- **Permission denied (publickey):** verify the correct `ansible_user` is set in inventory/group_vars.
- **Docker GPU access fails:** re-run `playbooks/docker.yml` and validate the NVIDIA driver + `nvidia-smi` works on the host.

---

End of README
