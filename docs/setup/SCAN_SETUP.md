# Configuration d'un projet pour scanner du code (SonarQube)

## Configuration d'un projet pour utiliser le module scan

### Explication

Pour comprendre comment marche le scan, voir [docs/modules/SCAN.md](../modules/SCAN.md). Pour comprendre comment le module de setup applique cette configuration (variable de token SonarQube, `ci_config_path`), voir [docs/modules/SETUP_PROJECT.md](../modules/SETUP_PROJECT.md).

Pour configurer un projet afin qu'il puisse être scanné par SonarQube, dans le dossier `cicd-configuration/setup/`, vous pouvez créer un fichier dédié au projet dans `setup/by_project` (ex : `setup/by_project/mon-project.scan.yml`) ou tout simplement le mettre dans le fichier `scan.yml`. Ce fichier doit respecter la forme suivante :

```yaml
- name: "NOM_DE_MON_PROJET_A_SETUP"
  id: ID_DE_MON_PROJET_A_SETUP
  type:
    - "SONARQUBE"
```

### Arguments possibles

| Argument     | Valeurs possibles     | Obligatoire | Comportement associé               |
|--------------|-----------------------|-------------|------------------------------------|
| `name:`      | Nom de mon projet/pipeline | oui         |Nom du projet gitlab à configurer pour le scan|
| `id:`        | ex : 22443 | oui         |ID du projet gitlab à configurer|
| `type:`      | Liste de champ YAML, seule valeur reconnue actuellement : `"SONARQUBE"` | oui         |Type(s) de scan à activer pour le projet. Seul `SONARQUBE` déclenche un traitement, toute autre valeur est ignorée|

### Exemple

```yaml
- name: "rcube-sources"
  id: 22859
  type:
    - "SONARQUBE"
```

### Ce que fait le setup, et ce qu'il reste à faire manuellement

Pour un projet listé avec `type: ["SONARQUBE"]`, le job `setup-scan-project` :

- positionne `ci_config_path` sur le projet (comme pour build/trigger) ;
- écrit sur le projet la variable `SONAR_TOKEN` (nom configurable via `SETUP_SONAR_TOKEN_VARIABLE_NAME`), avec la valeur de la variable d'environnement de même nom lue sur le job de setup lui-même — il faut donc que `cicd-configuration` dispose déjà de cette variable (typiquement en variable de groupe/instance) pour pouvoir la propager.

Le setup ne configure **ni** `SONAR_HOST_URL`, **ni** le fichier `sonar-project.properties` à la racine du projet cible, **ni** `LAUNCH_FEATURE` : ces trois éléments restent à la charge du projet cible et sont décrits dans les sections **Prérequis** et **Déclenchement** de [docs/modules/SCAN.md](../modules/SCAN.md).

---
