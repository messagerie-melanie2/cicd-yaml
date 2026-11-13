# Gitleaks module (détection de secrets)

Recherche les secrets committés par erreur dans le projet (tokens, mots de passe, clés privées, URLs de webhook...) avec [gitleaks](https://github.com/gitleaks/gitleaks). Comme le scan SonarQube, cette feature n'utilise aucun script Python de `cicd-script` : c'est un appel à l'exécutable `gitleaks` dans une image dédiée.

## Projets concernés

| Projet | Rôle dans la feature |
|------|----------------------|
| `cicd-yaml` | Définition du job (`features/gitleaks.yml`) |
| `cicd-docker` | Image utilisée par le job (`cicd-docker/gitleaks`, basée sur `zricethezav/gitleaks`), qui embarque le script `gitleaks_scan.sh` |
| Projet cible | Fichiers `.gitleaks.toml` et `.gitleaksignore` à la racine (optionnels, voir plus bas) |

## Fonctionnement

```mermaid
flowchart LR
    A{"GITLEAKS_SCAN_MODE"} -- auto --> B{"Pipeline de push ?"}
    B -- Oui --> D["diff : commits du push<br>CI_COMMIT_BEFORE_SHA..CI_COMMIT_SHA"]
    B -- Non --> H["history : tout l'historique"]
    A -- diff --> D
    A -- history --> H
    A -- dir --> R["dir : fichiers actuels uniquement"]
    D -- "commit précédent introuvable" --> H
    D --> J["Rapport JUnit<br>gitleaks-report.xml"]
    H --> J
    R --> J
```

Le job `gitleaks-scan` (stage `quality`, avec `needs: []` : il démarre dès le début de la pipeline) :

1. Démarre l'image `${REGISTRY_DOMAIN}${CICD_NAMESPACE}${GITLEAKS_PATH}:${GITLEAKS_TAG}`.
2. Récupère tout l'historique git du projet (`GIT_DEPTH: "0"`).
3. Lance `/entrypoint/gitleaks_scan.sh`, qui choisit quoi scanner selon `GITLEAKS_SCAN_MODE` :
   - **`auto`** (défaut) : sur un push, seulement les commits du push. Pour un schedule ou un lancement manuel, tout l'historique.
   - **`diff`** : seulement les commits entre `CI_COMMIT_BEFORE_SHA` et `CI_COMMIT_SHA`. Si le commit précédent est introuvable (par exemple après un force push), le script scanne tout l'historique à la place.
   - **`history`** : tout l'historique git.
   - **`dir`** : seulement les fichiers présents dans le dernier commit, sans l'historique.
4. Les secrets sont **masqués** dans les logs (`--redact`) : le log indique le fichier, la ligne, le commit et la règle déclenchée, mais jamais la valeur.
5. Le rapport `gitleaks-report.xml` (format JUnit) est publié en artefact. Les fuites apparaissent dans l'onglet **Tests** de la pipeline.
6. Le job est `allow_failure: true` : si un secret est trouvé, le job est en échec (orange) mais ne bloque pas le reste de la pipeline.

## Personnalisation par projet

gitleaks lit automatiquement ces fichiers s'ils sont à la racine du projet :

| Fichier | Rôle |
|---------|------|
| `.gitleaks.toml` | Règles personnalisées et allowlist (chemins, regex ou commits à ignorer). Pour garder les règles par défaut et y ajouter les siennes, mettre `[extend] useDefault = true` |
| `.gitleaksignore` | Liste d'empreintes (`Fingerprint`, affichée dans le log pour chaque fuite) à ignorer, une par ligne. C'est le moyen le plus simple d'écarter un faux positif |

Exemple de `.gitleaks.toml` :

```toml
[extend]
useDefault = true

[allowlist]
description = "Fichiers de test"
paths = ['''tests/fixtures/.*''']
```

## Que faire si un secret est trouvé ?

Supprimer le secret du code ne suffit pas : il reste dans l'historique git, et toute personne ayant accès au repo peut le récupérer.

1. **Révoquer ou régénérer le secret** à sa source (token GitLab, webhook, mot de passe...). C'est l'étape la plus importante.
2. Déclarer la nouvelle valeur en **variable CI/CD masquée** (projet ou groupe), plutôt que dans un fichier.
3. Ajouter l'empreinte de l'ancienne fuite dans `.gitleaksignore`, pour que les scans de l'historique ne la remontent plus.

## Déclenchement

Le job ne se lance que si `LAUNCH_FEATURE` contient `scan-secret-gitleaks` **et** que la pipeline vient d'un push, d'un schedule ou d'un lancement manuel (`CI_PIPELINE_SOURCE` = `push`, `schedule` ou `web`). Il ne se lance donc pas dans une pipeline déclenchée par un autre projet (`trigger`, `pipeline`), ni par l'API ou une merge request : sinon chaque trigger lancerait un scan de tout l'historique, alors que le projet déclencheur a déjà scanné ses propres commits.

Le scan tourne de deux façons, sans rien à configurer côté projet :

| Quand | Ce qui est scanné | Origine |
|-------|-------------------|---------|
| À chaque push | Les commits du push (mode `auto` → `diff`) | `scan-secret-gitleaks` fait partie de la valeur par défaut de `LAUNCH_FEATURE` |
| Tous les 5 jours | Tout l'historique git (`GITLEAKS_SCAN_MODE: "history"`) | Schedule **obligatoire** `scangitleaks`, créé sur chaque projet par le module setup-project |

Le scan à chaque push détecte un nouveau secret dès qu'il est poussé. Le schedule rattrape tout ce que le push ne voit pas : historique antérieur à l'activation de la feature, force push, nouvelles règles gitleaks après une montée de version de l'image.

Le scan peut aussi être lancé à la main via **Build > Pipelines > Run pipeline**, avec `LAUNCH_FEATURE = scan-secret-gitleaks`, ou depuis **Build > Pipeline schedules** (bouton ▶ du schedule `scangitleaks`).

### Schedule obligatoire `scangitleaks`

Le schedule est défini dans `SETUP_SCHEDULE_TYPE` et listé dans `SETUP_SCHEDULE_MANDATORY` (`"cleanlog,scangitleaks"`). Le job `setup-schedule-project` le crée donc sur **tous** les projets déclarés dans les fichiers de setup (`setup/**/*build.yml` et `setup/**/*triggers.yml`), comme le schedule `cleanlog`. Voir [SETUP_PROJECT.md](SETUP_PROJECT.md#branche---setup-schedule).

| Paramètre | Valeur |
|-----------|--------|
| Description | `[<branche>] [scan] Schedule scan des leaks de secrets dans git` |
| Branche | Branche par défaut du projet |
| Cron | `0 0 */5 * *` (à minuit, les 1, 6, 11, 16, 21, 26 et 31 du mois, heure de Paris) |
| Variables | `LAUNCH_FEATURE: "scan-secret-gitleaks"`, `GITLEAKS_SCAN_MODE: "history"`, `CI_COMMIT_MESSAGE: "[scan] Schedule scan des leaks de secrets dans git"` |

Comme `LAUNCH_FEATURE` ne contient que `scan-secret-gitleaks`, cette pipeline ne lance que `get-files-from-git` et `gitleaks-scan` (ni build, ni trigger, ni SonarQube).

Le schedule ne peut pas être retiré d'un projet, mais ses valeurs peuvent être surchargées dans le bloc `schedule:` du projet (fichier de setup), avec `type: "scangitleaks"` et **la même branche** :

```yaml
  projects:
    - name: "mon-projet"
      id: 12345
      schedule:
        - type: "scangitleaks"
          cron: "0 3 * * 1"   # tous les lundis à 3h
```

> **Attention :** ce schedule lance un scan de tout l'historique, qui peut être long sur un gros repo. Si besoin, limiter la taille des fichiers scannés avec `GITLEAKS_EXTRA_ARGS` (ex : `--max-target-megabytes 5`) dans les variables du schedule.

## Variables

| Variable | Défaut | Rôle |
|----------|--------|------|
| `GITLEAKS_PATH` | `cicd-docker/gitleaks` | Chemin de l'image dans le registry |
| `GITLEAKS_TAG` | `v8.28.0_1.0-prod` | Tag de l'image (`<version gitleaks>_<version image>`) |
| `GITLEAKS_SCAN_MODE` | `auto` (`history` dans le schedule `scangitleaks`) | `auto`, `diff`, `history` ou `dir` (voir plus haut) |
| `GITLEAKS_EXTRA_ARGS` | vide | Arguments gitleaks supplémentaires (ex : `--max-target-megabytes 5`) |
| `GITLEAKS_REPORT_PATH` | `gitleaks-report.xml` | Chemin du rapport JUnit (défini dans le job, à ne pas modifier) |

## Résultat

Un job `gitleaks-scan` en succès si aucun secret n'est trouvé, en échec non bloquant sinon, avec le détail des fuites dans le log (valeurs masquées) et dans l'onglet **Tests** de la pipeline.

---
