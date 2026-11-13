# Cloudflare Tunnel — CleanTask

Expose staging (puis prod) **sans ouvrir 80/443** sur le serveur :  
visiteur → Cloudflare (HTTPS) → tunnel sortant → Nginx `127.0.0.1:80` → app Docker `:8000`.

AWX ne crée **pas** le tunnel chez Cloudflare. Tu le crées une fois sur ton poste ;  
Ansible (`tunnel_setup.yml`) installe `cloudflared` et dépose le JSON sur la VM.

Pour le reste d’AWX (projet, inventaire, vault app) : [`GUIDE_AWX.md`](GUIDE_AWX.md).

---

## 1. Hostnames (staging)

Universal SSL Cloudflare ne couvre que `*.cleantask.co` (un seul label).

| Rôle | Hostname | Inventaire |
|---|---|---|
| App Django | `app-staging.cleantask.co` | `app_domain` |
| Site vitrine | `staging.cleantask.co` | `website_domain` |
| Doc | `docs-staging.cleantask.co` | `docs_domain` |

Prod (plus tard) : `app.cleantask.co`, `cleantask.co`, `docs.cleantask.co` — autre tunnel, autre vault.

---

## 2. Créer le tunnel (une fois, sur ton poste)

### 2.1 Installer `cloudflared` (si besoin)

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install cloudflared
```

### 2.2 Se connecter au compte Cloudflare

```bash
cloudflared tunnel login
```

Choisir la zone **`cleantask.co`**.  
→ certificat `~/.cloudflared/cert.pem` (autorise la création de tunnels pour ce domaine).

### 2.3 Créer le tunnel staging

```bash
cloudflared tunnel create cleantask-staging
```

Exemple de sortie :

```text
Tunnel credentials written to /home/vous/.cloudflared/6ff42ae2-765d-4adf-8112-31c55c1551ef.json
Created tunnel cleantask-staging with id 6ff42ae2-765d-4adf-8112-31c55c1551ef
```

Noter :

- **UUID** → `cloudflare_tunnel_id`
- **Fichier JSON** → secret vault (ne pas le committer en clair)

Vérifier : `cloudflared tunnel list` → `cleantask-staging`, colonne CONNECTIONS encore vide (normal).

### 2.4 Routes DNS (3 hostnames)

Le tunnel publie des CNAME proxied vers `<uuid>.cfargotunnel.com`.

```bash
cloudflared tunnel route dns cleantask-staging app-staging.cleantask.co
cloudflared tunnel route dns cleantask-staging staging.cleantask.co
cloudflared tunnel route dns cleantask-staging docs-staging.cleantask.co
```

Contrôle dans **Cloudflare → DNS** : trois CNAME, nuage orange.

### 2.5 SSL / TLS (zone)

Le navigateur parle **toujours en HTTPS** à Cloudflare.  
Entre Cloudflare et ton serveur, le tunnel transporte du **HTTP** vers Nginx
(`127.0.0.1:80`) : **aucun certificat** à installer sur la VM
(pas de Certbot / Let’s Encrypt).

Dans le tableau de bord Cloudflare → **SSL/TLS** (réglage de la zone
`cleantask.co`, une fois pour tout le domaine) :

| Réglage | Valeur | Pourquoi |
|---|---|---|
| Encryption mode | **Full** | Cloudflare accepte de parler HTTP à l’origine (notre Nginx local). *Flexible* est trop faible ; *Full (strict)* demanderait un cert valide sur le serveur — on n’en a pas. |
| Always Use HTTPS | **On** | Une visite en `http://…` est redirigée vers `https://…` côté Cloudflare. |
---

## 3. Brancher dans le dépôt `cleantask-awx`

### 3.1 UUID (inventaire, pas secret)

`inventories/staging/group_vars/all/env.yml` :

```yaml
cloudflare_tunnel_id: 6ff42ae2-765d-4adf-8112-31c55c1551ef   # plus A_RENSEIGNER
nginx_noindex: true
```

### 3.2 JSON (vault)

Chiffrer le contenu du fichier `~/.cloudflared/<uuid>.json` — **jamais** en clair dans Git.
Préférer rediriger le fichier (évite les espaces en fin de ligne au collage, qui cassent le vault) :

```bash
cd ~/Dev/cleantask-awx
.venv/bin/ansible-vault encrypt_string \
  --stdin-name 'vault_cloudflare_tunnel_credentials' \
  < ~/.cloudflared/<uuid>.json
# coller le bloc !vault dans inventories/staging/vault.yml
# Vérifier : la dernière ligne hex n’a pas d’espaces après, fichier termine par une newline
```

