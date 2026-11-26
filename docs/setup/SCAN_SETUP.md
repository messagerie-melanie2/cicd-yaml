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
  # Facultatif : clé du projet SonarQube (par défaut : chemin GitLab avec ':' à la place de '/')
  sonar_project_key: "snum:detn:groupe:projet"
```

### Arguments possibles

| Argument     | Valeurs possibles     | Obligatoire | Comportement associé               |
|--------------|-----------------------|-------------|------------------------------------|
| `name:`      | Nom de mon projet/pipeline | oui         |Nom du projet gitlab à configurer pour le scan|
| `id:`        | ex : 22443 | oui         |ID du projet gitlab à configurer|
| `type:`      | Liste de champ YAML, seule valeur reconnue actuellement : `"SONARQUBE"` | oui         |Type(s) de scan à activer pour le projet. Seul `SONARQUBE` déclenche un traitement, toute autre valeur est ignorée|
| `sonar_project_key:` | ex : `snum:detn:groupe:projet` | non |Clé du projet SonarQube. Par défaut, chemin GitLab du projet avec les `/` remplacés par des `:` (même règle que `before_scan.sh`). À renseigner uniquement si le projet committe son propre `sonar-project.properties` avec une autre clé|

### Exemple

```yaml
- name: "rcube-sources"
  id: 22859
  type:
    - "SONARQUBE"
```

### Ce que fait le setup, et ce qu'il reste à faire manuellement

Les projets ne sont plus créés automatiquement par SonarQube lors du premier scan : c'est le setup qui les crée via l'API SonarQube. Pour un projet listé avec `type: ["SONARQUBE"]`, le job `setup-scan-project` :

- positionne `ci_config_path` sur le projet (comme pour build/trigger) ;
- s'adresse à l'instance SonarQube de `SONAR_HOST_URL` (la même que celle du job de scan, définie dans `cicd-configuration/configuration/.default-conf.yml`), avec le token d'API dont le nom est donné par `SETUP_SONAR_API_TOKEN_VARIABLE_NAME` (ex : `SONAR_API_TOKEN_PROD` ou `SONAR_API_TOKEN_PREPROD`, variables CI/CD de `cicd-configuration`). Changer d'instance se fait donc uniquement dans `cicd-configuration` ;
- crée le projet sur SonarQube s'il n'existe pas encore (`api/projects/create`), avec la branche par défaut du projet GitLab comme branche principale ;
- si le projet existe déjà, aligne sa branche principale sur la branche par défaut GitLab (`api/project_branches/rename`) ;
- lie le projet SonarQube à son dépôt GitLab (`api/alm_settings/set_gitlab_binding`, avec l'`id` du projet GitLab), ou corrige le lien s'il pointe ailleurs. L'intégration GitLab utilisée est celle de `SETUP_SONAR_ALM_SETTING` (`gitlab` par défaut) ; si cette variable est vide, l'unique intégration GitLab disponible pour le projet (`api/alm_settings/list?project=<clé>`, qui ne demande que le droit *Administer* sur le projet). S'il y en a zéro ou plusieurs, le lien n'est pas fait (erreur dans les logs) ;
- écrit sur le projet la variable `SONAR_TOKEN` (nom fixe, attendu par le job de scan et par `sonar-scanner`), avec la valeur de la variable lue sur le job de setup lui-même, dont le nom est donné par `SETUP_SONAR_TOKEN_VARIABLE_NAME` (ex : `SONAR_TOKEN_PREPROD`) — il faut donc que `cicd-configuration` dispose déjà de cette variable (typiquement en variable de groupe/instance) pour pouvoir la propager.

Si `SONAR_HOST_URL` ou le token d'API est vide, le projet n'est pas créé sur SonarQube (erreur dans les logs du job) ; `SONAR_TOKEN` est tout de même écrite.

Le compte SonarQube du token d'API doit avoir le droit global *Create Projects*, et les droits *Browse* et *Administer* sur chaque projet. Ces deux derniers sont automatiques pour les projets créés par le setup, mais doivent être donnés à la main pour un projet déjà existant créé par un autre compte. Voir la section **Prérequis** de [docs/modules/SETUP_PROJECT.md](../modules/SETUP_PROJECT.md#prérequis).

Le setup ne configure **ni** `SONAR_HOST_URL` sur le projet (elle vient de la configuration), **ni** le fichier `sonar-project.properties` à la racine du projet cible, **ni** `LAUNCH_FEATURE` : ces éléments restent à la charge du projet cible et sont décrits dans les sections **Prérequis** et **Déclenchement** de [docs/modules/SCAN.md](../modules/SCAN.md).

---
