# CleanTask — Proposition d'infrastructure Ansible / AWX

Document de conception, avant implémentation. Il décrit l'architecture visée,
l'organisation du dépôt Ansible, les rôles et playbooks prévus, et l'ordre de
réalisation.

## 1. Décisions prises

| Sujet | Décision |
|---|---|
| Architecture de départ | **Un seul serveur** : application + base de données |
| Application | Docker Compose (`docker-compose.prod.yml` du dépôt `cleantask`) |
| PostgreSQL | **Installé directement sur le système** (paquets PGDG), pas dans Docker |
| Accès des conteneurs à la base | Nom `db.internal` → `host-gateway` |
| Réseau privé (prod) | **Hetzner Cloud Network**, dès le premier serveur |
| Construction de l'image | **Sur le serveur** (comme `deploy.sh` aujourd'hui) |
| Sauvegardes | **Hetzner Storage Box** |
| Médias | Disque local pour l'instant ; Hetzner Object Storage plus tard (demande une migration du code) |
| AWX | Sur la machine `ansible` (`192.168.10.114`) ; **tous les tests et déploiements passent par la console AWX** (pas `ansible-playbook` local) — voir [`GUIDE_AWX.md`](GUIDE_AWX.md) |
| Exposition HTTPS | **Tunnel Cloudflare en staging et en production** (§ 7) ; pas de Let's Encrypt ; DNS `cleantask.co` déjà sur Cloudflare |
| Site vitrine et documentation | **Sur le même serveur** que l'application, servis par Nginx en fichiers statiques (§ 7) |
| Secrets du staging | **Le même `.env` que le poste de développement** (§ 8.1) |
| Nommage | `ct-<env>-<rôle>-<nn>` (voir `docs/Infrastructure/SERVER_NAMING.md` du dépôt `cleantask`) |

## 2. Architecture

### 2.1 Aujourd'hui

| Machine | Environnement | Hébergement | Rôles |
|---|---|---|---|
| `ct-staging-app-01` | staging | VM Proxmox à la maison, `192.168.10.115`, utilisateur `ubuntu` | `app`, `db` |
| `ct-prod-app-01` | prod | Hetzner Cloud CPX42 + Cloud Network | `app`, `db` |

Sur chaque machine :

- Docker : `web`, `worker`, `worker-documents`, `beat`, `redis`, `redis-cache` ;
- PostgreSQL sur le système, joignable uniquement depuis `localhost` et le sous-réseau Docker ;
- `cloudflared` (tunnel Cloudflare) → Nginx en HTTP local, qui sert
  aussi le **site vitrine** (`cleantask-website`) et la **documentation**
  (`cleantask-docs`) en fichiers statiques ;
- sauvegardes vers la Storage Box.

| Site | Dépôt | Technologie | Prod | Staging |
|---|---|---|---|---|
| Application | `cleantask` | Django (Docker) | `app.cleantask.co` | `app-staging.cleantask.co` |
| Site vitrine | `cleantask-website` | HTML + Tailwind (statique) | `cleantask.co` (+ `www` → redirection) | `staging.cleantask.co` |
| Documentation | `cleantask-docs` | Docusaurus (statique) | `docs.cleantask.co` | `docs-staging.cleantask.co` |

### 2.2 Plus tard : haute disponibilité

- 2 serveurs qui font chacun **application + base** (`ct-prod-app-01`, `ct-prod-app-02`) ;
- PostgreSQL en cluster **Patroni** ;
- **3 membres etcd** : un sur chaque serveur + un petit serveur dédié (`ct-prod-etcd-01`) ;
- tout sur le même Cloud Network privé.

Prérequis de cette évolution, hors Ansible : médias sur Object Storage,
répartition de charge, Celery beat unique, Redis partagé (ou Sentinel),
`db.internal` pointé vers HAProxy ou une IP flottante.

Répartition de charge : un `cloudflared` sur chaque serveur d'application
(même tunnel) apporte la **tolérance aux pannes**, pas la répartition (Cloudflare
envoie le trafic au connecteur le plus proche). Pour répartir réellement :
Cloudflare Load Balancing (payant, environ 5 $/mois) ou load balancer Hetzner.

