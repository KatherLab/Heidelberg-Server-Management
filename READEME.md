# Heidelberg Server Management (Ansible)

This repository manages the lab GPU servers using Ansible from a macOS control machine.

## Servers (production)
| Name   | IP              |
|--------|------------------|
| Pixel  | 129.206.138.101 |
| Cosmos | 129.206.138.69  |
| Photon | 129.206.138.91  |

Admin policy:
- `jeff` is the only sudo-capable admin on all servers.
- Lab members are non-sudo users.
- Lab users are managed by Ansible via an encrypted Ansible Vault file.

---

## Repo layout

ansible/
ansible.cfg
inventories/production/hosts.ini
inventories/production/group_vars/all.yml
playbooks/
base.yml
users.yml
gpu_check.yml
docker.yml
site.yml
clear_pw_expiry.yml
vault/
lab_users.vault.yml (encrypted)