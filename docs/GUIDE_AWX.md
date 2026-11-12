# Guide AWX — Connecter le dépôt CleanTask

> **Objectif :** connecter le dépôt Ansible `cleantask-awx` à AWX, créer les
> objets nécessaires, et piloter `ct-staging-app-01` (puis la prod) **depuis
> la console AWX**.
>
> **Principe de travail :** modifier les fichiers sur votre poste → commit →
> push → lancer le job dans AWX.
>
> **Ne pas tester avec `ansible-playbook` sur le poste** une fois AWX en
> place : la console est la source d’exécution. Un run local contourne les
> jobs, les logs et les credentials AWX.
>
> Ce guide suppose qu’AWX est déjà installé (machine `ansible`,
> `192.168.10.114`).

## Sommaire

- [1. Circuit Git et AWX](#1-circuit-git-et-awx)
- [2. Les dépôts Git](#2-les-dépôts-git)
- [3. Préparer le dépôt avant le premier push](#3-préparer-le-dépôt-avant-le-premier-push)
- [4. Correspondance fichiers ↔ AWX](#4-correspondance-fichiers--awx)
- [5. Créer les objets dans AWX](#5-créer-les-objets-dans-awx)
- [6. Où sont les secrets](#6-où-sont-les-secrets)
- [7. Opérations courantes](#7-opérations-courantes)
- [8. Vérifier que AWX lit bien Git](#8-vérifier-que-awx-lit-bien-git)
- [9. Erreurs courantes](#9-erreurs-courantes)
- [10. Hors périmètre](#10-hors-périmètre)
- [11. Checklist de mise en service](#11-checklist-de-mise-en-service)

---

## 1. Circuit Git et AWX

### Le dépôt distant est la source de vérité

Le dossier `~/Dev/cleantask-awx/` sur le PC est une **copie de travail**.
AWX **ne lit jamais** ce disque. Il clone uniquement GitHub.

| Ordre | Depuis | Vers | Action |
|---|---|---|---|
| 1 | Vous, sur le poste | GitHub `cleantask/cleantask-awx` | Commit + push |
| 2 | GitHub | AWX | `git clone` / pull à la synchronisation |
| 3 | AWX | `ct-staging-app-01` (puis prod) | SSH + exécution des playbooks |

| Endroit | Rôle |
|---|---|
| Votre PC | Éditer les YAML, commit, push |
| **GitHub** | **Source de vérité : si ce n’est pas poussé, AWX ne l’a pas** |
| Checkout AWX | Copie **temporaire**, recréée à chaque job / sync |
| Serveurs | Résultat : Docker, PostgreSQL, Nginx, application… |

Il n’y a **aucun fichier à téléverser** dans l’interface AWX. Un bouton
**Lancer** signifie : **récupérer Git, puis exécuter le playbook**.

> **Point d’attention**
>
> Si vous changez un rôle uniquement sur le PC et oubliez `git push`, AWX
> rejoue **l’ancienne version**. Les serveurs suivent Git, pas le dossier local.

### Pourquoi la console AWX, pas la ligne de commande

| | `ansible-playbook` local | Job AWX |
|---|---|---|
| Visible dans la console | Non | Oui (historique, qui a lancé, journal) |
| Credentials | Clés du poste | Identifiants AWX |
| Même chemin staging / prod | Facile de se tromper d’inventaire | Inventaire choisi dans le modèle |
| AWX à jour | Non concerné | Oui |

Le geste normal : **modifier → commit → push → Lancer dans AWX**.

---

## 2. Les dépôts Git

AWX clone **uniquement les playbooks**. Le code de l’application est cloné
**par le serveur** (rôle `app`, plus tard), avec une deploy key dédiée.

| Dépôt | Qui le clone | Contenu |
|---|---|---|
| **`cleantask/cleantask-awx`** | **AWX** | Playbooks, rôles, inventaires |
| **`cleantask/cleantask`** | Le serveur (rôle `app`) | Django, Docker |
| **`cleantask/cleantask-website`** | Le serveur (rôle `static_sites`) | Site vitrine |
| **`cleantask/cleantask-docs`** | Le serveur (rôle `static_sites`) | Documentation |

AWX n’a **pas** besoin d’accès aux dépôts application / site / docs pour
lancer `server_setup.yml`.

---

## 3. Préparer le dépôt avant le premier push

### Arborescence attendue à la racine du dépôt GitHub

```text
ansible.cfg
collections/requirements.yml
server_setup.yml
deploy.yml                    ← plus tard
README.md
docs/
  PROPOSITION_ARCHITECTURE_ANSIBLE.md
  GUIDE_AWX.md
inventories/
  staging/
    hosts.yml
    group_vars/all/env.yml
  prod/                       ← plus tard
group_vars/all/
  main.yml
  vault.yml                   ← valeurs en !vault (encrypt_string), fichier versionné
  vault.yml.example
roles/
  preflight/ common/ network/ docker/ postgresql/
  app/ nginx/ cloudflared/ …  ← au fur et à mesure
```

### Avant le premier push (obligatoire)

1. **VM staging** : revenir au snapshot Proxmox si un run local a déjà
   modifié la machine (pour repartir propre via AWX).
2. **Chiffrer chaque secret** avec `encrypt_string` (voir ci-dessous) — **pas**
   `ansible-vault encrypt` sur tout le fichier.
3. Commit + push vers `git@github.com:cleantask/cleantask-awx.git`
   (`vault.yml` versionné, mais **uniquement** avec des blocs `!vault`).

### Chiffrer les secrets (`encrypt_string`)

On ne chiffre **pas** le fichier entier. On chiffre **chaque valeur** ; le
YAML reste lisible (noms des clés visibles), les secrets sont des blocs
`!vault`. Le **mot de passe du coffre** (celui que `encrypt_string` demande)
est le même pour toutes les valeurs ; il va dans l’identifiant AWX
`vault-cleantask`, jamais dans Git.

```bash
cd ~/Dev/cleantask-awx

# 1) Mot de passe du coffre Ansible (une seule fois) — à garder dans le
#    gestionnaire de mots de passe + credential AWX vault-cleantask.
#    Ce n’est PAS le mot de passe Postgres : c’est la clé qui chiffre les secrets.
openssl rand -base64 32
# → noter le résultat comme « mot de passe vault CleanTask »

# 2) Mot de passe PostgreSQL de l’application (ou tout autre secret)
SECRET=$(openssl rand -base64 24)
echo "$SECRET"   # le noter aussi si besoin (ex. debug psql) ; il sera dans vault.yml chiffré

# 3) Chiffrer cette valeur (Ansible demande le mot de passe du coffre du §1)
.venv/bin/ansible-vault encrypt_string --name 'vault_pg_password' "$SECRET"
```

La commande affiche un bloc à coller dans `group_vars/all/vault.yml` :

```yaml
vault_pg_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          6638…lignes…
```

Pour un secret déjà choisi (collé sans le laisser dans l’historique) :

```bash
.venv/bin/ansible-vault encrypt_string --stdin-name 'vault_pg_password'
# coller le secret, Entrée, puis Ctrl-D
```

Autres secrets plus tard : même coffre, autre `--name` :

```bash
.venv/bin/ansible-vault encrypt_string --name 'vault_django_secret_key' "$(openssl rand -base64 48)"
```

> **À ne pas faire :** `ansible-vault encrypt group_vars/all/vault.yml`
> (fichier entièrement opaque). On utilise **uniquement** `encrypt_string`.

### Utilisateur SSH sur le serveur

| Environnement | Hôte | `ansible_user` | Sudo |
|---|---|---|---|
| Staging | `ct-staging-app-01` (`192.168.10.115`) | `ubuntu` | NOPASSWD déjà en place |
| Prod | `ct-prod-app-01` | à définir | idem |

La clé **publique** d’AWX (identifiant Machine) doit être dans
`/home/ubuntu/.ssh/authorized_keys` sur la VM. Votre clé personnelle
(`~/.ssh/id_ed25519`) peut y rester pour le dépannage SSH hors AWX.

---

## 4. Correspondance fichiers ↔ AWX

### 4.1 Projet Git

| Fichier | Dans AWX |
|---|---|
| Tout le dépôt | **Projets → `cleantask-awx`**, SCM Git |
| `ansible.cfg` | Lu depuis le checkout ; l’inventaire par défaut est **ignoré** (AWX fournit le sien) |
| `collections/requirements.yml` | Installé **à la synchronisation du projet** |
| `README.md` / `docs/` | Documentation ; AWX ne les exécute pas |

### 4.2 Inventaire sourcé depuis le projet

| Fichier | Dans AWX |
|---|---|
| `inventories/staging/hosts.yml` | **Inventaires → `cleantask-staging` → Sources** → fichier `inventories/staging/hosts.yml` |
| `inventories/staging/group_vars/all/env.yml` | Chargé automatiquement (à côté de `hosts.yml` dans le clone) |
| `group_vars/all/main.yml` | Chargé automatiquement (à côté des playbooks) |
| `group_vars/all/vault.yml` | Chargé automatiquement ; **déchiffré** via l’identifiant Vault du modèle |

> **Ne créez pas les hôtes à la main** dans Hôtes. Sinon `group_vars` ne
> suivent plus Git.

### 4.3 Playbooks → modèles de job

| Playbook | Modèle AWX | Quand |
|---|---|---|
| `server_setup.yml` | `server-setup` | Installation / reconvergence du serveur (sans déployer l’app) |
| `deploy.yml` | `deploy` | Plus tard : nouvelle version de l’application |
| `deploy_sites.yml` | `deploy-sites` | Plus tard : site vitrine / doc |
| `backup_now.yml` / `restore_db.yml` | idem | Plus tard |

### 4.4 Rôles

**Aucun objet AWX par rôle.** Ils voyagent dans le clone Git et sont appelés
par les playbooks.

| Rôle (aujourd’hui) | Déclenché par |
|---|---|
| `preflight`, `common`, `network`, `docker`, `postgresql` | `server-setup` |

---

## 5. Créer les objets dans AWX

Respectez l’ordre. Tous les objets dans **la même organisation**.

| Ordre | Objet | Nom proposé |
|---|---|---|
| 5.1 | Organisation | `CleanTask` |
| 5.2 | Identifiants | `git-cleantask-awx`, `ssh-cleantask`, `vault-cleantask` |
| 5.3 | Projet | `cleantask-awx` |
| 5.4 | Inventaire | `cleantask-staging` (puis `cleantask-prod`) |
| 5.5 | Modèle de job | `server-setup` (puis les autres) |

### 5.1 Organisation

**Accès → Organisations → Ajouter**

| Champ | Valeur |
|---|---|
| Nom | `CleanTask` |

Utilisez `CleanTask` dans le champ Organisation de chaque objet suivant.

### 5.2 Identifiants

**Informations d’identification → Ajouter**, trois fois.

| Nom | Type | Rôle |
|---|---|---|
| `git-cleantask-awx` | **Contrôle de source** | Deploy key **lecture seule** sur `cleantask/cleantask-awx` |
| `ssh-cleantask` | **Machine** | SSH vers les serveurs en `ubuntu` + `sudo` |
| `vault-cleantask` | **Archivage sécurisé** | Mot de passe de `ansible-vault encrypt` |

#### Deux connexions SSH distinctes

| Connexion | Clé | Identifiant | Usage |
|---|---|---|---|
| AWX → GitHub | `git-cleantask-awx_ed25519` | `git-cleantask-awx` | Cloner le dépôt Ansible |
| AWX → serveurs | `awx-cleantask_ed25519` | `ssh-cleantask` | Se connecter en `ubuntu` |

Votre clé GitHub personnelle **ne va jamais dans AWX**.

#### Identifiant `git-cleantask-awx`

Sur le poste :

```bash
ssh-keygen -t ed25519 -C git-cleantask-awx -f git-cleantask-awx_ed25519 -N ""
cat git-cleantask-awx_ed25519.pub
```

GitHub → dépôt **`cleantask/cleantask-awx`** → **Settings → Deploy keys →
Add deploy key** : titre `awx`, coller le `.pub`, **Allow write access
décoché**.

Dans AWX :

| Champ | Valeur |
|---|---|
| Nom | `git-cleantask-awx` |
| Organisation | `CleanTask` |
| Type | Contrôle de source |
| Nom d’utilisateur | `git` |
| Clé privée SCM | Contenu complet de `git-cleantask-awx_ed25519` |
| Phrase de passe | Vide |

Puis `rm git-cleantask-awx_ed25519*` : AWX est le seul dépositaire.

#### Identifiant `ssh-cleantask`

Prérequis sur `ct-staging-app-01` :

- utilisateur `ubuntu` ;
- clé publique d’AWX dans `/home/ubuntu/.ssh/authorized_keys` ;
- `sudo` sans mot de passe (déjà le cas sur la VM actuelle).

```bash
ssh-keygen -t ed25519 -C awx-cleantask -f awx-cleantask_ed25519 -N ""
# coller awx-cleantask_ed25519.pub dans authorized_keys sur la VM
```

| Champ | Valeur |
|---|---|
| Nom | `ssh-cleantask` |
| Organisation | `CleanTask` |
| Type | Machine |
| Nom d’utilisateur | `ubuntu` (= `ansible_user` de l’inventaire) |
| Clé privée SSH | Contenu complet de `awx-cleantask_ed25519` |
| Méthode d’escalade | `sudo` |
| Mot de passe d’escalade | Vide (NOPASSWD) |

Après enregistrement, supprimer la clé privée du poste.

#### Identifiant `vault-cleantask`

| Champ | Valeur |
|---|---|
| Nom | `vault-cleantask` |
| Organisation | `CleanTask` |
| Type | Archivage sécurisé |
| Mot de passe | Celui saisi à chaque `encrypt_string` (mot de passe du **coffre**, §3) |
| Me le demander au lancement | **Décoché** |

### 5.3 Projet Git

**Projets → Ajouter**

| Champ | Valeur |
|---|---|
| Nom | `cleantask-awx` |
| Organisation | `CleanTask` |
| Type de contrôle de source | Git |
| URL | `git@github.com:cleantask/cleantask-awx.git` (SSH) |
| Branche | `main` |
| Identifiant | `git-cleantask-awx` |
| Mise à jour à la révision avant le lancement | **Oui** |
| Effacer le cache | Oui (clone propre à chaque job) |

**Enregistrer → Synchroniser.** Vert = clone OK. À cette sync, AWX installe
les collections de `collections/requirements.yml`.

### 5.4 Inventaire `cleantask-staging`

1. **Inventaires → Ajouter → Ajouter un inventaire** : nom `cleantask-staging`,
   organisation `CleanTask`.
2. **Sources → Ajouter** :

| Champ | Valeur |
|---|---|
| Nom | `depuis-git` |
| Source | Provenance d’un projet |
| Environnement d’exécution | **AWX EE version numérotée** (ex. `24.6.1`), **pas** `latest` |
| Projet | `cleantask-awx` |
| Fichier d’inventaire | Taper `inventories/staging/hosts.yml` puis Entrée (ne pas choisir « / (project root) ») |
| Options | Cocher **Remplacer**, **Remplacer les variables**, **Mettre à jour au lancement** |

3. **Synchroniser** la source → État **Réussi**.
4. Vérifier : **Hôtes** contient `ct-staging-app-01` ; **Groupes** contient
   `app_servers` (et `db_servers`).

#### Tester SSH sans playbook

Inventaire → Hôtes → cocher `ct-staging-app-01` → **Exécuter une commande** :

| Module | Élévation | Attendu |
|---|---|---|
| `ping` | Non | `pong` |
| `command` / `id` | **Oui** | `uid=0(root)` |

### 5.5 Modèle de job `server-setup`

**Modèles → Ajouter → Ajouter un modèle de job**

| Champ | Valeur |
|---|---|
| Nom | `server-setup` |
| Description | Convergence du serveur (preflight, common, network, docker, postgresql) — pas l’application |
| Organisation | `CleanTask` |
| Type de job | Exécuter |
| Inventaire | `cleantask-staging` |
| Projet | `cleantask-awx` |
| Environnement d’exécution | Même EE numéroté qu’au §5.4 |
| Playbook | `server_setup.yml` |
| Informations d’identification | **Deux pastilles** : Machine `ssh-cleantask` **et** Vault `vault-cleantask` |
| Options → Élévation des privilèges | **Cochée** |
| Limite | Vide ; « Me le demander au lancement » au choix |
| Variables | `---` ; demander au lancement **décoché** |

Équivalence (à ne plus utiliser pour les tests courants) :

```text
ansible-playbook server_setup.yml --ask-vault-pass
        →  modèle server-setup → Lancer
```

---

## 6. Où sont les secrets

| Secret | Emplacement | Usage AWX |
|---|---|---|
| Mot de passe PostgreSQL (et plus tard le reste) | `group_vars/all/vault.yml` : chaque valeur en `!vault` (`encrypt_string`) | Identifiant `vault-cleantask` (mot de passe du coffre) |
| Clé SSH vers les serveurs | **AWX seulement** (`ssh-cleantask`) | Jamais dans Git |
| Deploy key du dépôt Ansible | **AWX seulement** (`git-cleantask-awx`) | — |
| `.env` de l’application (staging ≈ `.env` de dev) | Plus tard : généré sur le serveur / credential AWX | Jamais en clair dans Git |
| Identifiants tunnel Cloudflare | Plus tard : vault ou fichiers chiffrés | — |

> Ne collez **jamais** le vault en clair dans les **Variables** d’un modèle.

---

## 7. Opérations courantes

| Besoin | Dans Git | Puis dans AWX |
|---|---|---|
| **Première install staging** | `vault.yml` chiffré, inventaire OK, push | Sync projet + inventaire → **`server-setup` → Lancer** |
| **Rejouer après un changement de rôle** | Commit + push | **`server-setup` → Lancer** |
| **Déployer l’application** | Plus tard (`deploy.yml`) | Modèle `deploy` |
| **Prod** | Inventaire `prod` | Même modèles, inventaire `cleantask-prod` |

---

## 8. Vérifier que AWX lit bien Git

1. Modifiez une ligne visible (commentaire dans `group_vars/all/main.yml`)
   **sans push**.
2. Relancez `server-setup` : l’ancienne version s’applique.
3. Push, puis relancez : la nouvelle version s’applique.

---

## 9. Erreurs courantes

| Symptôme | Cause / action |
|---|---|
| Sync projet en rouge | URL SSH, branche, ou deploy key `git-cleantask-awx` |
| Playbook introuvable | Le playbook n’est pas à la racine du dépôt GitHub (mauvais layout poussé) |
| `community.postgresql` manquant | `collections/requirements.yml` absent ou projet non resynchronisé |
| Hôte absent de l’inventaire | Source pas sur `inventories/staging/hosts.yml`, ou sync inventaire oubliée |
| Vault undecryptable | Pastille `Vault: vault-cleantask` absente du modèle |
| `Permission denied` SSH | Clé AWX absente de `authorized_keys`, ou user ≠ `ubuntu` |
| `Missing sudo password` | sudoers sans NOPASSWD pour `ubuntu` |
| `Failed to JSON parse … b''` | EE `latest` → choisir une version numérotée |
| Rien dans la console alors que la VM a changé | Un `ansible-playbook` a été lancé **hors AWX** |

---

## 10. Hors périmètre

- Installation d’AWX lui-même (déjà sur `ansible` / `192.168.10.114`).
- Création du tunnel Cloudflare (plus tard, rôle `cloudflared`).
- Haute disponibilité Patroni / etcd (plus tard).

---

## 11. Checklist de mise en service

### Dépôt Git

- [ ] Racine GitHub = contenu de `cleantask-awx/` (playbooks à la racine).
- [ ] `vault.yml` commité avec des blocs `!vault` uniquement ; aucune valeur en clair.
- [ ] `collections/requirements.yml` présent.
- [ ] Push sur `cleantask/cleantask-awx` (`main`).

### Serveur staging

- [ ] Snapshot Proxmox récent **avant** le premier job.
- [ ] Clé publique `ssh-cleantask` dans `authorized_keys` de `ubuntu`.
- [ ] `sudo -n true` OK pour `ubuntu`.

### AWX

- [ ] Organisation `CleanTask`.
- [ ] Identifiants `git-cleantask-awx`, `ssh-cleantask`, `vault-cleantask`.
- [ ] Projet `cleantask-awx` sync **vert**.
- [ ] Inventaire `cleantask-staging` sourcé, hôte `ct-staging-app-01` visible.
- [ ] `ping` + `id` (avec élévation) verts.
- [ ] Modèle `server-setup` créé avec SSH **et** Vault.
- [ ] Premier job **`server-setup`** lancé depuis la console → **Réussi**.
