# Scan module (SonarQube)

Lance une analyse de qualité de code SonarQube sur le projet à l'aide de `sonar-scanner-cli`. Contrairement aux autres modules, cette feature ne fait appel à aucun script Python de `cicd-script` : c'est un simple appel à l'exécutable `sonar-scanner` dans une image dédiée.

## Projets concernés

| Projet | Rôle dans la feature |
|------|----------------------|
| `cicd-yaml` | Définition du job (`features/scan.yml`) |
| `cicd-docker` | Image utilisée par le job (`cicd-docker/sonar-scanner-cli`, basée sur `sonarsource/sonar-scanner-cli`) |
| Projet cible | Fichier `sonar-project.properties` à la racine, qui décrit ce qui doit être analysé |
| Instance SonarQube | Reçoit l'analyse et calcule le Quality Gate |

## Fonctionnement

Le job `scan-sonarqube` (stage `detect`) :

1. Démarre l'image `${REGISTRY_DOMAIN}${CICD_NAMESPACE}${SCAN_SONARQUBE_PATH}:${SCAN_SONARQUBE_TAG}` (image `sonar-scanner-cli`).
2. Met en cache le répertoire `${CI_PROJECT_DIR}/.sonar` (`SONAR_USER_HOME`) ainsi que `sonar-scanner/` via une clé de cache par branche (`sonar-cache-$CI_COMMIT_REF_SLUG`), pour accélérer les analyses successives.
3. Récupère tout l'historique git du projet (`GIT_DEPTH: "0"`), nécessaire à SonarQube pour l'analyse du "new code" et le blame.
4. Exécute `sonar-scanner -Dsonar.host.url="${SONAR_HOST_URL}"`, qui lit automatiquement le fichier `sonar-project.properties` présent à la racine du projet cible (sources à analyser, exclusions, langage, clé/nom du projet SonarQube, etc.).
5. Le job est `allow_failure: true` : un échec du scan (ou un Quality Gate non passé) ne fait pas échouer la pipeline globale.

## Prérequis

- Avoir un fichier `sonar-project.properties` à la racine du projet cible, par exemple :

```properties
sonar.projectKey=snum:detn:groupe:projet
sonar.projectName=mon-projet
sonar.projectVersion=1.0
sonar.sources=.
sonar.sourceEncoding=UTF-8
sonar.exclusions=**/node_modules/**,**/vendor/**,**/dist/**,**/.git/**,**/.sonar/**
```

- Avoir la variable `SONAR_HOST_URL` définie (URL de l'instance SonarQube cible).
- Selon la configuration de l'instance SonarQube, une authentification peut être nécessaire (variable d'environnement `SONAR_TOKEN`, reconnue nativement par `sonar-scanner`) — elle n'est pas gérée explicitement par le job et doit être ajoutée en variable CI/CD du projet si besoin.

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
