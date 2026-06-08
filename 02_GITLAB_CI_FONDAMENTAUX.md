# Module 02 - GitLab CI : Fondamentaux Avancés

> **Durée** : ~5h | [← Module 01](./01_GITLAB_ARCHITECTURE.md) | [→ Module 03](./03_GITLAB_CI_AVANCE.md)

---

## Sommaire

1. [Anatomie du `.gitlab-ci.yml`](#1-anatomie-du-gitlab-ciyml)
2. [Stages, Jobs, et DAG](#2-stages-jobs-et-dag)
3. [Rules - Contrôle fin d'exécution](#3-rules--contrôle-fin-dexécution)
4. [Cache et Artifacts](#4-cache-et-artifacts)
5. [Variables - Hiérarchie et gestion](#5-variables--hiérarchie-et-gestion)
6. [Services et environnements de test](#6-services-et-environnements-de-test)
7. [Pipelines for Merge Requests](#7-pipelines-for-merge-requests)

---

## 1. Anatomie du `.gitlab-ci.yml`

### 1.1 Structure globale

```yaml
# ─────────────────────────────────────────────────────────────────
# SECTION GLOBALE - s'applique à tous les jobs sauf override
# ─────────────────────────────────────────────────────────────────

# Image Docker par défaut
default:
  image: python:3.12-slim
  
  # Retry automatique (réseau, runner crash)
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
      - api_failure
  
  # Timeout par défaut (surcharge le projet)
  timeout: 30 minutes
  
  # Scripts exécutés avant/après CHAQUE job
  before_script:
    - pip install --upgrade pip
  
  # Tags par défaut pour tous les jobs
  tags:
    - docker
    - linux

# Ordre des stages (définit le flux du pipeline)
stages:
  - .pre           # Stage réservé, toujours en premier
  - validate
  - build
  - test
  - security
  - package
  - deploy:staging
  - deploy:prod
  - .post          # Stage réservé, toujours en dernier

# Variables globales
variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  DOCKER_BUILDKIT: "1"
  FF_USE_NEW_BASH_EVAL_STRATEGY: "true"

# ─────────────────────────────────────────────────────────────────
# WORKFLOW - Contrôle quand un pipeline est créé
# ─────────────────────────────────────────────────────────────────
workflow:
  name: "$CI_COMMIT_REF_NAME - $CI_PIPELINE_SOURCE"
  rules:
    # Pipeline sur push de branche (sauf tags)
    - if: $CI_COMMIT_BRANCH && $CI_PIPELINE_SOURCE == "push"
    # Pipeline sur merge request
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    # Pipeline sur tag de release
    - if: $CI_COMMIT_TAG
    # Pipeline manuel (bouton "Run pipeline")
    - if: $CI_PIPELINE_SOURCE == "web"
    # Scheduled pipeline
    - if: $CI_PIPELINE_SOURCE == "schedule"
    # PAS de pipeline sur push qui crée une MR (doublon)
    - when: never
```

### 1.2 Variables CI/CD prédéfinies essentielles

```bash
# ── Identité commit/branche ─────────────────────────────────────
CI_COMMIT_SHA           # SHA complet du commit
CI_COMMIT_SHORT_SHA     # 8 premiers caractères
CI_COMMIT_REF_NAME      # Nom branche ou tag
CI_COMMIT_BRANCH        # Vide si tag
CI_COMMIT_TAG           # Vide si branche
CI_COMMIT_MESSAGE       # Message du commit
CI_COMMIT_AUTHOR        # "Prénom Nom <email>"

# ── Projet ───────────────────────────────────────────────────────
CI_PROJECT_ID           # ID numérique du projet
CI_PROJECT_NAME         # Nom du projet (slug)
CI_PROJECT_NAMESPACE    # Groupe ou utilisateur
CI_PROJECT_PATH         # group/sous-groupe/projet
CI_PROJECT_URL          # URL complète
CI_PROJECT_DIR          # Répertoire de travail dans le job

# ── Pipeline ─────────────────────────────────────────────────────
CI_PIPELINE_ID          # ID unique du pipeline
CI_PIPELINE_SOURCE      # push, merge_request_event, schedule, web, api...
CI_JOB_ID               # ID unique du job
CI_JOB_NAME             # Nom du job
CI_JOB_TOKEN            # Token temporaire pour API GitLab (scope limité)
CI_JOB_STATUS           # success, failed, running...

# ── MR spécifiques ───────────────────────────────────────────────
CI_MERGE_REQUEST_ID         # ID de la MR
CI_MERGE_REQUEST_IID        # IID (numéro interne)
CI_MERGE_REQUEST_SOURCE_BRANCH_NAME  # Branche source
CI_MERGE_REQUEST_TARGET_BRANCH_NAME  # Branche cible
CI_MERGE_REQUEST_TITLE      # Titre de la MR

# ── Registry Docker intégrée ─────────────────────────────────────
CI_REGISTRY             # registry.gitlab.example.com
CI_REGISTRY_IMAGE       # registry.gitlab.example.com/group/project
CI_REGISTRY_USER        # gitlab-ci-token
CI_REGISTRY_PASSWORD    # = CI_JOB_TOKEN
```

---

## 2. Stages, Jobs, et DAG

### 2.1 Pipeline linéaire vs DAG

Par défaut, GitLab exécute les jobs d'un même stage **en parallèle**, et passe au stage suivant quand tous sont terminés. Avec `needs:`, on active le **DAG (Directed Acyclic Graph)** qui permet des dépendances fines entre jobs.

```yaml
stages:
  - build
  - test
  - deploy

# ── Sans DAG (linéaire) ─────────────────────────────────────────
# build:frontend et build:backend se lancent ensemble
# Puis test:frontend et test:backend attendent TOUS les builds
# Puis deploy attend TOUS les tests

# ── Avec DAG (needs:) ───────────────────────────────────────────
build:frontend:
  stage: build
  script: npm run build
  artifacts:
    paths: [dist/]

build:backend:
  stage: build
  script: mvn package
  artifacts:
    paths: [target/]

test:frontend:
  stage: test
  needs:                      # ← N'attend QUE build:frontend
    - job: build:frontend
      artifacts: true         # Télécharge ses artifacts
  script: npm test

test:backend:
  stage: test
  needs:                      # ← N'attend QUE build:backend
    - job: build:backend
      artifacts: true
  script: mvn test

deploy:staging:
  stage: deploy
  needs:                      # ← Attend les deux tests
    - test:frontend
    - test:backend
  script: deploy.sh staging
```

```
Résultat avec DAG :

  t=0   [build:frontend]  [build:backend]
  t=2   [test:frontend]                    ← démarre dès build:frontend fini
  t=3                     [test:backend]   ← démarre dès build:backend fini
  t=5   [deploy:staging]                   ← attend les deux tests

  Gain vs linéaire : les tests démarrent immédiatement au lieu d'attendre
  tous les builds.
```

⚠️ **Piège DAG** - Avec `needs:`, un job peut s'exécuter dans un stage *antérieur* si ses dépendances sont satisfaites. Cela peut surprendre.

### 2.2 Parallel et Matrix

```yaml
# ── Parallel simple ────────────────────────────────────────────
test:unit:
  parallel: 5                  # Crée test:unit 1/5, 2/5, ..., 5/5
  script:
    - pytest tests/ --splits $CI_NODE_TOTAL --group $CI_NODE_INDEX

# ── Matrix - combinatoire ──────────────────────────────────────
test:compatibility:
  parallel:
    matrix:
      - PYTHON_VERSION: ["3.10", "3.11", "3.12"]
        DATABASE: ["postgres", "mysql"]
      - PYTHON_VERSION: ["3.12"]
        DATABASE: ["sqlite"]
  image: python:${PYTHON_VERSION}-slim
  services:
    - name: "${DATABASE}:15"
      alias: db
  script:
    - pytest tests/ --database $DATABASE

# Génère :
#   test:compatibility: [PYTHON_VERSION: 3.10, DATABASE: postgres]
#   test:compatibility: [PYTHON_VERSION: 3.10, DATABASE: mysql]
#   test:compatibility: [PYTHON_VERSION: 3.11, DATABASE: postgres]
#   ... (7 jobs au total)
```

### 2.3 Trigger et pipelines multi-projets

```yaml
# Déclencher un pipeline dans un autre projet
trigger:downstream:
  stage: deploy
  trigger:
    project: mygroup/deploy-infra
    branch: main
    strategy: depend       # Ce job attend la fin du pipeline aval
  variables:
    TRIGGERED_BY: "$CI_PROJECT_PATH"
    APP_VERSION: "$CI_COMMIT_TAG"
```

---

## 3. Rules - Contrôle fin d'exécution

### 3.1 Rules vs only/except

`only/except` est **déprécié**. `rules` est plus puissant et lisible. Ne jamais mélanger les deux dans un même job.

### 3.2 Conditions disponibles

```yaml
job:example:
  script: echo "hello"
  rules:
    # ── Conditions de base ──────────────────────────────────────
    - if: $CI_COMMIT_TAG                          # Si c'est un tag
    - if: $CI_COMMIT_BRANCH == "main"             # Branche exacte
    - if: $CI_COMMIT_BRANCH =~ /^release\/.*/     # Regex branche
    - if: $CI_PIPELINE_SOURCE == "schedule"       # Pipeline planifié
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main"
    
    # ── Fichiers modifiés (très utile en monorepo) ──────────────
    - changes:
        paths:
          - "src/api/**/*"
          - "requirements.txt"
        compare_to: "refs/heads/main"   # Compare à main (pas au commit précédent)
    
    # ── Combinaisons ────────────────────────────────────────────
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - "src/**/*"
      when: on_success     # Lancer seulement si du code source a changé

    # ── Actions disponibles ─────────────────────────────────────
    # when: on_success     (défaut)
    # when: on_failure     (lancer en cas d'échec d'un job précédent)
    # when: always         (lancer quoi qu'il arrive)
    # when: manual         (bouton dans l'UI)
    # when: delayed        (after: 30 minutes)
    # when: never          (ne pas lancer)
    
    # Avec allow_failure
    - if: $CI_COMMIT_BRANCH != "main"
      when: manual
      allow_failure: true   # Le pipeline reste vert même si ce job est skip

    # Variables injectées conditionnellement
    - if: $CI_COMMIT_BRANCH == "main"
      variables:
        DEPLOY_ENV: "production"
    - when: on_success
      variables:
        DEPLOY_ENV: "staging"
```

### 3.3 Patterns avancés avec rules

```yaml
# ── Pattern : MR + main + tags seulement ────────────────────────
.rules:default: &rules_default
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
    - when: never

# ── Pattern : Deployment graduel ────────────────────────────────
deploy:prod:
  script: deploy.sh prod
  environment: production
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/
      when: manual                  # Toujours manuel pour la prod
      allow_failure: false          # Bloquant
    - when: never

# ── Pattern : Nightly seulement ─────────────────────────────────
test:integration:heavy:
  script: pytest tests/integration/ --slow
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "nightly"

# ── Pattern : Skip CI ────────────────────────────────────────────
workflow:
  rules:
    - if: $CI_COMMIT_MESSAGE =~ /\[skip ci\]/i
      when: never
    - when: always
```

---

## 4. Cache et Artifacts

### 4.1 Différence fondamentale

```
CACHE                               ARTIFACTS
─────────────────────────────────   ─────────────────────────────────
Optimisation de performance         Transmission de données entre jobs
Partagé entre pipelines             Scoped au pipeline (ou downloadable)
Non garanti (best-effort)           Garantis (stockés sur GitLab/S3)
Clé de cache pour l'invalidation    TTL configurable
Ex: node_modules, .venv, .m2        Ex: binaires, rapports, images
```

### 4.2 Cache - Configuration avancée

```yaml
# ── Cache Python partagé ────────────────────────────────────────
test:python:
  variables:
    PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  cache:
    key:
      files:
        - requirements.txt     # Invalidé si requirements.txt change
      prefix: "pip-$CI_JOB_IMAGE"  # Séparé par image Python
    paths:
      - .cache/pip/
      - .venv/
    policy: pull-push          # Pull au début, push à la fin (défaut)
    # policy: pull             # Lecture seule (jobs de test en parallèle)
    # policy: push             # Écriture seule (job de build qui installe)
    when: always               # Mettre en cache même en cas d'échec

# ── Cache Node.js ───────────────────────────────────────────────
build:frontend:
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
    policy: pull-push

# ── Cache Maven (Nexus local) ───────────────────────────────────
build:java:
  variables:
    MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
    MAVEN_CLI_OPTS: "--batch-mode --no-transfer-progress"
  cache:
    key:
      files:
        - pom.xml
      prefix: maven
    paths:
      - .m2/repository/
    policy: pull-push
```

### 4.3 Artifacts - Configuration avancée

```yaml
build:app:
  stage: build
  script:
    - make build
    - make test-report
  artifacts:
    # ── Fichiers à conserver ────────────────────────────────────
    paths:
      - dist/
      - reports/
    
    # ── Exclusions ──────────────────────────────────────────────
    exclude:
      - dist/**/*.map         # Source maps trop lourdes
    
    # ── Durée de rétention ──────────────────────────────────────
    expire_in: 7 days         # "never", "1 hour", "30 days"...
    
    # ── Rapports parsés par GitLab UI ───────────────────────────
    reports:
      junit: reports/junit.xml           # Résultats de tests
      coverage_report:
        coverage_format: cobertura
        path: reports/coverage.xml
      sast: gl-sast-report.json          # Rapport SAST GitLab
      dependency_scanning: gl-dependency-scanning-report.json
      container_scanning: gl-container-scanning-report.json
      dast: gl-dast-report.json
      
    # ── Quand exposer ───────────────────────────────────────────
    when: always              # Même en cas d'échec (pour les rapports d'erreur)
    
    # ── Accessible depuis l'UI même sans être développeur ───────
    public: false             # Limiter l'accès aux membres du projet

# Consommer les artifacts d'un autre job
deploy:
  needs:
    - job: build:app
      artifacts: true         # Télécharge les artifacts de build:app
  script:
    - ls dist/               # Les fichiers sont disponibles
```

### 4.4 Artifacts de rapport - Intégration UI GitLab

```yaml
# Tests avec rapport JUnit → affichage dans l'UI de la MR
pytest:
  script:
    - pytest tests/ 
        --junitxml=report.xml 
        --cov=src 
        --cov-report=xml:coverage.xml
  coverage: '/TOTAL.*\s+(\d+%)$/'   # Regex pour extraire le % de coverage
  artifacts:
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

---

## 5. Variables - Hiérarchie et gestion

### 5.1 Ordre de priorité (du plus prioritaire au moins prioritaire)

```
1. Trigger variables         (déclenchement via API/trigger)
2. Scheduled pipeline vars   (Variables du pipeline planifié)
3. Manual pipeline vars      (Saisies lors du "Run pipeline" manuel)
4. Dotenv artifact           (Variables générées dynamiquement par un job)
5. Job variables             (variables: dans le job)
6. Capture CI_JOB_TOKEN      
7. Project CI/CD variables   (Settings > CI/CD > Variables)
8. Group CI/CD variables     (hérités du groupe)
9. Instance CI/CD variables  (Admin > CI/CD > Variables)
10. Global .gitlab-ci.yml    (variables: global)
11. Default variables        (intégrées GitLab)
```

### 5.2 Variables dynamiques via dotenv

```yaml
# Job qui génère des variables pour les jobs suivants
prepare:version:
  stage: .pre
  script:
    - VERSION=$(git describe --tags --abbrev=0 || echo "0.0.0-dev")
    - BUILD_DATE=$(date +%Y%m%d-%H%M%S)
    - IMAGE_TAG="${VERSION}-${CI_COMMIT_SHORT_SHA}"
    # ← Écrire dans un fichier .env (format KEY=VALUE)
    - |
      cat > build.env << EOF
      APP_VERSION=${VERSION}
      BUILD_DATE=${BUILD_DATE}
      IMAGE_TAG=${IMAGE_TAG}
      EOF
  artifacts:
    reports:
      dotenv: build.env       # GitLab injecte ces variables dans les jobs downstream

build:docker:
  stage: build
  needs:
    - job: prepare:version
      artifacts: true
  script:
    - echo "Building version $IMAGE_TAG"   # Variable injectée !
    - docker build -t $CI_REGISTRY_IMAGE:$IMAGE_TAG .
```

### 5.3 Masquage et protection des variables

```
Protection des variables :
┌──────────────┬───────────────┬──────────────────────────────────────┐
│ Protected    │ Masked        │ Comportement                         │
├──────────────┼───────────────┼──────────────────────────────────────┤
│ ✅ Oui       │ ✅ Oui        │ Uniquement branches/tags protégés,   │
│              │               │ valeur cachée dans les logs          │
├──────────────┼───────────────┼──────────────────────────────────────┤
│ ✅ Oui       │ ❌ Non        │ Branches/tags protégés seulement     │
├──────────────┼───────────────┼──────────────────────────────────────┤
│ ❌ Non       │ ✅ Oui        │ Tous jobs, valeur cachée dans logs   │
├──────────────┼───────────────┼──────────────────────────────────────┤
│ ❌ Non       │ ❌ Non        │ Tous jobs, visible dans logs         │
└──────────────┴───────────────┴──────────────────────────────────────┘

⚠️ Masked = la valeur ne doit pas apparaître en clair dans les logs
   Elle doit faire au moins 8 caractères et ne pas contenir de newlines
```

🔒 **Bonne pratique** - Stocker les secrets réels dans Vault ou un gestionnaire de secrets externe et les injecter dynamiquement :

```yaml
secrets:
  DATABASE_URL:
    vault: myapp/staging/database#url   # Format: path#field
    file: false                         # Injecté comme variable d'env
  JWT_SECRET:
    vault: myapp/staging/jwt#secret
```

---

## 6. Services et environnements de test

### 6.1 Services Docker dans les jobs

```yaml
test:integration:
  image: python:3.12-slim
  
  # Services lancés en sidecar (même réseau que le job)
  services:
    # ── Service simple ──────────────────────────────────────────
    - name: postgres:15-alpine
      alias: db                     # Hostname dans le job
      variables:
        POSTGRES_DB: testdb
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
      command: ["postgres", "-c", "fsync=off"]  # Perf test mode
    
    # ── Redis ───────────────────────────────────────────────────
    - name: redis:7-alpine
      alias: cache
    
    # ── Service depuis Nexus privé ───────────────────────────────
    - name: nexus.internal:8082/myapp/mock-service:latest
      alias: mock-api
      pull_policy: always
  
  variables:
    DATABASE_URL: "postgresql://test:test@db:5432/testdb"
    REDIS_URL: "redis://cache:6379"
    MOCK_API_URL: "http://mock-api:8080"
  
  before_script:
    # Attendre que les services soient prêts
    - apt-get update && apt-get install -y postgresql-client
    - until pg_isready -h db -U test; do sleep 1; done
    - pip install -r requirements.txt
  
  script:
    - pytest tests/integration/ -v
```

### 6.2 Environments - Tracking des déploiements

```yaml
deploy:staging:
  stage: deploy:staging
  script:
    - helm upgrade --install myapp ./chart -f values.staging.yaml
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop:staging     # Job de nettoyage
    auto_stop_in: 3 days      # Auto-suppression (pour review apps)
  
stop:staging:
  stage: deploy:staging
  script:
    - helm uninstall myapp
  environment:
    name: staging
    action: stop
  rules:
    - when: manual

# Review apps par MR
deploy:review:
  stage: deploy:staging
  script:
    - helm upgrade --install "review-$CI_MERGE_REQUEST_IID" ./chart
        --set ingress.host="review-$CI_MERGE_REQUEST_IID.example.com"
  environment:
    name: "review/$CI_MERGE_REQUEST_IID"
    url: "https://review-$CI_MERGE_REQUEST_IID.example.com"
    on_stop: stop:review
    auto_stop_in: 1 day
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

---

## 7. Pipelines for Merge Requests

### 7.1 Configuration MR Pipeline

```yaml
# Activer les pipelines MR (dans les Settings du projet)
# Settings > CI/CD > General > Merge request pipelines ✓

# Différencier le comportement selon le contexte
test:unit:
  script: pytest
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# Jobs uniquement sur MR (pas sur main push)
lint:changed-files:
  script:
    - git diff --name-only origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME | xargs pylint
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - "**/*.py"

# Interdire le merge si ce job échoue
# (configurer dans Settings > Merge requests > Status checks)
security:scan:
  script: trivy fs .
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  allow_failure: false         # Bloquant pour le merge
```

### 7.2 Merged Results vs Merge Trains

```
Merged Results Pipeline :
  ├─ Crée un commit temporaire : merge(source_branch, target_branch)
  ├─ Teste CE commit (pas juste la source)
  ├─ Plus fiable car teste ce qui sera réellement mergé
  └─ Activé dans Settings > CI/CD > Merged results pipelines

Merge Trains :
  ├─ Extension des Merged Results en pipeline "file d'attente"
  ├─ Plusieurs MR peuvent être dans le train simultanément
  ├─ Chaque MR teste merge(sa-branche, toutes-les-MR-devant-elle)
  ├─ Si un test échoue, les MR suivantes sont retraitées
  └─ Nécessite GitLab EE/Premium
```

---

## 📋 Récapitulatif Module 02

| Concept | Point clé |
|---------|-----------|
| DAG avec `needs:` | Parallélisme fin, démarre les jobs dès que leurs dépendances sont satisfaites |
| `rules:` | Remplace `only/except`, supporte conditions, changes, variables injectées |
| Cache | Performance uniquement, non garanti, clé sur fichiers de lockfile |
| Artifacts | Transmission inter-jobs, rapports parsés par GitLab UI, TTL requis |
| Dotenv artifacts | Variables dynamiques générées en runtime et injectées en downstream |
| Services | Containers sidecar, attendre leur readiness avant les scripts |
| Environments | Tracking des déploiements, review apps automatiques |
| Variables | Hiérarchie stricte, masking + protection pour les secrets |

---

[← Module 01](./01_GITLAB_ARCHITECTURE.md) | [→ Module 03 - Patterns Avancés](./03_GITLAB_CI_AVANCE.md)
