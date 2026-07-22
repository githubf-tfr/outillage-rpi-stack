# rpi-stack

Bibliothèque générique de templates Docker Compose pour services auto-hébergés.
Ce repo ne contient **aucune valeur concrète, aucun secret** — uniquement des
templates (`${VAR}`) et des fichiers `.example`. Les valeurs réelles sont
apportées par le projet consommateur (ex. `rpi-nomade`).

---

## Stacks disponibles

### `compose/portainer` — Portainer Business Edition (EE lts)

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

Variables requises (voir `compose/portainer/portainer.env.example`) :

| Variable | Rôle |
|---|---|
| `PORTAINER_CONTAINER_NAME` | Nom du conteneur (`portainer`, un seul conteneur dans la stack) |
| `PORTAINER_BIND_ADDR` | IP d'écoute de l'UI (port 9000) |
| `PORTAINER_NETWORK_SUBNET` | Sous-réseau Docker interne de la stack, en `/24` |
| `PORTAINER_NETWORK_IP` | IP fixe de Portainer dans ce sous-réseau (`.100`) |
| `PORTAINER_NETWORK_NAME` | Nom de réseau Docker (`net_portainer`) |
| `PORTAINER_NETWORK_IFACE` | Nom d'interface bridge (`net_portainer`, à garder identique par convention) |
| `PORTAINER_DATA_DIR` | Chemin hôte pour la persistance des données |

Secrets requis (voir `compose/portainer/secrets.example/`) :

| Fichier | Contenu |
|---|---|
| `secrets/portainer_admin_password` | Hash bcrypt du mot de passe admin |
| `secrets/license.env` | Clé de licence EE (`PORTAINER_LICENSE_KEY=…`) |

---

## Deux méthodes de déploiement

### 1. `compose/` — déploiement direct (`docker compose up -d`)

Pour les services qui doivent être bootstrappés **sans Portainer** — typiquement
parce que Portainer n'est pas encore en place. C'est le cas de Portainer
lui-même : il ne peut pas se déployer via sa propre API.

Le consommateur dépose le `compose.yaml` sur la cible avec son `.env` réel et
ses `secrets/` propres, puis lance `docker compose up -d`.

```
compose/
└── portainer/      ← aujourd'hui
```

### 2. Via l'API Portainer — *à venir*

Une fois Portainer en place, les stacks suivantes seront poussées via son API
(ex. rôle Ansible `portainer-stack`). Elles vivront dans une catégorie
distincte, créée quand le premier besoin réel se présente.

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
