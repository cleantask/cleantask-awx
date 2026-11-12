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
| `cleantask/cleantask` | Le serveur (rôle `app`, plus tard) |
| `cleantask/cleantask-website` / `cleantask-docs` | Le serveur (plus tard) |

---

## 2. Vault (`encrypt_string`)

Chiffrer **chaque valeur**, pas le fichier entier.  
Le mot de passe du **coffre** → identifiant AWX `vault-cleantask` (jamais dans Git).

```bash
cd ~/Dev/cleantask-awx

# Mot de passe du coffre (une fois) → noter + credential vault-cleantask
openssl rand -base64 32

# Secret Postgres, puis chiffrement
SECRET=$(openssl rand -base64 24)
.venv/bin/ansible-vault encrypt_string --name 'vault_pg_password' "$SECRET"
# coller le bloc !vault dans group_vars/all/vault.yml
```

Secret déjà choisi (ex. depuis le `.env` local pour le staging) :

```bash
.venv/bin/ansible-vault encrypt_string --stdin-name 'vault_django_secret_key'
# coller → Entrée → Ctrl-D
```

Deploy key application (multiligne) :

```bash
ssh-keygen -t ed25519 -C cleantask-app-deploy -f cleantask_app_deploy -N ""
# GitHub → cleantask/cleantask → Deploy keys (lecture seule) : coller le .pub
.venv/bin/ansible-vault encrypt_string --stdin-name 'vault_app_git_deploy_key' < cleantask_app_deploy
rm cleantask_app_deploy cleantask_app_deploy.pub
```

Liste complète des clés : `group_vars/all/vault.yml.example`.  
Ne pas faire : `ansible-vault encrypt group_vars/all/vault.yml`.

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

### 3.6 Modèle `deploy-staging` (application)

**Avant** : chiffrer les secrets app dans `vault.yml` (voir `vault.yml.example`),
commit + push, et ajouter une **deploy key lecture seule** sur
`cleantask/cleantask` (clé privée → `vault_app_git_deploy_key`).

**Modèles → Ajouter**

| Champ | Valeur |
|---|---|
| Nom | `deploy-staging` |
| Organisation | `CleanTask` |
| Type de job | Exécuter |
| Inventaire | `cleantask-staging` |
| Projet | `cleantask-awx` |
| Environnement d’exécution | même EE qu’en 3.4 |
| Playbook | `deploy.yml` |
| Informations d’identification | Machine `ssh-cleantask` **et** Vault `vault-cleantask` |
| Élévation des privilèges | **cochée** |
| Variables | optionnel : `version: main` (branche / tag / sha) |

Santé attendue : `http://127.0.0.1:8000/core/health/` → 200  
(nginx + tunnel Cloudflare = étape suivante).

---

## 4. Secrets

| Secret | Où |
|---|---|
| Postgres, Django, Redis, Outlook, deploy key app… | `vault.yml` en `!vault` + credential `vault-cleantask` |
| SSH serveurs | AWX `ssh-cleantask` seulement |
| Clone du dépôt **Ansible** | AWX `git-cleantask-awx` seulement |
| Clone du dépôt **application** | clé dans `vault_app_git_deploy_key` (sur le serveur) |

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
| Deploy : assert vault_* | secrets manquants dans `vault.yml` (voir `.example`) |
| Deploy : git clone fail | deploy key absente sur `cleantask/cleantask` |
| Deploy : health timeout | `docker compose … logs web` sur la VM |

---

## 6. Checklist

- [ ] `vault.yml` en `!vault` seulement, pushé sur `main`
- [ ] Snapshot Proxmox staging avant le 1er job
- [ ] Clé `ssh-cleantask` dans `authorized_keys` de `ubuntu`
- [ ] Org + 3 identifiants + projet sync vert
- [ ] Inventaire sourcé, `ping` + `id` OK
- [ ] Modèle `server-setup-staging` (SSH + Vault) → job **Réussi**
- [ ] Secrets app chiffrés + deploy key sur `cleantask/cleantask`
- [ ] Modèle `deploy-staging` → health localhost **200**
