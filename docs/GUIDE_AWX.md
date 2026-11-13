# Guide AWX — CleanTask

Modifier sur le PC → `git push` → lancer le job dans AWX.  
AWX clone GitHub ; il ne lit pas `~/Dev/cleantask-awx/`.  
Ne pas lancer `ansible-playbook` une fois AWX en place.

AWX : machine `ansible` (`192.168.10.114`).

---

## 1. Dépôts

| Dépôt | Qui clone |
|---|---|
| `cleantask/cleantask-awx` | **AWX** (playbooks) |
| `cleantask/cleantask` | Le serveur (rôle `app` / `deploy.yml`) |
| `cleantask/cleantask-website` / `cleantask-docs` | Le serveur (`deploy_sites.yml`) |

---

## 2. Vault (`encrypt_string`)

Chiffrer **chaque valeur**, pas le fichier entier.  
Le mot de passe du **coffre** → identifiant AWX `vault-cleantask` (jamais dans Git).  
Ne pas faire : `ansible-vault encrypt` sur tout le fichier vault.
Un vault **par environnement** :
`inventories/staging/vault.yml` et plus tard `inventories/prod/vault.yml`.
**Pas** sous `group_vars/` : AWX corrompt les blocs `!vault` à l’import d’inventaire.
Chargé au début du playbook (rôle `load_vault`).

```bash
cd ~/Dev/cleantask-awx

# Mot de passe du coffre (une fois) → noter + credential vault-cleantask
openssl rand -base64 32
```

### 2.1 Postgres (`server_setup`)

```bash
SECRET=$(openssl rand -base64 24)
.venv/bin/ansible-vault encrypt_string --name 'vault_pg_password' "$SECRET"
# coller le bloc !vault dans inventories/staging/vault.yml
```

### 2.2 Secrets app (staging = valeurs du `.env` local)

Fichier : **`inventories/staging/vault.yml`**  
(prod = autre fichier, autres valeurs — voir `inventories/prod/.../vault.yml.example`).

Pour chaque clé : prendre la valeur dans `~/Dev/cleantask/.env`, puis :

```bash
.venv/bin/ansible-vault encrypt_string --stdin-name 'NOM_VAULT'
# coller → Entrée → Ctrl-D → coller le bloc dans le vault **staging**
```

| Clé vault | Source `.env` | Obligatoire |
|---|---|---|
| `vault_django_secret_key` | `SECRET_KEY` | oui (boot) |
| `vault_django_secret_key_fallbacks` | `SECRET_KEY_FALLBACKS` | non |
| `vault_field_encryption_salt` | `FIELD_ENCRYPTION_SALT` | oui (boot) |
| `vault_redis_password` | `REDIS_PASSWORD` ou `openssl rand -hex 32` | oui (boot) |
| `vault_redis_cache_password` | `REDIS_CACHE_PASSWORD` ou autre `rand` | oui (boot) |
| `vault_outlook_client_id` | `OUTLOOK_CLIENT_ID` | oui (boot) |
| `vault_outlook_client_secret` | `OUTLOOK_CLIENT_SECRET` | oui (boot) |
| `vault_outlook_client_secret_expires_at` | `OUTLOOK_CLIENT_SECRET_EXPIRES_AT` | oui |
| `vault_postmark_server_token` | `POSTMARK_SERVER_TOKEN` | recommandé |
| `vault_gmail_client_id` | `GMAIL_CLIENT_ID` | recommandé |
| `vault_gmail_client_secret` | `GMAIL_CLIENT_SECRET` | recommandé |
| `vault_turnstile_secret_key` | `TURNSTILE_SECRET_KEY` | recommandé |
| `vault_stripe_secret_key` | `STRIPE_SECRET_KEY` | recommandé |
| `vault_stripe_webhook_secret` | `STRIPE_WEBHOOK_SECRET` | recommandé |
| `vault_cbeapi_api_key` | `CBEAPI_API_KEY` | recommandé |
| `vault_assistant_openrouter_api_key` | `ASSISTANT_OPENROUTER_API_KEY` | recommandé |

#### Vault vs inventaire (Turnstile / Stripe)

| `.env` | Vault (`vault_*`) | Inventaire (`env.yml`) |
|---|---|---|
| `TURNSTILE_SECRET_KEY` | `vault_turnstile_secret_key` | — |
| `TURNSTILE_SITE_KEY` | — | `app_turnstile_site_key` (déjà rempli) |
| `STRIPE_SECRET_KEY` | `vault_stripe_secret_key` | — |
| `STRIPE_WEBHOOK_SECRET` | `vault_stripe_webhook_secret` | — |
| `STRIPE_PUBLISHABLE_KEY` | — | `app_stripe_publishable_key` (déjà rempli) |
| `STRIPE_PRICE_*` | — | `app_stripe_price_*` (déjà rempli) |

