# Configuration d'un projet pour trigger d'autres projets

## Configuration d'un trigger

### Configuration dans le projet cicd-configuration

Pour comprendre comment marche le trigger et ces fonctionnalités, voir la documentation dans cicd-yaml.

#### Explication

Pour configurer un projet afin qu'il puisse trigger un autre projet, dans le dossier `cicd-configuration/setup`, vous pouvez créer un fichier dedié au projet dans `setup/by_project` (`ex : setup/by_project/mon-project.triggers.yml`) ou tout simplement le mettre dans le fichier `triggers.yml`. Ce fichier doit respecter la forme suivante :

```yaml
- type: "gitlab" #ou "jenkins"
  name: "NOM_DE_MON_PROJET_A_TRIGGER" #ou "NOM_DE_MA_PIPELINE_JENKINS"
  id : ID_DE_MON_PROJET_A_TRIGGER #'si type == gitlab'
  projects:
    - name: "NOM_DE_MON_PROJET"
      id: ID_DE_MON_PROJET
```

#### Arguments possibles

| Argument     | Valeurs possibles     | Type autorisé            | Obligatoire | Comportement associé               |
|--------------|-----------------------|--------------------------|-------------|------------------------------------|
| `type:`      | "gitlab" ou "jenkins" seulement | **gitlab** / **jenkins** | oui         |Type du projet à trigger            |
| `name:`      | Nom de mon projet/pipeline | **gitlab** / **jenkins** | oui         |Nom du projet gitlab à trigger où nom de la pipeline jenkins à trigger|
| `id:`      | ex : 22443 | **gitlab** | oui pour gitlab        |ID du projet gitlab à trigger|
| `projects:`      | Liste de champ YAML | **gitlab** / **jenkins** | oui         |Projets à configurer pour trigger le projet ou la pipeline correspondante|
| `projects:  name:`      | Nom de mon projet  | **gitlab** / **jenkins** | oui         |Nom du projet à configurer|
| `projects:  id:`      | ex : 22444  | **gitlab** / **jenkins** | oui         |Id du projet à configurer|
| `projects:  trigger_files:`      | Liste  | **gitlab** / **jenkins** | non         |Permet de choisir pour quel fichier le trigger est lancé|
| `projects:  branchs_only_trigger:`      | Liste  | **gitlab** / **jenkins** | non         |Permet de choisir pour quel branche le trigger est lancé|
| `projects:  branchs_mapping:`      | Liste de champ YAML  | **gitlab** / **jenkins** | non         |Permet de définir un mapping de branches lors du trigger|
| `projects:  change_ci:`      | Booléan  | **gitlab** / **jenkins** | non         |Si mis à False, empêche la configuration du CI/CD configuration file |
| `projects:  focus_trigger:`      | Booléan  | **gitlab** | non         |Permet de choisir si le trigger doit être précis en fonction des changements des fichiers|
| `projects:  additional_params:`      | JSON ex : `{"param1": "value1", "param2": "value2"}`  | **jenkins** | non         |Permet de rajouter des paramètres pour sa pipeline|
| `projects: schedule: `      | Liste  | non         |Permet de choisir quels types de schedules configurer pour le projet |
| `projects: schedule:  type:`      | ex : 'buildall'  | oui         |Type de schedule à configurer|
| `projects: schedule:  branch:`      | ex : 'main'  | non         |La branche sur lequel lancer le schedule |
| `projects: schedule:  cron:`      | ex: '0 20 * * *'  | non         |Le cron du schedule|
| `projects: schedule:  variables:`      | Liste de champ YAML | non         |Remplace ou crée des variables par défaut du schedule, doit être au format "MON_NOM_DE_VARIABLE" : "MA_VALEUR"|
| `dependencies:`      | Liste de champ YAML | **gitlab** | non         |Projets à mettre dans la liste d'autorisation des projets à configurer pour le bon fonctionnement du process|
| `dependencies: type:`      | "gitlab" seulement | **gitlab** | non         |Type du projet à mettre dans la liste d'autorisation|
| `dependencies: name:`      | Nom de mon projet | **gitlab** | non         |Nom du projet à mettre dans la liste d'autorisation|
| `dependencies: id:`      | ex : 22445  | **gitlab** | non         |Id du projet à mettre dans la liste d'autorisation|

#### Exemple

```yaml
- type: "gitlab"
  name: "mel-docker"
  id: 11665
  dependencies:
    - type: "gitlab"
      name: "gmcd-docs"
      id: 25419
  projects:
    - name: "cicd-trigger"
      id: 22822
      trigger_files: ["triggers.yml","*.py"]
      focus_trigger: True
      branchs_only_trigger: ["main","ht-reworkv2"]
      branchs_mapping: 
        main: 'prod'
        ht-reworkv2: 'preprod'
- type: "jenkins"
  name: "Gitlab Trigger Configuration"
  projects:
    - name: "cicd-trigger"
      id: 22822
      trigger_files: ["triggers.yml"]
      branchs_only_trigger: ["main"]
      branchs_mapping: 
        main: 'prod'
```

### Configuration du projet qui trigger (Optionnel)

#### Explication

Il est possible de rajouter un fichier `trigger_parameters.yml` à la racine du projet afin de pouvoir configurer directement vos triggers.

**Pré-requis** : Avoir au préalable fait l'étape de Mise en place dans le projet cicd-trigger.

Votre fichier `trigger_parameters.yml` peut avoir la forme suivante :

```yaml
- name: "NOM_DE_MON_PROJET_A_TRIGGER" #ou "NOM_DE_MA_PIPELINE_JENKINS"
  type: "gitlab" #ou "jenkins"
  mon_option: ma_valeur
```

Les options pouvant être rajoutés sont les mêmes que dans le projet cicd-trigger :  

- trigger_files
- branchs_only_trigger
- branchs_mapping
- focus_trigger

**ATTENTION :** Le nom du projet à trigger doit être exactement le même que celui assigné dans le projet cicd-trigger. 
#### Exemple

```yaml
- name: "mel-docker"
  type: "gitlab"
  trigger_files: ["triggers.yml","*.py"]
  focus_trigger: True
  branchs_only_trigger: ["main","ht-reworkv2"]
  branchs_mapping: 
    main: 'prod'
    ht-reworkv2: 'preprod'
- name: "Gitlab Trigger Configuration"
  type: "jenkins"
  trigger_files: ["triggers.yml"]
  branchs_only_trigger: ["main"]
  branchs_mapping: 
    main: 'prod'
```

---