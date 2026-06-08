# Module 07 - Pipeline Patterns & Architecture

> **Durée** : ~5h | [← Module 06](./06_SECURITE_DEVSECOPS.md) | [→ Module 08](./08_OBSERVABILITE_PERFORMANCE.md)

---

## Sommaire

1. [Stratégies de déploiement](#1-stratégies-de-déploiement)
2. [GitOps avec GitLab](#2-gitops-avec-gitlab)
3. [Release Management](#3-release-management)
4. [Multi-environnements et promotion](#4-multi-environnements-et-promotion)
5. [Rollback et gestion des incidents](#5-rollback-et-gestion-des-incidents)
6. [Feature Flags dans le pipeline](#6-feature-flags-dans-le-pipeline)

---

## 1. Stratégies de déploiement

### 1.1 Rolling Update (défaut Kubernetes)

```yaml
deploy:rolling:
  stage: deploy:staging
  image: bitnami/kubectl:latest
  script:
    - |
      kubectl set image deployment/myapp \
        myapp="$NEXUS_REGISTRY/$CI_PROJECT_PATH:$IMAGE_TAG" \
        --namespace myapp-staging
      
      # Attendre le rollout
      kubectl rollout status deployment/myapp \
        --namespace myapp-staging \
        --timeout=5m
  environment:
    name: staging
    url: https://staging.example.com
```

### 1.2 Blue/Green Deployment

```yaml
# Pattern Blue/Green avec Kubernetes et deux Services

deploy:blue-green:
  stage: deploy:prod
  image: bitnami/kubectl:latest
  script:
    - |
      # Détecter la couleur active
      ACTIVE_COLOR=$(kubectl get service myapp-prod \
        -o jsonpath='{.spec.selector.color}' \
        --namespace production)
      
      if [ "$ACTIVE_COLOR" = "blue" ]; then
        DEPLOY_COLOR="green"
        OLD_COLOR="blue"
      else
        DEPLOY_COLOR="blue"
        OLD_COLOR="green"
      fi
      
      echo "Active: $ACTIVE_COLOR → Deploying: $DEPLOY_COLOR"
      
      # Déployer la nouvelle version sur la couleur inactive
      kubectl set image deployment/myapp-${DEPLOY_COLOR} \
        myapp="$NEXUS_REGISTRY/$CI_PROJECT_PATH:$IMAGE_TAG" \
        --namespace production
      
      # Attendre que le nouveau déploiement soit prêt
      kubectl rollout status deployment/myapp-${DEPLOY_COLOR} \
        --namespace production --timeout=10m
      
      # Smoke test sur la nouvelle version (via service interne)
      kubectl run smoke-test --image=curlimages/curl:latest \
        --restart=Never \
        --rm -it \
        --namespace production \
        -- curl -f http://myapp-${DEPLOY_COLOR}-internal/health
      
      # Basculer le trafic vers la nouvelle couleur
      kubectl patch service myapp-prod \
        -p "{\"spec\":{\"selector\":{\"color\":\"${DEPLOY_COLOR}\"}}}" \
        --namespace production
      
      echo "✅ Traffic switched to $DEPLOY_COLOR"
      echo "DEPLOY_COLOR=$DEPLOY_COLOR" >> deploy.env
      echo "OLD_COLOR=$OLD_COLOR" >> deploy.env
  artifacts:
    reports:
      dotenv: deploy.env
  environment:
    name: production
    url: https://app.example.com
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+/
      when: manual

rollback:blue-green:
  stage: deploy:prod
  image: bitnami/kubectl:latest
  needs:
    - job: deploy:blue-green
      artifacts: true
  script:
    - |
      echo "Rolling back to $OLD_COLOR"
      kubectl patch service myapp-prod \
        -p "{\"spec\":{\"selector\":{\"color\":\"${OLD_COLOR}\"}}}" \
        --namespace production
      echo "✅ Rolled back to $OLD_COLOR"
  environment:
    name: production
    action: rollback
  when: manual
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+/
```

### 1.3 Canary Deployment

```yaml
# Pattern Canary avec Nginx Ingress et annotations

deploy:canary:
  stage: deploy:prod
  image: bitnami/kubectl:latest
  variables:
    CANARY_WEIGHT: "10"    # 10% du trafic vers la nouvelle version
  script:
    - |
      # Déployer la version canary (replicas réduits)
      cat << EOF | kubectl apply -f -
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp-canary
        namespace: production
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: myapp
            track: canary
        template:
          metadata:
            labels:
              app: myapp
              track: canary
          spec:
            containers:
            - name: myapp
              image: $NEXUS_REGISTRY/$CI_PROJECT_PATH:$IMAGE_TAG
      EOF
      
      # Configurer le poids du trafic via Ingress
      kubectl annotate ingress myapp \
        nginx.ingress.kubernetes.io/canary="true" \
        nginx.ingress.kubernetes.io/canary-weight="$CANARY_WEIGHT" \
        --overwrite \
        --namespace production
      
      echo "✅ Canary deployed with $CANARY_WEIGHT% traffic"
  environment:
    name: production/canary
    url: https://app.example.com
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+/
      when: manual

promote:canary:
  stage: deploy:prod
  needs: [deploy:canary]
  image: bitnami/kubectl:latest
  script:
    - |
      # Promouvoir la canary en production complète
      kubectl set image deployment/myapp \
        myapp="$NEXUS_REGISTRY/$CI_PROJECT_PATH:$IMAGE_TAG" \
        --namespace production
      kubectl rollout status deployment/myapp --namespace production --timeout=10m
      
      # Supprimer la canary
      kubectl delete deployment myapp-canary --namespace production
      kubectl annotate ingress myapp \
        nginx.ingress.kubernetes.io/canary- \
        --namespace production
      echo "✅ Canary promoted to production"
  environment:
    name: production
    url: https://app.example.com
  when: manual

cleanup:canary:
  stage: deploy:prod
  needs: [deploy:canary]
  image: bitnami/kubectl:latest
  script:
    - kubectl delete deployment myapp-canary --namespace production --ignore-not-found
    - kubectl annotate ingress myapp nginx.ingress.kubernetes.io/canary- --namespace production
    - echo "Canary cleaned up (rollback)"
  environment:
    name: production/canary
    action: stop
  when: manual
```

---

## 2. GitOps avec GitLab

### 2.1 Architecture GitOps Pull-Based (ArgoCD / Flux)

```
┌──────────────────────────────────────────────────────────────────┐
│                     GitLab CI Pipeline                           │
│  [Build] → [Test] → [Push image Nexus] → [Update manifest repo] │
└────────────────────────────────┬─────────────────────────────────┘
                                 │ git push (manifests)
                                 ▼
                    ┌────────────────────────┐
                    │   Repo GitOps          │
                    │   (Helm values /       │
                    │    K8s manifests)      │
                    └────────────┬───────────┘
                                 │ pull (polling / webhook)
                                 ▼
                    ┌────────────────────────┐
                    │   ArgoCD / Flux        │
                    │   (Kubernetes Cluster) │
                    └────────────────────────┘
```

### 2.2 Pipeline GitOps - Mise à jour automatique des manifests

```yaml
# Job qui met à jour le dépôt GitOps après un build réussi
update:gitops-repo:
  stage: deploy:staging
  image: alpine/git:latest
  needs:
    - job: build:image
      artifacts: true    # Récupère IMAGE_TAG depuis dotenv
  variables:
    GITOPS_REPO: "git@gitlab.example.com:devops/gitops-manifests.git"
    GITOPS_DIR: "/tmp/gitops"
  before_script:
    # Configurer SSH pour le push vers le repo GitOps
    - eval $(ssh-agent -s)
    - echo "$GITOPS_DEPLOY_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan gitlab.example.com >> ~/.ssh/known_hosts
    - git config --global user.email "ci@company.com"
    - git config --global user.name "GitLab CI"
  script:
    - |
      git clone "$GITOPS_REPO" "$GITOPS_DIR"
      cd "$GITOPS_DIR"
      
      # Mettre à jour le tag de l'image dans les values Helm
      APP_NAME=$(basename "$CI_PROJECT_PATH")
      VALUES_FILE="apps/${APP_NAME}/values.staging.yaml"
      
      if [ ! -f "$VALUES_FILE" ]; then
        echo "❌ Values file not found: $VALUES_FILE"
        exit 1
      fi
      
      # Remplacer le tag (utilise yq pour modifier le YAML proprement)
      apk add --no-cache yq
      yq e ".image.tag = \"$IMAGE_TAG\"" -i "$VALUES_FILE"
      
      # Commit et push
      git add "$VALUES_FILE"
      git diff --staged --quiet && echo "No changes to commit" && exit 0
      
      git commit -m "ci(${APP_NAME}): update image tag to ${IMAGE_TAG}

      Pipeline: $CI_PIPELINE_URL
      Triggered by: $GITLAB_USER_NAME
      Commit: $CI_COMMIT_SHA"
      
      git push origin main
      
      echo "✅ GitOps manifest updated: $VALUES_FILE → $IMAGE_TAG"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

### 2.3 GitLab Agent for Kubernetes (GitLab-native GitOps)

```yaml
# gitlab-agent/config.yaml (dans le repo k8s-config)
gitops:
  manifest_projects:
    - id: mygroup/k8s-manifests
      default_namespace: production
      paths:
        - glob: "apps/*/manifests/*.yaml"
        - glob: "apps/*/manifests/*.yml"
      reconcile_timeout: "3600s"
      dry_run_strategy: "none"
      prune: true
      prune_timeout: "60s"
      prune_propagation_policy: "foreground"
      inventory_policy: "must_match"
```

```yaml
# Utiliser l'agent dans un pipeline (push-based via tunnel)
deploy:via-agent:
  stage: deploy:prod
  image: bitnami/kubectl:latest
  script:
    - |
      # Le GitLab Agent expose un kubeconfig via CI_KUBERNETES_NAMESPACE
      kubectl config use-context "$KUBE_CONTEXT"
      kubectl apply -f k8s/production/
      kubectl rollout status deployment/myapp --namespace production
  environment:
    name: production
    kubernetes:
      namespace: production
```

---

## 3. Release Management

### 3.1 Pipeline de release complet

```yaml
# Pipeline déclenché sur tag vX.Y.Z
stages:
  - validate
  - build
  - test
  - security
  - package
  - release
  - deploy:prod

# Validation du format du tag
validate:tag-format:
  stage: validate
  script:
    - |
      if ! echo "$CI_COMMIT_TAG" | grep -qE '^v[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9.]+)?$'; then
        echo "❌ Tag format invalide: $CI_COMMIT_TAG"
        echo "   Format attendu: vX.Y.Z ou vX.Y.Z-alpha.1"
        exit 1
      fi
      echo "✅ Tag format valide: $CI_COMMIT_TAG"
  rules:
    - if: $CI_COMMIT_TAG

# Création de la Release GitLab automatique
create:release:
  stage: release
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  needs:
    - validate:tag-format
    - build:wheel         # ou build:image, build:jar...
  script:
    - |
      # Générer les release notes depuis le CHANGELOG
      if [ -f CHANGELOG.md ]; then
        # Extraire la section du tag courant
        NOTES=$(awk "/^## \[${CI_COMMIT_TAG}\]/{found=1; next} /^## \[/{if(found) exit} found{print}" CHANGELOG.md)
      else
        NOTES="Release $CI_COMMIT_TAG"
      fi
      echo "$NOTES" > release-notes.txt
  release:
    tag_name: "$CI_COMMIT_TAG"
    name: "Release $CI_COMMIT_TAG"
    description: "./release-notes.txt"
    assets:
      links:
        - name: "Docker Image"
          url: "https://nexus.internal/repository/docker-hosted-releases/#$CI_PROJECT_PATH:$CI_COMMIT_TAG"
        - name: "Helm Chart"
          url: "https://nexus.internal/repository/helm-hosted/"
        - name: "Python Package"
          url: "https://nexus.internal/repository/pypi-hosted/packages/$CI_PROJECT_NAME/$CI_COMMIT_TAG/"
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/

# Notification de release
notify:release:
  stage: release
  needs: [create:release]
  image: curlimages/curl:latest
  script:
    - |
      # Notification Teams
      curl -H "Content-Type: application/json" \
        -d "{
          \"title\": \"🚀 New Release: $CI_PROJECT_NAME $CI_COMMIT_TAG\",
          \"text\": \"Released by $GITLAB_USER_NAME\n\nPipeline: $CI_PIPELINE_URL\",
          \"themeColor\": \"0076D7\"
        }" \
        "$TEAMS_WEBHOOK_URL"
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/
  allow_failure: true
```

### 3.2 CHANGELOG automatique

```yaml
# Générer le CHANGELOG via conventional commits
generate:changelog:
  stage: .pre
  image: node:20-alpine
  before_script:
    - npm install -g conventional-changelog-cli
  script:
    - conventional-changelog -p angular -i CHANGELOG.md -s -r 0
    - |
      # Committer si des changements
      git config user.email "ci@company.com"
      git config user.name "GitLab CI"
      git add CHANGELOG.md
      git diff --staged --quiet || git commit -m "docs(changelog): update for $CI_COMMIT_TAG [skip ci]"
      git push "https://oauth2:$CI_TOKEN@$CI_SERVER_HOST/$CI_PROJECT_PATH.git" HEAD:$CI_COMMIT_BRANCH
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+/
  allow_failure: true
```

---

## 4. Multi-environnements et promotion

### 4.1 Pipeline de promotion entre environnements

```yaml
# Architecture : DEV → STAGING → PREPROD → PROD
# Chaque promotion est manuelle et traçable

variables:
  ENVIRONMENTS: "dev staging preprod prod"

# Template de déploiement réutilisable
.deploy:template:
  image: alpine/helm:3.14
  before_script:
    - echo "$KUBE_CONFIG" | base64 -d > ~/.kube/config
  script:
    - |
      echo "🚀 Deploying $IMAGE_TAG to $TARGET_ENV"
      helm upgrade --install myapp nexus-helm/myapp \
        --namespace "myapp-${TARGET_ENV}" \
        --create-namespace \
        --version "$CHART_VERSION" \
        --values "chart/values.${TARGET_ENV}.yaml" \
        --set image.tag="$IMAGE_TAG" \
        --wait --timeout 5m --atomic
      echo "✅ Deployed to $TARGET_ENV"
  after_script:
    - rm -f ~/.kube/config

deploy:dev:
  extends: .deploy:template
  stage: deploy:dev
  variables:
    TARGET_ENV: dev
    KUBE_CONFIG: "$KUBE_CONFIG_DEV"
  environment:
    name: dev
    url: https://dev.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy:staging:
  extends: .deploy:template
  stage: deploy:staging
  variables:
    TARGET_ENV: staging
    KUBE_CONFIG: "$KUBE_CONFIG_STAGING"
  needs: [deploy:dev]
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  when: manual

deploy:preprod:
  extends: .deploy:template
  stage: deploy:preprod
  variables:
    TARGET_ENV: preprod
    KUBE_CONFIG: "$KUBE_CONFIG_PREPROD"
  needs: [deploy:staging]
  environment:
    name: preprod
    url: https://preprod.example.com
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+/
  when: manual

deploy:prod:
  extends: .deploy:template
  stage: deploy:prod
  variables:
    TARGET_ENV: prod
    KUBE_CONFIG: "$KUBE_CONFIG_PROD"
  needs: [deploy:preprod]
  environment:
    name: production
    url: https://app.example.com
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/
  when: manual
  # Approval requis (GitLab EE : Protected Environments + Required Approvals)
```

---

## 5. Rollback et gestion des incidents

### 5.1 Rollback automatique vs manuel

```yaml
# Rollback Helm automatique (via --atomic dans le deploy)
# → Déjà couvert par "helm upgrade --atomic"

# Rollback manuel déclenché depuis l'UI GitLab
rollback:prod:
  stage: deploy:prod
  image: alpine/helm:3.14
  before_script:
    - echo "$KUBE_CONFIG_PROD" | base64 -d > ~/.kube/config
  script:
    - |
      # Lister l'historique
      echo "Helm history:"
      helm history myapp --namespace production
      
      # Rollback à la release précédente
      helm rollback myapp 0 \
        --namespace production \
        --wait \
        --timeout 5m
      
      # Vérifier
      helm status myapp --namespace production
      echo "✅ Rolled back successfully"
  environment:
    name: production
    action: rollback
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

# Rollback à une version spécifique (via variable pipeline)
rollback:to-version:
  stage: deploy:prod
  image: alpine/helm:3.14
  before_script:
    - echo "$KUBE_CONFIG_PROD" | base64 -d > ~/.kube/config
  script:
    - |
      if [ -z "$ROLLBACK_TO_TAG" ]; then
        echo "❌ Variable ROLLBACK_TO_TAG manquante"
        echo "   Relancer le pipeline avec ROLLBACK_TO_TAG=vX.Y.Z"
        exit 1
      fi
      
      echo "Rolling back to $ROLLBACK_TO_TAG"
      helm upgrade myapp nexus-helm/myapp \
        --namespace production \
        --reuse-values \
        --set image.tag="$ROLLBACK_TO_TAG" \
        --wait --timeout 5m
  environment:
    name: production
    action: rollback
  when: manual
  rules:
    - if: $CI_PIPELINE_SOURCE == "web"    # Seulement via "Run pipeline" manuel
```

### 5.2 Post-deploy health check

```yaml
.post-deploy-healthcheck:
  image: curlimages/curl:latest
  script:
    - |
      echo "Running health checks on $APP_URL..."
      MAX_RETRIES=20
      RETRY_INTERVAL=15
      
      for i in $(seq 1 $MAX_RETRIES); do
        HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$APP_URL/health")
        if [ "$HTTP_STATUS" = "200" ]; then
          echo "✅ Health check passed (attempt $i/$MAX_RETRIES)"
          exit 0
        fi
        echo "⏳ Attempt $i/$MAX_RETRIES: HTTP $HTTP_STATUS, retrying in ${RETRY_INTERVAL}s..."
        sleep $RETRY_INTERVAL
      done
      
      echo "❌ Health check failed after $MAX_RETRIES attempts"
      exit 1

healthcheck:staging:
  extends: .post-deploy-healthcheck
  stage: deploy:staging
  needs: [deploy:staging]
  variables:
    APP_URL: "https://staging.example.com"

healthcheck:prod:
  extends: .post-deploy-healthcheck
  stage: deploy:prod
  needs: [deploy:prod]
  variables:
    APP_URL: "https://app.example.com"
```

---

## 6. Feature Flags dans le pipeline

### 6.1 Unleash / GitLab Feature Flags

```yaml
# Activer/désactiver des features sans redéploiement
feature-flags:sync:
  stage: deploy:prod
  image: curlimages/curl:latest
  needs: [deploy:prod]
  script:
    - |
      # Activer la feature flag après déploiement réussi
      curl -X POST \
        -H "Authorization: Bearer $UNLEASH_API_TOKEN" \
        -H "Content-Type: application/json" \
        "https://unleash.example.com/api/admin/projects/default/features/new-checkout/environments/production/on"
      echo "✅ Feature flag 'new-checkout' enabled in production"
  rules:
    - if: $CI_COMMIT_TAG =~ /^v[0-9]+/
      when: manual
  allow_failure: true
```

---

## 📋 Récapitulatif Module 07

| Pattern | Cas d'usage | Points clés |
|---------|-------------|-------------|
| Rolling | Déploiement standard sans downtime | Simple, natif K8s |
| Blue/Green | Zero-downtime avec rollback instantané | Double infra, bascule réseau |
| Canary | Validation progressive sur % de trafic | Nécessite monitoring, promotion manuelle |
| GitOps Pull | Audit trail, séparation CI/CD | ArgoCD/Flux, repo manifest dédié |
| GitOps Push | Pipeline simple, contrôle direct | GitLab Agent, moins de friction |
| Release | Tags sémantiques + notes automatiques | Conventional commits, CHANGELOG |
| Promotion | Tracabilité entre envs, approbation humaine | Manual gate, Protected Environments |
| Rollback | Helm rollback ou re-déploiement par tag | Toujours garder N-1 accessible |

---

[← Module 06](./06_SECURITE_DEVSECOPS.md) | [→ Module 08 - Observabilité & Performance](./08_OBSERVABILITE_PERFORMANCE.md)
