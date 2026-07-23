# rpi-stack

Bibliothèque générique de templates Docker Compose pour services auto-hébergés.
Ce repo ne contient **aucune valeur concrète, aucun secret** — uniquement des
templates (`${VAR}`) et des fichiers `.example`. Les valeurs réelles sont
apportées par le projet consommateur (ex. `rpi-nomade`).

> **Conventions figées.** Structure, nommage et plan d'adressage réseau
> (ci-dessous) sont des règles actées avec l'utilisateur — pas des choix
> libres. Tout assistant (LLM ou autre) qui s'en écarte, ou qui propose une
> alternative en créant un nouveau service, doit d'abord obtenir l'accord
> explicite de l'utilisateur (détail dans `CLAUDE.md`).

---

## Plan d'adressage réseau (Docker)

Chaque stack a son propre sous-réseau Docker dédié, toujours en `/24` (cf.
`CLAUDE.md`), alloué depuis un bloc `/16` fixe selon la catégorie du
service. Renumérotée une 2ème fois le 2026-07-23 (fusion des catégories
infrastructure/services communs en un seul dossier `socle/`, cf.
`CLAUDE.md` « Structure ») :

| Bloc `/16` | Catégorie |
|---|---|
| `10.0.0.0/16` | Réseau `proxy` partagé (routage, cf. ci-dessous — pas un bloc de `/24` par stack) |
| `10.1.0.0/16` | Socle technique (Portainer, ddclient, Traefik, monitoring...) |
| `10.2.0.0/16` | Services métiers |

Allocations actuelles — **à tenir à jour à chaque nouveau service** :