Autres non-secrets (From/Reply, modèle assistant…) → aussi `env.yml`.  
`DATABASE_URL` / `SITE_URL` / `ALLOWED_HOSTS` : générés par le rôle (pas le `.env` local).

À la rotation du secret Entra : re-chiffrer secret + date, commit, `deploy-staging`.

### 2.3 Deploy key application (`cleantask/cleantask`)

Clé **différente** de `git-cleantask-awx` (celui-là = dépôt Ansible seulement).

```bash
cd ~/Dev/cleantask-awx
ssh-keygen -t ed25519 -C cleantask-app-deploy -f cleantask_app_deploy -N ""
cat cleantask_app_deploy.pub
```

1. GitHub → repo **`cleantask/cleantask`** → **Settings → Deploy keys → Add**  
   titre `awx-app`, coller le `.pub`, **Allow write access décoché**.
2. Chiffrer la **privée** :

```bash
.venv/bin/ansible-vault encrypt_string --stdin-name 'vault_app_git_deploy_key' < cleantask_app_deploy
# coller le bloc dans inventories/staging/vault.yml
rm cleantask_app_deploy cleantask_app_deploy.pub
```

3. Commit + push le vault staging → sync projet AWX.

Liste complète : `inventories/staging/vault.yml.example`.
---

## 3. Créer dans AWX (ordre)

Tous les objets dans l’organisation **`CleanTask`**.

| # | Objet | Nom |
|---|---|---|
| 3.1 | Organisation | `CleanTask` |
| 3.2 | Identifiants | `git-cleantask-awx`, `ssh-cleantask`, `vault-cleantask` |
| 3.3 | Projet | `cleantask-awx` |
| 3.4 | Inventaire | `cleantask-staging` |
| 3.5 | Modèle de job | `server-setup-staging` (prod : `server-setup-prod` plus tard) |
| 3.6 | Modèle de job | `deploy-staging` (après secrets vault app + deploy key) |

### 3.1 Organisation

**Accès → Organisations → Ajouter** → Nom : `CleanTask`.

### 3.2 Identifiants

Deux clés SSH **différentes** : une pour GitHub, une pour les serveurs.  
Votre clé GitHub perso ne va jamais dans AWX.

#### `git-cleantask-awx` (Contrôle de source)

```bash
ssh-keygen -t ed25519 -C git-cleantask-awx -f git-cleantask-awx_ed25519 -N ""
cat git-cleantask-awx_ed25519.pub
```

GitHub → `cleantask/cleantask-awx` → **Settings → Deploy keys** → coller le `.pub`, write **décoché**.

| Champ | Valeur |
|---|---|
| Nom | `git-cleantask-awx` |
| Organisation | `CleanTask` |
| Type | Contrôle de source |
| Nom d’utilisateur | `git` |
| Clé privée SCM | contenu de `git-cleantask-awx_ed25519` |
| Phrase de passe | vide |

Puis `rm git-cleantask-awx_ed25519*`.

#### `ssh-cleantask` (Machine)

Clé publique dans `/home/ubuntu/.ssh/authorized_keys` sur `ct-staging-app-01` (`192.168.10.115`). Sudo NOPASSWD déjà en place.

```bash
ssh-keygen -t ed25519 -C awx-cleantask -f awx-cleantask_ed25519 -N ""
# coller awx-cleantask_ed25519.pub dans authorized_keys sur la VM
```

| Champ | Valeur |
|---|---|
| Nom | `ssh-cleantask` |
| Organisation | `CleanTask` |
| Type | Machine |
| Nom d’utilisateur | `ubuntu` |
| Clé privée SSH | contenu de `awx-cleantask_ed25519` |
| Méthode d’escalade | `sudo` |
| Mot de passe d’escalade | vide |

Puis supprimer la clé privée du poste.

#### `vault-cleantask` (Archivage sécurisé)

| Champ | Valeur |
|---|---|
| Nom | `vault-cleantask` |
| Organisation | `CleanTask` |
| Type | Archivage sécurisé |
| Mot de passe | mot de passe du **coffre** (§2) |
| Me le demander au lancement | **décoché** |

### 3.3 Projet `cleantask-awx`

**Projets → Ajouter**

