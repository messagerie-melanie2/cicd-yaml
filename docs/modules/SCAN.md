# Scan module (SonarQube)

Lance une analyse de qualité de code SonarQube sur le projet à l'aide de `sonar-scanner-cli`. Contrairement aux autres modules, cette feature ne fait appel à aucun script Python de `cicd-script` : c'est un simple appel à l'exécutable `sonar-scanner` dans une image dédiée.

## Projets concernés

| Projet | Rôle dans la feature |
|------|----------------------|
| `cicd-yaml` | Définition du job (`features/scan.yml`) |
| `cicd-docker` | Image utilisée par le job (`cicd-docker/sonar-scanner-cli`, basée sur `sonarsource/sonar-scanner-cli`), qui embarque un `sonar-project.properties` par défaut et le script `before_scan.sh` |
| Projet cible | Fichier `sonar-project.properties` à la racine (optionnel, voir plus bas), qui décrit ce qui doit être analysé |
| Instance SonarQube | Reçoit l'analyse et calcule le Quality Gate |

## Fonctionnement

Le job `scan-sonarqube` (stage `detect`) :

1. Démarre l'image `${REGISTRY_DOMAIN}${CICD_NAMESPACE}${SCAN_SONARQUBE_PATH}:${SCAN_SONARQUBE_TAG}` (image `sonar-scanner-cli`).
2. Met en cache le répertoire `${CI_PROJECT_DIR}/.sonar` (`SONAR_USER_HOME`) ainsi que `sonar-scanner/` via une clé de cache par branche (`sonar-cache-$CI_COMMIT_REF_SLUG`), pour accélérer les analyses successives.
3. Récupère tout l'historique git du projet (`GIT_DEPTH: "0"`), nécessaire à SonarQube pour l'analyse du "new code" et le blame.
4. `before_script: source /entrypoint/before_scan.sh` — voir [Fichier `sonar-project.properties` par défaut](#fichier-sonar-projectproperties-par-défaut) ci-dessous : ce script garantit qu'un `sonar-project.properties` valide existe à la racine du projet avant l'analyse, sans que chaque projet ait besoin d'en committer un.
5. Exécute `sonar-scanner -Dsonar.host.url="${SONAR_HOST_URL}"`, qui lit automatiquement le fichier `sonar-project.properties` présent à la racine du projet cible (sources à analyser, exclusions, langage, clé/nom du projet SonarQube, etc.).
6. Le job est `allow_failure: true` : un échec du scan (ou un Quality Gate non passé) ne fait pas échouer la pipeline globale.

## Fichier `sonar-project.properties` par défaut

L'image `sonar-scanner-cli` (`cicd-docker/sonar-scanner-cli`) embarque, en plus de `sonar-scanner`, deux fichiers ajoutés par son `Dockerfile` :

| Fichier dans l'image | Rôle |
|---|---|
| `/usr/src/sonar-project.properties.default` | Modèle de `sonar-project.properties` par défaut, commun à tous les projets |
| `/entrypoint/before_scan.sh` | Script lancé en `before_script` par le job `scan-sonarqube` |

Contenu du modèle par défaut :

```properties
sonar.projectKey=projectname
sonar.qualitygate.wait=true
```

Au démarrage du job, `before_scan.sh` :

1. Vérifie si `sonar-project.properties` existe déjà à la racine du projet (`${CI_PROJECT_DIR}`).
   - **S'il existe** (committé par le projet), il est utilisé tel quel, sans aucune modification — c'est le moyen de personnaliser l'analyse (sources, exclusions, langage, etc.) pour un projet ayant des besoins spécifiques.
   - **S'il est absent**, le script copie le modèle par défaut à sa place, puis calcule `sonar.projectKey` à partir de `CI_PROJECT_PATH` (le chemin complet du projet GitLab, avec les `/` remplacés par des `:`, ex : `groupe/sous-groupe/projet` → `groupe:sous-groupe:projet`) et le substitue dans le fichier copié.
2. Affiche dans les logs du job la valeur de `sonar.projectKey` finalement utilisée.

Ainsi, la plupart des projets n'ont **rien à committer** pour être scannés : le `sonar-project.properties` par défaut (avec une clé de projet dérivée automatiquement du chemin GitLab, et l'attente du Quality Gate activée) suffit. Un projet ne doit committer son propre `sonar-project.properties` que s'il a un besoin spécifique (exclusions particulières, langage à forcer, sources hors racine, etc.), auquel cas ce fichier prend entièrement le dessus sur le modèle par défaut.

## Prérequis

- Avoir la variable `SONAR_HOST_URL` définie (URL de l'instance SonarQube cible).
- Selon la configuration de l'instance SonarQube, une authentification peut être nécessaire (variable d'environnement `SONAR_TOKEN`, reconnue nativement par `sonar-scanner`) — elle n'est pas gérée explicitement par le job et doit être ajoutée en variable CI/CD du projet si besoin (voir [docs/setup/SCAN_SETUP.md](../setup/SCAN_SETUP.md) pour l'automatiser via le module setup-project).
- Avoir un fichier `sonar-project.properties` à la racine du projet cible **uniquement** si les valeurs par défaut décrites ci-dessus ne conviennent pas, par exemple :

```properties
sonar.projectKey=snum:detn:groupe:projet
sonar.projectName=mon-projet
sonar.projectVersion=1.0
sonar.sources=.
sonar.sourceEncoding=UTF-8
sonar.exclusions=**/node_modules/**,**/vendor/**,**/dist/**,**/.git/**,**/.sonar/**
```

## Déclenchement

Le job ne se lance que si `LAUNCH_FEATURE` contient `scan-code-sonarqube` :

```
LAUNCH_FEATURE = scan-code-sonarqube
```

Ceci peut être fait ponctuellement via **CI/CD > Pipelines > Run pipeline**, ou ajouté durablement dans la configuration du projet (`cicd-configuration/configuration/by_project/.<projet>-conf.yml`) :

```yaml
variables:
  LAUNCH_FEATURE: "build-docker,trigger-project,scan-code-sonarqube"
```

## Résultat

Le résultat de l'analyse (issues, couverture, duplications, Quality Gate) est disponible directement dans l'instance SonarQube, sous le projet identifié par `sonar.projectKey`.

---