| Stack | Catégorie | Subnet privé | Réseau/interface | IP sur `net_proxy` |
|---|---|---|---|---|
| `portainer` | socle | `10.1.0.0/24` | `net_portainer` | — (routé via provider file, pas `net_proxy` — cf. ci-dessous) |
| `ddclient` | socle | `10.1.1.0/24` | `net_ddclient` | — (pas d'interface web, rien à router) |
| `traefik` | socle | `10.1.2.0/24` | `net_traefik` | `10.0.1.2` (créateur du réseau) |

Prochain `/24` libre : `10.1.3.0/24` (socle) ; `10.2.0.0/24` (services
métiers, bloc encore inutilisé).

### Réseau `proxy` partagé (routage)

Amendement à la règle « un `/24` par stack » (cf. `CLAUDE.md`) pour les
stacks de routage type Traefik : en plus de son `/24` privé, ce genre de
stack **crée** un réseau Docker bridge **partagé** que les stacks backend
**locales** rejoignent **en plus** de leur propre `/24`, en le déclarant
`external: true` chez elles (auto-découverte par labels via le *provider*
Docker de Traefik). Un backend **délocalisé** (NAS, autre Pi, cluster...)
n'a besoin de rien de tout ça — juste une IP:port dans la config `dynamic/`.

- **Sous-réseau fixe** : `10.0.0.0/16` (plus laissé à l'attribution libre de
  Docker depuis le 2026-07-23).
- **Adressage** : `10.0.X.Y`, où `X` = code de catégorie (`1` socle, `2`
  métier — mêmes chiffres que les blocs `/16` privés) et `Y` = le 3ème
  octet du `/24` privé du service (sa position dans sa catégorie). Ex.
  `traefik` (`10.1.2.0/24`, 3ème octet `2`) → `10.0.1.2`. **Aucune autre
  stack ne le rejoint aujourd'hui** — `ddclient` n'a pas d'interface web
  (rien à router), `portainer` est routé autrement (cf. « Pourquoi
  Portainer n'est pas sur `net_proxy` » ci-dessous) ; le mécanisme reste
  prêt pour un futur service qui en aurait vraiment besoin.
- **Point ouvert, pas encore tranché** : si une stack a besoin de plusieurs
  URLs Traefik distinctes (donc potentiellement plusieurs adresses sur ce
  réseau pour un seul service), le schéma à utiliser n'est pas décidé —
  piste envisagée mais non retenue non plus : un relais web local (reverse-
  proxy interne à la stack) pour ne présenter qu'une seule adresse à
  Traefik. À trancher au premier besoin réel.

| Réseau partagé | Créé par | Rejoint par |
|---|---|---|
| `net_proxy` | `traefik` | personne aujourd'hui — prêt pour une future stack locale qui voudrait être routée via Traefik avec découverte auto par labels (`external: true`) |

### Pourquoi Portainer n'est pas sur `net_proxy`

Décidé le 2026-07-23, après un premier essai (Portainer rejoignant
`net_proxy` + labels `traefik.*`) revenu en arrière — complexité pas
justifiée par le gain. Portainer veut du TLS Let's Encrypt, mais **pas**
via `net_proxy`/labels : il est déployé *avant* que `traefik` (et donc
`net_proxy`) n'existe, ce qui aurait imposé une dance en deux temps
(déployer Portainer, déployer Traefik, réappliquer le `compose.yaml` de
Portainer) rien que pour lui.

À la place, Traefik route vers Portainer via le **provider file** — exacte-
ment comme un backend délocalisé (cf. `socle/traefik/dynamic/portainer.yml.example`) :
une simple entrée `IP:port` pointant sur l'accès déjà publié de Portainer
(`PORTAINER_BIND_ADDR:9000`). Zéro changement dans son `compose.yaml`, zéro
label, zéro réseau partagé, zéro dépendance d'ordre de déploiement —
Portainer garde aussi son accès direct existant en fallback. Réutilisable
pour tout futur service qui veut du TLS Traefik sans les contraintes de
`net_proxy`.

### Ordre de déploiement

`net_proxy` n'existe qu'une fois `traefik` déployé. Ordre concret :
**Portainer** (bootstrap, toujours en premier) → **Traefik** (crée
`net_proxy`), déployé juste après, en 2ème. Compose refuse un réseau
`external` introuvable — pas de contournement possible sur l'ordre.

Aujourd'hui, aucune stack n'a besoin de rejoindre `net_proxy` (Portainer
utilise le provider file, cf. ci-dessus) — cette contrainte d'ordre ne
s'applique pour l'instant qu'à un futur service qui en aurait explicitement
besoin.

#### Rattacher une stack pré-existante à `net_proxy`

Pour exposer via Traefik une stack déjà déployée (locale), le contenu à
ajouter est **le même** quel que soit le mécanisme de déploiement — seule la
façon de le pousser change :

```yaml
services:
  monservice:
    networks:
      network: {}   # son /24 privé existant, inchangé
      proxy: {}     # ajout : rejoint le réseau partagé
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.monservice.rule=Host(`monservice.exemple.com`)"
      - "traefik.http.routers.monservice.entrypoints=websecure"
      - "traefik.http.routers.monservice.tls.certresolver=letsencrypt"

networks:
  network: {}       # inchangé
  proxy:
    external: true
    name: net_proxy # doit correspondre au nom cree par la stack traefik
```

- **Déploiement brut (`docker compose up -d`)** — éditer le `compose.yaml`
  de la stack ciblée sur la cible, puis relancer `docker compose up -d` dans
  son dossier. Compose connecte le conteneur en plus de son réseau existant,
  sans toucher aux volumes/données.
- **Déploiement via l'API Portainer** — même contenu YAML, mais mis à jour
  dans la Stack Portainer et redéployé via l'API (`PUT /api/stacks/{id}`,
  ce que fait le rôle `portainer-stack`) ; les variables passent par le
  champ *Environment variables* de la Stack plutôt que par un `.env` déposé
  à côté.
- **Dans les deux cas** : `net_proxy` doit déjà exister sur l'hôte (donc la
  stack `traefik` déjà déployée) avant l'update — sinon ça échoue en
  cherchant un réseau externe introuvable.

---

## Stacks disponibles

### `socle/portainer/` — Portainer Business Edition (EE lts)

Interface web de gestion Docker. Licence gratuite jusqu'à 3 nœuds ; la version
EE est utilisée car elle partage le même codebase que CE et déverrouille les
fonctionnalités sous licence sans rien changer au reste.

Particularités du template :

- Mot de passe admin bcrypt pré-configuré au démarrage (`--admin-password-file`)
  — pas de fenêtre de 5 min, pas de setup token à recopier depuis les logs.
- Exposition sur une IP précise (`PORTAINER_BIND_ADDR`) plutôt que `0.0.0.0`
  — Docker bypasse `ufw`/`nftables` via DNAT, le binding est la seule parade.
- Réseau bridge dédié (`net_portainer`) avec IP fixe — adressage stable pour
  les règles pare-feu, interface prévisible dans `ip link`/tcpdump.
- Volume bindé sur une partition dédiée (`PORTAINER_DATA_DIR`) — les données
  restent hors de `/var/lib/docker`, réservé au moteur.
- **Pas de `healthcheck`** — l'image est buildée `FROM scratch` (aucun
  wget/curl/shell dedans) et Portainer n'a pas de sous-commande CLI dédiée ;
  un healthcheck HTTP classique est donc impossible sans changer l'image
  (`-alpine`), pas fait ici. Détail en commentaire dans `compose.yaml`.