| Champ | Valeur |
|---|---|
| Nom | `cleantask-awx` |
| Organisation | `CleanTask` |
| Type de contrôle de source | Git |
| URL | `git@github.com:cleantask/cleantask-awx.git` |
| Branche | `main` |
| Identifiant (contrôle de source) | `git-cleantask-awx` |
| Environnement d’exécution | EE AWX par défaut (ou version numérotée) — **pas** l’URL Git |
| Options → Mettre à jour Révision au lancement | **coché** |
| Options → Nettoyer | **coché** |
| Options → Supprimer | décoché (re-clone complet, seulement en dépannage) |

**Enregistrer → Synchroniser** → vert = OK.  
`collections/requirements.yml` s’installe à cette sync.

### 3.4 Inventaire `cleantask-staging`

1. **Inventaires → Ajouter** : nom `cleantask-staging`, org `CleanTask`.
2. **Sources → Ajouter** :

| Champ | Valeur |
|---|---|
| Nom | `depuis-git` |
| Source | Provenance d’un projet |
| Environnement d’exécution | AWX EE **version numérotée** (pas `latest`) |
| Projet | `cleantask-awx` |
| Fichier d’inventaire | `inventories/staging/hosts.yml` (Entrée ; pas « project root ») |
| Options | Remplacer + Remplacer les variables + Mettre à jour au lancement |

3. **Synchroniser** → Hôtes : `ct-staging-app-01` ; Groupes : `app_servers`, `db_servers`.  
Ne pas créer les hôtes à la main.

**Test SSH** (Inventaire → Hôtes → cocher l’hôte → Exécuter une commande) :

| Module | Élévation | Attendu |
|---|---|---|
| `ping` | Non | `pong` |
| `command` / `id` | Oui | `uid=0(root)` |

### 3.5 Modèle `server-setup-staging`

**Modèles → Ajouter → Ajouter un modèle de job**

Un modèle **par environnement** (même playbook `server_setup.yml`, inventaire différent). Prod plus tard : `server-setup-prod` → inventaire `cleantask-prod`.

| Champ | Valeur |
|---|---|
| Nom | `server-setup-staging` |
| Organisation | `CleanTask` |
| Type de job | Exécuter |
| Inventaire | `cleantask-staging` |
| Projet | `cleantask-awx` |
| Environnement d’exécution | même EE numéroté qu’en 3.4 |
| Playbook | `server_setup.yml` |
| Informations d’identification | Machine `ssh-cleantask` **et** Vault `vault-cleantask` |
| Élévation des privilèges | **cochée** |

Puis **Lancer**.

PostgreSQL majeur = **18** (`pg_version` dans `group_vars/all`), identique
au laptop (`postgres:18` / CI). Partir d’une VM **sans** PostgreSQL (sinon
un ancien majeur garderait `:5432` et le 18 irait sur `:5433`). Upgrade
majeur en prod = `pg_upgrade` / dump-restore planifié, pas ce playbook.

### 3.6 Modèle `deploy-staging` (application)

Déploie l’app sur la VM (`deploy.yml` → rôle `app`) : clone Git,
`.env`, Compose (`db.internal` → Postgres hôte), migrate, health.

#### Prérequis

- [ ] `server-setup-staging` déjà **Réussi**
- [ ] Snapshot Proxmox récent (recommandé)
- [ ] Secrets §2.2 + §2.3 dans `inventories/staging/vault.yml` (`!vault` only)
- [ ] Vault staging poussé sur `main` ; **resync inventaire** AWX (Remplacer les variables) pour purger d’anciens vaults importés
- [ ] Deploy key **publique** sur `cleantask/cleantask`
- [ ] Projet AWX **synchronisé** (vert)

#### Créer le modèle

**Modèles → Ajouter → Ajouter un modèle de job**

Même playbook pour la prod plus tard : `deploy-prod` → inventaire `cleantask-prod`.

| Champ | Valeur |
|---|---|
| Nom | `deploy-staging` |
| Organisation | `CleanTask` |
| Type de job | Exécuter |
| Inventaire | `cleantask-staging` |
| Projet | `cleantask-awx` |
| Environnement d’exécution | même EE numéroté qu’en 3.4 (**pas** `latest`) |
| Playbook | `deploy.yml` |
| Informations d’identification | **deux pastilles** : Machine `ssh-cleantask` **et** Vault `vault-cleantask` |
| Options → Élévation des privilèges | **cochée** |
| Variables d’extra | vide (staging = branche `dev` via inventaire) |
| Demander les variables au lancement | **décoché** (sauf pour forcer un tag / sha avec `version: …`) |

