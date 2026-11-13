# CleanTask — infrastructure Ansible / AWX

Automates the CleanTask servers: staging (Proxmox) and production (Hetzner).

Architecture decisions live in [`docs/PROPOSITION_ARCHITECTURE_ANSIBLE.md`](docs/PROPOSITION_ARCHITECTURE_ANSIBLE.md).  
AWX day-to-day: [`docs/GUIDE_AWX.md`](docs/GUIDE_AWX.md).  
Cloudflare Tunnel (create + DNS + vault + `tunnel_setup`): [`docs/CLOUDFLARE_TUNNEL.md`](docs/CLOUDFLARE_TUNNEL.md).  
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
| `tunnel_setup.yml` | Nginx (loopback) + Cloudflare Tunnel |
| `deploy_sites.yml` | Marketing site + docs (static, Node build in Docker) |
| `seed_demo.yml` | Staging-only demo orgs (`create_demo_data`) |

After first `deploy` + tunnel: either `seed-demo-staging` (BrightClean /
SparkleHome + platform admin) or a manual `createsuperuser` — see
[`docs/GUIDE_AWX.md`](docs/GUIDE_AWX.md) § 3.8–3.9. Then `deploy-sites-staging`
for website/docs.

## Local usage (laptop only — not the server)

Day-to-day deploys run from **AWX** ([`docs/GUIDE_AWX.md`](docs/GUIDE_AWX.md)).  
On your PC you only need Ansible to **encrypt vault strings** (and rare local checks).

One-time setup on the laptop:

```bash
cd ~/Dev/cleantask-awx
python3 -m venv .venv
.venv/bin/pip install 'ansible-core>=2.16,<2.20'
.venv/bin/ansible-galaxy collection install -r collections/requirements.yml -p collections
```

Then encrypt a secret (example):

```bash
SECRET=$(openssl rand -base64 24)
.venv/bin/ansible-vault encrypt_string --name 'vault_pg_password' "$SECRET"
# Paste the !vault block into inventories/staging/vault.yml
# (prod = inventories/prod/.../vault.yml). Do not encrypt the whole file.
```

## Groups

| Group | Today | Later (HA) |
|---|---|---|
| `app_servers` | `ct-staging-app-01` | `ct-prod-app-01`, `ct-prod-app-02` |
| `db_servers` | same host | same hosts (Patroni) |
| `dcs_servers` | empty | app servers + etcd arbiter |
