# Clean-registry module

Nettoie la registry Docker d'un projet (images/tags devenus inutiles) pour éviter la saturation du stockage GitLab. Le module couvre deux cas distincts : les images « fantômes » (`ghost`, plus définies dans les Dockerfiles du projet) et les images de branches de développement obsolètes (`dev`).

## Projets concernés

| Projet | Rôle dans la feature |
|------|----------------------|
| `cicd-yaml` | Définition des jobs (`features/clean-registry.yml`) |
| `cicd-script` | Logique Python (`clean_registry/`), réutilise `find_dockerfiles_r` du module `build_docker` |
| `cicd-configuration` | Variables du projet + schedules |

## Fonctionnement du script

```mermaid
flowchart LR
    A[Récupération de la registry du projet] --> B["Scan des Dockerfiles du repo<br>(find_dockerfiles_r, comme build-docker)"]
    B --> C{Mode demandé}
    C -- --delete-ghost-image --> D[clean_ghost_images]
    C -- --delete-dev-image --> E[clean_dev_images]
```

Les deux modes partagent la même étape préalable : récupération de la registry du projet (`get_registry_info`) et de la liste des Dockerfiles réellement présents dans le repo (`find_dockerfiles_r`, la même fonction que celle utilisée par le module `build-docker`), pour connaître le nom et la version attendus de chaque image.

### Mode `--delete-ghost-image` (images fantômes)

1. Pour chaque dépôt (repository) de la registry, récupère la liste de ses tags et la liste des branches du projet.
2. Un tag est considéré « fantôme avec branche dev » (`filter_ghost_tags_with_dev_branch`) s'il ne correspond à la version d'aucun Dockerfile actuellement présent dans le repo pour ce dépôt.
3. Parmi ces tags fantômes, ceux dont le nom correspond à une branche de développement encore active (`check_if_is_dev_branch`, i.e. une branche qui n'est pas dans `BUILDER_PROJECT_BRANCHS`) sont exclus — ils seront potentiellement traités par le mode `dev`, pas ici.
4. Les tags fantômes restants, et non listés dans `TAG_TO_KEEP`, sont supprimés (`delete_tag_in_repository`).
5. Un dépôt entier est supprimé (`delete_repository_in_registry`) s'il ne correspond plus à aucun Dockerfile du repo (`repository_not_present`).
6. Dans tous les cas, un dépôt ou un tag dont le nom de dépôt figure dans `REPOSITORIES_WHITELIST` est conservé.

### Mode `--delete-dev-image` (images de dev obsolètes)

1. Pour chaque dépôt de la registry, chaque tag est comparé aux branches existantes du projet.
2. Un tag est marqué à supprimer s'il ne correspond à aucune branche existante (`is_current_dev_tag = False`) et n'est pas dans `TAG_TO_KEEP`.
3. Les tags à supprimer d'un dépôt whitelisté (`REPOSITORIES_WHITELIST`) sont conservés.

## Prérequis

- `ENABLE_BUILD` doit valoir `yes` pour le projet (configuration mise en place par le module `build-docker`, voir [BUILD_SETUP.md](../setup/BUILD_SETUP.md)).
- Un token d'API GitLab avec les droits suffisants sur la registry (`CICD_API_TOKEN`).

## Déclenchement

Deux jobs indépendants (stage `clean`), déclenchés par tag dans le message de commit :

| Job | Condition | Mode |
|-----|-----------|------|
| `clean-registry-ghost-image` | `ci-clean-nobuild` dans le message de commit | `--delete-ghost-image` |
| `clean-registry-dev-image` | `ci-clean-dev` dans le message de commit | `--delete-dev-image` |

## (optionnel) Configuration du schedule

Les types de schedule `cleanghostimage` et `cleandevimage` (définis par défaut via `SETUP_BUILD_SCHEDULE_TYPE` dans `cicd-configuration`) permettent de programmer ces nettoyages automatiquement, à ajouter dans `cicd-configuration/setup/build.yml` :

```yaml
- name: "mon-repo-docker"
  id: <project_id>
  schedule:
    - type: "cleanghostimage"   # commit "ci-clean-nobuild" tous les jours à 2h
    - type: "cleandevimage"     # commit "ci-clean-dev" tous les jours à 1h30
```

### Variables (`cicd-configuration/configuration/by_project/.<projet>-conf.yml`)

```yaml
variables:
  #================== Features variables =================#
  CLEAN_REGISTRY_LOG_LEVEL: "DEBUG"        # Défaut : INFO
  BUILDER_PROJECT_BRANCHS: "main,preprod"  # Défaut : main -- branches "stables" du projet (hors dev)
  TAG_TO_KEEP: "latest-main,latest-prod"   # Défaut : latest-main -- tags jamais supprimés
  REPOSITORIES_WHITELIST: "mon-image"      # Défaut : vide -- dépôts jamais touchés par le nettoyage
```

## Résultat

- **Ghost image** : les dépôts et tags qui ne correspondent plus à aucun Dockerfile du projet sont supprimés de la registry.
- **Dev image** : les tags qui ne correspondent plus à aucune branche existante sont identifiés en log (suppression effective désactivée dans le code actuel).

---