**Enregistrer → Lancer.**

#### Vérifier

| Contrôle | Attendu |
|---|---|
| Job AWX | **Réussi** |
| Sur la VM : `curl -fsS -H 'Host: localhost' -H 'X-Forwarded-Proto: https' http://127.0.0.1:8000/core/health/` | HTTP **200** |
| Sur la VM : `cd /opt/cleantask && docker compose -f docker-compose.prod.yml -f docker-compose.override.yml ps` | `web`, `worker`, `redis`, … Up |

Par défaut staging déploie la branche **`dev`** (`inventories/staging/.../env.yml`).  
Prod plus tard : `app_version: main`. Override ponctuel : `-e version=<tag|sha>`.

`GUNICORN_WORKERS` : staging = **3** ; prod CPX42 = **8**  
(voir `cleantask` → `docs/Infrastructure/PERF_TEST_CPX42.md`).

### 3.6 Nginx + Cloudflare Tunnel (`tunnel_setup.yml`)

Après `deploy-staging` (app sur `:8000`) : créer le tunnel + routes DNS, vault, puis job  
`tunnel-setup-staging`.

Procédure complète (hostnames, SSL, vérifs, prod) :  
[`CLOUDFLARE_TUNNEL.md`](CLOUDFLARE_TUNNEL.md).

### 3.7 Site vitrine + documentation (`deploy_sites.yml`)

Après `tunnel_setup` (Nginx sert déjà les vhosts), publier les sites statiques.

**Prérequis :** GitHub n’accepte **qu’une deploy key par clé publique** —
il faut **deux clés** distinctes (`vault_website_git_deploy_key`,
`vault_docs_git_deploy_key`), chacune en lecture seule sur son repo.

| Champ | Valeur |
|---|---|
| Nom | `deploy-sites-staging` |
| Inventaire | `cleantask-staging` |
| Projet | `cleantask-awx` |
| Playbook | `deploy_sites.yml` |
| Identifiants | Machine + Vault |
| Élévation | **cochée** |
| Extra vars (optionnel) | `site: website` / `docs` / `all` (défaut) |

Le rôle build dans un conteneur Node jetable, publie sous
`/var/www/cleantask-website|docs/releases/<sha>` puis bascule `current`.

| URL | Attendu |
|---|---|
| `https://staging.cleantask.co/` | site vitrine |
| `https://docs-staging.cleantask.co/` | documentation |

### 3.8 Seed démo staging (`seed_demo.yml`) — recommandé

**Staging seulement.** Ne pas créer de job prod avec ce playbook.

Après `deploy-staging`, peupler BrightClean / SparkleHome via
`manage.py create_demo_data` (même script que le laptop). Le rôle injecte
`CLEANTASK_ALLOW_DEMO_SEED=1` **uniquement** sur le `compose run` (pas dans
le `.env` durable). Sans `--reset` : orgs déjà présentes = skip ; comptes
existants jamais écrasés.

#### `vault_demo_password` — un mot de passe pour tous les comptes démo

Ce n’est **pas** un mot de passe SSH / Django `SECRET_KEY` / Postgres.
C’est le **mot de passe applicatif unique** partagé par **tous** les
utilisateurs créés par le seed (`CLEANTASK_DEMO_PASSWORD`) :

| Compte | Exemple d’email | Même mot de passe ? |
|---|---|---|
| Platform admin | (email du seed, ex. `jerome.pauwels@cleantask.co`) | oui → `vault_demo_password` |
| Owners / managers / cleaners BrightClean | `*@brightclean.be` | oui |
| Owners / managers / cleaners SparkleHome | `*@sparklehome.be` | oui |

Login = **email** + ce mot de passe (pas le username).  
Choisis une valeur assez forte (staging est public), chiffre-la dans le
vault, et range-la dans le gestionnaire d’équipe. Le job AWX ne la
réaffiche pas dans la sortie (seulement les emails créés).

Changer `vault_demo_password` **après** un seed déjà fait ne met **pas**
à jour les comptes existants — il faudrait un wipe / re-seed (hors
scope AWX, voir fin de §).

#### Prérequis

- [ ] `deploy-staging` déjà **Réussi** (image avec le gate
  `CLEANTASK_ALLOW_DEMO_SEED`)
- [ ] `vault_demo_password` chiffré dans `inventories/staging/vault.yml`

```bash
cd ~/Dev/cleantask-awx
.venv/bin/ansible-vault encrypt_string --stdin-name 'vault_demo_password'
# coller le !vault dans inventories/staging/vault.yml → commit / push → sync AWX
```

