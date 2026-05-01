# Gateway Traefik — Centre Informatique

Reverse proxy central pour les projets déployés sur le VPS. Les applications **ne publient pas** leurs ports sur Internet : seul Traefik écoute en **80** / **443** ; le routage se fait via le **provider Docker** (labels sur les services) et le réseau **`traefik-public`**.

## Prérequis sur le VPS

- Utilisateur avec accès à **Docker** (ex. `ci` dans le groupe `docker`).
- Réseau externe créé une fois :

  ```bash
  docker network create traefik-public
  ```

- **UFW** (ou équivalent) : ports **80/tcp** et **443/tcp** ouverts vers le VPS.
- Un **nom de domaine** dont les enregistrements DNS pointent vers l’IP du VPS (nécessaire pour des certificats Let’s Encrypt valides en production).

## Fichiers

| Fichier | Rôle |
|---------|------|
| `docker-compose.yml` | Service Traefik, ports, montages. |
| `traefik.yml` | Points d’entrée `web` / `websecure`, provider Docker, TLS minimal. |
| `docker-compose.yml` (section `command`) | Paramètres **Let’s Encrypt** (e-mail, `caServer`, challenge TLS) — variables lues depuis `.env` par Compose. |
| `dynamic/middleware-security-headers.yml` | Middleware **`security-headers@file`** (HSTS modéré, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`, etc.), appliqué par défaut aux services Docker. |
| `.env` | Variables sensibles (copie de `.env.example`) — **non versionné**. |
| `acme.json` | Stockage des certificats Let’s Encrypt — **non versionné**, droits **600**. |

## Déploiement sur le VPS

1. Cloner le dépôt (ou mettre à jour avec `git pull`).

2. Aller dans ce dossier :

   ```bash
   cd gateway
   ```

3. Créer `.env` à partir du modèle et **renseigner un vrai e-mail** pour Let’s Encrypt :

   ```bash
   cp .env.example .env
   nano .env
   ```

4. Créer `acme.json` sur l’hôte (fichier vide, droits stricts) :

   ```bash
   touch acme.json
   chmod 600 acme.json
   ```

5. Démarrer la stack :

   ```bash
   docker compose up -d
   ```

6. Vérifier les journaux si besoin :

   ```bash
   docker compose logs -f traefik
   ```

## Dashboard Traefik (API)

Par défaut, le dashboard est servi en **`api.insecure: true`** sur le port **8080 du conteneur**, mappé sur le VPS uniquement sur **`127.0.0.1:8080`** (pas d’exposition publique).

Depuis ton PC, **tunnel SSH** :

```bash
ssh -L 8080:127.0.0.1:8080 ci@ADRESSE_DU_VPS
```

Puis ouvre dans le navigateur : **http://127.0.0.1:8080/dashboard/**

## Let’s Encrypt — production vs staging

- **Production** : `ACME_CA_SERVER=https://acme-v02.api.letsencrypt.org/directory` — certificats reconnus ; quotas stricts en cas d’erreurs répétées.
- **Staging** : URL dans les commentaires de `.env.example` — idéal pour valider la stack sans risquer le blocage.

## Branchement d’une autre stack Docker

Guide détaillé (réseau, labels, exemple complet, pièges) : **`BRANCHEMENT.md`**.

En résumé : réseau externe **`traefik-public`**, **`traefik.enable=true`**, **`traefik.docker.network=traefik-public`**, routeur **`websecure`** + **`tls.certresolver=citls`**, et **`loadbalancer.server.port`** = port d’écoute **dans** le conteneur.

`exposedByDefault` est **`false`** sur ce gateway : sans **`traefik.enable=true`**, un conteneur n’est **pas** publié par Traefik.

## En-têtes de sécurité (global)

Le provider **file** charge `dynamic/middleware-security-headers.yml`. L’**entryPoint `websecure`** déclare `http.middlewares: security-headers@file` : Traefik **préfixe** ce middleware à **toutes** les routes HTTP attachées à **:443** (y compris celles découvertes via Docker). Les middlewares définis sur un routeur Docker s’appliquent **après** (voir [doc EntryPoints — http.middlewares](https://doc.traefik.io/traefik/v3.4/reference/install-configuration/entrypoints/)).

Réglages notables :

- **HSTS** : 1 an, sous-domaines inclus, **sans** `preload` par défaut (évite de vous engager sur hstspreload.org tant que le domaine n’est pas figé). Tu peux passer `stsPreload: true` dans le YAML quand c’est pertinent.
- **`X-Robots-Tag: noindex, ...`** : évite d’indexer par erreur des environnements de dev / VPS scolaire. Pour un site public à référencer, retire ou adapte cette ligne dans `middleware-security-headers.yml`.
- **CSP** (`Content-Security-Policy`) : volontairement **absente** ici, car elle casse vite des frontends variés (scripts inline, CDN). Les projets peuvent ajouter un middleware CSP **par route** via labels si besoin.

## Dépannage — `Router uses a nonexistent certificate resolver`

Le résolveur s’appelle **`citls`** et est défini dans **`gateway/docker-compose.yml`** (`--certificatesresolvers.citls…`). Docker Compose substitue **`${ACME_EMAIL}`** et **`${ACME_CA_SERVER}`** depuis **`gateway/.env`**. Dans les labels applicatifs, utiliser **`certresolver=citls`**. Ne pas s’appuyer sur **`${VAR}` dans `traefik.yml`** pour l’URL ACME : utilise le `.env` du gateway interpolé par Compose.

Corrige `.env`, sans **espace en fin de ligne** sur `ACME_CA_SERVER`, puis :

```bash
docker compose up -d --force-recreate
```

## Dépannage — `client version 1.24 is too old`

Avec **Docker Engine récent**, les versions de Traefik **strictement inférieures à 3.6** utilisaient une API Docker trop basse ; la variable **`DOCKER_API_VERSION` ne suffit pas** (le client embarqué dans Traefik ne la prenait pas en compte pour ces appels). Ce dépôt utilise **`traefik:v3.6`**, qui négocie correctement l’API avec le démon. Après mise à jour de l’image : `docker compose pull && docker compose up -d` dans `gateway/`.

## Mise à jour Traefik

```bash
docker compose pull
docker compose up -d
```