### 2.3 Ce qu'on prépare dès maintenant pour cette évolution

| Préparé maintenant | Implémenté plus tard |
|---|---|
| Groupes d'inventaire `app_servers`, `db_servers`, `dcs_servers` (vide) | Rôles `etcd`, `patroni`, `haproxy` |
| Rôles séparés : `postgresql` ne connaît pas l'application | Playbooks de bascule, redémarrage, réplica |
| `db_host`, interface privée, `postgres_listen_addresses`, réseaux `pg_hba` en variables | Répartition de charge, Redis Sentinel |
| `celery_beat_host` : beat sur un seul hôte | Beat qui suit le primaire Patroni |
| `common` découpé (`base.yml` réutilisable sur le serveur etcd) | `arbiter_setup.yml` |

## 3. Principes d'exploitation

- **Structure** : `inventories/<env>/` pour ce qui change, `group_vars/all/`
  partagé. Une clé n'est jamais aux deux endroits (celle de `group_vars/all`
  gagnerait en silence).
- **`A_RENSEIGNER`** dans l'inventaire : tout playbook s'arrête au preflight.
- **Rôle `preflight`** en tête de chaque playbook.
- `any_errors_fatal`, `force_handlers: true`, versions figées.
- **`common`** : fail2ban, durcissement SSH en fichier séparé,
  `unattended-upgrades` sans PostgreSQL et sans redémarrage automatique,
  fuseau `Europe/Brussels`.
- **`network`** : nftables (sans `flush ruleset`, pour ne pas casser Docker),
  ports privés listés dans `private_tcp_ports`, Cloud Network en prod.
