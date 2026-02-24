# Installation de la CI dans un Gitlab

## Mise en place d'un groupe CICD

Pour faciliter l'installation, il est conseillé de créer un groupe CICD dans votre Gitlab. Ce groupe sera composé de plusieurs projets :  
* **cicd-yaml** : Contient les yaml constituant la CI avec les templates de la configuration à mettre en place.  
* **cicd-script** : Contient les script utilisés par la CI pour ces features.  
* **cicd-docker** : Contient des images dockers utilisées par la CI pour ces features  
* **cicd-configuration** : Contient la configuration des projets permettant de personnaliser la CI et configurer automatiquement les autres projets pour qu'ils puissent utiliser la CI.  

## Initialisation de la CI

### Initialisation du projet cicd-configuration

Pour initialiser le projet cicd-configuration, il faut récupérer ce qu'il y'a dans cicd-configuration.template dans ce repo et créez votre propre configuration en fonction de votre environnement. Les fichiers devront avoir le même nom avec le .template en moins (ex : .default-conf.yml au lieu de .default-conf.template.yml).  

A noté que vous pouvez mettre la configuration dans n'importe quel projet tant que la configuration est à la racine mais il est vivement conseillé de le séparer dans un projet différent.  

Ensuite, il faut respecter les étapes suivantes :  

1) Configurer les variables ci suivantes dans Settings/CI/CD/Variables/Project variables :   
* **DOCKERHUB_TOKEN** : Un token d'accès à dockerhub qui permet le pull des images  
* **CICD_GITLAB_ADMIN_TOKEN** : Un token utilisateur admin de gitlab pour permettre la configuration des projets. A savoir qu'on peut changer son nom dans la configuration avec la variable `SETUP_GITLAB_TOKEN_NAME`.  
* **CICD_CONFIGURATION_PATH** : Le chemin du projet cicd-configuration (ex : `cicd/cicd-configuration`).  
* **SETUP** : La variable permettant au projet d'utiliser la pipeline de configuration automatique des projets. Elle doit avoir pour valeur `yes`.  

2) Configurer l'option CI/CD configuration file dans Settings/CI/CD/General pipelines et mettre `.gitlab-ci.yml@cicd/cicd-yaml`. (La valeur dependra de votre organisation de Groupe gitlab)  

### Initialisation du projet cicd-docker

Pour pouvoir initialiser le projet cicd-docker et donc permettre l'utilisation des images nécéssaires à la CI (image builder, image python...). Il faut respecter les étapes suivantes :  

1) Configurer les variables ci suivantes dans Settings/CI/CD/Variables/Project variables :  
* **DOCKERHUB_TOKEN** : Un token d'accès à dockerhub qui permet le pull des images  
* **CICD_CONFIGURATION_PATH** : Le chemin du projet cicd-configuration (ex : `cicd/cicd-configuration`).  

2) Configurer l'option CI/CD configuration file dans Settings/CI/CD/General pipelines et mettre `.gitlab-ci.yml@cicd/cicd-yaml`. (La valeur dependra de votre organisation de Groupe gitlab)  

3) Lancer une pipeline avec comme variable `INIT=yes`  

---