# Configuration d'un projet pour lancer un script Python

Ce tutoriel explique comment faire tourner un script Python d'un projet GitLab via la feature **launch-script** (lancement ponctuel ou planifié par schedule).

Pour comprendre le fonctionnement du job et la liste des variables, voir [docs/modules/LAUNCH_SCRIPT.md](../modules/LAUNCH_SCRIPT.md). Pour comprendre comment le module de setup applique la configuration (CI, variables, schedules), voir [docs/modules/SETUP_PROJECT.md](../modules/SETUP_PROJECT.md).

## Vue d'ensemble

| Étape | Où | Quoi |
|-------|----|------|
| 1 | Projet cible | Écrire le script (`script/main.py`) et, si besoin, `script/requirements_pip` |
| 2 | `cicd-configuration/setup` | Déclarer le projet pour qu'il utilise `cicd-yaml` |
| 3 | `cicd-configuration/configuration/by_project` | Définir les variables `LAUNCH_SCRIPT_*` |
| 4 | `cicd-configuration/setup` | (Optionnel) Ajouter un schedule pour lancer le script automatiquement |
| 5 | GitLab | Tester avec **Run pipeline** |

## 1. Préparer le script dans le projet

### Arborescence

```
mon-projet/
└── script/
    ├── main.py            # Point d'entrée (LAUNCH_SCRIPT_MODULE: "main")
    ├── requirements_pip   # Dépendances pip en plus de l'image python-process (optionnel)
    └── ...                # Autres modules du script
```

- Le dossier (`script` ici) correspond à `LAUNCH_SCRIPT_FOLDER`.
- Le script est lancé avec `python3 -m <LAUNCH_SCRIPT_MODULE>` **depuis ce dossier** : les imports se font donc depuis `script/` (ex : `from mes_fonctions import ...`).
- Le dossier `script/lib/` est rempli par le job avec la librairie `cicd-script`. Mieux vaut ne pas en avoir dans le repo : s'il existe, les fichiers de même nom (`helper.py`, `gitlab_helper.py`, `global_vars.py`, `__init__.py`) sont écrasés. Il peut être utile d'ajouter `script/lib/` au `.gitignore` pour les tests en local.
- `requirements_pip` est optionnel (si absent, aucune installation n'est faite). C'est un fichier classique au format `requirements.txt`. Les paquets `requests`, `pyyaml`, `environs` et `jinja2` sont déjà présents dans l'image : inutile de les y remettre.

### Exemple de `main.py`

```python
# coding=utf-8
import argparse
import logging
import os

# Disponible grâce à la copie de cicd-script/lib (GET_CICD_MODULE != "no")
from lib.gitlab_helper import get_issues
from lib.helper import send_message

logging.basicConfig(level=os.environ.get("MON_SCRIPT_LOG_LEVEL", "INFO"))
logger = logging.getLogger(__name__)

def main(args):
    issues = get_issues(args.token, args.project_id, {"state": "opened"})
    logger.info(f"{len(issues)} issues ouvertes")
    send_message(os.environ.get("MON_SCRIPT_CHANNEL_URL", ""), f"{len(issues)} issues ouvertes")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--project-id", required=True)
    parser.add_argument("--token", required=True)
    main(parser.parse_args())
```

Les arguments sont fournis par `LAUNCH_SCRIPT_ARGUMENT`, et toute variable CI/CD est lisible via `os.environ`.

## 2. Déclarer le projet dans cicd-configuration

Le projet doit utiliser `cicd-yaml` comme fichier de CI. C'est le module setup-project qui s'en charge, à condition que le projet soit déclaré dans un fichier de setup (`setup/triggers.yml`, `setup/build.yml`, ou un fichier dédié dans `setup/by_project/`).

Si le projet est déjà déclaré (pour du build ou du trigger), passer à l'étape suivante. Sinon, l'ajouter dans la liste `projects:` d'un trigger existant, par exemple celui de la documentation (voir [TRIGGER_SETUP.md](TRIGGER_SETUP.md) pour le détail des champs) :

```yaml
# cicd-configuration/setup/by_project/gmcd-docs.triggers.yml
- type: "gitlab"
  name: "gmcd-docs"
  id: 25419
  projects:
    - name: "mon-projet"
      id: 12345
      trigger_files: ["*.md", "docs_parameters.yml"]
      branchs_mapping:
        main: 'prod'
```

## 3. Définir les variables du script

Créer (ou compléter) le fichier `cicd-configuration/configuration/by_project/.<nom-du-projet>-conf.yml`. Le nom doit correspondre exactement à `CI_PROJECT_NAME`, car il est inclus automatiquement par le `.gitlab-ci.yml` de `cicd-yaml`.

```yaml
variables:
  # Dossier et module à lancer
  LAUNCH_SCRIPT_FOLDER: "script"
  LAUNCH_SCRIPT_MODULE: "main"
  LAUNCH_SCRIPT_ARGUMENT: "--project-id ${CI_PROJECT_ID} --token ${CICD_GMCD_TOKEN}"

  # Optionnel : commandes shell lancées après le script
  # LAUNCH_SCRIPT_OTHER_COMMANDS: "ls -la && cat rapport.txt"

  # Optionnel : désactiver la copie de cicd-script/lib
  # GET_CICD_MODULE: "no"

  # Optionnel : ne pas installer requirements_pip même s'il existe
  # INSTALL_OTHER_PACKAGE: "no"

  # Variables propres au script
  MON_SCRIPT_LOG_LEVEL: "DEBUG"
```

> **Attention :** ne pas ajouter `launch-python-script` dans `LAUNCH_FEATURE` ici, sinon le script se lancera à chaque push. Le lancement passe par un schedule (étape 4) ou par **Run pipeline** (étape 5).

> **Secrets :** les tokens et URLs de webhook ne doivent pas être écrits en clair dans ce fichier. Les déclarer en variables CI/CD masquées (projet ou groupe) et y faire référence (ex : `${CICD_GMCD_TOKEN}`).

## 4. (Optionnel) Planifier le script avec un schedule

Dans le fichier de setup où le projet est déclaré (étape 2), ajouter un bloc `schedule:` :

```yaml
  projects:
    - name: "mon-projet"
      id: 12345
      # ...
      schedule:
        - type: "launchscript"
          branch: "refs/heads/main"
          cron: "0 10 * * *"
          description: "Lancement quotidien de mon script"
          variables:
            LAUNCH_FEATURE: "launch-python-script"
```

| Champ | Obligatoire | Rôle |
|-------|-------------|------|
| `type:` | oui | Identifiant du schedule. `launchscript` n'a pas de valeurs par défaut (ce n'est pas un type prédéfini) : `cron` et `description` doivent donc être renseignés |
| `branch:` | non | Branche sur laquelle lancer le schedule (défaut : branche par défaut du projet) |
| `cron:` | oui | Fréquence de lancement (ex : `0 0 1 * *` = le 1er de chaque mois à minuit) |
| `description:` | oui | Nom du schedule dans GitLab (préfixé automatiquement par `[<branche>]`). Sert aussi de clé : changer la description crée un nouveau schedule |
| `variables:` | oui | Doit contenir `LAUNCH_FEATURE: "launch-python-script"`. D'autres variables peuvent être ajoutées pour surcharger la configuration du projet |

