# Heidelberg Server Management (Ansible)

This repository manages the Heidelberg Lab GPU servers using Ansible
from a macOS control machine.

------------------------------------------------------------------------

# Infrastructure Overview

## Production GPU Servers

| Hostname | IP Address      | GPU Type       |
|----------|-----------------|----------------|
| Pixel    | 129.206.138.101 | RTX 5070  12GB |
| Cosmos   | 129.206.138.69  | RTX 5070  12GB |
| Photon   | 129.206.138.91  | RTX 4090  24GB |

## Additional DL Nodes with Goodaccess VPN connection

| Hostname | IP Address   | GPU Type      |
|----------|--------------|---------------|
| DL0      | 172.24.4.103 | RTX 6000 24GB |
| DL2      | 172.24.4.97  | RTX 6000 24GB |
| DL3      | 172.24.4.101 | RTX 8000 48GB |

------------------------------------------------------------------------

# Governance & Security Model

## Admin Policy

-   `jeff` is the only sudo-capable administrator.
-   Lab members are **non-sudo users**.
-   All system-level changes are performed via Ansible.
-   Manual changes on single servers are forbidden (prevents
    configuration drift).

## Package Management Policy

### System-Level Packages (APT)

Examples: - system libraries - docker - drivers - kernel modules - OS
utilities

Rules: 1. No lab member receives sudo. 2. All system packages must be
installed via Ansible. 3. Never manually `apt install` on one server
only. 4. All changes must be committed to git.

### User-Space Software

Users should install via:

-   Conda / Mamba (preferred)
-   pip –user
-   local builds under $HOME

Never use `sudo` for personal experiments.

------------------------------------------------------------------------

# Repository Structure

    Heidelberg-Server-Management/
    │
    ├── README.md
    ├── ADMIN-SOP.md
    │
    └── ansible/
        ├── ansible.cfg
        ├── inventories/
        │   └── production/
        │       ├── hosts.ini
        │       └── group_vars/
        │           └── all.yml
        ├── playbooks/
        │   ├── base.yml
        │   ├── users.yml
        │   ├── docker.yml
        │   ├── gpu_check.yml
        │   ├── hosts.yml
        │   ├── audit_packages.yml
        │   └── site.yml
        └── vault/
            └── lab_users.vault.yml (encrypted)

------------------------------------------------------------------------

# macOS Control Machine Setup

Install Ansible:

``` bash
brew install ansible
```

Test:

``` bash
ansible --version
```

Ensure SSH access to all servers as `jeff`.

------------------------------------------------------------------------

# Core Operational Commands

## Test Connectivity

``` bash
ansible all -m ping
```

## Apply Baseline System State

``` bash
ansible-playbook playbooks/base.yml --ask-become-pass
```

## Manage Users

``` bash
ansible-playbook playbooks/users.yml \
  --vault-password-file .secrets/vault_pass.txt \
  --ask-become-pass
```

## GPU Verification

``` bash
ansible-playbook playbooks/gpu_check.yml
```

## Docker + NVIDIA Runtime

``` bash
ansible-playbook playbooks/docker.yml --ask-become-pass
```

------------------------------------------------------------------------

# Name Resolution Management

Managed via `playbooks/hosts.yml` which enforces:

    129.206.138.101 pixel
    129.206.138.69 cosmos
    129.206.138.91 photon
    172.24.4.103 dl0
    172.24.4.97 dl2
    172.24.4.101 dl3

Apply with:

``` bash
ansible-playbook playbooks/hosts.yml --ask-become-pass
```

------------------------------------------------------------------------

# Package Drift Detection

Audit manually installed packages:

``` bash
ansible-playbook playbooks/audit_packages.yml --ask-become-pass
```

Compare outputs locally to identify discrepancies.

Never fix drift manually — always update Ansible and reapply.

------------------------------------------------------------------------

# Internal SSH Between Servers

1.  Generate SSH key for `jeff` on one server.
2.  Distribute public key via Ansible `authorized_key` module.
3.  Verify passwordless SSH across cluster.

------------------------------------------------------------------------

# Maintenance Schedule

Monthly: - Run base.yml - Run gpu_check.yml - Audit package drift

Before major experiments: - Verify GPU health - Verify docker runtime -
Confirm persistence mode enabled

------------------------------------------------------------------------

End of README
