# Heidelberg Lab – Administrator SOP

This document is for the admin (`jeff`) only.

------------------------------------------------------------------------

# Golden Rules

1.  Never manually modify only one server.
2.  All system-level changes must go through Ansible.
3.  Keep servers consistent.
4.  Vault password must never be committed to git.
5.  Docker group membership = root-level power.

------------------------------------------------------------------------

# Scenario Playbooks

## Scenario 1 – Add New Lab Member

1.  Generate SHA-512 password hash:

``` bash
pip install passlib
python - <<'PY'
from passlib.hash import sha512_crypt
import getpass
pw = getpass.getpass("Password: ")
print(sha512_crypt.hash(pw))
PY
```

2.  Edit vault:

``` bash
ansible-vault edit ansible/vault/lab_users.vault.yml
```

3.  Apply:

``` bash
ansible-playbook playbooks/users.yml \
  --vault-password-file .secrets/vault_pass.txt \
  --ask-become-pass
```

------------------------------------------------------------------------

## Scenario 2 – Add New Server

1.  Install Ubuntu.
2.  Create `jeff` with sudo.
3.  Enable SSH.
4.  Add to inventory.
5.  Bootstrap:

``` bash
ansible-playbook playbooks/base.yml --limit newnode --ask-become-pass
ansible-playbook playbooks/users.yml --limit newnode --vault-password-file .secrets/vault_pass.txt --ask-become-pass
```

------------------------------------------------------------------------

## Scenario 3 – One User Requests sudo Package

Decision tree:

If system-level: - Add to Ansible (group_vars/all.yml) - Commit change -
Apply to all servers

If user-level: - Instruct user to install via conda or pip –user

Never grant sudo.

------------------------------------------------------------------------

## Scenario 4 – Server Drift Detected

1.  Run audit_packages.yml
2.  Compare outputs
3.  Decide: install everywhere OR remove everywhere
4.  Update Ansible
5.  Reapply playbooks

------------------------------------------------------------------------

## Scenario 5 – Docker Issues

Reapply:

``` bash
ansible-playbook playbooks/docker.yml --ask-become-pass
```

Verify:

``` bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

------------------------------------------------------------------------

## Scenario 6 – SSH Issues Between Nodes

1.  Re-distribute cluster SSH key.
2.  Verify /etc/hosts entries.
3.  Confirm permissions on \~/.ssh (700) and authorized_keys (600).

------------------------------------------------------------------------

# Disaster Recovery Philosophy

If something breaks: - Do NOT manually patch. - Fix playbook. - Reapply
configuration.

Ansible is the source of truth.

------------------------------------------------------------------------

End of Admin SOP
