# Branchement d’un service au gateway Traefik (CI)

Ce document décrit comment exposer **un conteneur** (ou une stack Compose) derrière le Traefik du VPS, **sans** publier directement les ports de l’application sur Internet.

**Prérequis côté infrastructure** (déjà en place sur le VPS CI) :

- Traefik écoute en **80** / **443** (UFW ouvert).
- Réseau Docker externe **`traefik-public`** (créé une fois : `docker network create traefik-public`).
- Un **nom de domaine** (ou sous-domaine) dont le **DNS** pointe vers l’**IP publique du VPS** (nécessaire pour un certificat Let’s Encrypt **production** valide).
- Sur ce gateway, le résolveur ACME s’appelle **`letsencrypt`** (voir `gateway/docker-compose.yml`).

---

## 1. Réseau Docker obligatoire

Le service **doit** être sur le réseau **`traefik-public`** (même réseau que Traefik), pour que le reverse proxy atteigne le conteneur **en IP interne**.

Dans ton `docker-compose.yml` :

```yaml
networks:
  traefik-public:
    external: true
```

Et sur **chaque** service exposé :

```yaml
services:
  mon-app:
    # ...
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

Si ton stack a **plusieurs** réseaux (ex. base de données sur un réseau privé), tu peux attacher le service aux **deux** : `traefik-public` + `backend-internal`.

---

## 2. Activer la découverte Traefik

Sur ce gateway, **`exposedByDefault`** est à **`false`**. Sans label explicite, Traefik **ignore** le conteneur.

Ajoute au minimum :

```yaml
labels:
  - traefik.enable=true
```

---

## 3. Indiquer le réseau vu par Traefik

Quand un conteneur est sur **plusieurs** réseaux, Traefik doit savoir **lequel** utiliser pour joindre le backend.

```yaml
labels:
  - traefik.docker.network=traefik-public
```

---

## 4. Router HTTPS + TLS (Let’s Encrypt)

Convention : remplace **`mon-projet`** par un préfixe **unique** sur le serveur (évite les collisions entre étudiants / projets). Remplace **`api.mon-domaine.org`** par ton **vrai** FQDN (DNS déjà en place).

```yaml
labels:
  - traefik.enable=true
  - traefik.docker.network=traefik-public
  - traefik.http.routers.mon-projet-app.rule=Host(`api.mon-domaine.org`)
  - traefik.http.routers.mon-projet-app.entrypoints=websecure
  - traefik.http.routers.mon-projet-app.tls=true
  - traefik.http.routers.mon-projet-app.tls.certresolver=letsencrypt
  - traefik.http.services.mon-projet-app.loadbalancer.server.port=8000
```

**À adapter** :

| Élément | Rôle |
|--------|------|
| `mon-projet-app` | Nom **unique** du routeur / service côté labels (lettres, chiffres, `-`). |
| `Host(...)` | Hôte attendu dans l’URL ; doit correspondre au **certificat** demandé. |
| `entrypoints=websecure` | Trafic **HTTPS** sur le port **443** du VPS. |
| `certresolver=letsencrypt` | Doit **exactement** correspondre au nom du résolveur sur le gateway. |
| `loadbalancer.server.port` | **Port d’écoute du processus dans le conteneur** (ex. 8000 Django, 3000 Node, 80 nginx). |

L’entrypoint **`web`** (port 80) redirige vers **HTTPS** ; pour une app publique, configure **`websecure`** + **TLS** comme ci-dessus.

---

## 5. Exemple minimal complet (`docker-compose.yml`)

À copier dans un dossier dédié (ex. `~/deploy/mon-projet/`), puis `docker compose up -d`.

```yaml
services:
  whoami:
    image: traefik/whoami
    restart: unless-stopped
    networks:
      - traefik-public
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      - traefik.http.routers.whoami-demo.rule=Host(`whoami.example.com`)
      - traefik.http.routers.whoami-demo.entrypoints=websecure
      - traefik.http.routers.whoami-demo.tls=true
      - traefik.http.routers.whoami-demo.tls.certresolver=letsencrypt
      - traefik.http.services.whoami-demo.loadbalancer.server.port=80

networks:
  traefik-public:
    external: true
```

Remplace **`whoami.example.com`** par un nom qui **résout vers le VPS**, puis teste : `https://whoami.example.com` dans le navigateur.

---

## 6. Conventions DNS et noms

- **Un FQDN par routeur** (sauf configuration avancée avec plusieurs règles / priorités).
- Évite les `Host()` trop génériques en partage : préfère **`projet-XX.domaine`**, **`api-projet.domaine`**, etc.
- Avant de multiplier les sous-domaines, vérifie que le **certificat** et le **rate limit** Let’s Encrypt restent raisonnables (erreurs répétées = blocage temporaire).

---

## 7. En-têtes de sécurité (déjà globaux)

Le gateway applique un middleware **`security-headers@file`** sur l’entrypoint **`websecure`** (HSTS modéré, `X-Frame-Options`, `Referrer-Policy`, etc.). Tes routes **HTTPS** en bénéficient **sans** label supplémentaire.

Tu peux **ajouter** tes propres middlewares (rate limit, Basic Auth, stripPrefix…) via des labels ; ils s’exécutent **en plus** de ceux du gateway (ordre : voir doc Traefik sur les chaînes de middlewares).

---

## 8. Erreurs fréquentes à éviter

| Problème | Conséquence |
|----------|-------------|
| Oublier **`traefik.enable=true`** | Traefik ne crée **aucune** route. |
| Oublier **`traefik.docker.network`** avec plusieurs réseaux | Erreurs **502** / mauvaise IP cible. |
| Deux routeurs avec le **même** nom (`traefik.http.routers.meme-nom`) | Conflit de config imprévisible. |
| **`Host()`** qui ne correspond pas au DNS | Certificat Let’s Encrypt **impossible** ou mauvais site. |
| Publier **`ports: "8080:8080"`** sur l’hôte pour l’app **en plus** de Traefik | Surface d’attaque élargie, contournement du proxy ; en CI, préfère **uniquement** le réseau Docker + Traefik. |
| Mauvais **`loadbalancer.server.port`** | **502** (Traefik joint le mauvais port dans le conteneur). |

---

## 9. Tests Let’s Encrypt **staging**

Pour éviter de taper les quotas Let’s Encrypt en phase de test, le **gateway** peut utiliser l’URL **staging** dans son `.env` (`ACME_CA_SERVER`, voir `.env.example`). Les certificats ne seront **pas** reconnus par les navigateurs (normal). Repasse en **production** une fois la config validée.

---

## 10. Vérifications rapides

1. `docker compose ps` — le conteneur est **Up**.
2. `docker network inspect traefik-public` — ton service y figure.
3. Logs Traefik : `docker compose logs traefik` (depuis le dossier `gateway/`) — erreurs ACME ou de routage.
4. Depuis ton PC : `curl -I https://ton-sous-domaine` — **HTTP 200** (ou 30x) et en-têtes présents.

---

_Pour le déploiement du gateway lui-même : voir `gateway/README.md`._