#### Créer le modèle

| Champ | Valeur |
|---|---|
| Nom | `seed-demo-staging` |
| Inventaire | `cleantask-staging` |
| Projet | `cleantask-awx` |
| Playbook | `seed_demo.yml` |
| Identifiants | Machine + Vault |
| Élévation | **cochée** |
| Extra vars (optionnel) | `single_org: true` / `no_jobs: true` |
| Demander au lancement | optionnel (pour `single_org` / `no_jobs`) |

**Lancer.** Sortie AWX : liste des emails ; connexion avec
`vault_demo_password`.

| URL | Qui |
|---|---|
| `https://app-staging.cleantask.co/login/` | n’importe quel email démo + `vault_demo_password` |
| `https://app-staging.cleantask.co/platform/` | platform admin seedé (si l’email n’existait pas) |

`--reset` (wipe total) **n’est pas** exposé ici : réservé au laptop
(`DEBUG` + `CLEANTASK_DISPOSABLE_*`). Pour vider staging : snapshot Proxmox
+ recreate DB / redeploy, ou procédure manuelle hors AWX.

### 3.9 Premier login prod (ou staging sans seed) — `createsuperuser`

**`deploy.yml` ne crée aucun utilisateur.** Prod n’a **pas** de seed démo.

```bash
ssh ubuntu@…          # VM cible
cd /opt/cleantask
sudo docker compose \
  -f docker-compose.prod.yml \
  -f docker-compose.override.yml \
  exec web python manage.py createsuperuser
```

| URL | Qui |
|---|---|
| `https://app.cleantask.co/login/` (prod) | compte créé à la main |
| `https://app.cleantask.co/platform/` | platform admin (superuser) |

> Compte applicatif créé à la main : ne pas le mettre dans le vault Ansible
> (sauf `vault_demo_password` pour le seed staging § 3.8).

---

## 4. Secrets

| Secret | Où |
|---|---|
| Postgres, Django, Redis, Outlook, deploy key app… | `inventories/<env>/.../vault.yml` (`!vault`) + credential `vault-cleantask` |
| JSON credentials Cloudflare Tunnel | `vault_cloudflare_tunnel_credentials` (+ `cloudflare_tunnel_id` en inventaire) |
| SSH serveurs | AWX `ssh-cleantask` seulement |
| Clone du dépôt **Ansible** | AWX `git-cleantask-awx` seulement |
| Clone app | `vault_app_git_deploy_key` |
| Clone website / docs | `vault_website_git_deploy_key` / `vault_docs_git_deploy_key` (clés **distinctes**) |
| Mot de passe **partagé** de tous les comptes démo staging | `vault_demo_password` → `CLEANTASK_DEMO_PASSWORD` (`seed-demo-staging` seulement ; pas prod) |

Jamais de secret en clair dans les Variables d’un modèle.

---

## 5. Erreurs courantes

| Symptôme | Action |
|---|---|
| Sync projet rouge | URL SSH / branche / deploy key |
| Playbook introuvable | fichier absent à la racine GitHub |
| Collection manquante | resync projet (`collections/requirements.yml`) |
| Hôte absent | source = `inventories/staging/hosts.yml` + sync |
| Vault undecryptable | pastille Vault sur le modèle |
| SSH `Permission denied` | clé AWX dans `authorized_keys`, user `ubuntu` |
| `Missing sudo password` | NOPASSWD pour `ubuntu` |
| `Failed to JSON parse … b''` | EE `latest` → version numérotée |
| Deploy : assert vault_* | secrets manquants dans le vault de l’env (voir `.example`) |
| Deploy : git clone fail | deploy key absente sur `cleantask/cleantask` |
| Deploy : health timeout | `docker compose … logs web` sur la VM |

---

## 6. Checklist

- [ ] Vault staging en `!vault` seulement, poussé sur `main`
- [ ] Snapshot Proxmox staging avant le 1er job
- [ ] Clé `ssh-cleantask` dans `authorized_keys` de `ubuntu`
- [ ] Org + 3 identifiants + projet sync vert
- [ ] Inventaire sourcé, `ping` + `id` OK
- [ ] Modèle `server-setup-staging` (SSH + Vault) → job **Réussi**
- [ ] Secrets app chiffrés + 3 deploy keys (app / website / docs)
- [ ] Modèle `deploy-staging` → health localhost **200**
- [ ] Modèle `deploy-sites-staging` → staging.cleantask.co + docs-staging OK
- [ ] `vault_demo_password` + modèle `seed-demo-staging` (optionnel mais utile)
