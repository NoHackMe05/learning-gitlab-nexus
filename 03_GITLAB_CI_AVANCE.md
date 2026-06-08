# Module 03 - GitLab CI : Patterns Avancés

> **Durée** : ~5h | [← Module 02](./02_GITLAB_CI_FONDAMENTAUX.md) | [→ Module 04](./04_NEXUS_ARCHITECTURE.md)

---

## Sommaire

1. [Templates et réutilisabilité](#1-templates-et-réutilisabilité)
2. [Includes et hiérarchie de configuration](#2-includes-et-hiérarchie-de-configuration)
3. [Pipelines parents-enfants](#3-pipelines-parents-enfants)
4. [Gestion avancée des secrets](#4-gestion-avancée-des-secrets)
5. [Component Catalog - la nouvelle approche](#5-component-catalog--la-nouvelle-approche)
6. [Patterns monorepo](#6-patterns-monorepo)
7. [Optimisations avancées](#7-optimisations-avancées)

---

## 1. Templates et réutilisabilité

### 1.1 `extends` - Héritage de configuration

```yaml
# Définir un template (préfixe . = caché, non exécuté)
.job:base:
  image: python:3.12-slim
  before_script:
    - pip install -r requirements.txt
  retry:
    max: 2
    when: [runner_system_failure, stuck_or_timeout_failure]
  tags: [docker, linux]

# Héritage simple
test:unit:
  extends: .job:base
  stage: test
  script:
    - pytest tests/unit/

# Héritage avec override - les valeurs sont fusionnées (deep merge)
test:integration:
  extends: .job:base
  stage: test
  # Override de before_script (remplace complètement)
  before_script:
    - pip install -r requirements.txt
    - pip install -r requirements-dev.txt
  script:
    - pytest tests/integration/
  # Tags additionnels (fusionnés avec le parent ? Non ! Override complet)
  # Pour étendre un tableau, utiliser YAML anchor
  tags: [docker, linux, integration]
```

### 1.2 YAML Anchors - Réutilisation dans le même fichier

```yaml
# Définition d'un anchor
.cache:pip: &cache_pip
  key:
    files: [requirements.txt]
    prefix: "pip"
  paths: [.cache/pip/, .venv/]
  policy: pull-push

.rules:mrs-and-main: &rules_mrs_and_main
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  - when: never

# Utilisation
test:unit:
  cache:
    <<: *cache_pip        # Merge l'anchor dans le job
  rules: *rules_mrs_and_main  # Inject les rules

test:coverage:
  cache:
    <<: *cache_pip
    policy: pull          # Override la policy seule
  rules: *rules_mrs_and_main
```

### 1.3 Différence extends vs YAML anchors

```
YAML Anchors (&/*) :
  ✅ Standard YAML, lisible
  ✅ Fonctionne dans le même fichier
  ❌ Ne traverse pas les `include:`
  ❌ Pas de merge profond (scalaires écrasés, tableaux aussi)

extends: :
  ✅ Traverse les includes (partage entre fichiers)
  ✅ Merge profond intelligent (hash merge, pas array merge)
  ✅ Héritage multiple : extends: [.template1, .template2]
  ✅ Résolution dans GitLab (pas juste YAML)
  ❌ Plus verbeux pour les cas simples
```

---

## 2. Includes et hiérarchie de configuration

### 2.1 Types d'includes

```yaml
include:
  # ── Fichier local dans le même repo ────────────────────────────
  - local: '/ci/templates/build.yml'
  - local: '/ci/templates/test.yml'
  
  # ── Fichier dans un autre projet GitLab ────────────────────────
  - project: 'devops/ci-templates'
    ref: 'v2.1.0'              # Tag, branche ou SHA
    file:
      - '/templates/docker-build.yml'
      - '/templates/deploy-k8s.yml'
  
  # ── URL externe (HTTPS) ─────────────────────────────────────────
  - remote: 'https://raw.githubusercontent.com/org/repo/main/ci.yml'
  
  # ── Templates GitLab intégrés (Auto DevOps, etc.) ───────────────
  - template: 'Auto-DevOps.gitlab-ci.yml'
  - template: 'Security/SAST.gitlab-ci.yml'
  - template: 'Security/Dependency-Scanning.gitlab-ci.yml'
  
  # ── Include conditionnel ────────────────────────────────────────
  - local: '/ci/deploy-prod.yml'
    rules:
      - if: $CI_COMMIT_TAG =~ /^v[0-9]+/
```

### 2.2 Architecture de templates d'entreprise

```
Dépôt : devops/ci-templates
  ├── templates/
  │   ├── jobs/
  │   │   ├── docker-build.yml      # Build + push d'image vers Nexus
  │   │   ├── docker-scan.yml       # Trivy container scanning
  │   │   ├── python-test.yml       # pytest + coverage
  │   │   ├── node-build.yml        # npm ci + build
  │   │   ├── helm-deploy.yml       # Déploiement Kubernetes
  │   │   ├── sast-semgrep.yml      # SAST avec Semgrep
  │   │   └── notify-teams.yml      # Notification MS Teams/Slack
  │   ├── workflows/
  │   │   ├── default-workflow.yml  # Règles workflow standard
  │   │   └── release-workflow.yml  # Workflow pour les releases
  │   └── stages/
  │       └── standard-stages.yml   # Définition des stages communs
  └── CHANGELOG.md
```

```yaml
# templates/jobs/docker-build.yml
# Template réutilisable pour build Docker + push Nexus

.docker:build:
  stage: build
  image: docker:24-cli
  services:
    - name: docker:24-dind
      variables:
        DOCKER_TLS_CERTDIR: "/certs"
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
    DOCKER_BUILDKIT: "1"
    # Variables attendues du projet consommateur :
    # NEXUS_REGISTRY, NEXUS_USER, NEXUS_PASSWORD (CI Variables)
    BUILD_CONTEXT: "."
    DOCKERFILE: "Dockerfile"
  before_script:
    - echo "$NEXUS_PASSWORD" | docker login $NEXUS_REGISTRY -u $NEXUS_USER --password-stdin
  script:
    - |
      # Calcul du tag
      if [ -n "$CI_COMMIT_TAG" ]; then
        TAG="$CI_COMMIT_TAG"
      else
        TAG="${CI_COMMIT_SHORT_SHA}"
      fi
      FULL_IMAGE="$NEXUS_REGISTRY/$CI_PROJECT_PATH:$TAG"
      LATEST_IMAGE="$NEXUS_REGISTRY/$CI_PROJECT_PATH:latest"
      
      # Build avec cache
      docker build \
        --cache-from "$LATEST_IMAGE" \
        --build-arg BUILDKIT_INLINE_CACHE=1 \
        --build-arg APP_VERSION="${TAG}" \
        --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        -t "$FULL_IMAGE" \
        -t "$LATEST_IMAGE" \
        -f "$DOCKERFILE" \
        "$BUILD_CONTEXT"
      
      docker push "$FULL_IMAGE"
      docker push "$LATEST_IMAGE"
      
      # Exporter le tag pour les jobs downstream
      echo "IMAGE_TAG=${TAG}" >> build.env
      echo "FULL_IMAGE=${FULL_IMAGE}" >> build.env
  artifacts:
    reports:
      dotenv: build.env
    expire_in: 1 hour
```

```yaml
# Utilisation dans un projet
include:
  - project: 'devops/ci-templates'
    ref: 'v2.1.0'
    file: '/templates/jobs/docker-build.yml'

build:docker:
  extends: .docker:build
  variables:
    BUILD_CONTEXT: "./backend"
    DOCKERFILE: "backend/Dockerfile.prod"
```

---

## 3. Pipelines parents-enfants

### 3.1 Concept et cas d'usage

```
Pipeline Parent
  │
  ├── trigger:frontend ──► Pipeline Enfant Frontend
  │                          ├── lint
  │                          ├── test
  │                          └── build
  │
  ├── trigger:backend  ──► Pipeline Enfant Backend
  │                          ├── test:unit
  │                          ├── test:integration
  │                          └── build:image
  │
  └── trigger:infra    ──► Pipeline Enfant Infra
                             ├── terraform:plan
                             └── terraform:apply
```

### 3.2 Implémentation

```yaml
# .gitlab-ci.yml (parent)
stages:
  - validate
  - trigger
  - finalize

validate:yaml:
  stage: validate
  script:
    - gitlab-ci-lint ci/frontend.yml
    - gitlab-ci-lint ci/backend.yml

trigger:frontend:
  stage: trigger
  trigger:
    include: ci/frontend.yml    # Fichier dans le même repo
    strategy: depend            # Attend la fin du pipeline enfant
  rules:
    - changes:
        - frontend/**/*
        - ci/frontend.yml

trigger:backend:
  stage: trigger
  trigger:
    include: ci/backend.yml
    strategy: depend
  rules:
    - changes:
        - backend/**/*
        - ci/backend.yml
  # Passer des variables au pipeline enfant
  variables:
    PARENT_PIPELINE_ID: "$CI_PIPELINE_ID"
    DEPLOY_ENV: "$DEPLOY_ENV"

notify:success:
  stage: finalize
  script: notify.sh "Pipeline complet"
  when: on_success
  needs:
    - trigger:frontend
    - trigger:backend
```

```yaml
# ci/backend.yml (enfant)
stages:
  - test
  - build
  - deploy

# Accéder aux variables du parent
test:unit:
  stage: test
  script:
    - echo "Parent pipeline: $PARENT_PIPELINE_ID"
    - pytest tests/unit/

# Le pipeline enfant peut lui-même déclencher d'autres pipelines
# (pipelines "petits-enfants" - jusqu'à 2 niveaux de profondeur)
```

---

## 4. Gestion avancée des secrets

### 4.1 Intégration HashiCorp Vault

```yaml
# Prérequis : configurer l'instance Vault dans GitLab
# Admin > Settings > CI/CD > Variables > Vault server URL

job:with:vault:
  image: python:3.12-slim
  id_tokens:
    VAULT_ID_TOKEN:
      aud: https://vault.example.com   # Audience du JWT
  secrets:
    # Format : path/to/secret:field
    DB_PASSWORD:
      vault: myapp/$CI_ENVIRONMENT_NAME/database:password
      file: false                      # Comme variable d'env
    SSH_PRIVATE_KEY:
      vault: myapp/shared/ssh:private_key
      file: true                       # Écrit dans un fichier temporaire
  script:
    - echo "Connexion à la DB..."
    - python -c "import os; print(os.environ['DB_PASSWORD'][:3] + '***')"
```

### 4.2 OIDC / JWT - Authentification sans secret statique

```yaml
# GitLab peut générer des JWT tokens pour s'authentifier
# auprès de services tiers (AWS, GCP, Vault) sans stocker de clés

deploy:aws:
  image: amazon/aws-cli
  id_tokens:
    AWS_OIDC_TOKEN:
      aud: sts.amazonaws.com
  script:
    - |
      # Échange du JWT GitLab contre des credentials AWS temporaires
      export AWS_ROLE_ARN="arn:aws:iam::123456789:role/GitLabDeployRole"
      CREDS=$(aws sts assume-role-with-web-identity \
        --role-arn "$AWS_ROLE_ARN" \
        --role-session-name "gitlab-$CI_JOB_ID" \
        --web-identity-token "$AWS_OIDC_TOKEN" \
        --duration-seconds 3600)
      
      export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r .Credentials.AccessKeyId)
      export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r .Credentials.SecretAccessKey)
      export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r .Credentials.SessionToken)
      
      # Déploiement avec les credentials temporaires
      aws s3 sync dist/ s3://my-bucket/
```

### 4.3 External Secrets via init container (K8s)

```yaml
# Pour les runners Kubernetes : injecter des secrets Vault via sidecar
deploy:k8s:
  image: bitnami/kubectl:latest
  script:
    - kubectl apply -f k8s/
  # Le pod spec peut être enrichi pour inclure Vault Agent
  variables:
    KUBERNETES_POD_ANNOTATIONS_1: "vault.hashicorp.com/agent-inject: 'true'"
    KUBERNETES_POD_ANNOTATIONS_2: "vault.hashicorp.com/role: 'gitlab-runner'"
    KUBERNETES_POD_ANNOTATIONS_3: "vault.hashicorp.com/agent-inject-secret-db: 'myapp/data/db'"
```

---

## 5. Component Catalog - la nouvelle approche

### 5.1 GitLab CI/CD Components (GitLab 16.x+)

Les Components remplacent les templates projet/remote par un système plus structuré et versionné.

```yaml
# Utilisation d'un component
include:
  - component: gitlab.example.com/devops/catalog/docker-build@v1.2.0
    inputs:
      registry: nexus.internal:8082
      dockerfile: Dockerfile.prod
      context: ./backend

  - component: gitlab.example.com/devops/catalog/pytest@v2.0.0
    inputs:
      python_version: "3.12"
      coverage_threshold: "80"
```

```yaml
# Définition d'un component (dans le dépôt catalog)
# Fichier : templates/docker-build.yml

spec:
  inputs:
    registry:
      description: "URL du registry Docker"
      type: string
    dockerfile:
      description: "Chemin vers le Dockerfile"
      type: string
      default: "Dockerfile"
    context:
      description: "Build context"
      type: string
      default: "."
    tags:
      description: "Tags runner"
      type: array
      default: ["docker"]

---
# Le job du component
".docker:build":
  stage: build
  image: docker:24-cli
  tags: $[[ inputs.tags ]]
  script:
    - docker build 
        -f "$[[ inputs.dockerfile ]]" 
        -t "$[[ inputs.registry ]]/$CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA"
        "$[[ inputs.context ]]"
    - docker push "$[[ inputs.registry ]]/$CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA"
```

---

## 6. Patterns monorepo

### 6.1 Détection des changements par service

```yaml
# Dans un monorepo : services/frontend, services/backend, services/worker

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - when: never

# ── Détection des changements ──────────────────────────────────
.rules:frontend:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        paths: ["services/frontend/**/*", "shared/ui/**/*"]
        compare_to: "refs/heads/main"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      changes:
        paths: ["services/frontend/**/*", "shared/ui/**/*"]
        compare_to: "refs/heads/main"
    - when: never

.rules:backend:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        paths: ["services/backend/**/*", "shared/lib/**/*"]
        compare_to: "refs/heads/main"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      changes:
        paths: ["services/backend/**/*", "shared/lib/**/*"]
        compare_to: "refs/heads/main"
    - when: never

# ── Jobs conditionnels ─────────────────────────────────────────
build:frontend:
  extends: .rules:frontend
  script: cd services/frontend && npm run build

test:frontend:
  extends: .rules:frontend
  needs: [build:frontend]
  script: cd services/frontend && npm test

build:backend:
  extends: .rules:backend
  script: cd services/backend && docker build .

test:backend:
  extends: .rules:backend
  needs: [build:backend]
  script: pytest services/backend/
```

### 6.2 Pipeline parent-enfant pour monorepo large

```yaml
# .gitlab-ci.yml (orchestrateur)
stages:
  - detect
  - trigger

# Générer dynamiquement le pipeline en fonction des changements
detect:changes:
  stage: detect
  image: python:3.12-slim
  script:
    - pip install pyyaml
    - python ci/detect-changes.py > generated-pipeline.yml
  artifacts:
    paths: [generated-pipeline.yml]
    expire_in: 1 hour

trigger:dynamic:
  stage: trigger
  trigger:
    include:
      - artifact: generated-pipeline.yml
        job: detect:changes
    strategy: depend
```

```python
# ci/detect-changes.py
import subprocess
import yaml
import os

def get_changed_dirs():
    result = subprocess.run(
        ["git", "diff", "--name-only", "origin/main...HEAD"],
        capture_output=True, text=True
    )
    changed = result.stdout.splitlines()
    dirs = set()
    for f in changed:
        parts = f.split("/")
        if len(parts) > 1 and parts[0] == "services":
            dirs.add(parts[1])
    return dirs

services = get_changed_dirs()
pipeline = {"stages": ["build", "test", "deploy"], "jobs": {}}

for service in services:
    pipeline["jobs"][f"build:{service}"] = {
        "stage": "build",
        "script": [f"cd services/{service} && make build"]
    }
    pipeline["jobs"][f"test:{service}"] = {
        "stage": "test",
        "needs": [f"build:{service}"],
        "script": [f"cd services/{service} && make test"]
    }

print(yaml.dump(pipeline))
```

---

## 7. Optimisations avancées

### 7.1 Réduction du temps de pipeline

```yaml
# ── Fail fast - lancer le lint en premier ─────────────────────
lint:
  stage: .pre           # Toujours en premier
  script: flake8 src/
  allow_failure: false  # Bloque tout si ça échoue

# ── Paralleliser le maximum ─────────────────────────────────────
test:unit:
  parallel: 4
  script:
    - pytest tests/unit/ --splits=$CI_NODE_TOTAL --group=$CI_NODE_INDEX

# ── Cache agressif ──────────────────────────────────────────────
build:deps:
  stage: .pre
  script:
    - pip install -r requirements.txt
  cache:
    key: "$CI_COMMIT_REF_SLUG-deps"
    paths: [.venv/]
    policy: push          # Ce job crée le cache

test:unit:
  needs: [build:deps]
  cache:
    key: "$CI_COMMIT_REF_SLUG-deps"
    paths: [.venv/]
    policy: pull          # Les tests lisent seulement
```

### 7.2 Interruptible pipelines

```yaml
# Annuler les vieux pipelines quand un nouveau push arrive
workflow:
  auto_cancel:
    on_new_commit: interruptible    # Annule les jobs marqués interruptible

test:unit:
  interruptible: true   # Peut être annulé par un nouveau pipeline

deploy:prod:
  interruptible: false  # NE PAS annuler un déploiement en cours !
```

### 7.3 Artifacts légers et sélectifs

```yaml
# ⚠️ Anti-pattern : artifacts trop larges
build:bad:
  artifacts:
    paths:
      - "**/*"          # JAMAIS ça ! Upload de node_modules, .git, etc.
    expire_in: never    # JAMAIS ça en production !

# ✅ Bon pattern : sélectif et TTL court
build:good:
  artifacts:
    paths:
      - dist/           # Seulement le build output
      - "*.whl"         # Package Python
    exclude:
      - "**/*.pyc"
      - "**/__pycache__"
    expire_in: 1 day    # TTL court pour les artifacts de build intermédiaires
    when: on_success

# Pour les rapports de test : durée plus longue
test:
  artifacts:
    reports:
      junit: junit.xml
    expire_in: 30 days  # Garder l'historique des tests
    when: always        # Même en cas d'échec
```

### 7.4 Mesure de performance du pipeline

```yaml
# Job de fin qui mesure le temps global
.post:metrics:
  stage: .post
  script:
    - |
      DURATION=$(($(date +%s) - ${CI_PIPELINE_CREATED_AT:-$(date +%s)}))
      echo "Pipeline duration: ${DURATION}s"
      # Envoyer à Prometheus Pushgateway
      cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/gitlab_pipeline
      gitlab_pipeline_duration_seconds{project="$CI_PROJECT_NAME",ref="$CI_COMMIT_REF_NAME"} $DURATION
      EOF
  when: always
  allow_failure: true
```

---

## 📋 Récapitulatif Module 03

| Concept | Point clé |
|---------|-----------|
| `extends` | Héritage inter-fichiers, merge profond, préférer aux YAML anchors pour les templates partagés |
| Includes | 4 types (local, project, remote, template), inclure avec `rules:` depuis 16.x |
| Parent-enfant | Isolation, parallélisme, max 2 niveaux de profondeur |
| Components | Futur des templates GitLab, inputs typés, versionnés dans un Catalog |
| Secrets OIDC | Éviter les secrets statiques, JWT pour AWS/GCP/Vault |
| Monorepo | `changes:` + `compare_to:` pour les changements précis, pipeline dynamique pour les grands monorepos |
| Performance | `interruptible`, cache `pull-only` sur les jobs parallèles, `fail-fast` avec `.pre` |

---

[← Module 02](./02_GITLAB_CI_FONDAMENTAUX.md) | [→ Module 04 - Nexus Architecture](./04_NEXUS_ARCHITECTURE.md)