- **Ne rejoint pas `net_proxy`** — pas de label `traefik.*`, pas de réseau
  partagé. Pour du TLS Let's Encrypt, Traefik route vers lui via le
  *provider file* (IP:port déjà publiée, cf. `socle/traefik/dynamic/portainer.yml.example`
  et « Pourquoi Portainer n'est pas sur `net_proxy` » plus haut) — l'accès
  direct `PORTAINER_BIND_ADDR:9000` reste utilisable en fallback.

Variables requises (voir `socle/portainer/portainer.env.example`) :

| Variable | Rôle |
|---|---|
| `PORTAINER_CONTAINER_NAME` | Nom du conteneur (`portainer`, un seul conteneur dans la stack) |
| `PORTAINER_BIND_ADDR` | IP d'écoute de l'UI (port 9000) |
| `PORTAINER_NETWORK_SUBNET` | Sous-réseau Docker interne de la stack, en `/24` |
| `PORTAINER_NETWORK_IP` | IP fixe de Portainer dans ce sous-réseau (`.100`) |
| `PORTAINER_NETWORK_NAME` | Nom de réseau Docker (`net_portainer`) |
| `PORTAINER_NETWORK_IFACE` | Nom d'interface bridge (`net_portainer`, à garder identique par convention) |
| `PORTAINER_DATA_DIR` | Chemin hôte pour la persistance des données |

Secrets requis (voir `socle/portainer/secrets.example/`) :

| Fichier | Contenu |
|---|---|
| `secrets/portainer_admin_password` | Hash bcrypt du mot de passe admin |
| `secrets/license.env` | Clé de licence EE (`PORTAINER_LICENSE_KEY=…`) |

---

### `socle/ddclient/` — DNS dynamique (linuxserver.io)

Met à jour un enregistrement DNS (ex. OVH DynHost) avec l'IP publique de
l'hôte. Image multi-arch, `ddclient` ≥ 3.10.0 (`protocol=ovh` natif).

Particularités du template :

- Le fichier de config applicatif complet (`ddclient.conf`,
  protocole/identifiants/domaine) n'est pas découpable en `${VAR}` Compose
  — voir `ddclient.conf.example` et la section « Cas particulier » du
  `CLAUDE.md`.
- **Pas de `healthcheck`** — ddclient n'écoute aucun port, ne sert aucune
  page web, rien à sonder depuis l'extérieur du process. Détail en
  commentaire dans `compose.yaml`.

Variables requises (voir `socle/ddclient/ddclient.env.example`) :

| Variable | Rôle |
|---|---|
| `DDCLIENT_CONTAINER_NAME` | Nom du conteneur (`ddclient`, un seul conteneur dans la stack) |
| `DDCLIENT_PUID` / `DDCLIENT_PGID` / `DDCLIENT_TZ` | Utilisateur hôte et fuseau horaire (image linuxserver.io) |
| `DDCLIENT_CONFIG_PATH` | Chemin hôte vers le `ddclient.conf` réel |
| `DDCLIENT_NETWORK_SUBNET` | Sous-réseau Docker interne de la stack, en `/24` |
| `DDCLIENT_NETWORK_IP` | IP fixe de ddclient dans ce sous-réseau (`.100`) |
| `DDCLIENT_NETWORK_NAME` | Nom de réseau Docker (`net_ddclient`) |
| `DDCLIENT_NETWORK_IFACE` | Nom d'interface bridge (`net_ddclient`, à garder identique par convention) |

---

### `socle/traefik/` — reverse proxy + TLS (socle technique)

Expose les futurs services métiers derrière TLS (ACME/Let's Encrypt). Fait
partie du socle technique au même titre que Portainer/ddclient — sert
d'autres stacks plutôt que d'être un service métier lui-même.

Particularités du template :

- Config statique (`traefik.yml`) et dynamique (`dynamic/*.yml`) en fichiers
  complets, pas en `${VAR}` — même logique que `ddclient.conf` (voir « Cas
  particulier » du `CLAUDE.md`).
- Deux réseaux Docker : son `/24` privé classique, **et** `net_proxy`, réseau
  partagé qu'il crée pour router les stacks locales (voir « Réseau `proxy`
  partagé » ci-dessus) — pas encore rejoint par aucune stack aujourd'hui. Un
  backend délocalisé (NAS, autre Pi...) **ou même un service local déjà
  publié sur l'hôte** (ex. Portainer, cf. `dynamic/portainer.yml.example`)
  n'a besoin d'aucun des deux réseaux — juste une entrée `dynamic/*.yml`
  avec une IP:port.
- `exposedByDefault: false` — un conteneur n'est routé que s'il porte
  `traefik.enable=true` en label, jamais de découverte automatique.
- Dashboard jamais exposé nu (`api.insecure: false`), routé via un middleware
  `basicAuth` (`dynamic/dashboard.yml.example`).
- `docker.sock` monté en lecture seule pour l'auto-découverte — accès
  équivalent root sur l'hôte, même arbitrage déjà accepté pour Portainer.
- TLS : challenge HTTP-01 par défaut (port 80 joignable) ; alternative DNS-01
  via OVH documentée en commentaire dans `traefik.yml.example` (pas de port
  80 exposé, certs wildcard, mais identifiants API OVH différents de ceux de
  `ddclient`).
- **`healthcheck` via `traefik healthcheck --ping`** — sous-commande CLI
  native (pas un ping ICMP) : GET HTTP réel sur `/ping`, servi par
  l'entrypoint interne `traefik` (port 8080, non publié vers l'hôte).
  Marche sans wget/curl dans l'image ; nécessite `ping: {}` en config
  statique (déjà dans `traefik.yml.example`).

