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

## Structure

1 dossier de catégorie = 1 mécanisme de déploiement ; à l'intérieur,
**1 dossier = 1 service** (`compose.yaml` + `<service>.env.example` +
`secrets.example/` si le service a des secrets).

- **`compose/`** — services déployés par un simple `docker compose up -d`
  brut (pas via l'API Portainer). Aujourd'hui : `compose/portainer/`.
  Portainer y vit spécifiquement parce que c'est un cas particulier :
  **il ne peut pas se déployer lui-même via sa propre API** (il n'existe pas
  encore au moment de son propre déploiement) — c'est le stack *bootstrap*.
  Toute future stack qui, elle, sera poussée via l'API Portainer (une fois
  Portainer en place) n'a **pas sa place ici** — elle vivra dans une autre
  catégorie, à créer *quand le premier besoin réel se présente* (pas avant
  — le squelette accueille, le contenu vient à l'usage, même logique que
  `rpi-nomade`/`CLAUDE.md`).

```
rpi-stack/
├── CLAUDE.md
└── compose/
    └── portainer/
        ├── compose.yaml            # template, ${VAR} uniquement
        ├── portainer.env.example   # modele des valeurs non secretes
        └── secrets.example/        # modele des secrets (noms de fichiers, valeurs factices)
```

## Convention de nommage des variables

Chaque variable est préfixée par le nom du service en MAJUSCULES
(`PORTAINER_BIND_ADDR`, `PORTAINER_NETWORK_SUBNET`, ...) — évite les
collisions si le contenu de plusieurs `.env` de services est un jour
concaténé côté consommateur.

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

Premier consommateur connu : `rpi-nomade` (`services/portainer/`), Phase 2 —
Portainer déployé en bootstrap via `rpi-stage`
(`docker-compose-stack` + `docker_compose_stack_env_file`, ajouté le
2026-07-22 pour ce besoin).

## Pas de README séparé

Comme `rpi-nomade` (règle "un seul fichier de doc générique"), ce repo n'a
qu'un seul fichier d'explication : ce `CLAUDE.md`. S'il grossit au point de
ne plus être qu'un contexte pour session mais une vraie doc utilisateur,
envisager un `README.md` à ce moment-là — pas avant.