- **`postgresql`** : mémoire calculée à partir de la RAM (surcharge staging).
- **`pgbackrest`** : restauration à la minute vers la Storage Box (SFTP port 23,
  sous-compte, clés d'hôte épinglées).
- **`pgdump`** chiffré avec **`age`** (clé privée jamais sur le serveur),
  horodatage de dernière réussite, refus si le disque est trop plein.
- **`nginx`** : durcissement partagé, `return 444` par défaut, page de
  maintenance par fichier drapeau.
- **Application** : une seule app multi-organisations dans Docker Compose
  (pas une instance OS par client) ; un seul jeu de secrets (vault classique).
- **Opérations destructrices** avec confirmation explicite (`-e confirm_restore=<nom>`).
- **Restauration** : vérification de la sauvegarde, maintenance, puis écrasement.
- **AWX** : un inventaire par environnement, modèles qui demandent
  l'inventaire, credentials séparés, collections installées à la synchronisation ;
  tous les runs passent par la console (voir [`GUIDE_AWX.md`](GUIDE_AWX.md)).
- **Documentation** : README avec tableaux rôles / playbooks, guide AWX.

## 4. Organisation du dépôt

Racine du dépôt : `~/Dev/cleantask-awx/`.

```
cleantask-awx/
├── ansible.cfg
├── collections/requirements.yml
├── README.md
├── .gitignore              # .venv/, vault en clair, *.retry
├── docs/
│   ├── PROPOSITION_ARCHITECTURE_ANSIBLE.md
│   ├── GUIDE_INSTALLATION.md
│   └── GUIDE_AWX.md
├── inventories/
│   ├── staging/
│   │   ├── hosts.yml
│   │   ├── vault.yml              # !vault — hors group_vars (load_vault)
│   │   ├── vault.yml.example
│   │   └── group_vars/all/env.yml # non-secrets seulement
│   └── prod/
│       ├── hosts.yml
│       ├── vault.yml
│       ├── vault.yml.example
│       └── group_vars/all/env.yml
├── group_vars/all/
│   └── main.yml            # fuseau, PostgreSQL… (pas de secrets)
├── roles/
│   ├── preflight/
│   ├── common/
│   ├── network/
│   ├── docker/
│   ├── postgresql/
│   ├── pgbackrest/
│   ├── pgdump/
│   ├── media_backup/       # restic des médias vers la Storage Box
│   ├── app/                # compose, .env, réseau Docker, déploiement
│   ├── static_sites/       # site vitrine + documentation
│   ├── nginx/
│   ├── cloudflared/        # staging
│   └── exporters/          # plus tard
├── server_setup.yml
├── deploy.yml
├── deploy_sites.yml
├── rollback.yml
├── maintenance.yml
├── backup_now.yml
├── restore_db.yml
└── restore_test.yml
```

Inventaire de départ (staging) :

```yaml
all:
  children:
    app_servers:
      hosts:
        ct-staging-app-01:
          ansible_host: 192.168.10.115
    db_servers:
      hosts:
        ct-staging-app-01:
    dcs_servers:
      hosts: {}
  vars:
    ansible_user: ubuntu
```

## 5. Rôles

| Rôle | Ce qu'il fait |
|---|---|
| `preflight` | Variables obligatoires renseignées (pas de `A_RENSEIGNER`), adresse distincte par machine, espace disque, mémoire. Plus tard : rôle Patroni. |
| `common` | `base.yml` : paquets de base, fail2ban, durcissement SSH, `unattended-upgrades`, fuseau, synchronisation de l'heure. `main.yml` : ce qui est propre aux serveurs d'application. |
| `network` | Pare-feu nftables et réseau privé. **Ne vide que sa propre table** : un `flush ruleset` effacerait les règles de Docker et couperait le réseau des conteneurs. |
| `docker` | Docker Engine et le plugin Compose depuis le dépôt officiel, versions figées, rotation des logs. |
| `postgresql` | PostgreSQL PGDG, réglages calculés à partir de la RAM, base et rôle `cleantask`, `listen_addresses` = `localhost` + passerelle Docker, `pg_hba.conf` limité au sous-réseau Docker en `scram-sha-256`. |
| `pgbackrest` | Sauvegarde complète hebdomadaire, incrémentales quotidiennes, journaux en continu vers la Storage Box. |
| `pgdump` | `pg_dump` quotidien chiffré `age`, envoyé sur la Storage Box puis supprimé du serveur. |
| `media_backup` | Sauvegarde restic chiffrée des médias vers la Storage Box, rétention, vérification périodique. |
| `app` | Clone du dépôt (clé de déploiement), `.env` généré depuis le vault, **sous-réseau Docker fixe** (pour `pg_hba`), `extra_hosts: db.internal:host-gateway`, puis la logique de `deploy.sh` (voir § 6.2). |
| `static_sites` | Clone `cleantask-website` et `cleantask-docs`, construit chaque site **dans un conteneur Node jetable** (pas de Node sur le système), publie dans `/var/www/<site>/releases/<commit>` puis bascule le lien `current` (publication atomique, retour arrière immédiat). Les URL de l'environnement sont injectées au build (§ 7.4). |
| `nginx` | Écoute en HTTP sur `127.0.0.1` uniquement (seul `cloudflared` lui parle). Reverse proxy de l'application vers `127.0.0.1:8000`, vhosts statiques du site vitrine et de la doc, **IP réelle du visiteur depuis `CF-Connecting-IP`** (§ 7.3), durcissement, `444` par défaut, page de maintenance. |
| `cloudflared` | Tunnel Cloudflare vers Nginx, en staging et en production, un tunnel par serveur. Version figée. Identifiants du tunnel **chiffrés dans le vault**. |
| `exporters` | node_exporter et postgres_exporter sur l'IP privée (plus tard). |

## 6. Playbooks

### 6.1 Liste

| Playbook | Quand | Ce qu'il fait |
|---|---|---|
| `server_setup.yml` | Installation, changement de configuration, reconstruction | Le serveur, sans l'application : preflight, common, network, docker, postgresql, sauvegardes, nginx / tunnel. |
| `deploy.yml` | Nouvelle version | Voir § 6.2. `-e version=<branche, tag ou commit>`. |
| `deploy_sites.yml` | Nouvelle version du site vitrine ou de la doc | `-e site=website|docs|all`, `-e version=…`. Build, publication atomique, contrôle HTTP. Indépendant du déploiement de l'application. |
| `rollback.yml` | Déploiement raté | Relance l'image du commit précédent. Ne défait pas les migrations : une migration irréversible se traite à part. |
| `maintenance.yml` | Intervention | `-e maintenance=on|off` : fichier drapeau lu par Nginx, sans rechargement. |
| `backup_now.yml` | Avant une opération risquée | Sauvegarde pgBackRest + `pg_dump` + médias, et vérifie qu'ils sont arrivés. |
| `restore_db.yml` | Données endommagées | `-e confirm_restore=cleantask`, date optionnelle. Vérifie la sauvegarde, maintenance, arrêt des workers, restauration, migrations, remise en service. |
| `restore_test.yml` | Régulièrement | Restaure la dernière sauvegarde dans une base jetable et vérifie qu'elle est lisible, sans toucher la production. |

### 6.2 Déploiement (`deploy.yml`, rôle `app`)

Reprend `deploy.sh` en l'enrichissant :

1. preflight ;
2. si le marqueur `.deployed` contient déjà le commit demandé : rien à faire ;
3. `git fetch` + checkout du commit ;
4. construction de l'image, **étiquetée par commit** (`cleantask:<sha>`) ;
5. `migrate`, `check_encryption_keys`, `check_outlook_client_secret` ;
6. `docker compose up -d` avec la nouvelle étiquette ;
7. contrôle de santé (`/core/health/`) ;
8. écriture du marqueur, et conservation des N dernières images pour `rollback.yml`.

## 7. Noms de domaine et exposition HTTPS

### 7.1 Noms DNS

| Site | Prod | Staging |
|---|---|---|
| Application | `app.cleantask.co` | `app-staging.cleantask.co` |
| Site vitrine | `cleantask.co`, `www.cleantask.co` → redirection | `staging.cleantask.co` |
| Documentation | `docs.cleantask.co` | `docs-staging.cleantask.co` |

**Un seul niveau sous `cleantask.co`** (`app-staging.cleantask.co`, et non
`app.staging.cleantask.co`) : le certificat gratuit de Cloudflare (Universal
SSL) ne couvre que `cleantask.co` et `*.cleantask.co`. Un deuxième niveau
demanderait un certificat payant (Advanced Certificate Manager).

Les noms `*.cleantask.lovelyhome.io` restent ceux du poste de développement.

Garde-fous du staging :

- en-tête `X-Robots-Tag: noindex, nofollow` sur les trois sites, pour ne pas
  être indexé par les moteurs de recherche ;
- **Cloudflare Access** pour réserver les trois sites à l'équipe, en laissant
  passer `/cleaner/api/` (app mobile) ;
- **e-mails comme en dev / prod** : même logique que le `.env` de
  développement (SMTP / Gmail / Outlook selon la config), pas de boîte de
  capture obligatoire ;
- **uniquement des données de démo**, jamais de données réelles de clients
  (évite d'écrire à de vrais destinataires par accident).

### 7.2 Tunnel Cloudflare, en staging et en production

Un tunnel par serveur, avec trois règles d'entrée (application, site vitrine,
documentation) vers Nginx. Gratuit (plan gratuit Cloudflare, sans limite de
tunnels ni de trafic).

- Cloudflare présente au navigateur un certificat géré et renouvelé par lui :
  **pas de Let's Encrypt, nulle part**.
- `cloudflared` ouvre une connexion **sortante** : **aucun port entrant** ouvert
  (ni 80 ni 443), ni sur le serveur Hetzner, ni sur la box ou OPNsense pour le
  staging. L'IP du serveur n'apparaît pas dans le DNS.
- `cloudflared` → Nginx en HTTP sur `127.0.0.1`.
- Protection anti-DDoS de Cloudflare incluse.
- Les noms sont des enregistrements CNAME vers le tunnel, dans la zone
  `cleantask.co` déjà gérée par Cloudflare.
- Mêmes playbooks et même rôle pour les deux environnements.

| Alternative écartée | Pourquoi |
|---|---|
| DNS seul + Let's Encrypt | Ports 80/443 ouverts à Internet, IP publique, certificats à renouveler ; seul avantage : pas de dépendance à Cloudflare |
| Proxy Cloudflare + certificat sur le serveur | Même dépendance à Cloudflare que le tunnel, mais un port ouvert et un certificat à gérer |
| OPNsense + ACME (staging) | Ouvre le réseau domestique, publie l'IP de la maison |

Contraintes acceptées :

- **si Cloudflare est indisponible, les sites le sont aussi** ;
- limite de 100 Mo par envoi sur le plan gratuit, sans impact pour les photos et
  documents ;
- WebSockets : pris en charge par le tunnel.

### 7.3 IP réelle des visiteurs

Avec le tunnel, toutes les requêtes arrivent depuis `cloudflared`, en local.
Sans réglage, **django-axes** verrait la même IP pour tout le monde.

**Côté Django (fait dans le dépôt `cleantask`) :**

- `apps/core/client_ip.py` : `get_client_ip(request)` ;
- `TRUST_PROXY_HEADERS` : `False` en local (défaut), `True` en production
  (`config/settings/production.py`) ;
- django-axes utilise `AXES_CLIENT_IP_CALLABLE = apps.core.client_ip.get_client_ip` ;
- Turnstile (inscription) et les journaux d'impersonation utilisent le même helper ;
- les en-têtes ne sont lus que si le pair TCP est `127.0.0.1` / `::1` (Nginx) ;
- tests : `apps/core/tests/test_client_ip.py`.

**Côté Nginx (à faire avec le rôle Ansible) :**

```
real_ip_header CF-Connecting-IP;
set_real_ip_from 127.0.0.1;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

### 7.4 URL propres à chaque environnement

Chaque site doit pointer vers les autres sites **du même environnement**
(le site vitrine de staging vers l'application de staging, etc.).

| Site | Comment l'URL est réglée aujourd'hui | Ce qu'il faut |
|---|---|---|
| Application | `SITE_URL`, `DOCS_URL` dans `.env` ; `CSRF_TRUSTED_ORIGINS` / `ALLOWED_HOSTS` dans les settings | Générés depuis l'inventaire (`app_domain`, `docs_domain`, `website_domain`) |
| Documentation | Variables `DOCS_URL` et `APP_URL` au build Docusaurus (`docusaurus.config.ts`) | Passées par le rôle `static_sites`, rien à changer dans le dépôt |
| Site vitrine | URL de l'application **codée en dur** dans les pages HTML (`https://app.cleantask.lovelyhome.io`, et parfois `https://app.cleantask.co`) | **Modification du dépôt `cleantask-website`** : un marqueur (par exemple `__APP_URL__`) remplacé au build par le rôle `static_sites` |

Variables d'inventaire (`inventories/<env>/group_vars/all/env.yml`) :

```yaml
app_domain: app-staging.cleantask.co
website_domain: staging.cleantask.co
docs_domain: docs-staging.cleantask.co
```

## 8. Secrets

- `inventories/<env>/vault.yml` : **chaque secret** chiffré
  avec `ansible-vault encrypt_string` (blocs `!vault`), un fichier par
  environnement — détail dans [`GUIDE_AWX.md`](GUIDE_AWX.md) §2.
- Contenu : `SECRET_KEY` et anciennes clés, clés de chiffrement des champs,
  mot de passe PostgreSQL, OAuth Gmail / Outlook, Stripe, Sentry, accès Storage
  Box, mots de passe restic, identifiants du tunnel.
- **Les clés de chiffrement ne sont jamais régénérées** : les refresh tokens
  Gmail / Outlook en base sont chiffrés avec elles. Le vault est leur seule
  source.
- Le mot de passe du vault et la clé privée `age` sont conservés **hors des
  serveurs** (gestionnaire de mots de passe + copie hors ligne) et dans les
  credentials AWX.
- Aucun secret dans les logs : `no_log: true` sur les tâches qui les manipulent.

### 8.1 Staging = le `.env` de développement

Pour le staging, la source des secrets est **le `.env` du poste de
développement** (`~/Dev/cleantask/.env`) : mêmes `SECRET_KEY`, mêmes clés de
chiffrement, mêmes clients OAuth, Stripe, Turnstile, etc. Comme ça, un token
chiffré ou un flux OAuth testé en local se comporte pareil sur
`ct-staging-app-01`.

Ce fichier **ne va pas dans Git**. Il est placé sur le serveur par Ansible
(credential / vault AWX), jamais commité.

Même avec « le même `.env` », quelques clés **doivent être adaptées** à la
machine de staging, sinon l'application pointe encore vers le poste local :

| Clé | Pourquoi la changer sur le staging |
|---|---|
| `DATABASE_URL` | PostgreSQL de la VM, pas celui du poste |
| `REDIS_URL`, `CACHE_URL`, `LOCKS_REDIS_URL`, `CELERY_BROKER_URL` | Redis de la VM (Docker) |
| `SITE_URL`, `DOCS_URL`, `ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS` | `app-staging` / `docs-staging` / `staging.cleantask.co` |
| `TRUST_PROXY_HEADERS` | `True` derrière le tunnel (en local le défaut est `False`) |
| `DEBUG` | `False` sur le staging (le `.env` de dev a en général `True`) |
| `DJANGO_SETTINGS_MODULE` | `config.settings.production` dans Docker |

En pratique : Ansible part du `.env` de dev (secrets), puis **écrase** ces
clés d'hôte depuis `inventories/staging/group_vars/all/env.yml`. Le reste est
identique.

## 9. AWX

| Élément AWX | Contenu |
|---|---|
| Projet | Dépôt Git `cleantask-awx` |
| Inventaires | `cleantask-staging`, `cleantask-prod`, sourcés depuis `inventories/` |
| Credentials | Clé SSH machine, mot de passe Ansible Vault, clé `age` (restauration) |
| Modèles | Un par playbook ; inventaire demandé au lancement ; confirmations en questionnaire (survey) pour les opérations destructrices |
| Planifications | `restore_test.yml` hebdomadaire sur staging |

## 10. Ordre de réalisation

1. **AWX d’abord** (voir [`GUIDE_AWX.md`](GUIDE_AWX.md)) : organisation,
   credentials, projet GitHub `cleantask/cleantask-awx`, inventaire staging,
   modèle `server-setup-staging`. Le vault doit être **chiffré** avant le push.
   Remettre la VM au snapshot si un run local l’a déjà modifiée.
2. **Staging de base via la console** : job `server-setup-staging` (`preflight`,
   `common`, `network`, `docker`, `postgresql`), puis rôles `app`,
   `static_sites`, `nginx`, `cloudflared` et données de démo.
3. **Sauvegardes** : `pgbackrest`, `pgdump`, `media_backup`, `restore_db.yml`,
   `restore_test.yml` et un vrai test de restauration.
4. **Production** : `ct-prod-app-01` sur Hetzner, mêmes playbooks, inventaire `prod`.
5. **Plus tard** : etcd, Patroni, HAProxy, Object Storage.

Conseil : prendre un **snapshot Proxmox** de `ct-staging-app-01` avant chaque
essai de playbook.

## 11. Questions ouvertes

- Rôle Nginx : activer `real_ip` + `CF-Connecting-IP` (§ 7.3).
- Site vitrine : accord pour remplacer l'URL codée en dur par un marqueur injecté au build.
- Hébergement Git du dépôt `cleantask-awx` (GitHub, Origin…) et clé de déploiement d'AWX.
- Supervision : où collecter les métriques des exporters (Prometheus / Grafana, ou service externe) ?
- Rétention des sauvegardes (proposition : journaux 14 jours, `pg_dump` 30 jours, médias 30 jours + 12 mensuelles).
