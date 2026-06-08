# Module 01 - Architecture GitLab & Runners

> **Durée** : ~4h | **Niveau** : Avancé | [← Index](./00_INDEX.md) | [→ Module 02](./02_GITLAB_CI_FONDAMENTAUX.md)

---

## Sommaire

1. [Topologie GitLab en production](#1-topologie-gitlab-en-production)
2. [Les Runners - Types et Executors](#2-les-runners--types-et-executors)
3. [Registration et configuration avancée](#3-registration-et-configuration-avancée)
4. [Autoscaling des Runners](#4-autoscaling-des-runners)
5. [Sécurisation de l'infrastructure Runner](#5-sécurisation-de-linfrastructure-runner)
6. [Monitoring et troubleshooting](#6-monitoring-et-troubleshooting)

---

## 1. Topologie GitLab en production

### 1.1 Architecture Reference (GitLab HA)

📋 **Concept** - GitLab est une application monolithique modulaire. En production sérieuse, les composants sont séparés :

```
┌─────────────────────────────────────────────────────────────────┐
│                        LOAD BALANCER (HAProxy / Nginx)          │
└──────────────┬────────────────────────┬────────────────────────┘
               │                        │
      ┌────────▼────────┐     ┌─────────▼────────┐
      │  GitLab App 1   │     │  GitLab App 2    │  (Puma + Sidekiq)
      │  (Rails + NGINX)│     │  (Rails + NGINX) │
      └────────┬────────┘     └─────────┬────────┘
               │                        │
      ┌────────▼────────────────────────▼────────┐
      │              Services partagés            │
      │  ┌──────────┐  ┌──────────┐  ┌────────┐  │
      │  │PostgreSQL│  │  Redis   │  │Gitaly  │  │
      │  │ (Primary)│  │ Sentinel │  │Cluster │  │
      │  │(Replica) │  │          │  │        │  │
      │  └──────────┘  └──────────┘  └────────┘  │
      │  ┌──────────┐  ┌──────────────────────┐  │
      │  │  Minio/  │  │   Elasticsearch      │  │
      │  │  Object  │  │   (Advanced Search)  │  │
      │  │  Storage │  └──────────────────────┘  │
      │  └──────────┘                             │
      └───────────────────────────────────────────┘
               │
      ┌────────▼──────────────────────────────────┐
      │              GitLab Runners               │
      │  [Shell]  [Docker]  [K8s]  [Custom]       │
      └───────────────────────────────────────────┘
```

### 1.2 Composants critiques

| Composant | Rôle | Scaling |
|-----------|------|---------|
| **Puma** | Serveur web Rails (HTTP) | Horizontal (plusieurs nodes) |
| **Sidekiq** | Worker asynchrone (jobs CI, emails, webhooks) | Horizontal + queues dédiées |
| **Gitaly** | gRPC pour accès Git | Cluster (HA avec Praefect) |
| **PostgreSQL** | Base de données principale | Primary + replicas (Patroni) |
| **Redis** | Cache, sessions, queues Sidekiq | Sentinel ou Cluster |
| **Object Storage** | Artifacts, LFS, uploads, packages | S3-compatible (Minio, AWS S3, etc.) |

### 1.3 Configuration recommandée `/etc/gitlab/gitlab.rb` (extraits)

```ruby
## ── Accès externe ──────────────────────────────────────────────────
external_url 'https://gitlab.example.com'

## ── PostgreSQL externe ─────────────────────────────────────────────
postgresql['enable'] = false
gitlab_rails['db_host'] = 'postgres.internal'
gitlab_rails['db_port'] = 5432
gitlab_rails['db_database'] = 'gitlabhq_production'
gitlab_rails['db_username'] = 'gitlab'
gitlab_rails['db_password'] = ENV['DB_PASSWORD']  # via env ou vault

## ── Redis externe ──────────────────────────────────────────────────
redis['enable'] = false
gitlab_rails['redis_host'] = 'redis.internal'
gitlab_rails['redis_port'] = 6379

## ── Object Storage (Minio / S3) ────────────────────────────────────
gitlab_rails['object_store']['enabled'] = true
gitlab_rails['object_store']['proxy_download'] = true
gitlab_rails['object_store']['connection'] = {
  'provider' => 'AWS',
  'region' => 'us-east-1',
  'aws_access_key_id' => ENV['S3_ACCESS_KEY'],
  'aws_secret_access_key' => ENV['S3_SECRET_KEY'],
  'endpoint' => 'http://minio.internal:9000',
  'path_style' => true
}
gitlab_rails['artifacts_object_store_enabled'] = true
gitlab_rails['artifacts_object_store_bucket'] = 'gitlab-artifacts'

## ── Sidekiq - queues dédiées CI ────────────────────────────────────
sidekiq['routing_rules'] = [
  ["feature_category=continuous_integration", "pipeline_processing:2"],
  ["*", "default:1"]
]

## ── TLS (Let's Encrypt ou cert custom) ─────────────────────────────
letsencrypt['enable'] = false
nginx['ssl_certificate'] = '/etc/ssl/certs/gitlab.crt'
nginx['ssl_certificate_key'] = '/etc/ssl/private/gitlab.key'

## ── Rate limiting ───────────────────────────────────────────────────
gitlab_rails['rate_limit_requests_per_period'] = 100
gitlab_rails['rate_limit_period'] = 60
```

⚠️ **Piège** - Ne jamais stocker les artifacts sur disque local sur un cluster multi-nodes : ils doivent obligatoirement être sur Object Storage pour être accessibles depuis n'importe quel node Puma/Sidekiq.

---

## 2. Les Runners - Types et Executors

### 2.1 Portée des Runners

```
┌─────────────────────────────────────────────────────┐
│                   Instance Runner                    │
│         (tous les projets de l'instance)             │
│  ┌──────────────────────────────────────────────┐   │
│  │              Group Runner                    │   │
│  │     (tous les projets du groupe)             │   │
│  │  ┌─────────────────────────────────────┐    │   │
│  │  │         Project Runner              │    │   │
│  │  │  (un seul projet spécifique)        │    │   │
│  │  └─────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

💡 **Tip** - Pour les projets sensibles (secrets de prod, certificats), utilisez des **Project Runners dédiés** avec `protected` + `locked` pour éviter qu'un fork ou un autre projet y accède.

### 2.2 Types d'Executors - Comparatif complet

| Executor | Isolation | Perf | Cas d'usage | Limites |
|----------|-----------|------|-------------|---------|
| **shell** | ❌ Aucune | ⚡⚡⚡ | Déploiements simples, accès hardware | Pollution env, sécu faible |
| **docker** | ✅ Bonne | ⚡⚡ | Build généraliste, test isolation | Nécessite Docker socket ou DinD |
| **docker+machine** | ✅ Bonne | ⚡⚡ | Autoscaling cloud (AWS, GCP, Azure) | Docker Machine deprecated |
| **kubernetes** | ✅✅ Forte | ⚡⚡ | Cloud native, autoscaling natif | Latence pod startup |
| **docker-autoscaler** | ✅ Bonne | ⚡⚡ | Autoscaling nouvelle génération | Config plus complexe |
| **custom** | Variable | Variable | Cas exotiques (VMs, LXC, etc.) | À implémenter soi-même |
| **instance** | ✅✅ Forte | ⚡⚡⚡ | Nesting, pas de DinD | GitLab Runner 15.11+ |

### 2.3 Executor Docker - Configurations critiques

```toml
# /etc/gitlab-runner/config.toml

concurrent = 10          # Nombre max de jobs simultanés sur ce runner
check_interval = 3       # Polling interval (secondes)
shutdown_timeout = 30    # Attente d'arrêt propre

[session_server]
  session_timeout = 1800

[[runners]]
  name = "docker-runner-prod"
  url = "https://gitlab.example.com"
  token = "TOKEN_ICI"
  executor = "docker"
  
  [runners.docker]
    image = "alpine:3.18"          # Image par défaut si non spécifiée dans job
    privileged = false             # ⚠️ NE PAS mettre à true sans raison sérieuse
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    
    # Volumes montés dans TOUS les jobs
    volumes = [
      "/cache",                    # Cache local
      "/var/run/docker.sock:/var/run/docker.sock",  # ⚠️ Voir note sécurité
    ]
    
    # Réseau dédié (isolation)
    network_mode = "gitlab-runner-net"
    
    # Ressources limitées
    memory = "2g"
    memory_swap = "2g"
    cpus = "2"
    
    # Pull policy
    pull_policy = ["always", "if-not-present"]  # Essaie always, fallback sur cache
    
    # DNS custom (pour Nexus interne)
    dns = ["192.168.1.10"]
    
    # Variables injectées dans tous les jobs
    environment = [
      "NEXUS_URL=https://nexus.internal",
      "DOCKER_BUILDKIT=1"
    ]
    
    # Services annexes disponibles dans les jobs
    allowed_services = ["docker:dind", "postgres:*", "redis:*"]
    
  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "minio.internal:9000"
      AccessKey = "MINIO_ACCESS"
      SecretKey = "MINIO_SECRET"
      BucketName = "gitlab-runner-cache"
      Insecure = false
```

🔒 **Sécurité - Docker socket vs DinD**

```
Option 1 - Docker socket mount (pratique, dangereux)
  volumes = ["/var/run/docker.sock:/var/run/docker.sock"]
  → Le job peut faire TOUT ce que Docker peut faire sur le host
  → Équivalent root sur le host - À ÉVITER en production

Option 2 - Docker-in-Docker (DinD)
  services: ["docker:24-dind"]
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  → Container isolé, mais nécessite privileged: true
  → Acceptable si le runner host est lui-même isolé (VM dédiée)

Option 3 - Kaniko (recommandé en K8s)
  image: gcr.io/kaniko-project/executor:debug
  → Build d'images sans Docker, sans root, sans socket
  → Idéal pour environnements Kubernetes
```

### 2.4 Executor Kubernetes - Configuration

```toml
[[runners]]
  name = "k8s-runner"
  url = "https://gitlab.example.com"
  token = "TOKEN"
  executor = "kubernetes"

  [runners.kubernetes]
    host = ""                     # Vide = utilise le kubeconfig in-cluster
    namespace = "gitlab-runners"
    namespace_overwrite_allowed = ""
    
    # Image par défaut
    image = "alpine:3.18"
    
    # Pull secrets (pour Nexus registry privé)
    image_pull_secrets = ["nexus-registry-secret"]
    
    # Resources par défaut
    [runners.kubernetes.pod_labels]
      app = "gitlab-runner"
      
    [runners.kubernetes.pod_annotations]
      "cluster-autoscaler.kubernetes.io/safe-to-evict" = "false"
    
    # CPU/RAM par défaut (overridable par job)
    cpu_request = "100m"
    cpu_limit = "2000m"
    memory_request = "128Mi"
    memory_limit = "2Gi"
    
    # Tolérations (pour nodes dédiés CI)
    [[runners.kubernetes.pod_spec]]
      name = "ci-tolerations"
      patch_type = "strategic"
      patch = '''
        tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "gitlab-runner"
          effect: "NoSchedule"
      '''
```

---

## 3. Registration et configuration avancée

### 3.1 Registration avec `gitlab-runner register` (nouvelle méthode token)

Depuis GitLab 16.0, les runner tokens sont **rotatables** et le workflow a changé :

```bash
# Étape 1 - Créer le runner dans l'UI GitLab (Settings > CI/CD > Runners > New runner)
# Copier le token "runner authentication token" (glrt-xxxx)

# Étape 2 - Registration
gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.example.com" \
  --token "glrt-xxxxxxxxxxxxxxxxxxxx" \
  --executor "docker" \
  --docker-image "alpine:3.18" \
  --description "docker-runner-prod-01" \
  --tag-list "docker,linux,prod" \
  --run-untagged false \
  --locked false \
  --access-level "not_protected"

# Étape 3 - Vérifier
gitlab-runner verify --delete  # Supprime les runners qui ne répondent plus
gitlab-runner list
```

### 3.2 Tags - Stratégie de tagging

Les tags sont le mécanisme de **routing** entre jobs et runners. Concevoir une taxonomie claire :

```
Format recommandé : <env>-<executor>-<arch>-<spécialisation>

Exemples :
  prod-docker-amd64          → Runner Docker en prod, architecture x86
  prod-k8s-amd64-gpu         → Runner K8s avec GPU
  staging-docker-arm64       → Runner ARM (Apple M1, Graviton)
  shared-shell-deploy        → Runner shell pour déploiements
  secure-docker-amd64        → Runner isolé pour jobs avec secrets sensibles
```

```yaml
# Utilisation dans le pipeline
build:image:
  tags:
    - prod-docker-amd64
  script:
    - docker build .

deploy:prod:
  tags:
    - shared-shell-deploy    # Runner avec accès SSH au serveur cible
  script:
    - ansible-playbook deploy.yml
```

### 3.3 Variables de configuration avancées

```toml
[[runners]]
  # Limit de jobs simultanés pour CE runner spécifiquement
  limit = 5
  
  # Output limit (évite les logs de 100MB qui saturent)
  output_limit = 4096  # KB, défaut 4MB
  
  # Timeout global par job (surcharge la valeur projet)
  [runners.custom_build_dir]
    enabled = true
    
  # Feature flags
  [runners.feature_flags]
    FF_USE_NEW_BASH_EVAL_STRATEGY = true
    FF_USE_DIRECT_DOWNLOAD = true
    FF_SKIP_NOOP_JOBS_IN_PIPELINE_PROCESSING = true
```

---

## 4. Autoscaling des Runners

### 4.1 Docker Autoscaler (nouvelle génération, recommandé)

Remplace Docker Machine (déprécié). Utilise **Fleeting** pour gérer des pools de VMs.

```toml
[[runners]]
  name = "autoscaler-aws"
  executor = "docker-autoscaler"
  
  [runners.autoscaler]
    plugin = "aws"               # ou "gcp", "azure"
    capacity_per_instance = 2    # Jobs par VM
    max_use_count = 10           # Recycle la VM après N jobs (fresh env)
    max_instances = 20
    
    [runners.autoscaler.plugin_config]
      name = "gitlab-runner-autoscaler"
      region = "eu-west-3"
      instance_type = "c6i.xlarge"
      ami_id = "ami-xxxxxxxxx"   # AMI avec Docker pré-installé
      
      # Spot instances pour réduire les coûts
      use_spot = true
      spot_price = "0.15"
      
    [runners.autoscaler.connector_config]
      username = "ec2-user"
      use_external_addr = false
      
  [runners.autoscaler.policy]
    idle_count = 2               # VMs toujours chaudes
    idle_time = "20m0s"
    scale_factor = 1.5
    scale_factor_limit = 0
```

### 4.2 Autoscaling Kubernetes - Keda / Cluster Autoscaler

Pour K8s, l'autoscaling se fait au niveau de l'infra Kubernetes :

```yaml
# HPA via KEDA pour scaler les runner pods sur métriques GitLab
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: gitlab-runner-scaler
  namespace: gitlab-runners
spec:
  scaleTargetRef:
    name: gitlab-runner-deployment
  minReplicaCount: 1
  maxReplicaCount: 20
  cooldownPeriod: 300
  triggers:
    - type: gitlab
      metadata:
        gitlabApiURL: "https://gitlab.example.com"
        projectID: "42"
        # Scaler sur les jobs en attente
        pendingJobsThreshold: "2"
      authenticationRef:
        name: gitlab-trigger-auth
```

---

## 5. Sécurisation de l'infrastructure Runner

### 5.1 Isolation réseau

```
Architecture sécurisée recommandée :

  Internet
      │
  [Firewall]
      │
  [GitLab Instance] ◄──── HTTPS pull ────► [Runners]
      │                                         │
      │                                    [Réseau privé CI]
      │                              ┌──────────┴──────────┐
      │                           [Nexus]              [SonarQube]
      │                              │
      └──── Deploy SSH/kubectl ──► [Targets prod/staging]
```

```bash
# iptables pour runner Docker - bloquer l'accès au metadata AWS depuis les jobs
iptables -I DOCKER-USER -d 169.254.169.254 -j DROP

# Autoriser seulement Nexus et GitLab depuis les containers
iptables -I DOCKER-USER -d 192.168.1.0/24 -j ACCEPT
iptables -I DOCKER-USER -j DROP
```

### 5.2 Secrets dans les Runners - Bonnes pratiques

🔒 **Ne jamais** mettre de secrets dans `config.toml` en clair. Utiliser :

```bash
# Option 1 - Variables CI/CD GitLab (chiffrées au repos)
# UI : Settings > CI/CD > Variables > Add variable (masked + protected)

# Option 2 - Vault Agent Injector (pour K8s runners)
# Le pod runner récupère les secrets via annotations Vault

# Option 3 - Fichier .env chiffré (sops + age)
sops --decrypt secrets.enc.env > /tmp/runner-secrets.env
source /tmp/runner-secrets.env
```

### 5.3 Hardening du runner

```bash
# Créer un utilisateur dédié non-root
useradd -r -s /bin/false gitlab-runner

# Limiter les capabilities Docker
# Dans config.toml :
# cap_drop = ["ALL"]
# cap_add = ["NET_BIND_SERVICE"]  # seulement si nécessaire

# Apparmor profile pour les containers CI
# docker run --security-opt apparmor=gitlab-runner-profile ...

# Read-only root filesystem (quand possible)
# security_opt = ["no-new-privileges:true"]
# readonly_rootfs = true
# tmpfs = ["/tmp:rw,noexec,nosuid,size=512m"]
```

---

## 6. Monitoring et troubleshooting

### 6.1 Métriques Prometheus du Runner

```toml
# Activer le endpoint métriques
listen_address = ":9252"  # dans config.toml (hors [[runners]])
```

```yaml
# Scrape config Prometheus
scrape_configs:
  - job_name: 'gitlab-runner'
    static_configs:
      - targets: ['runner-host:9252']
```

Métriques clés :
- `gitlab_runner_jobs{state="running"}` - Jobs en cours
- `gitlab_runner_jobs{state="failed"}` - Jobs échoués
- `gitlab_runner_job_duration_seconds` - Durée des jobs (histogram)
- `gitlab_runner_errors_total` - Erreurs runner
- `gitlab_runner_version_info` - Version du runner

### 6.2 Troubleshooting courant

```bash
# Logs runner en temps réel
journalctl -u gitlab-runner -f

# Mode debug (très verbeux, JAMAIS en prod longtemps)
gitlab-runner --debug run

# Vérifier la connexion au GitLab
gitlab-runner verify

# Tester manuellement un job (utile en debug)
gitlab-runner exec docker mon-job-name \
  --docker-image alpine:3.18 \
  --env CI_COMMIT_REF_NAME=main

# Nettoyer les containers orphelins
docker ps -a | grep gitlab-runner | awk '{print $1}' | xargs docker rm -f

# Voir les logs d'un job spécifique depuis le runner
gitlab-runner --log-level debug run --single-run
```

⚠️ **Piège fréquent - "Job is stuck"**

```
Causes communes :
1. Aucun runner avec les tags requis n'est disponible/online
2. Runner en pause (vérifier Settings > Runners > Edit)
3. Job bloqué sur "pending" → runner saturé (concurrent trop bas)
4. Protected branch + runner non marqué "protected"
5. Tag "only protected branches" + pipeline sur MR non-protégée

Vérification rapide :
  Admin > CI/CD > Runners → voir statut, tags, jobs en cours
  Admin > Jobs → filtrer par "stuck" ou "pending"
```

---

## 📋 Récapitulatif Module 01

| Concept | Point clé |
|---------|-----------|
| Architecture GitLab | Composants séparés en prod, Object Storage obligatoire pour multi-node |
| Runner scope | Instance > Group > Project, toujours préférer le scope le plus restreint pour les secrets |
| Executor choice | K8s pour cloud-native, Docker pour isolation, Shell uniquement pour déploiements contrôlés |
| Docker socket | Équivalent root - préférer Kaniko ou DinD avec VM dédiée |
| Autoscaling | Docker Autoscaler (Fleeting) remplace Docker Machine, KEDA pour K8s |
| Tags | Taxonomie `env-executor-arch-spéc`, toujours taguer les runners sensibles |
| Sécurité | Isoler réseau, pas de secrets en clair, iptables pour bloquer metadata cloud |

---

[← Index](./00_INDEX.md) | [→ Module 02 - GitLab CI Fondamentaux](./02_GITLAB_CI_FONDAMENTAUX.md)
