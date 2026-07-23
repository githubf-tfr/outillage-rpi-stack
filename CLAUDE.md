# CLAUDE.md — rpi-stack

## Sujet de ce repo

`rpi-stack` est une **bibliothèque générique de templates Docker Compose**
pour des services auto-hébergés communs (Portainer, DNS, sync, reverse-proxy,
etc.). Il joue pour les *stacks Docker* le même rôle que `rpi-format`/
`rpi-cis`/`rpi-stage` jouent pour l'imaging/le durcissement/le provisioning
réseau : un outil **générique et réutilisable**, consommé de l'extérieur par
un projet concret (ex. `rpi-nomade`) qui, lui, apporte les valeurs et les
secrets propres à son déploiement.

**Règle d'or : zéro code, zéro secret, zéro valeur concrète.** Ce repo ne
contient que des `compose.yaml` (syntaxe Compose native `${VAR}`/
`${VAR:-défaut}`, jamais de valeur en dur) et des fichiers `*.example`
(modèles vides/factices). Aucune logique de déploiement (pas de script, pas
de rôle Ansible) — ça, c'est le rôle du consommateur et de ses propres
outils (ex. `rpi-stage`, rôle `docker-compose-stack` ou `portainer-stack`).

**Conventions figées — pas de déviation sans accord explicite.** La
structure (catégories + 1 dossier = 1 service), le nommage des
variables/conteneurs/volumes/réseaux et le plan d'adressage réseau (cf. plus bas et
`README.md`) sont des décisions actées avec l'utilisateur, pas des choix
libres. Tout assistant (LLM ou autre) qui modifie ce repo, ou qui s'en
inspire pour créer un nouveau service, doit **obtenir l'accord explicite
de l'utilisateur avant de s'écarter de ces conventions** — y compris en
proposant une alternative "meilleure" de sa propre initiative.

## Structure

