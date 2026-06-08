# Module 05 — Nexus : Intégration CI/CD

> **Durée** : ~4h | [← Module 04](./04_NEXUS_ARCHITECTURE.md) | [→ Module 06](./06_SECURITE_DEVSECOPS.md)

---

## Sommaire

1. [Python — pip et twine](#1-python--pip-et-twine)
2. [Maven — Java/Kotlin/Scala](#2-maven--javakotlinscala)
3. [npm — JavaScript/TypeScript](#3-npm--javascripttypescript)
4. [Docker — Build et Push](#4-docker--build-et-push)
5. [Helm Charts](#5-helm-charts)
6. [Artifacts Raw (binaires génériques)](#6-artifacts-raw-binaires-génériques)
7. [Stratégies de versioning](#7-stratégies-de-versioning)

---

## 1. Python — pip et twine

### 1.1 Configuration pip côté runner

```ini
# pip.conf (à créer dans le job ou dans l'image Docker)
[global]
index-url = https://nexus.internal/repository/pypi-group/simple/
trusted-host = nexus.internal

# Avec authentification
[global]
index-url = https://ci-reader:${NEXUS_TOKEN}@nexus.internal/repository/pypi-group/simple/
```

### 1.2 Jobs CI/CD Python complets

```yaml
# Variables globales du projet (Settings > CI/CD > Variables)
# NEXUS_URL = https://nexus.internal
# NEXUS_USER = ci-reader (masked)
# NEXUS_PASSWORD = <token> (masked)
# NEXUS_WRITER_USER = ci-writer (masked)
# NEXUS_WRITER_PASSWORD = <token> (masked)

variables:
  PIP_INDEX_URL: "https://${NEXUS_USER}:${NEXUS_PASSWORD}@nexus.internal/repository/pypi-group/simple/"
  PIP_TRUSTED_HOST: "nexus.internal"
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  TWINE_REPOSITORY_URL: "https://nexus.internal/repository/pypi-hosted/"
  TWINE_USERNAME: "$NEXUS_WRITER_USER"
  TWINE_PASSWORD: "$NEXUS_WRITER_PASSWORD"

# ── Installer les dépendances ───────────────────────────────────
build:deps:
  stage: .pre
  image: python:3.12-slim
  script:
    - python -m venv .venv
    - .venv/bin/pip install --upgrade pip wheel setuptools
    - .venv/bin/pip install -r requirements.txt
  cache:
    key:
      files: [requirements.txt, requirements-dev.txt]
    paths: [.cache/pip/, .venv/]
    policy: push

# ── Tests ───────────────────────────────────────────────────────
test:unit:
  stage: test
  image: python:3.12-slim
  needs: [build:deps]
  cache:
    key:
      files: [requirements.txt, requirements-dev.txt]
    paths: [.cache/pip/, .venv/]
    policy: pull
  script:
    - .venv/bin/pytest tests/unit/ 
        --junitxml=reports/junit.xml
        --cov=src
        --cov-report=xml:reports/coverage.xml
        --cov-report=term-missing
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    reports:
      junit: reports/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: reports/coverage.xml
    paths: [reports/]
    expire_in: 30 days

# ── Build wheel ─────────────────────────────────────────────────
build:wheel:
  stage: build
  image: python:3.12-slim
  needs: [test:unit]
  script:
    - .venv/bin/pip install build
    - python -m build --wheel --sdist
    - ls -la dist/
  artifacts:
    paths: [dist/]
    expire_in: 7 days
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+/

# ── Publish vers Nexus PyPI hosted ──────────────────────────────
publish:pypi:
  stage: package
  image: python:3.12-slim
  needs: [build:wheel]
  script:
    - pip install twine
    - |
      twine upload \
        --repository-url "$TWINE_REPOSITORY_URL" \
        --username "$TWINE_USERNAME" \
        --password "$TWINE_PASSWORD" \
        --non-interactive \
        dist/*
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+/
```

---

## 2. Maven — Java/Kotlin/Scala

### 2.1 Configuration Maven (`settings.xml`)

```xml
<!-- ~/.m2/settings.xml ou généré dynamiquement dans le pipeline -->
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">
  
  <!-- Serveurs d'authentification -->
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>${env.NEXUS_WRITER_USER}</username>
      <password>${env.NEXUS_WRITER_PASSWORD}</password>
    </server>
    <server>
      <id>nexus-snapshots</id>
      <username>${env.NEXUS_WRITER_USER}</username>
      <password>${env.NEXUS_WRITER_PASSWORD}</password>
    </server>
    <server>
      <id>nexus-group</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
  </servers>
  
  <!-- Mirror : toute requête Maven passe par Nexus -->
  <mirrors>
    <mirror>
      <id>nexus-all</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.internal/repository/maven-group/</url>
    </mirror>
  </mirrors>
  
  <!-- Profil par défaut -->
  <profiles>
    <profile>
      <id>nexus</id>
      <repositories>
        <repository>
          <id>nexus-group</id>
          <url>https://nexus.internal/repository/maven-group/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>nexus-group</id>
          <url>https://nexus.internal/repository/maven-group/</url>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
  
  <activeProfiles>
    <activeProfile>nexus</activeProfile>
  </activeProfiles>
  
</settings>
```

### 2.2 `pom.xml` — Distribution management

```xml
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <name>Nexus Release Repository</name>
    <url>https://nexus.internal/repository/maven-hosted-releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <name>Nexus Snapshot Repository</name>
    <url>https://nexus.internal/repository/maven-hosted-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

### 2.3 Pipeline Maven complet

```yaml
variables:
  MAVEN_OPTS: >-
    -Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository
    -Djava.awt.headless=true
    -Dmaven.color=false
  MAVEN_CLI_OPTS: >-
    --batch-mode
    --no-transfer-progress
    --fail-at-end
    -Dstyle.color=never
  SETTINGS_XML: "$CI_PROJECT_DIR/settings.xml"

# Générer settings.xml dynamiquement (sécurité : pas en dur dans le repo)
.prepare:maven:
  before_script:
    - |
      cat > settings.xml << 'EOF'
      <settings>
        <servers>
          <server>
            <id>nexus-releases</id>
            <username>${env.NEXUS_WRITER_USER}</username>
            <password>${env.NEXUS_WRITER_PASSWORD}</password>
          </server>
          <!-- ... -->
        </servers>
        <mirrors>
          <mirror>
            <id>nexus</id>
            <mirrorOf>*</mirrorOf>
            <url>https://nexus.internal/repository/maven-group/</url>
          </mirror>
        </mirrors>
      </settings>
      EOF

build:java:
  extends: .prepare:maven
  stage: build
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn $MAVEN_CLI_OPTS -s settings.xml clean package -DskipTests
  cache:
    key:
      files: [pom.xml]
      prefix: maven-$CI_JOB_IMAGE
    paths: [.m2/repository/]
  artifacts:
    paths: [target/*.jar]
    expire_in: 1 day

test:java:
  extends: .prepare:maven
  stage: test
  image: maven:3.9-eclipse-temurin-21
  needs: [build:java]
  services:
    - name: postgres:15-alpine
      alias: db
      variables:
        POSTGRES_DB: testdb
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
  script:
    - mvn $MAVEN_CLI_OPTS -s settings.xml test
  cache:
    key:
      files: [pom.xml]
      prefix: maven-$CI_JOB_IMAGE
    paths: [.m2/repository/]
    policy: pull
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml
    when: always

deploy:nexus:
  extends: .prepare:maven
  stage: package
  image: maven:3.9-eclipse-temurin-21
  needs: [test:java]
  script:
    - mvn $MAVEN_CLI_OPTS -s settings.xml deploy -DskipTests
  cache:
    key:
      files: [pom.xml]
      prefix: maven-$CI_JOB_IMAGE
    paths: [.m2/repository/]
    policy: pull
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH  # Snapshots sur main
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+/              # Releases sur tag
```

---

## 3. npm — JavaScript/TypeScript

### 3.1 Configuration npm

```bash
# .npmrc (à la racine du projet ou généré dynamiquement)
registry=https://nexus.internal/repository/npm-group/
//nexus.internal/repository/npm-group/:_auth=${NPM_AUTH_TOKEN}
//nexus.internal/repository/npm-hosted/:_auth=${NPM_AUTH_TOKEN}
email=ci@company.com
always-auth=true

# Générer le token Base64 :
# echo -n "ci-writer:password" | base64
```

### 3.2 Pipeline npm complet

```yaml
variables:
  NPM_CONFIG_REGISTRY: "https://nexus.internal/repository/npm-group/"
  NODE_ENV: "development"

build:frontend:
  stage: build
  image: node:20-alpine
  before_script:
    # Configurer l'authentification Nexus pour npm
    - |
      NPM_TOKEN=$(echo -n "$NEXUS_WRITER_USER:$NEXUS_WRITER_PASSWORD" | base64)
      echo "//nexus.internal/repository/npm-group/:_auth=${NPM_TOKEN}" > .npmrc
      echo "//nexus.internal/repository/npm-hosted/:_auth=${NPM_TOKEN}" >> .npmrc
      echo "always-auth=true" >> .npmrc
    - npm ci --prefer-offline  # Utilise package-lock.json strict
  script:
    - npm run build
    - npm run test -- --coverage --ci
  cache:
    key:
      files: [package-lock.json]
      prefix: npm
    paths: [node_modules/]
  artifacts:
    paths: [dist/]
    reports:
      junit: coverage/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

publish:npm:
  stage: package
  image: node:20-alpine
  needs: [build:frontend]
  before_script:
    - |
      NPM_TOKEN=$(echo -n "$NEXUS_WRITER_USER:$NEXUS_WRITER_PASSWORD" | base64)
      echo "//nexus.internal/repository/npm-hosted/:_auth=${NPM_TOKEN}" > .npmrc
  script:
    # Vérifier que la version n'existe pas déjà
    - |
      PACKAGE_NAME=$(node -p "require('./package.json').name")
      PACKAGE_VERSION=$(node -p "require('./package.json').version")
      if npm view "$PACKAGE_NAME@$PACKAGE_VERSION" --registry="$NPM_CONFIG_REGISTRY" 2>/dev/null; then
        echo "Version $PACKAGE_VERSION already exists!"
        exit 1
      fi
    - npm publish --registry="https://nexus.internal/repository/npm-hosted/"
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+/
```

---

## 4. Docker — Build et Push

### 4.1 Pattern Docker complet avec Nexus

```yaml
variables:
  NEXUS_REGISTRY: "nexus-docker.internal"
  NEXUS_REGISTRY_PORT: "8082"
  IMAGE_NAME: "${NEXUS_REGISTRY}:${NEXUS_REGISTRY_PORT}/${CI_PROJECT_PATH}"
  DOCKER_BUILDKIT: "1"

# Calcul des tags selon le contexte
prepare:image-tags:
  stage: .pre
  image: alpine:3.18
  script:
    - apk add --no-cache git
    - |
      if [ -n "$CI_COMMIT_TAG" ]; then
        # Release : tag sémantique complet
        TAGS="${IMAGE_NAME}:${CI_COMMIT_TAG}"
        # Ajouter aussi les tags majeur et majeur.mineur
        MAJOR=$(echo $CI_COMMIT_TAG | cut -d. -f1 | tr -d 'v')
        MINOR=$(echo $CI_COMMIT_TAG | cut -d. -f1-2 | tr -d 'v')
        TAGS="$TAGS,${IMAGE_NAME}:${MAJOR},${IMAGE_NAME}:${MINOR},${IMAGE_NAME}:latest"
      elif [ "$CI_COMMIT_BRANCH" = "$CI_DEFAULT_BRANCH" ]; then
        TAGS="${IMAGE_NAME}:main-${CI_COMMIT_SHORT_SHA},${IMAGE_NAME}:edge"
      else
        BRANCH_SLUG=$(echo "$CI_COMMIT_BRANCH" | tr '/' '-' | tr '[:upper:]' '[:lower:]')
        TAGS="${IMAGE_NAME}:${BRANCH_SLUG}-${CI_COMMIT_SHORT_SHA}"
      fi
      echo "IMAGE_TAGS=$TAGS" > build.env
      echo "PRIMARY_TAG=${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA}" >> build.env
      cat build.env
  artifacts:
    reports:
      dotenv: build.env

build:image:
  stage: build
  image: docker:24-cli
  needs: [prepare:image-tags]
  services:
    - name: docker:24-dind
      variables:
        DOCKER_TLS_CERTDIR: "/certs"
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - echo "$NEXUS_PASSWORD" | docker login "$NEXUS_REGISTRY:$NEXUS_REGISTRY_PORT" 
        -u "$NEXUS_USER" --password-stdin
  script:
    - |
      # Build avec cache depuis Nexus
      docker buildx create --use --name builder 2>/dev/null || true
      
      # Construire les arguments --tag dynamiquement
      TAG_ARGS=""
      for tag in $(echo "$IMAGE_TAGS" | tr ',' '\n'); do
        TAG_ARGS="$TAG_ARGS --tag $tag"
      done
      
      docker buildx build \
        $TAG_ARGS \
        --cache-from "type=registry,ref=${IMAGE_NAME}:buildcache" \
        --cache-to "type=registry,ref=${IMAGE_NAME}:buildcache,mode=max" \
        --build-arg APP_VERSION="${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}" \
        --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        --build-arg GIT_COMMIT="$CI_COMMIT_SHA" \
        --label "org.opencontainers.image.version=${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}" \
        --label "org.opencontainers.image.created=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        --label "org.opencontainers.image.source=$CI_PROJECT_URL" \
        --push \
        .
      
      echo "✅ Image(s) poussée(s) :"
      echo "$IMAGE_TAGS" | tr ',' '\n'
```

### 4.2 Dockerfile optimisé pour les pipelines CI

```dockerfile
# ── Stage 1 : Builder ───────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /build

# Installer les dépendances (layer cacheable)
COPY requirements.txt .
# Utiliser Nexus comme source pip
ARG PIP_INDEX_URL=https://nexus.internal/repository/pypi-group/simple/
ARG PIP_TRUSTED_HOST=nexus.internal
RUN pip install --upgrade pip \
    && pip install --user --no-cache-dir -r requirements.txt

# Copier et builder l'application
COPY src/ ./src/
RUN pip install --user --no-cache-dir -e .

# ── Stage 2 : Runner (image minimale) ──────────────────────────
FROM python:3.12-slim AS runner

# Sécurité : utilisateur non-root
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copier uniquement le nécessaire depuis le builder
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser src/ ./src/

# Métadonnées OCI
ARG APP_VERSION=unknown
ARG BUILD_DATE=unknown
ARG GIT_COMMIT=unknown
LABEL org.opencontainers.image.version="$APP_VERSION" \
      org.opencontainers.image.created="$BUILD_DATE" \
      org.opencontainers.image.revision="$GIT_COMMIT"

USER appuser

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 5. Helm Charts

### 5.1 Configuration Nexus pour Helm

```bash
# Nexus : créer un repo Helm hosted
curl -X POST -u admin:password \
  https://nexus.internal/service/rest/v1/repositories/helm/hosted \
  -H "Content-Type: application/json" \
  -d '{
    "name": "helm-hosted",
    "online": true,
    "storage": {"blobStoreName": "blob-default", "strictContentTypeValidation": true}
  }'
```

### 5.2 Pipeline Helm complet

```yaml
build:helm:
  stage: build
  image: alpine/helm:3.14
  script:
    # Lint du chart
    - helm lint ./chart/ --strict
    
    # Rendre le template et valider
    - helm template myapp ./chart/ 
        --values chart/values.staging.yaml
        --debug > /dev/null
    
    # Package avec la version du tag ou du commit
    - |
      if [ -n "$CI_COMMIT_TAG" ]; then
        CHART_VERSION="${CI_COMMIT_TAG#v}"
        APP_VERSION="$CI_COMMIT_TAG"
      else
        CHART_VERSION="0.0.0-${CI_COMMIT_SHORT_SHA}"
        APP_VERSION="$CI_COMMIT_SHORT_SHA"
      fi
      helm package ./chart/ \
        --version "$CHART_VERSION" \
        --app-version "$APP_VERSION" \
        --destination ./helm-packages/
    
    - ls -la helm-packages/
  artifacts:
    paths: [helm-packages/]
    expire_in: 7 days

publish:helm:
  stage: package
  image: curlimages/curl:8.5.0
  needs: [build:helm]
  script:
    - |
      for chart in helm-packages/*.tgz; do
        echo "Uploading $chart to Nexus..."
        curl --fail \
          --user "${NEXUS_WRITER_USER}:${NEXUS_WRITER_PASSWORD}" \
          --upload-file "$chart" \
          "https://nexus.internal/repository/helm-hosted/"
        echo "✅ Uploaded: $chart"
      done
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+/
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy:k8s:
  stage: deploy:staging
  image: alpine/helm:3.14
  needs: [publish:helm]
  environment:
    name: staging
    url: https://staging.example.com
  before_script:
    # Ajouter le repo Helm Nexus
    - helm repo add nexus-helm 
        "https://nexus.internal/repository/helm-hosted/"
        --username "$NEXUS_USER"
        --password "$NEXUS_PASSWORD"
    - helm repo update
    - echo "$KUBE_CONFIG_STAGING" | base64 -d > ~/.kube/config
  script:
    - |
      helm upgrade --install myapp nexus-helm/myapp \
        --namespace myapp-staging \
        --create-namespace \
        --version "${CHART_VERSION:-0.0.0-${CI_COMMIT_SHORT_SHA}}" \
        --values chart/values.staging.yaml \
        --set image.tag="$CI_COMMIT_SHORT_SHA" \
        --set image.registry="$NEXUS_REGISTRY" \
        --wait \
        --timeout 5m \
        --atomic  # Rollback automatique en cas d'échec
```

---

## 6. Artifacts Raw (binaires génériques)

```yaml
# Upload d'un binaire quelconque vers Nexus Raw
publish:binary:
  stage: package
  image: curlimages/curl:8.5.0
  needs: [build:app]
  script:
    - |
      VERSION="${CI_COMMIT_TAG:-${CI_COMMIT_SHORT_SHA}}"
      ARTIFACT_PATH="myapp/${VERSION}/myapp-${VERSION}-linux-amd64.tar.gz"
      
      # Créer l'archive
      tar czf "myapp-${VERSION}-linux-amd64.tar.gz" dist/
      
      # Upload vers Nexus raw
      curl --fail \
        --user "${NEXUS_WRITER_USER}:${NEXUS_WRITER_PASSWORD}" \
        --upload-file "myapp-${VERSION}-linux-amd64.tar.gz" \
        "https://nexus.internal/repository/raw-hosted-binaries/${ARTIFACT_PATH}"
      
      # Générer le checksum
      sha256sum "myapp-${VERSION}-linux-amd64.tar.gz" > checksums.txt
      curl --fail \
        --user "${NEXUS_WRITER_USER}:${NEXUS_WRITER_PASSWORD}" \
        --upload-file checksums.txt \
        "https://nexus.internal/repository/raw-hosted-binaries/myapp/${VERSION}/checksums.txt"
      
      echo "✅ Published: https://nexus.internal/repository/raw-hosted-binaries/${ARTIFACT_PATH}"
```

---

## 7. Stratégies de versioning

### 7.1 Semantic Versioning automatique

```yaml
# Utilisation de semantic-release ou bump2version

prepare:version:
  stage: .pre
  image: python:3.12-slim
  script:
    - pip install bump2version
    - |
      # Lire le message du commit pour déterminer le bump
      COMMIT_MSG="$CI_COMMIT_MESSAGE"
      
      if echo "$COMMIT_MSG" | grep -q "BREAKING CHANGE\|!:"; then
        BUMP="major"
      elif echo "$COMMIT_MSG" | grep -qE "^feat(\(.+\))?!?:"; then
        BUMP="minor"
      else
        BUMP="patch"
      fi
      
      # Lire la version actuelle
      CURRENT=$(python -c "import configparser; c=configparser.ConfigParser(); c.read('setup.cfg'); print(c['metadata']['version'])")
      
      # Calculer la prochaine version
      bump2version --dry-run --list $BUMP | grep new_version | sed 's/new_version=//' > next-version.txt
      echo "NEXT_VERSION=$(cat next-version.txt)" >> build.env
      echo "BUMP_TYPE=$BUMP" >> build.env
  artifacts:
    reports:
      dotenv: build.env
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### 7.2 Convention de nommage des images Docker

```
Pattern recommandé :

Pour les développements :
  nexus.internal:8083/group/project:branch-sha8
  nexus.internal:8083/myapp/api:feature-auth-a1b2c3d4

Pour la production (tagged releases) :
  nexus.internal:8082/group/project:1.2.3          ← Tag exact
  nexus.internal:8082/group/project:1.2             ← Alias mineur
  nexus.internal:8082/group/project:1               ← Alias majeur
  nexus.internal:8082/group/project:latest          ← Dernier stable

Pour le cache de build :
  nexus.internal:8082/group/project:buildcache      ← Cache BuildKit

⚠️ Ne jamais utiliser :latest comme seul tag en production !
   Toujours conserver le tag sémantique ou le SHA pour les rollbacks.
```

---

## 📋 Récapitulatif Module 05

| Format | Clé d'intégration |
|--------|-------------------|
| pip | `PIP_INDEX_URL` avec authentification, `twine` pour publish |
| Maven | `settings.xml` dynamique, mirror `*` vers Nexus Group |
| npm | `.npmrc` avec `_auth` Base64, `--prefer-offline` pour le cache |
| Docker | Login avant build/push, tags multiples, buildcache dans Nexus |
| Helm | `helm repo add` + `helm upgrade --atomic` pour rollback auto |
| Raw | curl upload, checksum automatique |
| Versioning | Ne jamais deployer latest seul, SemVer avec aliases de tags |

---

[← Module 04](./04_NEXUS_ARCHITECTURE.md) | [→ Module 06 — Sécurité DevSecOps](./06_SECURITE_DEVSECOPS.md)