Plusieurs schedules peuvent cohabiter (ex : un par branche ou avec des variables différentes) en ajoutant plusieurs entrées dans `schedule:`.

Une fois la modification mergée dans `cicd-configuration`, le job `setup-schedule-project` crée ou met à jour le schedule sur le projet.

## 5. Tester

1. Dans le projet cible, aller dans **Build > Pipelines > Run pipeline**.
2. Choisir la branche et ajouter la variable `LAUNCH_FEATURE` = `launch-python-script`.
3. Lancer la pipeline : les jobs `get-files-from-git` puis `launch-python-script` doivent apparaître. La commande exacte lancée est affichée dans les logs (`python3 -m ...`).

Une fois le schedule créé (étape 4), il peut aussi être lancé à la main depuis **Build > Pipeline schedules** (bouton ▶).

## Exemple complet : grafana-backup-dashboard

Sauvegarde mensuelle `prod` → `legacy` d'un repo de dashboards Grafana, sans librairie `cicd-script`.

`cicd-configuration/configuration/by_project/.grafana-backup-dashboard-conf.yml` :

```yaml
variables:
  LAUNCH_SCRIPT_FOLDER: "script"
  LAUNCH_SCRIPT_MODULE: "main"
  LAUNCH_SCRIPT_ARGUMENT: "--project-id ${CI_PROJECT_ID} --token ${CICD_GMCD_TOKEN}"

  GIT_SYNC_SOURCE_BRANCH: "prod"
  GIT_SYNC_TARGET_BRANCH: "legacy"
  GIT_SYNC_MERGE_COMMIT_MESSAGE: "[monitoring#35] backup auto prod -> legacy"
  GIT_SYNC_CLEAN_COMMIT_MESSAGE: "[monitoring#35] retrait ci après sync"
  CLEAN_CI_FILE: "yes"
  GET_CICD_MODULE: "no"
```

`cicd-configuration/setup/by_project/gmcd-docs.triggers.yml` :

```yaml
    - name: "grafana-backup-dashboard"
      id: 30152
      trigger_files: ["*.md", "docs_parameters.yml"]
      branchs_mapping:
        main: 'prod'
      schedule:
      - type: "launchscript"
        branch: "refs/heads/main"
        cron: "0 0 1 * *"
        description: "Monthly backup prod -> legacy for gitsync-grafana"
        variables:
          LAUNCH_FEATURE: "launch-python-script"
```

## Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| Le job `launch-python-script` n'apparaît pas | `LAUNCH_FEATURE` ne contient pas `launch-python-script` (attention : `launch-script` ne suffit pas) | Corriger la valeur dans le schedule ou dans **Run pipeline** |
| Aucun job ne se lance sur le projet | Le projet n'utilise pas `cicd-yaml`, ou le message de commit contient `no-build` | Vérifier l'étape 2 et le chemin du fichier CI dans **Settings > CI/CD** |
| Un fichier du `lib/` du projet a un contenu inattendu | Il porte le même nom qu'un fichier de `cicd-script/lib/` et a été écrasé | Renommer le fichier ou mettre `GET_CICD_MODULE: "no"` |
| Log « Pas de fichier requirements_pip » et `No module named <paquet>` | Le fichier n'est pas dans `LAUNCH_SCRIPT_FOLDER` (ou mal nommé) | Le placer dans `<LAUNCH_SCRIPT_FOLDER>/requirements_pip` |
| `No module named ...` | `LAUNCH_SCRIPT_MODULE` mal écrit, ou chemin relatif à la mauvaise racine | Le module est résolu depuis `LAUNCH_SCRIPT_FOLDER` (ex : `main` et non `script.main`) |
| `No module named 'lib'` | `GET_CICD_MODULE` est à `no` alors que le script importe `lib` | Retirer `GET_CICD_MODULE: "no"` |
| Les variables du fichier `by_project` ne sont pas prises en compte | Le nom du fichier ne correspond pas à `CI_PROJECT_NAME` | Renommer en `.<CI_PROJECT_NAME>-conf.yml` |

---