**1 dossier de catégorie → 1 dossier = 1 service** à l'intérieur
(`compose.yaml` + `<service>.env.example`/`<fichier>.example` +
`secrets.example/` si le service a des secrets découpables). Décidé le
2026-07-23, à l'occasion de `traefik/` : 3 catégories, **les mêmes** que les
blocs `/16` du plan d'adressage réseau (cf. plus bas et `README.md`) —
`infra/` (infrastructure), `sc/` (services communs), `metier/` (services
métiers, vide pour l'instant).

**Important : cet axe de catégorisation est différent de celui rejeté plus
tôt.** Ce repo a explicitement écarté un découpage par *mécanisme de
déploiement* (`compose/` brut vs `stacks/` API Portainer, cf. historique
Git) — ce choix-là reste entièrement au consommateur, indépendant de la
structure. Le découpage `infra/`/`sc/`/`metier/` catégorise par **rôle du
service**, pas par mécanisme : un service peut vivre dans n'importe laquelle
des 3 catégories quel que soit son mode de déploiement réel.

Portainer reste malgré tout un cas particulier *fonctionnel* (documenté dans
son propre `compose.yaml`, pas dans l'arborescence) : il ne peut pas se
déployer via sa propre API puisqu'il n'existe pas encore au moment de son
propre déploiement — c'est donc toujours lui qui sera lancé en `docker
compose up -d` brut côté consommateur, quel que soit l'endroit où il vit
dans ce repo.

```
rpi-stack/
├── CLAUDE.md
├── infra/
│   ├── portainer/
│   │   ├── compose.yaml            # template, ${VAR} uniquement
│   │   ├── portainer.env.example   # modele des valeurs non secretes
│   │   └── secrets.example/        # modele des secrets (noms de fichiers, valeurs factices)
│   └── ddclient/
│       ├── compose.yaml            # template, ${VAR} uniquement
│       ├── ddclient.env.example    # modele des valeurs non secretes
│       └── ddclient.conf.example   # modele du fichier de config complet (voir note ci-dessous)
├── sc/
│   └── traefik/
│       ├── compose.yaml            # template, ${VAR} uniquement
│       ├── traefik.env.example     # modele des valeurs non secretes
│       ├── traefik.yml.example     # modele de la config statique (voir note ci-dessous)
│       └── dynamic/                # modeles de config dynamique (routes), voir "Reseau proxy partage"
└── metier/                         # vide pour l'instant -- premier service metier ici
```

## Convention de nommage des variables

Chaque variable est préfixée par le nom du service en MAJUSCULES
(`PORTAINER_BIND_ADDR`, `DDCLIENT_CONFIG_PATH`, ...) — évite les collisions
si le contenu de plusieurs `.env` de services est un jour concaténé côté
consommateur.

## Cas particulier : config applicative non découpable en ${VAR}

Certains services (ex. `ddclient`) attendent un **fichier de config
applicatif complet** (pas juste des valeurs Compose comme ports/volumes) --
Compose ne sait templater que son propre `compose.yaml`, jamais le contenu
d'un fichier monté à l'intérieur du conteneur. Dans ce cas : le fichier
entier reste un `*.example` ici (structure + valeurs factices, ex.
`ddclient.conf.example`), le `compose.yaml` le monte via un chemin `${VAR}`
(ex. `${DDCLIENT_CONFIG_PATH}`), et **le consommateur fournit le fichier
réel** à cet emplacement — pas de découpage plus fin possible sans ajouter
un mécanisme de templating (entrypoint, `envsubst`...), volontairement évité
ici (zéro code, cf. règle d'or ci-dessus).

## Conventions des `compose.yaml`

Ces règles s'appliquent à tout nouveau template (elles ont été rétrofittées
sur `infra/portainer/` pour rester cohérentes dès le début). Un exemple de
référence à 2 conteneurs (serveur + bdd) illustrant l'ensemble de ces règles
vit dans `exemple/` (`compose.yaml` / `exemple.env.example` /
`secrets.example/`) — **volontairement à la racine, hors catégorie** :
n'étant pas un service réellement déployé, il n'a pas sa place dans
`infra/`/`sc/`/`metier/`.

- **Réseau en `/24`** — chaque stack a son propre sous-réseau Docker dédié,
  toujours en `/24` (à documenter en commentaire à côté de la variable de
  subnet, ex. `PORTAINER_NETWORK_SUBNET`), alloué depuis un bloc `/16` fixe
  selon la catégorie du service : infrastructure `10.0.0.0/16`, services
  communs `10.1.0.0/16`, services métiers `10.2.0.0/16`. Le registre des
  allocations `/24` actuelles vit dans `README.md` (« Plan d'adressage
  réseau ») — à tenir à jour à chaque nouveau service, pour éviter les
  collisions de sous-réseau.
- **IP fixe par conteneur, en variable** — chaque conteneur a une IP fixe
  dans ce `/24`, jamais de dépendance à la DNS interne Docker. Convention
  d'adressage : `.100` = serveur/web, `.10` = bdd (à rappeler en commentaire
  à côté de chaque `ipv4_address`).
- **Nom de réseau et nom d'interface = deux variables distinctes** —
  formalisme `net_xxx` pour les deux (`name:` du réseau Docker et
  `driver_opts.com.docker.network.bridge.name`). À garder identiques par
  convention (documenté en commentaire), mais ce sont deux variables
  séparées : libre au consommateur de les découpler s'il a un jour besoin
  de forcer une valeur différente.
- **Nom des volumes** — formalisme `nomdelastack_xxx` (ex. `portainer_data`),
  documenté en commentaire ; les noms de volumes restent en dur dans le
  compose (comme le veut la syntaxe Compose), pas en `${VAR}`.
- **Nom de conteneur, en variable** — `container_name` est toujours une
  `${VAR}` (ex. `PORTAINER_CONTAINER_NAME`), formalisme `nomdelastack_xxx`.
  Exception de nommage (pas de découplage variable/valeur) : si la stack n'a
  qu'un seul conteneur, pas de suffixe — juste `nomdelastack` (ex.
  Portainer : `PORTAINER_CONTAINER_NAME=portainer`).
- **`healthcheck`, quand c'est réellement possible** — décidé le 2026-07-23,
  à l'occasion de `traefik/` : ajouter un `healthcheck:` quand le conteneur
  offre un moyen fiable de se sonder *depuis l'intérieur de lui-même*
  (ex. `traefik healthcheck --ping`, sous-commande CLI native). **Ne pas**
  en ajouter un artificiel juste pour en avoir un — si l'image ne fournit
  aucun outil exploitable (ex. Portainer, image `scratch` sans wget/curl/
  shell) ou si le service n'a rien à sonder (ex. ddclient, ne sert aucun
  port), documenter l'absence en commentaire directement dans le
  `compose.yaml`, avec la raison technique précise plutôt qu'une supposition
  — pas de check maison en shell/log-parsing pour combler le vide (ce serait
  du code, cf. règle d'or).

## Réseau `proxy` partagé (amendement à la règle du `/24` par stack)

Décidé le 2026-07-23, à l'occasion de `traefik/`. Une stack de routage
(reverse-proxy type Traefik) a besoin de découvrir les conteneurs d'autres
stacks locales pour les exposer — ça ne rentre pas dans « chaque stack a son
propre `/24`, point ». Amendement : une stack de ce type peut, **en plus**
de son `/24` dédié classique, **créer** un réseau Docker **partagé** (bridge,
sans IPAM géré/pas de `/24` dédié — Docker attribue un sous-réseau libre) que
les stacks backend **locales** rejoignent **en plus** de leur propre `/24`
privé, en le déclarant `external: true` chez elles (même nom de réseau).

- La stack qui **crée** le réseau partagé le fait sans `external: true` (cf.
  `sc/traefik/compose.yaml`, réseau `proxy`) — elle en est le *bootstrap*, même
  logique que Portainer pour les stacks Docker en général.
- Nom de ce réseau partagé, en variable dédiée (ex.
  `TRAEFIK_PROXY_NETWORK_NAME=net_proxy`) — distinct du `/24` privé de la
  stack (`TRAEFIK_NETWORK_NAME`/`TRAEFIK_NETWORK_SUBNET`), qui reste géré
  normalement.
- Un backend **délocalisé** (hors Docker/hors hôte : NAS, autre Pi,
  cluster...) n'a besoin d'aucun réseau partagé — juste d'une IP:port
  joignable (cf. `sc/traefik/dynamic/exemple-delocalise.yml.example`).
- Cet amendement ne s'applique **qu'aux stacks qui en ont explicitement
  besoin** (routage/proxy) — la règle par défaut (un `/24` dédié, point)
  reste la norme pour tout le reste.

## Comment un projet consomme ce repo

Même mécanisme que `rpi-format`/`rpi-cis`/`rpi-stage` : détection d'un repo
local (dossier/lien à la racine du projet consommateur) puis d'un repo frère
(`../../outillage/rpi-stack`), sinon clone au commit épinglé — à documenter
dans le `README.md` du consommateur au même endroit que les 3 autres outils.

Le consommateur :
- lit le `compose.yaml` d'un service ici **tel quel** (ex. Ansible
  `lookup('file', ...)`) ;
- fournit ses **propres** valeurs non secrètes dans un fichier `.env` réel
  (copié/adapté depuis le `.example` d'ici, mais gardé **dans l'arbo du
  consommateur**, jamais ici) — déposé à côté du `compose.yaml` sur la
  cible (ex. rôle `docker-compose-stack` de `rpi-stage`,
  `docker_compose_stack_env_file`) ;
- fournit ses **propres** secrets dans `secrets/` (jamais `secrets.example/`
  d'ici), toujours gitignorés côté consommateur.

Consommateurs connus : `rpi-nomade`
- `services/portainer/` (Phase 2) — Portainer déployé en bootstrap via
  `rpi-stage` (`docker-compose-stack` + `docker_compose_stack_env_file`,
  ajouté le 2026-07-22 pour ce besoin).
- `services/ddclient/` (Phase 2 également, 2ème play de `stage/playbook.yml`
  — pas de phase séparée, décision du 2026-07-22, cf. `KANBAN.md` de
  `rpi-nomade`) — poussé via l'API Portainer (`portainer-stack`, avec
  `portainer-api-token`/`portainer-endpoint` pour les accès).

## Dépôt GitHub

- **URL** : https://github.com/githubf-tfr/outillage-rpi-stack
- **Branche principale** : `main` (unique branche, tout va sur `main`)
- Créé le 2026-07-22, premier commit : template Portainer bootstrap.

## `CLAUDE.md` vs `README.md`

Les deux coexistent, avec des rôles distincts :
- **`CLAUDE.md`** (ce fichier) — conventions et contexte pour qui modifie ce
  repo (règles de nommage, structure, mécanisme de consommation) ; pas
  destiné à cataloguer chaque stack en détail.
- **`README.md`** — doc utilisateur : catalogue des stacks disponibles
  (une fiche par service, variables/secrets requis) et comment les
  consommer, pour qui découvre le repo sans contexte de session.