Variables requises (voir `socle/traefik/traefik.env.example`) :

| Variable | Rôle |
|---|---|
| `TRAEFIK_CONTAINER_NAME` | Nom du conteneur (`traefik`, un seul conteneur dans la stack) |
| `TRAEFIK_BIND_ADDR` | IP d'écoute des ports 80/443 |
| `TRAEFIK_HTTP_PORT` / `TRAEFIK_HTTPS_PORT` | Ports hôte publiés vers 80/443 |
| `TRAEFIK_STATIC_CONFIG_PATH` | Chemin hôte vers `traefik.yml` réel |
| `TRAEFIK_DYNAMIC_CONFIG_DIR` | Dossier hôte vers les `dynamic/*.yml` réels |
| `TRAEFIK_ACME_DIR` | Dossier hôte des certificats ACME (`acme.json` à pré-créer en `chmod 600`) |
| `TRAEFIK_NETWORK_SUBNET` | Sous-réseau Docker interne de la stack, en `/24` |
| `TRAEFIK_NETWORK_IP` | IP fixe de Traefik dans ce sous-réseau (`.100`) |
| `TRAEFIK_NETWORK_NAME` | Nom de réseau Docker (`net_traefik`) |
| `TRAEFIK_NETWORK_IFACE` | Nom d'interface bridge (`net_traefik`, à garder identique par convention) |
| `TRAEFIK_PROXY_NETWORK_NAME` | Nom du réseau partagé de routage (`net_proxy`), créé par cette stack |
| `TRAEFIK_PROXY_NETWORK_SUBNET` | Sous-réseau fixe du réseau partagé (`10.0.0.0/16`) |
| `TRAEFIK_PROXY_IP` | IP fixe de Traefik lui-même sur `net_proxy` (`10.0.2.0`, formalisme `10.0.X.Y`) |

Secrets requis (voir `socle/traefik/secrets.example/`) :

| Fichier | Contenu |
|---|---|
| `secrets/ovh_dns.env` | Identifiants API OVH (`OVH_ENDPOINT`/`OVH_APPLICATION_KEY`/`OVH_APPLICATION_SECRET`/`OVH_CONSUMER_KEY`) pour le provider DNS-01 "ovh" de lego — DIFFÉRENTS des identifiants DynHost de `ddclient` ; sans effet si `dnsChallenge` n'est pas activé dans `traefik.yml` |

---

## Déploiement

Ce repo ne contient que des templates `compose.yaml`, indépendants du
mécanisme utilisé pour les lancer — c'est au consommateur de choisir,
service par service : `docker compose up -d` brut, ou poussé via l'API
Portainer (rôle Ansible `portainer-stack`). Portainer lui-même est un cas
particulier fonctionnel : il ne peut pas se déployer via sa propre API
(il n'existe pas encore au moment de son propre déploiement), donc toujours
lancé en `docker compose up -d` brut côté consommateur.

---

## Comment consommer ce repo

Même mécanique que `rpi-format` / `rpi-cis` / `rpi-stage` : le projet
consommateur détecte d'abord un dossier/lien local, puis un repo frère
(`../../outillage/rpi-stack`), sinon clone au commit épinglé.

Le consommateur :

1. Lit le `compose.yaml` d'un service **tel quel** (ex. `lookup('file', …)` Ansible).
2. Fournit un fichier `.env` réel adapté du `.example` — déposé à côté du
   `compose.yaml` sur la cible, jamais committé ici.
3. Fournit ses propres secrets dans `secrets/` — jamais `secrets.example/` d'ici,
   toujours gitignorés côté consommateur.
