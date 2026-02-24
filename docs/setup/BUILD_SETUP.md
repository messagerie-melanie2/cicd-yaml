# Configuration d'un projet pour build des dockers 

## Configuration d'un projet pour utiliser le module build-docker

### Explication

Pour configurer un projet afin qu'il puisse build des images dockers, dans le dossier `cicd-configuration/setup/`, vous pouvez créer un fichier dedié au projet dans `setup/by_project` (ex : `setup/by_project/mon-project.build.yml`) ou tout simplement le mettre dans le fichier `build.yml`. Ce fichier doit respecter la forme suivante :

```yaml
- name: "NOM_DE_MON_PROJET_A_SETUP"
  id : ID_DE_MON_PROJET_A_SETUP
```

### Arguments possibles

| Argument     | Valeurs possibles     | Obligatoire | Comportement associé               |
|--------------|-----------------------|-------------|------------------------------------|
| `name:`      | Nom de mon projet/pipeline | oui         |Nom du projet gitlab à configurer pour build des images dockers|
| `id:`      | ex : 22443 | oui pour gitlab        |ID du projet gitlab à configurer|
| `instance_to_allow::`      | Liste de champ YAML | oui         |Instance (Groupe ou Project) à mettre dans les allowlists de la CI pour permettre la communication entre elles|
| `instance_to_allow:  instance_type:`      | 'project' (par défaut) ou 'group'  | oui         |Type de l'instance à autoriser|
| `instance_to_allow:  name:`      | Nom de mon instance  | oui         |Nom de l'instance à autoriser|
| `instance_to_allow:  id:`      | ex : 22444  | oui         |Id de l'instance à autoriser|
| `schedule: `      | Liste  | non         |Permet de choisir quels types de schedules configurer pour le projet |
| `schedule:  type:`      | ex : 'buildall'  | oui         |Type de schedule à configurer|
| `schedule:  branch:`      | ex : 'main'  | non         |La branche sur lequel lancer le schedule |
| `schedule:  cron:`      | ex: '0 20 * * *'  | non         |Le cron du schedule|
| `schedule:  variables:`      | Liste de champ YAML | non         |Remplace ou crée des variables par défaut du schedule, doit être au format "MON_NOM_DE_VARIABLE" : "MA_VALEUR"|

### Exemple

```yaml
- name: "cicd-docker"
  id: 27188
  instance_to_allow:
    - instance_type : 'group'
      name : 'CICD'
      id : 4
  schedule:
    - type : "buildall"
      branch: "refs/heads/preprod"
      cron: "0 20 * * *"
    - type : "buildall"
      branch: "refs/heads/prod"
      cron: "0 22 * * *"
    - type : "cleanlog"
      variables:
        CLEANLOG_WEEKS_LIMIT: 1
```

---