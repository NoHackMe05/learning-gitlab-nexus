# Module 09 - Labs Pratiques

> **Durée** : ~6h | [← Module 08](./08_OBSERVABILITE_PERFORMANCE.md) | [← Index](./00_INDEX.md)

---

## Vue d'ensemble des labs

| Lab | Titre | Durée | Prérequis |
|-----|-------|-------|-----------|
| [Lab 01](#lab-01--installation-et-configuration-dun-runner-docker) | Runner Docker - Installation & Config | 45 min | Docker, accès GitLab |
| [Lab 02](#lab-02--pipeline-ci-python-complet) | Pipeline CI Python complet | 60 min | Lab 01 |
| [Lab 03](#lab-03--intégration-nexus--pypi--docker) | Intégration Nexus (PyPI + Docker) | 60 min | Nexus déployé, Lab 02 |
| [Lab 04](#lab-04--sécurité--sast--secret-detection--container-scan) | DevSecOps pipeline | 45 min | Lab 03 |
| [Lab 05](#lab-05--templates-réutilisables-et-includes) | Templates & Includes | 45 min | Lab 02 |
| [Lab 06](#lab-06--pipeline-multi-projets-et-gitops) | Multi-projets & GitOps | 60 min | Lab 03, K8s accessible |
| [Lab 07](#lab-07--déploiement-bluegreen-et-rollback) | Blue/Green & Rollback | 45 min | Lab 06, Helm |
| [Lab 08](#lab-08--pipeline-production-grade-complet) | Pipeline production-grade complet | 90 min | Tous les labs précédents |

---

## Lab 01 - Installation et configuration d'un Runner Docker

### Objectifs
- Installer GitLab Runner sur une VM ou un conteneur
- Enregistrer un runner Docker avec tags
- Vérifier le fonctionnement avec un premier pipeline

### Étapes

**Étape 1 - Installer GitLab Runner**

```bash
# Sur Ubuntu/Debian
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# Vérifier
gitlab-runner --version

# Sur Docker (alternative)
docker run -d \
  --name gitlab-runner \
  --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

**Étape 2 - Enregistrer le Runner**

```bash
# Dans l'UI GitLab :
# Votre projet > Settings > CI/CD > Runners > New project runner
# Cocher : Run untagged jobs = NON
# Tags : docker,linux,lab
# Copier le token glrt-xxxx

# Enregistrer
sudo gitlab-runner register \
  --non-interactive \
  --url "https://votre-gitlab.com" \
  --token "glrt-VOTRE_TOKEN" \
  --executor "docker" \
  --docker-image "alpine:3.18" \
  --description "lab-docker-runner" \
  --tag-list "docker,linux,lab" \
  --run-untagged=false
```

**Étape 3 - Vérifier la configuration**

```bash
cat /etc/gitlab-runner/config.toml
# Ajouter les limits
sudo vim /etc/gitlab-runner/config.toml
# → concurrent = 4
# → [runners.docker] memory = "1g", cpus = "1"

sudo gitlab-runner restart
sudo gitlab-runner status
```

**Étape 4 - Premier pipeline de test**

Créer le fichier `.gitlab-ci.yml` dans un projet de test :

```yaml
# .gitlab-ci.yml - Pipeline de vérification Lab 01
stages:
  - verify

verify:runner:
  stage: verify
  tags:
    - docker
    - lab
  script:
    - echo "✅ Runner opérationnel !"
    - echo "Runner: $CI_RUNNER_DESCRIPTION"
    - echo "Executor: Docker"
    - docker --version
    - cat /etc/os-release
    - echo "Job ID: $CI_JOB_ID"
    - echo "Pipeline ID: $CI_PIPELINE_ID"
```

**Vérification attendue** : Pipeline vert dans l'UI GitLab, logs affichant les informations du runner.

---

## Lab 02 - Pipeline CI Python complet

### Objectifs
- Créer un pipeline multi-stages pour une application Python FastAPI
- Implémenter cache, artifacts, rapports de tests
- Utiliser le DAG avec `needs:`

### Application de test

```bash
# Cloner ou créer le projet
mkdir lab02-fastapi && cd lab02-fastapi
git init && git remote add origin https://gitlab.example.com/training/lab02.git
```

```
lab02-fastapi/
├── src/
│   ├── __init__.py
│   └── main.py
├── tests/
│   ├── unit/
│   │   └── test_main.py
│   └── integration/
│       └── test_api.py
├── requirements.txt
├── requirements-dev.txt
└── .gitlab-ci.yml
```

```python
# src/main.py
from fastapi import FastAPI

app = FastAPI(title="Lab 02 API")

@app.get("/health")
def health():
    return {"status": "ok", "version": "1.0.0"}

@app.get("/items/{item_id}")
def get_item(item_id: int, name: str = "default"):
    return {"id": item_id, "name": name}
```

```python
# tests/unit/test_main.py
from fastapi.testclient import TestClient
from src.main import app

client = TestClient(app)

def test_health():
    resp = client.get("/health")
    assert resp.status_code == 200
    assert resp.json()["status"] == "ok"

def test_get_item():
    resp = client.get("/items/42?name=test")
    assert resp.status_code == 200
    assert resp.json() == {"id": 42, "name": "test"}
```

```
# requirements.txt
fastapi==0.109.0
uvicorn==0.27.0

# requirements-dev.txt
pytest==7.4.4
pytest-cov==4.1.0
httpx==0.26.0
ruff==0.1.14
mypy==1.8.0
```

### Pipeline à implémenter

```yaml
# .gitlab-ci.yml - À COMPLÉTER pendant le lab

stages:
  - .pre
  - lint
  - test
  - build

default:
  image: python:3.12-slim
  tags: [docker, lab]
  before_script:
    - python -m venv .venv
    - source .venv/bin/activate

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

# TODO : Implémenter les jobs suivants
# 1. install:deps  (stage .pre, cache push)
# 2. lint:ruff     (stage lint, needs install:deps, cache pull)
# 3. lint:mypy     (stage lint, needs install:deps, cache pull)
# 4. test:unit     (stage test, needs install:deps, parallel:2)
# 5. build:package (stage build, needs test:unit, artifacts wheel)
```

### Solution complète

```yaml
# .gitlab-ci.yml - SOLUTION Lab 02

stages:
  - .pre
  - lint
  - test
  - build

default:
  image: python:3.12-slim
  tags: [docker, lab]

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  VENV: ".venv"

install:deps:
  stage: .pre
  before_script: []
  script:
    - python -m venv $VENV
    - $VENV/bin/pip install --upgrade pip
    - $VENV/bin/pip install -r requirements.txt -r requirements-dev.txt
  cache:
    key:
      files: [requirements.txt, requirements-dev.txt]
      prefix: "py312"
    paths: [$VENV/, .cache/pip/]
    policy: push
  interruptible: true

lint:ruff:
  stage: lint
  needs: [install:deps]
  cache:
    key:
      files: [requirements.txt, requirements-dev.txt]
      prefix: "py312"
    paths: [$VENV/]
    policy: pull
  script:
    - $VENV/bin/ruff check src/ tests/
    - $VENV/bin/ruff format --check src/ tests/
  interruptible: true

lint:mypy:
  stage: lint
  needs: [install:deps]
  cache:
    key:
      files: [requirements.txt, requirements-dev.txt]
      prefix: "py312"
    paths: [$VENV/]
    policy: pull
  script:
    - $VENV/bin/mypy src/ --ignore-missing-imports
  interruptible: true
  allow_failure: true   # Mypy peut être strict pour ce lab

test:unit:
  stage: test
  needs: [lint:ruff]     # DAG : attend lint avant les tests
  parallel: 2
  cache:
    key:
      files: [requirements.txt, requirements-dev.txt]
      prefix: "py312"
    paths: [$VENV/]
    policy: pull
  script:
    - $VENV/bin/pytest tests/unit/
        --junitxml=reports/junit.xml
        --cov=src
        --cov-report=xml:reports/coverage.xml
        --cov-report=term-missing
        --splits=$CI_NODE_TOTAL
        --group=$CI_NODE_INDEX
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    reports:
      junit: reports/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: reports/coverage.xml
    expire_in: 7 days
  interruptible: true

build:package:
  stage: build
  needs: [test:unit]
  cache:
    key:
      files: [requirements.txt, requirements-dev.txt]
      prefix: "py312"
    paths: [$VENV/]
    policy: pull
  script:
    - $VENV/bin/pip install build
    - python -m build --wheel
    - ls -la dist/
  artifacts:
    paths: [dist/*.whl]
    expire_in: 1 day
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

**Vérifications** :
- Pipeline vert complet
- Rapport de tests visible dans l'UI GitLab (MR > Tests)
- Coverage visible dans l'UI (badge sur la MR)
- Artifact `.whl` téléchargeable

---

## Lab 03 - Intégration Nexus : PyPI + Docker

### Objectifs
- Publier un package Python vers Nexus PyPI hosted
- Builder et pousser une image Docker vers Nexus Registry
- Consommer depuis Nexus dans un job CI

### Prérequis
- Nexus déployé et accessible (`https://nexus.lab.internal`)
- Repos créés : `pypi-hosted`, `pypi-group`, `docker-hosted-releases`
- Variables CI configurées : `NEXUS_URL`, `NEXUS_USER`, `NEXUS_PASSWORD`, `NEXUS_WRITER_USER`, `NEXUS_WRITER_PASSWORD`

### Étape 1 - Dockerfile

```dockerfile
# Dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app

ARG PIP_INDEX_URL=https://nexus.lab.internal/repository/pypi-group/simple/
ARG PIP_TRUSTED_HOST=nexus.lab.internal

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

COPY src/ ./src/

FROM python:3.12-slim
RUN useradd -r -u 1000 appuser
WORKDIR /app
COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local
COPY --from=builder --chown=appuser:appuser /app/src ./src
USER appuser
EXPOSE 8000
HEALTHCHECK CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"
CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Étape 2 - Pipeline étendu

```yaml
# Ajouter à votre .gitlab-ci.yml du Lab 02

variables:
  NEXUS_REGISTRY: "nexus.lab.internal:8082"
  TWINE_REPOSITORY_URL: "$NEXUS_URL/repository/pypi-hosted/"
  TWINE_USERNAME: "$NEXUS_WRITER_USER"
  TWINE_PASSWORD: "$NEXUS_WRITER_PASSWORD"

publish:pypi:
  stage: package
  needs: [build:package]
  script:
    - pip install twine
    - twine upload --non-interactive dist/*
  rules:
    - if: $CI_COMMIT_TAG

build:docker:
  stage: package
  image: docker:24-cli
  services:
    - name: docker:24-dind
      variables:
        DOCKER_TLS_CERTDIR: "/certs"
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
    IMAGE: "$NEXUS_REGISTRY/$CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA"
  before_script:
    - echo "$NEXUS_PASSWORD" | docker login "$NEXUS_REGISTRY" -u "$NEXUS_USER" --password-stdin
  script:
    - docker build
        --build-arg PIP_INDEX_URL="https://$NEXUS_USER:$NEXUS_PASSWORD@nexus.lab.internal/repository/pypi-group/simple/"
        --build-arg PIP_TRUSTED_HOST="nexus.lab.internal"
        -t "$IMAGE"
        .
    - docker push "$IMAGE"
    - echo "IMAGE=$IMAGE" >> docker.env
  artifacts:
    reports:
      dotenv: docker.env
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

**Vérifications** :
- Package visible dans Nexus > Browse > pypi-hosted
- Image visible dans Nexus > Browse > docker-hosted-releases
- `docker pull nexus.lab.internal:8082/training/lab03:latest` fonctionne

---

## Lab 04 - Sécurité : SAST + Secret Detection + Container Scan

### Objectifs
- Intégrer les templates de sécurité GitLab
- Configurer Gitleaks pour la détection de secrets
- Tester le blocage sur une vulnérabilité délibérée

### Étape 1 - Ajouter intentionnellement un "secret"

```python
# src/config.py - À NE PAS FAIRE EN PROD, c'est pour le lab !
# Ce fichier va déclencher Gitleaks

API_KEY = "sk-prod-a1b2c3d4e5f6789012345678901234567890abcd"  # noqa
DATABASE_URL = "postgresql://admin:SuperSecret123@db.prod:5432/myapp"
```

### Étape 2 - Pipeline sécurisé

```yaml
# .gitlab-ci.yml - Ajouts sécurité

include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml

stages:
  - .pre
  - secrets    # Avant tout
  - lint
  - test
  - security
  - build

# Secret detection en .pre = fail fast
secret_detection:
  stage: .pre   # Override le stage du template
  variables:
    SECRET_DETECTION_HISTORIC_SCAN: "false"

# Container scan après le build Docker
scan:container:
  stage: security
  needs:
    - job: build:docker
      artifacts: true
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image
        --exit-code 1
        --severity CRITICAL
        --ignore-unfixed
        "$IMAGE"
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
    when: always
  allow_failure: true   # Pour ce lab, ne pas bloquer
```

**Exercice** : Observer le job `secret_detection` échouer, puis corriger en déplaçant les secrets dans des variables CI/CD GitLab.

---

## Lab 05 - Templates réutilisables et Includes

### Objectifs
- Créer un dépôt de templates CI partagés
- Utiliser `extends` et `include` entre projets
- Implémenter un Component simple

### Étape 1 - Créer le dépôt de templates

```
Créer : devops/ci-templates (nouveau projet GitLab)

Structure :
  ci-templates/
  ├── jobs/
  │   ├── python-base.yml
  │   ├── docker-build.yml
  │   └── deploy-k8s.yml
  └── workflows/
      └── default.yml
```

```yaml
# ci-templates/jobs/python-base.yml

.python:install:
  image: python:${PYTHON_VERSION:-3.12}-slim
  variables:
    PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
    VENV: ".venv"
  before_script:
    - python -m venv $VENV
    - $VENV/bin/pip install --upgrade pip
  cache:
    key:
      files: [requirements.txt]
      prefix: "py-$PYTHON_VERSION"
    paths: [$VENV/, .cache/pip/]

.python:test:
  extends: .python:install
  script:
    - $VENV/bin/pip install -r requirements.txt -r requirements-dev.txt
    - $VENV/bin/pytest ${TEST_PATH:-tests/} 
        --junitxml=reports/junit.xml
        --cov=src
        --cov-report=xml:reports/coverage.xml
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    reports:
      junit: reports/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: reports/coverage.xml
    expire_in: 7 days
```

### Étape 2 - Consommer le template dans un projet

```yaml
# .gitlab-ci.yml du projet applicatif

include:
  - project: 'devops/ci-templates'
    ref: 'main'
    file:
      - '/jobs/python-base.yml'
      - '/jobs/docker-build.yml'

stages: [test, build]

test:unit:
  extends: .python:test
  stage: test
  variables:
    PYTHON_VERSION: "3.12"
    TEST_PATH: "tests/unit/"
  tags: [docker, lab]

build:image:
  extends: .docker:build
  stage: build
  needs: [test:unit]
  tags: [docker, lab]
```

---

## Lab 06 - Pipeline multi-projets et GitOps

### Objectifs
- Déclencher un pipeline downstream depuis un pipeline upstream
- Implémenter la mise à jour automatique d'un dépôt GitOps
- Observer la traçabilité end-to-end

### Architecture du lab

```
Projet A (app)         Projet B (infra/manifests)
  │                         │
  ├── build                 ├── staging/values.yaml
  ├── test                  └── prod/values.yaml
  └── trigger:deploy ──────► update:manifests
                                  │
                             [ArgoCD / Flux observe]
                                  │
                            [Kubernetes Cluster]
```

### Pipeline Projet A (trigger)

```yaml
# Projet app/.gitlab-ci.yml

trigger:update-manifests:
  stage: deploy:staging
  needs:
    - job: build:docker
      artifacts: true
  trigger:
    project: training/lab06-manifests
    branch: main
    strategy: depend
  variables:
    APP_IMAGE_TAG: "$CI_COMMIT_SHORT_SHA"
    TRIGGERED_BY_PIPELINE: "$CI_PIPELINE_URL"
    TRIGGERED_BY_USER: "$GITLAB_USER_LOGIN"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### Pipeline Projet B (manifests)

```yaml
# Projet lab06-manifests/.gitlab-ci.yml

update:staging-values:
  stage: update
  image: alpine/git:latest
  before_script:
    - git config user.email "ci@company.com"
    - git config user.name "GitLab CI"
    - eval $(ssh-agent -s)
    - echo "$DEPLOY_KEY" | ssh-add -
    - mkdir -p ~/.ssh
    - ssh-keyscan $CI_SERVER_HOST >> ~/.ssh/known_hosts
  script:
    - |
      apk add --no-cache yq
      
      echo "Updating image tag to $APP_IMAGE_TAG"
      yq e ".image.tag = \"$APP_IMAGE_TAG\"" -i staging/values.yaml
      
      git add staging/values.yaml
      git diff --staged --quiet && echo "No changes" && exit 0
      
      git commit -m "ci: update app image to $APP_IMAGE_TAG

      Triggered by: $TRIGGERED_BY_PIPELINE
      By user: $TRIGGERED_BY_USER"
      
      git push "ssh://git@$CI_SERVER_HOST/$CI_PROJECT_PATH.git" HEAD:main
      echo "✅ Manifest updated"
```

---

## Lab 07 - Déploiement Blue/Green et Rollback

### Objectifs
- Implémenter un déploiement Blue/Green sur Kubernetes
- Tester le basculement de trafic
- Effectuer un rollback manuel

### Prérequis
- Accès à un cluster Kubernetes (minikube, kind, ou cluster dédié lab)
- Helm et kubectl installés
- Ingress Controller (Nginx)

### Manifests Kubernetes

```yaml
# k8s/deployment-blue.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab07-blue
  namespace: lab07
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lab07
      color: blue
  template:
    metadata:
      labels:
        app: lab07
        color: blue
    spec:
      containers:
      - name: app
        image: nexus.lab.internal:8082/training/lab07:PLACEHOLDER
        ports:
        - containerPort: 8000
---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: lab07-prod
  namespace: lab07
spec:
  selector:
    app: lab07
    color: blue       # ← Changer ici pour basculer
  ports:
  - port: 80
    targetPort: 8000
```

### Pipeline Blue/Green

```yaml
# Adapter le Lab 03 avec le pattern Blue/Green du Module 07
# Étapes :
# 1. build:docker
# 2. deploy:inactive-color   (déploie sur la couleur inactive)
# 3. smoke:test              (curl sur le service interne)
# 4. switch:traffic          (patch le service)
# 5. rollback (manuel)
```

**Exercice** :
1. Déployer la v1 (color=blue)
2. Builder et déployer la v2 sur green
3. Observer le smoke test
4. Basculer le trafic vers green
5. Simuler une erreur et effectuer un rollback vers blue

---

## Lab 08 - Pipeline Production-Grade Complet

### Objectifs
- Assembler tous les concepts vus dans la formation
- Créer un pipeline complet avec toutes les best practices
- Valider sur une application réelle

### Cahier des charges du pipeline

```
Le pipeline doit :

✅ Avoir une section workflow avec auto-cancel des vieux pipelines
✅ Utiliser des includes depuis le dépôt de templates (Lab 05)
✅ Implémenter le DAG avec needs: pour paralléliser au maximum
✅ Cache correct (push sur install, pull sur les workers parallèles)
✅ Scans sécurité : SAST, secret detection, container scan
✅ Variables dynamiques via dotenv artifacts
✅ Build Docker multi-stage avec push vers Nexus
✅ Deploy staging automatique sur main
✅ Deploy prod manuel sur tag vX.Y.Z
✅ Post-deploy health check
✅ Notification de release
✅ Rollback manuel disponible
✅ Tous les artifacts avec TTL appropriés
✅ Rapports de tests visibles dans GitLab UI
```

### Pipeline final - Squelette à compléter

```yaml
# .gitlab-ci.yml - Pipeline Production-Grade
# LAB 08 - À COMPLÉTER

include:
  - project: 'devops/ci-templates'
    ref: 'main'
    file:
      - '/jobs/python-base.yml'
      - '/jobs/docker-build.yml'
      - '/jobs/deploy-k8s.yml'
      - '/workflows/default.yml'
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml

workflow:
  # TODO : auto-cancel + règles workflow

stages:
  - .pre
  - lint
  - test
  - security
  - build
  - package
  - deploy:staging
  - verify:staging
  - deploy:prod
  - verify:prod
  - .post

variables:
  # TODO : variables globales (Nexus, Docker, etc.)

# TODO : Implémenter tous les jobs selon le cahier des charges
```

### Critères de validation

```
Pipeline vert sur :
  ✅ push sur feature branch        → lint + test + security uniquement
  ✅ push sur main                  → tout sauf deploy:prod
  ✅ création de tag v1.0.0         → pipeline complet avec deploy:prod manuel

Vérifications fonctionnelles :
  ✅ Package Python visible dans Nexus PyPI hosted
  ✅ Image Docker visible dans Nexus Docker Registry
  ✅ Application accessible sur staging après déploiement
  ✅ Health check vert après déploiement staging
  ✅ Release GitLab créée automatiquement sur tag
  ✅ Rollback fonctionnel (tester depuis l'UI)

Qualité du pipeline :
  ✅ Durée totale < 15 minutes sur main branch
  ✅ Aucun secret en clair dans le .gitlab-ci.yml
  ✅ Tous les jobs sensibles avec tags dédiés
  ✅ Artifacts avec expire_in approprié (pas de never)
```

---

## Troubleshooting Guide - Problèmes courants des labs

```
❌ "Job is stuck in pending"
   → Vérifier les tags du runner vs les tags du job
   → Vérifier que le runner est "active" dans Settings > CI/CD > Runners
   → Vérifier concurrent dans config.toml

❌ "Cannot connect to Docker daemon"
   → L'executor Docker nécessite DinD comme service
   → Vérifier DOCKER_HOST = tcp://docker:2376
   → Vérifier DOCKER_TLS_CERTDIR = "/certs"

❌ "pip : 401 Unauthorized" sur Nexus
   → Vérifier NEXUS_USER / NEXUS_PASSWORD dans les CI Variables
   → Tester manuellement : curl -u user:pass https://nexus/repository/pypi-group/simple/

❌ "docker login failed" sur Nexus Registry
   → Vérifier que le port Docker est correct (8082 vs 8085)
   → Vérifier que le compte a le rôle nx-ci-reader (pull) ou nx-ci-writer (push)

❌ "Cache not found" à chaque run
   → Vérifier que la clé du cache est identique entre push et pull
   → Vérifier la configuration S3 du runner (si cache partagé)
   → Vérifier que le job qui fait "push" s'est bien exécuté avant

❌ "artifact not found"
   → Vérifier que needs: inclut artifacts: true
   → Vérifier que le job upstream a bien produit l'artifact
   → Vérifier expire_in (artifact peut avoir expiré)

❌ Gitleaks bloque sur des faux positifs
   → Ajouter une règle dans .gitleaks.toml > [allowlist]
   → Utiliser # gitleaks:allow en commentaire de la ligne

❌ Trivy "FATAL: failed to download vulnerability DB"
   → Configurer un miroir local : TRIVY_DB_REPOSITORY pointant vers Nexus
   → Utiliser un cache : --cache-dir .trivy-cache + configurer le cache GitLab
```

---

## 📋 Récapitulatif de la Formation

Vous avez maintenant couvert :

| Module | Compétence acquise |
|--------|--------------------|
| 01 | Architecture GitLab HA, Runners, Autoscaling, Sécurité |
| 02 | DAG, Rules, Cache/Artifacts, Variables, MR Pipelines |
| 03 | Templates, Includes, Parent-enfant, Monorepo, Secrets OIDC |
| 04 | Nexus types de repos, Docker, Blob Stores, IaC, HA |
| 05 | Intégration pip/Maven/npm/Docker/Helm dans les pipelines |
| 06 | SAST, SCA, Container Scan, DAST, Secret Detection, IaC Scan |
| 07 | Rolling/Blue-Green/Canary, GitOps, Release Management |
| 08 | Métriques Prometheus, Grafana, SLOs, Optimisation |
| 09 | 8 labs progressifs du runner au pipeline production-grade |

**La suite logique** :
- Passer les certifications GitLab CI/CD et GitLab Security
- Explorer GitLab EE : Merge Trains, Protected Environments, DORA Metrics
- Approfondir Sonatype Lifecycle (Nexus IQ) pour le SCA avancé
- Mettre en place les SLOs en production avec Sloth + Prometheus

---

[← Module 08](./08_OBSERVABILITE_PERFORMANCE.md) | [← Index général](./00_INDEX.md)
