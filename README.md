# GitLab CI présentation 

Ce projet utilise une **pipeline GitLab CI/CD modulaire**, organisée autour d'un fichier parent `.gitlab-ci.yml` qui inclut plusieurs blocs YAML spécifiques à chaque fonctionnalité.

---

## Structure générale

```
.gitlab-ci.yml           # Fichier parent principal
features/
├── init.yml             # Bloc pour l'initialisation de cicd-yaml
├── build-docker.yml     # Bloc pour la construction des images Docker
├── trigger-project.yml  # Bloc pour le trigger inter-projets
├── setup-project.yml    # Bloc pour la configuration des projets
├── clean-log.yml        # Bloc pour le nettoyage des logs gitlab
├── clean-registry.yml   # Bloc pour le nettoyage des images Docker
├── create-issue.yml     # Bloc pour la création d'issues Gitlab
├── launch-script.yml    # Bloc générique pour lancer un script cicd-script à la demande
├── detect-debt.yml      # Bloc pour la détection de dette technique interne
├── scan.yml             # Bloc pour l'analyse de qualité de code (SonarQube)
├── gitleaks.yml         # Bloc pour la détection de secrets (gitleaks)
```

### Fichier parent `.gitlab-ci.yml`

Le fichier parent est responsable de :

* Inclure les blocs YAML spécifiques pour chaque feature via `include`
* Definir le workflow général de la pipeline (changement de variables en fonction du type de lancement, création d'argument générique qu'on peut mettre en message de commit etc..)
* Définir un tag par défaut pour les runners gitlab.
* Définir les **stages** globaux de la pipeline (voir [Stages](#stages) plus bas)
* Définir le stage en commun entre toutes les features, la récupération du projet de script cicd-script :

```yaml
include:
  [...]
  - local: "features/build-docker.yml"
    rules:
      - exists:
          - features/build-docker.yml
  [...]

workflow:
  name: "$PIPELINE_DISPLAY_NAME"
  rules:
    # --------------------- 
    # --- NAME PIPELINE ---
    # ---------------------
    - if: $CI_PIPELINE_SOURCE =~ /^(schedule|web)/
      variables:
        PIPELINE_DISPLAY_NAME: "[$CI_PIPELINE_SOURCE] $CI_COMMIT_MESSAGE"
    [...]

default:
  tags:
    - $RUNNER_TAGS

stages:
  - prepare
  - build
  [...]

get-files-from-git:
  stage: prepare
  [...]
```

### Stages

Les jobs s'enchaînent grâce à leurs `needs` (et non grâce à l'ordre des stages) : les stages servent surtout à les regrouper dans l'interface GitLab. Un job sans `needs` attend en revanche la fin de tous les stages précédents ; les jobs qui n'ont besoin de rien d'autre que le repo (scans) ont donc `needs: []` pour démarrer immédiatement.

| Stage | Jobs | Feature |
|-------|------|---------|
| `prepare` | `get-files-from-git` | Commun à toutes les features |
| `build` | `fill-jsonnet-from-python`, `convert-jsonnet-to-json`, `execute-jobs-from-yaml` | build-docker |
| `quality` | `scan-sonarqube`, `gitleaks-scan` | scan, gitleaks |
| `trigger` | `trigger` | trigger-project |
| `maintenance` | `clean-job`, `clean-registry-*`, `create-issue`, `detect-debt-*`, `launch-python-script` | clean-log, clean-registry, create-issue, detect-debt, launch-script |
| `admin` | `setup-*-project`, `*-init` | setup-project, init |

Pour ajouter une feature, la ranger dans le stage qui correspond à son rôle plutôt que de créer un nouveau stage.

### Configuration par projet

Avant les features, le `.gitlab-ci.yml` inclut les fichiers de variables du projet `cicd-configuration` (`CICD_CONFIGURATION_PATH`, branche `main`), dans cet ordre :

| Ordre | Fichier | S'applique à |
|-------|---------|--------------|
| 1 | `configuration/.default-conf.yml` | Tous les projets |
| 2 | `configuration/by_project/.<CI_PROJECT_NAME>-conf.yml` | Tous les projets qui portent ce **nom**, quel que soit leur groupe |
| 3 | `configuration/by_path/<CI_PROJECT_NAMESPACE>/.<CI_PROJECT_NAME>-conf.yml` | Un seul projet, identifié par son **chemin complet** |

Chaque fichier est optionnel (il n'est inclus que s'il existe). Quand une même variable est définie dans plusieurs fichiers, c'est le dernier inclus qui l'emporte : `by_path` > `by_project` > `.default-conf.yml`.

`by_project` suffit tant que le nom du projet est unique. `by_path` sert quand plusieurs projets ont le même nom (ex : `PROJET/sources` et `AUTRE/sources`) :

- `configuration/by_project/.sources-conf.yml` s'applique à **tous** les projets nommés `sources` ;
- `configuration/by_path/<chemin du groupe>/PROJET/.sources-conf.yml` ne s'applique qu'à `PROJET/sources`.

`CI_PROJECT_NAMESPACE` est le chemin complet du groupe parent tel qu'il apparaît dans l'URL GitLab du projet, groupes parents compris (ex : `groupe/sous-groupe/PROJET` pour le projet `groupe/sous-groupe/PROJET/sources`). Il est aussi affiché dans les logs du job `get-files-from-git`.

### Blocs YAML modulaires

Chaque fichier inclus contient uniquement la configuration nécessaire à une fonctionnalité particulière :

* **init.yml** : Initialisation des images python-process et jsonnet-folder pour pouvoir lancer les différentes features (voir [docs/INIT.md](docs/INIT.md)).
* **build-docker.yml** : Construction des images Docker, push vers le registry et deploiement dans l'infra (voir [docs/modules/BUILD_DOCKER.md](docs/modules/BUILD_DOCKER.md)).
* **trigger-project.yml** : Trigger d'un projet vers un autre projet Gitlab ou tout autre webhooks (ex: Jenkins) (voir [docs/modules/TRIGGER.md](docs/modules/TRIGGER.md)).
* **setup-project.yml** : Configuration des projets pour permettre les différentes features (Build, Trigger et Scan) (voir [docs/modules/SETUP_PROJECT.md](docs/modules/SETUP_PROJECT.md) pour le fonctionnement du module, [docs/setup/BUILD_SETUP.md](docs/setup/BUILD_SETUP.md), [docs/setup/TRIGGER_SETUP.md](docs/setup/TRIGGER_SETUP.md) et [docs/setup/SCAN_SETUP.md](docs/setup/SCAN_SETUP.md) pour la configuration d'un projet).
* **clean-log.yml** : Nettoyage périodique des logs pour éviter la saturation de Gitlab (voir [docs/modules/CLEAN_LOG.md](docs/modules/CLEAN_LOG.md)).
* **clean-registry.yml** : Nettoyage périodique des images en fonction de leurs statuts (Image plus build ou image d'une branche dev supprimé) pour éviter la saturation de Gitlab (voir [docs/modules/CLEAN_REGISTRY.md](docs/modules/CLEAN_REGISTRY.md)).
* **create-issue.yml** : Création automatique d'une ou plusieurs issues Gitlab depuis la pipeline, avec assignation, template de description et méta-issue optionnelle (voir [docs/modules/CREATE_ISSUE.md](docs/modules/CREATE_ISSUE.md)).
* **launch-script.yml** : Bloc générique permettant de lancer n'importe quel module Python pouvant utiliser cicd-script à la demande, via les variables `LAUNCH_SCRIPT_MODULE` / `LAUNCH_SCRIPT_ARGUMENT` / `LAUNCH_SCRIPT_OTHER_COMMANDS` (voir [docs/modules/LAUNCH_SCRIPT.md](docs/modules/LAUNCH_SCRIPT.md) pour le fonctionnement du module et [docs/setup/LAUNCH_SCRIPT_SETUP.md](docs/setup/LAUNCH_SCRIPT_SETUP.md) pour la configuration d'un projet).
* **detect-debt.yml** : Analyse périodique de la dette technique du à la montée de version de certaines images et pas de leurs enfants (voir [docs/modules/DETECT_DEBT.md](docs/modules/DETECT_DEBT.md)).
* **scan.yml** : Analyse de qualité de code du projet via SonarQube (voir [docs/modules/SCAN.md](docs/modules/SCAN.md)).
* **gitleaks.yml** : Détection des secrets committés par erreur (tokens, mots de passe, clés...) via gitleaks (voir [docs/modules/GITLEAKS.md](docs/modules/GITLEAKS.md)).

### Arguments spécifiques

Si le message du commit (et donc la variable `$CI_COMMIT_MESSAGE`) contient l'un (ou plusieurs) des arguments listés ci-dessous, le comportement de la pipeline associée est modifié.

| Objectif          | Argument                  | Comportement associé |
|-------------------|---------------------------|----------------------|
| Opérationnel      | `no-build`                | Ne lance pas de pipeline associé au push |
| Reconstruction    | `ci-all`                  | Reconstruction de toutes les images, que les fichiers associés à ces images aient été modifiés ou non|
| Reconstruction    | `ci-check-before-push`    | Push de l'image vers la registry seulement si des différences entre l'image construite par la pipeline et l'image déjà stockée sur la registry sont constatées|
| Reconstruction    | `ci-branch-dev`    | Lance la reconstruction sur la pipeline de développement|
| Nettoyage         | `ci-clean-dev`            | Lancement du module clean-registry mode suppression image dev|
| Nettoyage         | `ci-clean-nobuild`        | Lancement du module clean-registry mode suppression image no build|
| Nettoyage         | `ci-clean-log`        | Lancement du module clean-log |

#### Exemples
- `ci-all && ci-check-before-push` pour forcer la reconstruction de toutes les images, sans les push si c'est inutile
- `ci-clean-nobuild && ci-clean-dev` pour nettoyer les anciennes versions, et les version de dev obselète.

### Features pilotées par `LAUNCH_FEATURE`

D'autres features ne se basent pas sur le message de commit mais sur le contenu (une liste séparée par des virgules) de la variable `$LAUNCH_FEATURE` :

| Feature | Valeur `LAUNCH_FEATURE` | Comportement associé |
|---------|--------------------------|-----------------------|
| Trigger | `trigger-project` | Déclenche les pipelines des projets configurés pour être trigger (valeur incluse par défaut) |
| Détection de dette | `detect-debt` | Lance l'analyse de dette technique interne |
| Scan de code | `scan-code-sonarqube` | Lance l'analyse SonarQube du projet |
| Détection de secrets | `scan-secret-gitleaks` | Lance la recherche de secrets avec gitleaks |
| Création d'issue | `create-issue` | Crée les issues Gitlab décrites par les variables `CREATE_ISSUE_ISSUE_*` |
| Script à la demande | `launch-python-script` | Lance le module Python (ou la commande) défini par `LAUNCH_SCRIPT_MODULE` |

La valeur par défaut de `LAUNCH_FEATURE` est `build-docker,trigger-project,scan-code-sonarqube,scan-secret-gitleaks` : les autres features doivent être ajoutées explicitement (en configuration projet ou ponctuellement via **Run pipeline**), par exemple `LAUNCH_FEATURE: "build-docker,trigger-project,create-issue"`.

En plus du scan à chaque push, la détection de secrets tourne via un **schedule obligatoire** `scangitleaks` qui scanne tout l'historique de chaque projet tous les 5 jours (voir [docs/modules/GITLEAKS.md](docs/modules/GITLEAKS.md#schedule-obligatoire-scangitleaks)).

---

## Avantages de cette approche

* **Lisibilité** : Chaque bloc est autonome et facile à comprendre.
* **Réutilisabilité** : Les blocs peuvent être inclus dans d’autres pipelines.
* **Maintenance simplifiée** : Modification d’une feature sans toucher au reste de la pipeline.
* **Scalabilité** : Ajout facile de nouvelles fonctionnalités CI/CD en créant simplement un nouveau bloc YAML.

---

## Bonnes pratiques

* Nommer les fichiers YAML de manière claire (`build-*.yml`, `clean-*.yml`).
* Garder les jobs modulaires et indépendants.
* Documenter chaque bloc YAML avec un commentaire sur son rôle.
