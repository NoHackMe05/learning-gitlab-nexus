# Module 04 - Nexus Repository Manager : Architecture

> **Durée** : ~3h | [← Module 03](./03_GITLAB_CI_AVANCE.md) | [→ Module 05](./05_NEXUS_INTEGRATION.md)

---

## Sommaire

1. [Types de repositories](#1-types-de-repositories)
2. [Formats supportés](#2-formats-supportés)
3. [Blob Stores et stockage](#3-blob-stores-et-stockage)
4. [Sécurité et contrôle d'accès](#4-sécurité-et-contrôle-daccès)
5. [Configuration via REST API / IaC](#5-configuration-via-rest-api--iac)
6. [Haute disponibilité et clustering](#6-haute-disponibilité-et-clustering)

---

## 1. Types de repositories

### 1.1 Les trois types fondamentaux

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │ Unique point d'entrée
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    GROUP REPOSITORY                              │
│           (agrège proxy + hosted, lecture seule)                │
└───────┬─────────────────────────────────────────┬──────────────┘
        │                                         │
        ▼                                         ▼
┌───────────────────┐                 ┌───────────────────────────┐
│  PROXY REPOSITORY │                 │    HOSTED REPOSITORY      │
│  (cache miroir)   │                 │    (stockage interne)     │
│                   │                 │                           │
│  Maven Central    │                 │  Vos artifacts internes   │
│  PyPI             │                 │  Vos images Docker        │
│  npm registry     │                 │  Vos Helm charts          │
│  Docker Hub       │                 │  Releases officielles     │
└─────────┬─────────┘                 └───────────────────────────┘
          │
          ▼
    [Internet / Source Upstream]
```

### 1.2 Comportement du Group Repository

```
Requête de l'artifact "requests==2.31.0"
  │
  ├─► Group cherche dans : [pypi-hosted, pypi-proxy-pypi, pypi-proxy-torch]
  │     (ordre défini dans la config du group)
  │
  ├─ 1. pypi-hosted : artifact présent ? OUI → retourner
  │
  ├─ 2. pypi-proxy-pypi : artifact en cache ? OUI → retourner
  │                                           NON → fetch depuis PyPI + cacher
  │
  └─ 3. pypi-proxy-torch : (non atteint si trouvé avant)
```

### 1.3 Repositories à créer par environnement

```
Configuration recommandée pour une organisation :

Maven :
  maven-hosted-releases      → Artifacts de release internes (RELEASE uniquement)
  maven-hosted-snapshots     → Artifacts snapshot internes (SNAPSHOT uniquement)
  maven-proxy-central        → Cache de Maven Central
  maven-proxy-spring         → Cache du repo Spring
  maven-group                → [releases, snapshots, proxy-central, proxy-spring]

Python (pip) :
  pypi-hosted                → Packages internes
  pypi-proxy-pypi            → Cache de PyPI officiel
  pypi-group                 → [pypi-hosted, pypi-proxy-pypi]

Docker :
  docker-hosted-releases     → Images de production (port 8082)
  docker-hosted-snapshots    → Images de dev/test (port 8083)
  docker-proxy-hub           → Cache Docker Hub (port 8084)
  docker-group               → [releases, snapshots, proxy-hub] (port 8085)

npm :
  npm-hosted                 → Packages internes
  npm-proxy-npmjs            → Cache de npmjs.com
  npm-group                  → [npm-hosted, npm-proxy-npmjs]

Helm :
  helm-hosted                → Charts internes
  helm-proxy-stable          → Cache du repo Helm stable

Raw :
  raw-hosted-binaries        → Binaires, scripts, archives diverses
```

---

## 2. Formats supportés

### 2.1 Configuration Maven (Hosted + Proxy)

```
Via UI : Repository > Create repository > maven2(hosted)

Paramètres clés :
  Name : maven-hosted-releases
  Version policy : Release          ← IMPORTANT : bloque les SNAPSHOTs
  Layout policy : Strict
  Deployment policy : Disable redeploy  ← Immuabilité des releases
  
  Name : maven-hosted-snapshots
  Version policy : Snapshot
  Deployment policy : Allow redeploy  ← Les snapshots peuvent être réécrites

  Name : maven-proxy-central
  Remote storage : https://repo1.maven.org/maven2/
  Maximum component age : 1440 (24h)   ← Durée du cache
  Maximum metadata age : 1440
  Negative cache enabled : true
  Negative cache TTL : 1440
```

### 2.2 Configuration Docker Registry

```
⚠️ Docker nécessite des ports dédiés (pas de path-based routing) :

Hosted port 8082 → Production images
Hosted port 8083 → Dev/staging images
Proxy  port 8084 → Docker Hub cache
Group  port 8085 → Point d'entrée unique

Configuration Docker hosted :
  HTTP port : 8082
  Enable Docker V1 API : false (V2 only)
  Allow anonymous pull : false
  Force basic authentication : true

Configuration Docker proxy :
  Remote storage : https://registry-1.docker.io
  Docker index : Use Docker Hub (pour la recherche)
  Cache foreign layers : true
```

```nginx
# Configuration Nginx reverse proxy pour Docker Nexus
# Nécessaire pour TLS et le routage par port

server {
    listen 443 ssl;
    server_name nexus-docker.internal;
    
    ssl_certificate /etc/ssl/certs/nexus.crt;
    ssl_certificate_key /etc/ssl/private/nexus.key;
    
    # Docker prod (port 8082)
    location / {
        proxy_pass http://nexus:8082;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Taille max des layers Docker (important !)
        client_max_body_size 10g;
        proxy_read_timeout 600;
        proxy_connect_timeout 600;
        
        # Chunked transfer (nécessaire pour Docker)
        chunked_transfer_encoding on;
    }
}
```

### 2.3 Politique de cleanup - Indispensable en production

```
Sans cleanup policy, Nexus grossit indéfiniment !

Créer via : System > Cleanup Policies > Create Cleanup Policy

Policy : "cleanup-docker-snapshots"
  Format : Docker
  Criteria :
    ✅ Component age : > 30 days    (artifacts non utilisés depuis 30j)
    ✅ Component usage : not downloaded in 60 days
    ✅ Release type : Pre-release/Snapshot versions only
    ☐ Asset name regex : (optionnel, ex: ".*-SNAPSHOT.*")

Policy : "cleanup-maven-snapshots"
  Format : Maven 2
  Release type : Snapshots only
  Component age : > 7 days

Policy : "cleanup-all-undownloaded"
  Format : All formats
  Component usage : not downloaded in 180 days

Attacher les policies aux repos :
  Repository > Edit > Cleanup policies > Ajouter la policy

Exécuter le cleanup (tâche) :
  System > Tasks > Create task > "Admin - Compact blob store"
  → Cleanup supprime les métadonnées, Compact libère l'espace disque réel
```

---

## 3. Blob Stores et stockage

### 3.1 Types de Blob Stores

```
File (défaut) :
  → Répertoire local sur le filesystem Nexus
  → Simple mais pas HA (single point of failure)
  → Chemin : /nexus-data/blobs/<nom>/

S3 (Pro) :
  → Compatible AWS S3, MinIO, etc.
  → Nécessite Nexus Pro pour le support officiel
  → En OSS : possible via montage S3FS (non recommandé)

Azure Blob (Pro) :
  → Microsoft Azure Blob Storage
```

### 3.2 Stratégie de Blob Stores

```
Recommandation pour une organisation :

  blob-default       → Tous les repos sauf exceptions (File local ou NFS)
  blob-docker        → Dédié Docker (gros volumes, IOPS élevées)
  blob-releases      → Artifacts de release (backup prioritaire)
  blob-temp          → Snapshots et builds temporaires

Avantages de la séparation :
  ✅ Politiques de backup différentes
  ✅ Monitoring d'usage par type
  ✅ Possibilité de migrer un type de repo vers S3 sans toucher les autres
  ✅ Soft quotas par blob store
```

### 3.3 Soft Quotas

```
System > Blob Stores > Edit > Soft Quota

Types :
  spaceRemainingQuota  : Alerte si espace RESTANT < X MB
  spaceUsedQuota       : Alerte si espace UTILISÉ > X MB

Configuration recommandée par taille :
  < 100GB total  : spaceRemainingQuota 10GB
  100-500GB      : spaceRemainingQuota 20GB
  > 500GB        : spaceRemainingQuota 10% de la capacité

Les quotas déclenchent uniquement des alertes (pas de blocage) !
Surveiller via l'API ou les métriques Prometheus.
```

---

## 4. Sécurité et contrôle d'accès

### 4.1 Modèle de sécurité Nexus

```
Utilisateur
  └─► Rôle(s)
         └─► Privilège(s)
                └─► Action(s) sur Repository/Format/Composant
```

### 4.2 Rôles et privilèges recommandés

```
Rôles à créer :

nx-ci-reader :
  Privilèges :
    - nx-repository-view-*-*-read     (lecture tous formats)
    - nx-repository-view-*-*-browse   (navigation)
  Usage : comptes de service CI pour télécharger les dépendances

nx-ci-writer :
  Privilèges :
    - nx-repository-view-*-hosted-*   (écriture sur hosted uniquement)
    - nx-repository-view-*-*-read
  Usage : comptes de service CI pour publier les artifacts

nx-docker-puller :
  Privilèges :
    - nx-repository-view-docker-*-read
    - nx-repository-view-docker-*-browse
  Usage : nœuds Kubernetes pour puller les images

nx-helm-releaser :
  Privilèges :
    - nx-repository-view-helm-helm-hosted-*
  Usage : pipeline de release Helm
```

### 4.3 Comptes de service CI

```bash
# Créer via l'API REST Nexus

# Créer l'utilisateur
curl -X POST \
  "https://nexus.internal/service/rest/v1/security/users" \
  -H "Content-Type: application/json" \
  -u "admin:${NEXUS_ADMIN_PASSWORD}" \
  -d '{
    "userId": "ci-reader",
    "firstName": "CI",
    "lastName": "Reader",
    "emailAddress": "ci-reader@company.com",
    "password": "'"${CI_READER_PASSWORD}"'",
    "status": "active",
    "roles": ["nx-ci-reader"]
  }'

# Créer l'utilisateur writer
curl -X POST \
  "https://nexus.internal/service/rest/v1/security/users" \
  -H "Content-Type: application/json" \
  -u "admin:${NEXUS_ADMIN_PASSWORD}" \
  -d '{
    "userId": "ci-writer",
    "firstName": "CI",
    "lastName": "Writer",
    "emailAddress": "ci-writer@company.com",
    "password": "'"${CI_WRITER_PASSWORD}"'",
    "status": "active",
    "roles": ["nx-ci-writer"]
  }'
```

### 4.4 Tokens d'authentification (User Tokens)

```
Les User Tokens sont des tokens révocables qui évitent d'utiliser
le mot de passe réel dans les CI/CD pipelines.

Activer : Security > User Tokens > Enable user tokens ✓

Génération pour un compte de service :
  1. Se connecter avec le compte ci-reader/ci-writer
  2. Profile utilisateur > User Token > Access User Token
  3. Copier le token code et le token pass
  4. Stocker dans les CI Variables GitLab (masked + protected)

Usage dans les pipelines :
  NEXUS_USER = <token_code>
  NEXUS_PASSWORD = <token_pass>
```

---

## 5. Configuration via REST API / IaC

### 5.1 API REST Nexus - Opérations essentielles

```bash
BASE_URL="https://nexus.internal"
AUTH="-u admin:${ADMIN_PASSWORD}"

# ── Lister les repositories ─────────────────────────────────────
curl $AUTH "$BASE_URL/service/rest/v1/repositories" | jq '.[].name'

# ── Créer un repo Maven hosted ──────────────────────────────────
curl -X POST $AUTH \
  "$BASE_URL/service/rest/v1/repositories/maven/hosted" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "maven-hosted-releases",
    "online": true,
    "storage": {
      "blobStoreName": "blob-releases",
      "strictContentTypeValidation": true,
      "writePolicy": "DENY"
    },
    "cleanup": {
      "policyNames": ["cleanup-maven-releases"]
    },
    "maven": {
      "versionPolicy": "RELEASE",
      "layoutPolicy": "STRICT",
      "contentDisposition": "ATTACHMENT"
    }
  }'

# ── Créer un repo PyPI proxy ────────────────────────────────────
curl -X POST $AUTH \
  "$BASE_URL/service/rest/v1/repositories/pypi/proxy" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "pypi-proxy-pypi",
    "online": true,
    "storage": {
      "blobStoreName": "blob-default",
      "strictContentTypeValidation": true
    },
    "proxy": {
      "remoteUrl": "https://pypi.org",
      "contentMaxAge": 1440,
      "metadataMaxAge": 1440
    },
    "negativeCache": {
      "enabled": true,
      "timeToLive": 1440
    },
    "httpClient": {
      "blocked": false,
      "autoBlock": true
    }
  }'

# ── Rechercher un composant ─────────────────────────────────────
curl $AUTH \
  "$BASE_URL/service/rest/v1/search?repository=pypi-hosted&name=mypackage" \
  | jq '.items[] | {name, version, assets}'

# ── Supprimer un composant spécifique ───────────────────────────
COMPONENT_ID="<id_from_search>"
curl -X DELETE $AUTH \
  "$BASE_URL/service/rest/v1/components/$COMPONENT_ID"
```

### 5.2 Provisioning Ansible

```yaml
# roles/nexus/tasks/create_repos.yml
---
- name: Create Maven hosted release repository
  uri:
    url: "{{ nexus_url }}/service/rest/v1/repositories/maven/hosted"
    method: POST
    user: "{{ nexus_admin_user }}"
    password: "{{ nexus_admin_password }}"
    force_basic_auth: yes
    body_format: json
    body:
      name: "maven-hosted-releases"
      online: true
      storage:
        blobStoreName: "blob-releases"
        strictContentTypeValidation: true
        writePolicy: "DENY"
      maven:
        versionPolicy: "RELEASE"
        layoutPolicy: "STRICT"
    status_code: [200, 201]
  register: result
  failed_when: result.status not in [200, 201]

- name: Configure cleanup policy
  uri:
    url: "{{ nexus_url }}/service/rest/v1/cleanup-policies"
    method: POST
    user: "{{ nexus_admin_user }}"
    password: "{{ nexus_admin_password }}"
    force_basic_auth: yes
    body_format: json
    body:
      name: "cleanup-releases-old"
      format: "ALL_FORMATS"
      mode: "deletion"
      criteria:
        lastBlobUpdated: "180"
        lastDownloaded: "90"
    status_code: [200, 201]
```

---

## 6. Haute disponibilité et clustering

### 6.1 Nexus Pro HA - Architecture

```
                    [Load Balancer]
                    /              \
          [Nexus Node 1]    [Nexus Node 2]
                    \              /
              [Base de données H2 / PostgreSQL*]
                         |
              [Blob Store partagé (NFS / S3)]

* PostgreSQL requis pour le clustering Nexus Pro
```

### 6.2 Configuration de base pour la résilience (OSS)

```properties
# nexus.properties - configuration JVM et réseau

# ── Mémoire ──────────────────────────────────────────────────────
# -Xms et -Xmx dans bin/nexus.vmoptions
# Recommandation :
#   4 GB RAM disponible  → -Xms1g -Xmx1g
#   8 GB RAM disponible  → -Xms2g -Xmx2g  (pour usage modéré)
#   16 GB RAM disponible → -Xms4g -Xmx4g  (pour usage intensif Docker)

# ── Data directory ───────────────────────────────────────────────
nexus-work=/nexus-data

# ── Sessions ─────────────────────────────────────────────────────
nexus.sessionTimeout=1440   # Minutes
```

```yaml
# Kubernetes Deployment pour Nexus OSS
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nexus
  namespace: nexus
spec:
  replicas: 1                    # OSS = 1 seul replica
  selector:
    matchLabels:
      app: nexus
  template:
    spec:
      securityContext:
        runAsUser: 200            # Utilisateur nexus
        fsGroup: 200
      containers:
        - name: nexus
          image: sonatype/nexus3:3.66.0
          resources:
            requests:
              memory: "4Gi"
              cpu: "1000m"
            limits:
              memory: "8Gi"
              cpu: "4000m"
          env:
            - name: INSTALL4J_ADD_VM_PARAMS
              value: "-Xms2g -Xmx2g -XX:MaxDirectMemorySize=2g"
          ports:
            - name: http
              containerPort: 8081
            - name: docker-prod
              containerPort: 8082
            - name: docker-dev
              containerPort: 8083
          livenessProbe:
            httpGet:
              path: /service/rest/v1/status
              port: 8081
            initialDelaySeconds: 180
            periodSeconds: 30
            failureThreshold: 6
          readinessProbe:
            httpGet:
              path: /service/rest/v1/status
              port: 8081
            initialDelaySeconds: 120
            periodSeconds: 10
          volumeMounts:
            - name: nexus-data
              mountPath: /nexus-data
  volumeClaimTemplates:
    - metadata:
        name: nexus-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 500Gi
```

---

## 📋 Récapitulatif Module 04

| Concept | Point clé |
|---------|-----------|
| Types de repos | Hosted (interne), Proxy (cache upstream), Group (agrégateur) |
| Strategy group | Ordre de recherche = priorité des repos dans le group |
| Docker | Ports dédiés obligatoires, Nginx reverse proxy pour TLS |
| Cleanup policies | INDISPENSABLE, sans ça Nexus grossit sans limite |
| Blob Stores | Séparer par type (docker, releases, temp) pour des politiques différentes |
| Sécurité | Comptes de service dédiés CI, User Tokens révocables (pas de mot de passe) |
| IaC | REST API pour tout configurer, Ansible pour l'automatisation |

---

[← Module 03](./03_GITLAB_CI_AVANCE.md) | [→ Module 05 - Nexus Intégration CI/CD](./05_NEXUS_INTEGRATION.md)
