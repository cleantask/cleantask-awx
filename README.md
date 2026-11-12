# CleanTask — infrastructure Ansible / AWX

Automates the CleanTask servers: staging (Proxmox) and production (Hetzner).

Architecture decisions live in [`docs/PROPOSITION_ARCHITECTURE_ANSIBLE.md`](docs/PROPOSITION_ARCHITECTURE_ANSIBLE.md).
Server naming: see `cleantask` → `docs/Infrastructure/SERVER_NAMING.md`.

## Layout

| Path | Purpose |
|---|---|
| `inventories/staging/` | Proxmox VM `ct-staging-app-01` |
| `inventories/prod/` | Hetzner (to come) |
| `group_vars/all/` | Shared settings + vault |
| `roles/` | What to install |
| `server_setup.yml` | Server only (Docker, PostgreSQL, …) |
| `deploy.yml` | Application (Compose, migrate, health) |

## Local usage

```bash
cd ~/Dev/cleantask-awx
python3 -m venv .venv && .venv/bin/pip install 'ansible-core>=2.16,<2.20'
.venv/bin/ansible-galaxy collection install -r collections/requirements.yml -p collections

# Secrets: encrypt each value (see docs/GUIDE_AWX.md), e.g.
#   SECRET=$(openssl rand -base64 24)
#   .venv/bin/ansible-vault encrypt_string --name 'vault_pg_password' "$SECRET"
# Paste the !vault block into group_vars/all/vault.yml — do not
# `ansible-vault encrypt` the whole file.

# Prefer running jobs from the AWX console (docs/GUIDE_AWX.md).
```

## Groups

| Group | Today | Later (HA) |
|---|---|---|
| `app_servers` | `ct-staging-app-01` | `ct-prod-app-01`, `ct-prod-app-02` |
| `db_servers` | same host | same hosts (Patroni) |
| `dcs_servers` | empty | app servers + etcd arbiter |
