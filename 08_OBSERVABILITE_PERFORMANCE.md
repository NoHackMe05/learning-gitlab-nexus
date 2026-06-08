# Module 08 — Observabilité & Performance des Pipelines

> **Durée** : ~3h | [← Module 07](./07_PIPELINE_PATTERNS.md) | [→ Module 09](./09_LABS_PRATIQUES.md)

---

## Sommaire

1. [Métriques GitLab CI — Tableau de bord](#1-métriques-gitlab-ci--tableau-de-bord)
2. [Prometheus + Grafana pour GitLab et Nexus](#2-prometheus--grafana-pour-gitlab-et-nexus)
3. [Optimisation des pipelines](#3-optimisation-des-pipelines)
4. [Monitoring Nexus en production](#4-monitoring-nexus-en-production)
5. [Alerting et SLOs CI/CD](#5-alerting-et-slos-cicd)

---

## 1. Métriques GitLab CI — Tableau de bord

### 1.1 Métriques natives GitLab (API)

```bash
# ── Durée moyenne des pipelines (30 derniers jours) ────────────
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.example.com/api/v4/projects/$PROJECT_ID/pipelines?per_page=100&status=success" \
  | jq '[.[] | .duration] | add / length | . / 60 | "Average pipeline duration: \(.) minutes"'

# ── Taux de succès des pipelines ───────────────────────────────
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.example.com/api/v4/projects/$PROJECT_ID/pipelines?per_page=100" \
  | jq '
    group_by(.status) |
    map({status: .[0].status, count: length}) |
    .[] |
    "\(.status): \(.count)"
  '

# ── Jobs les plus lents ─────────────────────────────────────────
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.example.com/api/v4/projects/$PROJECT_ID/jobs?per_page=100&status=success" \
  | jq 'sort_by(-.duration) | .[:10] | .[] | "\(.name): \(.duration)s"'
```

### 1.2 Exporter les métriques vers Prometheus (script Python)

```python
#!/usr/bin/env python3
"""Exporteur Prometheus pour les métriques GitLab CI."""

import time
import os
import requests
from prometheus_client import start_http_server, Gauge, Counter, Histogram

GITLAB_URL = os.environ["GITLAB_URL"]
GITLAB_TOKEN = os.environ["GITLAB_TOKEN"]
PROJECT_IDS = os.environ.get("PROJECT_IDS", "").split(",")

# Métriques Prometheus
pipeline_duration = Histogram(
    "gitlab_pipeline_duration_seconds",
    "Pipeline duration in seconds",
    ["project", "ref", "status"],
    buckets=[60, 120, 300, 600, 900, 1800, 3600]
)
pipeline_status = Counter(
    "gitlab_pipeline_total",
    "Total pipelines by status",
    ["project", "ref", "status"]
)
job_duration = Histogram(
    "gitlab_job_duration_seconds",
    "Job duration in seconds",
    ["project", "job_name", "stage"],
    buckets=[10, 30, 60, 120, 300, 600]
)
runner_jobs_running = Gauge(
    "gitlab_runner_jobs_running",
    "Number of running jobs per runner",
    ["runner_name"]
)


def fetch_pipelines(project_id: str, per_page: int = 50):
    resp = requests.get(
        f"{GITLAB_URL}/api/v4/projects/{project_id}/pipelines",
        headers={"PRIVATE-TOKEN": GITLAB_TOKEN},
        params={"per_page": per_page, "order_by": "updated_at"}
    )
    return resp.json() if resp.status_code == 200 else []


def collect_metrics():
    for project_id in PROJECT_IDS:
        pipelines = fetch_pipelines(project_id.strip())
        project_name = project_id.strip()

        for pipeline in pipelines:
            if pipeline.get("duration"):
                pipeline_duration.labels(
                    project=project_name,
                    ref=pipeline.get("ref", "unknown"),
                    status=pipeline.get("status", "unknown")
                ).observe(pipeline["duration"])

            pipeline_status.labels(
                project=project_name,
                ref=pipeline.get("ref", "unknown"),
                status=pipeline.get("status", "unknown")
            ).inc()


if __name__ == "__main__":
    start_http_server(9900)
    print("GitLab CI Prometheus exporter running on :9900")
    while True:
        collect_metrics()
        time.sleep(60)
```

---

## 2. Prometheus + Grafana pour GitLab et Nexus

### 2.1 Configuration Prometheus — scrape configs

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # GitLab instance metrics
  - job_name: 'gitlab'
    metrics_path: '/-/metrics'
    bearer_token: "$GITLAB_METRICS_TOKEN"
    static_configs:
      - targets: ['gitlab.internal:443']
    scheme: https
    tls_config:
      insecure_skip_verify: false

  # GitLab Runners
  - job_name: 'gitlab-runners'
    static_configs:
      - targets:
          - 'runner-01.internal:9252'
          - 'runner-02.internal:9252'
          - 'runner-k8s.internal:9252'

  # Nexus Repository Manager
  - job_name: 'nexus'
    metrics_path: '/service/metrics/prometheus'
    basic_auth:
      username: 'prometheus'
      password: '$NEXUS_METRICS_PASSWORD'
    static_configs:
      - targets: ['nexus.internal:8081']

  # GitLab CI custom exporter
  - job_name: 'gitlab-ci-exporter'
    static_configs:
      - targets: ['ci-exporter:9900']
```

### 2.2 Dashboard Grafana — Panels essentiels CI/CD

```json
// Panel 1 : Durée moyenne des pipelines par branche (dernières 24h)
{
  "title": "Pipeline Duration (avg) by Branch",
  "type": "timeseries",
  "targets": [
    {
      "expr": "histogram_quantile(0.95, sum(rate(gitlab_pipeline_duration_seconds_bucket[1h])) by (le, ref))",
      "legendFormat": "p95 - {{ref}}"
    },
    {
      "expr": "histogram_quantile(0.50, sum(rate(gitlab_pipeline_duration_seconds_bucket[1h])) by (le, ref))",
      "legendFormat": "p50 - {{ref}}"
    }
  ]
}
```

```promql
# ── PromQL essentiels pour GitLab CI ──────────────────────────

# Taux de succès des pipelines (dernière heure)
sum(rate(gitlab_pipeline_total{status="success"}[1h]))
/
sum(rate(gitlab_pipeline_total[1h]))

# Jobs en cours par runner
gitlab_runner_jobs{state="running"}

# Durée p95 des jobs par nom (top 10 les plus lents)
topk(10,
  histogram_quantile(0.95,
    sum(rate(gitlab_job_duration_seconds_bucket[1h])) by (le, job_name)
  )
)

# Runners surchargés (utilisation > 80%)
(gitlab_runner_jobs{state="running"} / gitlab_runner_concurrent) > 0.8

# Temps d'attente moyen en queue (stuck + pending)
avg_over_time(gitlab_runner_jobs{state="waiting_for_capacity"}[30m])
```

### 2.3 Alertes Grafana/Alertmanager

```yaml
# alerting-rules.yml
groups:
  - name: gitlab-ci
    rules:
      - alert: GitLabPipelineFailureRateHigh
        expr: |
          sum(rate(gitlab_pipeline_total{status="failed"}[15m]))
          /
          sum(rate(gitlab_pipeline_total[15m])) > 0.3
        for: 10m
        labels:
          severity: warning
          team: devops
        annotations:
          summary: "GitLab pipeline failure rate > 30%"
          description: "Failure rate is {{ $value | humanizePercentage }} over the last 15m"

      - alert: GitLabRunnerSaturated
        expr: |
          (gitlab_runner_jobs{state="running"} / gitlab_runner_concurrent) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Runner {{ $labels.runner }} is saturated (>90%)"

      - alert: GitLabJobStuck
        expr: gitlab_runner_jobs{state="waiting_for_capacity"} > 10
        for: 15m
        labels:
          severity: critical
        annotations:
          summary: "More than 10 jobs waiting for runner capacity"

  - name: nexus
    rules:
      - alert: NexusDiskSpaceLow
        expr: |
          nexus_blob_store_available_space_bytes
          / nexus_blob_store_total_space_bytes < 0.15
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Nexus blob store {{ $labels.name }} has less than 15% free space"

      - alert: NexusDown
        expr: up{job="nexus"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Nexus Repository Manager is unreachable"
```

---

## 3. Optimisation des pipelines

### 3.1 Diagnostic des pipelines lents

```bash
# Script d'analyse des pipelines lents via API GitLab
#!/bin/bash

PROJECT_ID=$1
TOKEN=$2
GITLAB_URL=${3:-"https://gitlab.example.com"}
LOOKBACK_DAYS=${4:-7}

echo "=== Pipeline Performance Analysis (last $LOOKBACK_DAYS days) ==="

# Récupérer les pipelines
SINCE=$(date -d "$LOOKBACK_DAYS days ago" --iso-8601=seconds)
PIPELINES=$(curl -s \
  --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/$PROJECT_ID/pipelines?per_page=100&updated_after=$SINCE&status=success" \
)

# Durée moyenne
echo "$PIPELINES" | jq -r '
  [.[] | .duration] |
  {
    count: length,
    avg: (add / length / 60 | round),
    max: (max / 60 | round),
    min: (min / 60 | round)
  } |
  "Pipelines: \(.count) | Avg: \(.avg)m | Max: \(.max)m | Min: \(.min)m"
'

# Top 5 pipelines les plus longs
echo -e "\nTop 5 slowest pipelines:"
echo "$PIPELINES" | jq -r '
  sort_by(-.duration) | .[:5] |
  .[] | "  \(.id) (\(.ref)): \(.duration / 60 | round)m - \(.web_url)"
'
```

### 3.2 Techniques d'optimisation

```yaml
# ── Technique 1 : Fail fast avec lint en .pre ──────────────────
lint:
  stage: .pre
  script: make lint
  allow_failure: false
  interruptible: true

# ── Technique 2 : Cache agressif pour les deps ─────────────────
# Séparer install (push) des tests (pull)
install:deps:
  stage: .pre
  script:
    - pip install -r requirements.txt -r requirements-dev.txt
  cache:
    key: { files: [requirements.txt, requirements-dev.txt] }
    paths: [.venv/]
    policy: push   # Créer le cache
  interruptible: true

test:unit:
  needs: [install:deps]
  cache:
    key: { files: [requirements.txt, requirements-dev.txt] }
    paths: [.venv/]
    policy: pull   # Lire seulement (permet la parallélisation)
  parallel: 4      # 4 workers

# ── Technique 3 : DAG pour éliminer les attentes inutiles ──────
# → Voir Module 02

# ── Technique 4 : Images Docker légères et pré-cachées ─────────
# Créer des images "CI" avec les dépendances pré-installées
# Stockées dans Nexus, rarement reconstruites

# image: nexus.internal:8082/ci-images/python-ci:3.12  
# (contient pytest, coverage, ruff, mypy pré-installés)

# ── Technique 5 : Interruptible + auto-cancel ──────────────────
workflow:
  auto_cancel:
    on_new_commit: interruptible

# Marquer tous les jobs non-critiques comme interruptible
.interruptible: &interruptible
  interruptible: true
  retry:
    max: 1
    when: [runner_system_failure]

test:unit:
  <<: *interruptible
  script: pytest

# ── Technique 6 : Artifacts minimalistes ──────────────────────
# ❌ Mauvais
build:bad:
  artifacts:
    paths: ["**/*"]          # Tout uploader = lent
    expire_in: never

# ✅ Bon
build:good:
  artifacts:
    paths: ["dist/*.whl"]    # Seulement le livrable
    exclude: ["**/*.pyc", "**/__pycache__/**"]
    expire_in: 1 day         # TTL court

# ── Technique 7 : Règles changes: ciblées ─────────────────────
# Éviter les pipelines inutiles en monorepo
test:backend:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        paths: ["backend/**/*", "shared/lib/**/*"]
        compare_to: "refs/heads/main"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - when: never
```

### 3.3 Métriques de performance cibles (SLOs CI/CD)

```
Objectifs recommandés :

Pipeline main branch :
  p50 duration < 10 minutes
  p95 duration < 20 minutes
  Success rate > 95%

Pipeline MR :
  p50 duration < 8 minutes  (feedback rapide pour les devs)
  p95 duration < 15 minutes

Jobs individuels :
  lint/format  : < 2 minutes
  unit tests   : < 5 minutes
  build image  : < 8 minutes
  SAST         : < 10 minutes
  deploy       : < 5 minutes

Disponibilité :
  Runners      : > 99.5% uptime
  Nexus        : > 99.9% uptime (critique pour les builds)
```

---

## 4. Monitoring Nexus en production

### 4.1 Métriques Nexus exposées via Prometheus

```bash
# Activer les métriques Prometheus dans Nexus
# System > Capabilities > Create capability > Metrics
# Type : Metrics (capability)
# Exposed : ✅

# Endpoint : https://nexus.internal/service/metrics/prometheus
# Authentification requise (compte avec privilege nx-metrics-all)

# Métriques disponibles :
#
# nexus_analytics_usage_request_seconds  → Latences des requêtes
# nexus_blob_store_available_space_bytes → Espace disque disponible
# nexus_blob_store_total_space_bytes     → Capacité totale
# nexus_blob_store_blob_count            → Nombre de blobs
# jvm_memory_used_bytes                  → Mémoire JVM
# jvm_threads_live_threads               → Threads actifs
# http_server_requests_seconds           → Latences HTTP
```

```promql
# ── PromQL pour Nexus ─────────────────────────────────────────

# Espace utilisé par blob store (%)
1 - (
  nexus_blob_store_available_space_bytes
  / nexus_blob_store_total_space_bytes
)

# Taux de requêtes HTTP Nexus (par méthode)
sum(rate(http_server_requests_seconds_count{job="nexus"}[5m])) by (method, status)

# Latence p95 des requêtes Nexus
histogram_quantile(0.95,
  sum(rate(http_server_requests_seconds_bucket{job="nexus"}[5m])) by (le, uri)
)

# Mémoire heap JVM utilisée
jvm_memory_used_bytes{job="nexus", area="heap"}
/ jvm_memory_max_bytes{job="nexus", area="heap"}

# Threads bloqués (signe de problème de perf)
jvm_threads_states_threads{job="nexus", state="blocked"}
```

### 4.2 Tâches de maintenance automatisées

```yaml
# Pipeline planifié : maintenance Nexus hebdomadaire
# Schedulé : tous les dimanches à 2h du matin

nexus:maintenance:
  stage: .pre
  image: curlimages/curl:latest
  script:
    - |
      echo "=== Nexus Weekly Maintenance ==="
      
      # 1. Déclencher le cleanup de tous les repos
      echo "Running cleanup tasks..."
      curl -X POST \
        -u "admin:$NEXUS_ADMIN_PASSWORD" \
        "https://nexus.internal/service/rest/v1/tasks/$(curl -s -u admin:$NEXUS_ADMIN_PASSWORD https://nexus.internal/service/rest/v1/tasks | jq -r '.items[] | select(.type=="repository.cleanup") | .id')/run"
      
      sleep 60
      
      # 2. Compacter les blob stores
      echo "Compacting blob stores..."
      for TASK_ID in $(curl -s \
        -u "admin:$NEXUS_ADMIN_PASSWORD" \
        "https://nexus.internal/service/rest/v1/tasks" \
        | jq -r '.items[] | select(.type=="blobstore.compact") | .id'); do
        curl -X POST \
          -u "admin:$NEXUS_ADMIN_PASSWORD" \
          "https://nexus.internal/service/rest/v1/tasks/${TASK_ID}/run"
        echo "Started compact task: $TASK_ID"
      done
      
      # 3. Rebuild des index des repos
      echo "Rebuilding search index..."
      curl -X POST \
        -u "admin:$NEXUS_ADMIN_PASSWORD" \
        "https://nexus.internal/service/rest/v1/tasks/$(curl -s -u admin:$NEXUS_ADMIN_PASSWORD https://nexus.internal/service/rest/v1/tasks | jq -r '.items[] | select(.type=="rebuild.asset.uploadMetadata") | .id' | head -1)/run"
      
      echo "=== Maintenance tasks triggered ==="
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "nexus-maintenance"
  allow_failure: true
```

---

## 5. Alerting et SLOs CI/CD

### 5.1 Définition des SLIs/SLOs pour CI/CD

```
Service Level Indicators (SLIs) mesurables :

1. Pipeline Success Rate
   SLI : (successful_pipelines / total_pipelines) * 100
   SLO : > 95% sur main branch, sur fenêtre glissante 7j

2. Pipeline Duration P95
   SLI : percentile(95, pipeline_duration_seconds)
   SLO : < 1200s (20 minutes) sur main branch

3. Nexus Availability
   SLI : uptime(nexus_health_endpoint)
   SLO : > 99.5% par mois (< 3.6h downtime/mois)

4. Nexus Response Time P99
   SLI : percentile(99, nexus_request_duration_seconds)
   SLO : < 5s pour les opérations de download

5. Runner Queue Time
   SLI : avg(job_waiting_for_capacity_duration)
   SLO : < 60s en moyenne (hors périodes de burst)
```

### 5.2 SLO Monitoring avec Sloth (Prometheus)

```yaml
# sloth-slos.yaml — Génère les règles Prometheus d'error budget
version: "prometheus/v1"
service: "cicd-platform"
slos:
  - name: "pipeline-success-rate"
    objective: 95
    description: "GitLab main branch pipeline success rate"
    sli:
      events:
        error_query: |
          sum(rate(gitlab_pipeline_total{ref="main",status=~"failed|canceled"}[{{.window}}]))
        total_query: |
          sum(rate(gitlab_pipeline_total{ref="main"}[{{.window}}]))
    alerting:
      name: GitLabPipelineSuccessRateSLO
      labels:
        team: devops
      annotations:
        runbook: "https://wiki.example.com/runbooks/cicd-failure-rate"
      page_alert:
        labels:
          severity: critical
      ticket_alert:
        labels:
          severity: warning

  - name: "nexus-availability"
    objective: 99.5
    description: "Nexus Repository Manager availability"
    sli:
      events:
        error_query: |
          sum(rate(http_server_requests_seconds_count{job="nexus",status=~"5.."}[{{.window}}]))
        total_query: |
          sum(rate(http_server_requests_seconds_count{job="nexus"}[{{.window}}]))
    alerting:
      name: NexusAvailabilitySLO
      page_alert:
        labels:
          severity: critical
      ticket_alert:
        labels:
          severity: warning
```

### 5.3 Dashboard Grafana — Template JSON clé

```json
// Panels essentiels pour un dashboard CI/CD ops
[
  {
    "title": "Pipeline Success Rate (7d)",
    "type": "stat",
    "fieldConfig": {"defaults": {"unit": "percentunit", "thresholds": {
      "steps": [
        {"color": "red", "value": 0},
        {"color": "yellow", "value": 0.9},
        {"color": "green", "value": 0.95}
    }}},
    "targets": [{
      "expr": "sum(rate(gitlab_pipeline_total{status='success'}[7d])) / sum(rate(gitlab_pipeline_total[7d]))"
    }]
  },
  {
    "title": "Active Runners",
    "type": "table",
    "targets": [{
      "expr": "gitlab_runner_jobs",
      "legendFormat": "{{runner}} - {{state}}"
    }]
  },
  {
    "title": "Nexus Disk Usage by Blob Store",
    "type": "bargauge",
    "targets": [{
      "expr": "1 - (nexus_blob_store_available_space_bytes / nexus_blob_store_total_space_bytes)",
      "legendFormat": "{{name}}"
    }]
  }
]
```

---

## 📋 Récapitulatif Module 08

| Domaine | Outil | Métrique clé |
|---------|-------|-------------|
| Pipelines CI | GitLab API + Prometheus | Success rate, p95 duration |
| Runners | gitlab-runner metrics `:9252` | Jobs running/waiting, saturation |
| Nexus | `/service/metrics/prometheus` | Disk usage, latences, uptime |
| SLOs | Sloth + Prometheus | Error budget consommé |
| Alerting | Alertmanager | Seuils sur failure rate, disk, queue |
| Optimisation | Cache policy pull-only, DAG, interruptible | Réduction p95 duration |

🚀 **Objectif** : pipeline main < 15 min en p95, Nexus > 99.5% uptime, taux de succès > 95%.

---

[← Module 07](./07_PIPELINE_PATTERNS.md) | [→ Module 09 — Labs Pratiques](./09_LABS_PRATIQUES.md)