Voir aussi `inventories/staging/vault.yml.example`.

### 3.3 Push + sync AWX

```bash
git add inventories/staging/group_vars/all/env.yml inventories/staging/vault.yml
git commit -m "Staging Cloudflare tunnel id + vault credentials"
git push
```

Dans AWX : synchroniser le **projet** et l’**inventaire** `cleantask-staging`.

---

## 4. Job AWX `tunnel-setup-staging`

Prérequis : `server-setup-staging` et `deploy-staging` déjà **Réussi** (app sur `127.0.0.1:8000`).

| Champ | Valeur |
|---|---|
| Nom | `tunnel-setup-staging` |
| Inventaire | `cleantask-staging` |
| Projet | `cleantask-awx` |
| EE | même EE numéroté que les autres jobs |
| Playbook | `tunnel_setup.yml` |
| Identifiants | Machine `ssh-cleantask` **et** Vault `vault-cleantask` |
| Élévation des privilèges | **cochée** |

Le playbook applique :

1. Rôle **`nginx`** — listen `127.0.0.1:80`, vhosts app / website / docs, `/ws/`, `CF-Connecting-IP`
2. Rôle **`cloudflared`** — paquet, JSON, `config.yml` (3 hostnames → `http://127.0.0.1:80`), service systemd

---

## 5. Vérifications

| Contrôle | Attendu |
|---|---|
| Job AWX | **Réussi** |
| `cloudflared tunnel list` (poste) | 1 connexion active pour `cleantask-staging` |
| Sur la VM : `curl -fsS -H 'Host: app-staging.cleantask.co' http://127.0.0.1/core/health/` | HTTP **200** |
| Navigateur : `https://app-staging.cleantask.co/` | app CleanTask (souvent redirect `/login/`) |
| `https://staging.cleantask.co/` / docs | placeholder jusqu’au rôle `static_sites` |

**Pas de compte encore** : le tunnel n’ouvre que l’app. Créer le platform admin  
(`createsuperuser`) — voir [`GUIDE_AWX.md` § 3.8](GUIDE_AWX.md).  
Site vitrine / docs : job `deploy-sites-staging` (`deploy_sites.yml`).

Sur la VM (lecture seule) :

```bash
systemctl is-active nginx cloudflared
sudo nginx -t
```

---

## 6. Chaîne de confiance IP (app)

1. Cloudflare pose **`CF-Connecting-IP`**
2. Nginx `real_ip` + headers → Docker
3. Django `TRUST_PROXY_HEADERS=True` (déjà dans `roles/app/templates/env.j2`)

Sans ça, axes / Turnstile voient tout le monde en `127.0.0.1`.

---

## 7. Production (rappel)

Quand la VM prod existe :

1. `cloudflared tunnel create cleantask-prod` (autre UUID, autre JSON)
2. Routes DNS pour `app.cleantask.co`, `cleantask.co`, `docs.cleantask.co`
3. `inventories/prod/.../env.yml` + `inventories/prod/vault.yml` (vault **séparé**)
4. Job `tunnel-setup-prod` → même `tunnel_setup.yml`, inventaire prod

Pas de Load Balancer tant qu’il n’y a qu’**un** serveur d’app.

---

## 8. Dépannage

| Symptôme | Piste |
|---|---|
| Assert « A_RENSEIGNER » / vault manquant | UUID dans `env.yml` + `vault_cloudflare_tunnel_credentials` chiffré |
| Tunnel list sans connexion | Job `tunnel_setup` pas lancé, ou `cloudflared` down sur la VM |
| 502 / vide via HTTPS | App pas up (`deploy-staging`) ; tester curl Nginx local §5 |
| Mauvais Host / cookie | `ALLOWED_HOSTS` / `SITE_URL` = `app_domain` (redeploy app si domaine changé) |
| Certificat navigateur | DNS pas proxied (nuage gris) ou hostname hors `*.cleantask.co` |

---

## 9. Fichiers concernés

| Fichier | Rôle |
|---|---|
| `tunnel_setup.yml` | Play nginx + cloudflared |
| `roles/nginx/` | Vhosts CleanTask |
| `roles/cloudflared/` | Tunnel → Nginx loopback |
| `inventories/staging/group_vars/all/env.yml` | `cloudflare_tunnel_id`, domaines |
| `inventories/staging/vault.yml` | `vault_cloudflare_tunnel_credentials` |
