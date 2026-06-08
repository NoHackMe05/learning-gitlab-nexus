# Module 06 - Sécurité & DevSecOps

> **Durée** : ~5h | [← Module 05](./05_NEXUS_INTEGRATION.md) | [→ Module 07](./07_PIPELINE_PATTERNS.md)

---

## Sommaire

1. [SAST - Static Application Security Testing](#1-sast--static-application-security-testing)
2. [SCA - Software Composition Analysis](#2-sca--software-composition-analysis)
3. [Container Scanning](#3-container-scanning)
4. [DAST - Dynamic Application Security Testing](#4-dast--dynamic-application-security-testing)
5. [Secret Detection](#5-secret-detection)
6. [IaC Scanning](#6-iac-scanning)
7. [Compliance Framework et Security Dashboard](#7-compliance-framework-et-security-dashboard)
8. [Intégration Nexus IQ / Lifecycle](#8-intégration-nexus-iq--lifecycle)

---

## 1. SAST - Static Application Security Testing

### 1.1 GitLab SAST intégré (templates)

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml

# Le template ajoute automatiquement les jobs selon les langages détectés :
# Python   → semgrep, bandit
# Java     → semgrep
# JS/TS    → semgrep, eslint-security
# Go       → semgrep, gosec
# Ruby     → brakeman
# PHP      → phpcs-security-audit
# etc.

# Personnalisation du template
variables:
  SAST_EXCLUDED_PATHS: "tests/, docs/, node_modules/"
  SAST_EXCLUDED_ANALYZERS: ""        # ex: "bandit, gosec"
  SAST_SEVERITY_THRESHOLD: "Medium"  # Ignorer les vulnérabilités < Medium
  SEARCH_MAX_DEPTH: 10
  
  # Semgrep spécifique
  SEMGREP_TIMEOUT: "300"
  SEMGREP_RULES: >-
    p/python
    p/security-audit
    p/owasp-top-ten
```

### 1.2 Semgrep - Configuration avancée

```yaml
# SAST custom avec Semgrep (plus de contrôle que le template)
sast:semgrep:
  stage: security
  image: returntocorp/semgrep:latest
  variables:
    SEMGREP_RULES: >-
      p/python
      p/django
      p/flask
      p/security-audit
      p/owasp-top-ten
      p/cwe-top-25
  script:
    - |
      semgrep scan \
        --config="$SEMGREP_RULES" \
        --config="semgrep-rules/"  \
        --json \
        --output gl-sast-report.json \
        --error \
        --severity=ERROR \
        --exclude="tests/" \
        --exclude="*.test.py" \
        src/
  artifacts:
    reports:
      sast: gl-sast-report.json
    paths: [gl-sast-report.json]
    when: always
    expire_in: 30 days
  allow_failure: false  # Bloquer si des HIGH/CRITICAL trouvés
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# Règles Semgrep custom dans semgrep-rules/
# rules/no-hardcoded-secrets.yml
```

```yaml
# semgrep-rules/company-rules.yml
rules:
  - id: no-hardcoded-api-key
    patterns:
      - pattern: |
          $API_KEY = "..."
      - metavariable-regex:
          metavariable: $API_KEY
          regex: (api_key|apikey|api-key|secret|password|passwd|pwd)
    message: "Hardcoded API key or secret detected: $API_KEY"
    severity: ERROR
    languages: [python, javascript, java]

  - id: sql-injection-string-format
    patterns:
      - pattern: |
          cursor.execute("..." % ...)
      - pattern: |
          cursor.execute(f"...{...}...")
    message: "Potential SQL injection via string formatting"
    severity: ERROR
    languages: [python]
```

---

## 2. SCA - Software Composition Analysis

### 2.1 GitLab Dependency Scanning

```yaml
include:
  - template: Security/Dependency-Scanning.gitlab-ci.yml

variables:
  DS_EXCLUDED_PATHS: "tests/"
  # Forcer l'utilisation de Nexus pour les checks
  PIP_INDEX_URL: "https://nexus.internal/repository/pypi-group/simple/"
  
  # Seuil de sévérité
  DS_SEVERITY_THRESHOLD: "High"
```

### 2.2 Trivy - SCA multiformat

```yaml
sca:trivy:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  variables:
    # Cache la DB de vulnérabilités dans Nexus (évite les pulls constants)
    TRIVY_CACHE_DIR: ".trivy-cache"
    TRIVY_DB_REPOSITORY: "nexus.internal:8082/aquasec/trivy-db:2"  # Mirror dans Nexus
  script:
    - |
      # Scan du filesystem (dépendances)
      trivy fs \
        --format template \
        --template "@/contrib/gitlab.tpl" \
        --output gl-dependency-scanning-report.json \
        --severity HIGH,CRITICAL \
        --exit-code 1 \
        --ignore-unfixed \
        .
  cache:
    paths: [.trivy-cache/]
  artifacts:
    reports:
      dependency_scanning: gl-dependency-scanning-report.json
    when: always
    expire_in: 30 days
  allow_failure: false
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### 2.3 Politique de dépendances bloquantes

```yaml
# Job qui vérifie les licences et bloque sur les licences interdites
sca:license-check:
  stage: security
  image: python:3.12-slim
  script:
    - pip install pip-licenses
    - pip install -r requirements.txt
    - |
      # Exporter les licences
      pip-licenses --format=json --output-file=licenses.json
      
      # Vérifier les licences interdites
      FORBIDDEN_LICENSES="GPL-3.0,AGPL-3.0,SSPL-1.0"
      python3 << 'EOF'
      import json, sys
      with open('licenses.json') as f:
          packages = json.load(f)
      forbidden = ['GPL-3.0', 'AGPL-3.0', 'SSPL-1.0']
      violations = [p for p in packages if any(l in p['License'] for l in forbidden)]
      if violations:
          print("❌ Forbidden licenses found:")
          for v in violations:
              print(f"  - {v['Name']} ({v['Version']}): {v['License']}")
          sys.exit(1)
      print("✅ All licenses approved")
      EOF
  artifacts:
    paths: [licenses.json]
    expire_in: 30 days
  allow_failure: true  # Warning mais pas bloquant
```

---

## 3. Container Scanning

### 3.1 GitLab Container Scanning (template)

```yaml
include:
  - template: Security/Container-Scanning.gitlab-ci.yml

container_scanning:
  variables:
    CS_IMAGE: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
    CS_SEVERITY_THRESHOLD: "High"
    CS_TRIVY_OPTIONS: "--ignore-unfixed"
    
    # Utiliser Nexus comme mirror pour la DB Trivy
    CS_REGISTRY_USER: "$NEXUS_USER"
    CS_REGISTRY_PASSWORD: "$NEXUS_PASSWORD"
```

### 3.2 Trivy Container Scan avancé

```yaml
scan:container:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  needs:
    - job: build:image
      artifacts: true    # Récupère $FULL_IMAGE depuis dotenv
  variables:
    TRIVY_CACHE_DIR: ".trivy-cache"
    TRIVY_USERNAME: "$NEXUS_USER"
    TRIVY_PASSWORD: "$NEXUS_PASSWORD"
  script:
    - |
      echo "🔍 Scanning image: $FULL_IMAGE"
      
      # Scan complet avec output JSON pour GitLab
      trivy image \
        --format template \
        --template "@/contrib/gitlab.tpl" \
        --output gl-container-scanning-report.json \
        --severity CRITICAL,HIGH \
        --exit-code 1 \
        --ignore-unfixed \
        --timeout 10m \
        "$FULL_IMAGE"
      
      # Rapport human-readable pour les logs
      trivy image \
        --severity CRITICAL,HIGH \
        --ignore-unfixed \
        "$FULL_IMAGE"
  cache:
    paths: [.trivy-cache/]
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
    paths: [gl-container-scanning-report.json]
    when: always
    expire_in: 30 days
  allow_failure: false
```

### 3.3 Hardening de l'image Docker

```dockerfile
# Checklist sécurité Dockerfile

# ✅ 1. Utiliser une image de base minimale et officielle
FROM python:3.12-slim
# Encore mieux pour la production critique :
# FROM gcr.io/distroless/python3-debian12

# ✅ 2. Ne pas exécuter en root
RUN groupadd -r -g 1000 appuser && useradd -r -u 1000 -g appuser appuser

# ✅ 3. Ne pas installer de packages inutiles
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# ✅ 4. Copier avec des permissions appropriées
COPY --chown=appuser:appuser src/ /app/src/

# ✅ 5. Utiliser COPY plutôt que ADD (sauf archives)
# ❌ ADD https://... /app/file  → risque SSRF
# ✅ COPY ./file /app/file

# ✅ 6. Pas de secrets dans le Dockerfile
# ❌ ENV API_KEY=secret123
# ✅ Utiliser des secrets Docker au build ou des variables d'env runtime

# ✅ 7. Définir un utilisateur non-root
USER appuser

# ✅ 8. Read-only rootfs si possible
# (géré au runtime : docker run --read-only)

# ✅ 9. HEALTHCHECK
HEALTHCHECK --interval=30s --timeout=10s CMD python -c "..."

# ✅ 10. Labels de traçabilité
LABEL org.opencontainers.image.source="$CI_PROJECT_URL"
```

---

## 4. DAST - Dynamic Application Security Testing

### 4.1 OWASP ZAP via GitLab DAST template

```yaml
include:
  - template: Security/DAST.gitlab-ci.yml

dast:
  stage: security
  environment:
    name: dast-review
    url: $DAST_WEBSITE
  variables:
    DAST_WEBSITE: "https://staging.example.com"
    DAST_FULL_SCAN_ENABLED: "false"   # true pour un scan complet (plus long)
    DAST_BROWSER_SCAN: "true"         # Scan basé sur le navigateur (plus moderne)
    DAST_AUTH_URL: "https://staging.example.com/login"
    DAST_USERNAME: "$DAST_USER"
    DAST_PASSWORD: "$DAST_PASSWORD"
    DAST_USERNAME_FIELD: "input[name=username]"
    DAST_PASSWORD_FIELD: "input[name=password]"
    DAST_SUBMIT_FIELD: "button[type=submit]"
    DAST_FIRST_SUBMIT_FIELD: ""
    DAST_EXCLUDE_URLS: "/logout,/api/v1/health"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

### 4.2 DAST avec ZAP custom (plus de contrôle)

```yaml
dast:zap:custom:
  stage: security
  image: owasp/zap2docker-stable:latest
  needs:
    - deploy:staging    # Le DAST nécessite un environnement déployé
  variables:
    TARGET_URL: "https://staging.example.com"
    ZAP_REPORT_DIR: "$CI_PROJECT_DIR/zap-reports"
  script:
    - mkdir -p "$ZAP_REPORT_DIR"
    - |
      # Scan de base (rapide)
      zap-baseline.py \
        -t "$TARGET_URL" \
        -g gen.conf \
        -r "$ZAP_REPORT_DIR/zap-report.html" \
        -J "$ZAP_REPORT_DIR/zap-report.json" \
        -x "$ZAP_REPORT_DIR/zap-report.xml" \
        -l WARN \
        -I  # Continue même si des alertes trouvées
    
    # Convertir en format GitLab
    - python3 zap-to-gitlab.py "$ZAP_REPORT_DIR/zap-report.json" > gl-dast-report.json
  artifacts:
    reports:
      dast: gl-dast-report.json
    paths: [zap-reports/]
    when: always
    expire_in: 30 days
  allow_failure: true   # DAST peut avoir des faux positifs
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

---

## 5. Secret Detection

### 5.1 GitLab Secret Detection (pre-push et pipeline)

```yaml
include:
  - template: Security/Secret-Detection.gitlab-ci.yml

secret_detection:
  variables:
    SECRET_DETECTION_HISTORIC_SCAN: "false"   # true pour scanner tout l'historique git
    SECRET_DETECTION_EXCLUDED_PATHS: "tests/"
```

### 5.2 Gitleaks - Configuration avancée

```yaml
detect:secrets:
  stage: .pre      # En premier, fail fast si secret détecté
  image:
    name: zricethezav/gitleaks:latest
    entrypoint: [""]
  script:
    - |
      gitleaks detect \
        --source . \
        --config .gitleaks.toml \
        --report-format json \
        --report-path gl-secret-detection-report.json \
        --redact \
        --verbose \
        --no-git=false
  artifacts:
    reports:
      secret_detection: gl-secret-detection-report.json
    when: always
    expire_in: 30 days
  allow_failure: false   # BLOQUER si secret trouvé
```

```toml
# .gitleaks.toml
title = "Company Secret Detection Rules"

[extend]
useDefault = true          # Inclure les règles par défaut Gitleaks

# Règles custom
[[rules]]
id = "company-api-key"
description = "Company internal API key"
regex = '''company_[a-zA-Z0-9]{32,}'''
tags = ["api-key", "company"]
severity = "CRITICAL"

[[rules]]
id = "nexus-credentials"
description = "Nexus credentials in files"
regex = '''nexus\.(user|password|token)\s*=\s*[^\s]{8,}'''
tags = ["nexus", "credentials"]

# Allowlist globale
[allowlist]
description = "Global allowlist"
regexes = [
  '''NEXUS_USER=\$\{''',           # Variables de template (pas des valeurs)
  '''password.*\$\{.*\}''',         # Variables shell
  '''example\.com|placeholder''',
]
paths = [
  '''^tests/fixtures/.*''',
  '''\.gitleaks\.toml''',
]
```

---

## 6. IaC Scanning

### 6.1 GitLab IaC Scanning (Terraform, K8s, Docker)

```yaml
include:
  - template: Security/SAST-IaC.gitlab-ci.yml

kics-iac-sast:
  variables:
    KICS_EXCLUDED_PATHS: "tests/, examples/"
    SAST_SEVERITY_THRESHOLD: "Medium"
```

### 6.2 Checkov - Scan Terraform et K8s

```yaml
scan:iac:checkov:
  stage: security
  image: bridgecrew/checkov:latest
  script:
    - |
      # Scan Terraform
      checkov -d terraform/ \
        --framework terraform \
        --output cli \
        --output json \
        --output-file-path checkov-tf-report.json \
        --compact \
        --skip-check CKV_AWS_144,CKV2_AWS_5  # Faux positifs connus
      
      # Scan Kubernetes manifests
      checkov -d k8s/ \
        --framework kubernetes \
        --output cli \
        --output json \
        --output-file-path checkov-k8s-report.json \
        --compact
      
      # Scan Dockerfile
      checkov -f Dockerfile \
        --framework dockerfile \
        --output json \
        --output-file-path checkov-docker-report.json
  artifacts:
    paths:
      - checkov-tf-report.json
      - checkov-k8s-report.json
      - checkov-docker-report.json
    when: always
    expire_in: 30 days
  allow_failure: true
```

---

## 7. Compliance Framework et Security Dashboard

### 7.1 Pipeline de sécurité complet consolidé

```yaml
# Consolidation de TOUS les scans sécurité
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  - template: Security/DAST.gitlab-ci.yml

stages:
  - .pre
  - validate
  - build
  - test
  - security      # Stage dédié sécurité
  - package
  - deploy:staging
  - dast          # DAST après déploiement
  - deploy:prod
  - .post

# Rapport de synthèse sécurité
security:summary:
  stage: .post
  image: python:3.12-slim
  script:
    - pip install jinja2
    - |
      python3 << 'EOF'
      import json, glob, sys
      
      reports = {}
      total_critical = 0
      total_high = 0
      
      for report_file in glob.glob("gl-*-report.json"):
          report_type = report_file.replace("gl-", "").replace("-report.json", "")
          try:
              with open(report_file) as f:
                  data = json.load(f)
              vulns = data.get("vulnerabilities", [])
              critical = len([v for v in vulns if v.get("severity", "").upper() == "CRITICAL"])
              high = len([v for v in vulns if v.get("severity", "").upper() == "HIGH"])
              reports[report_type] = {"total": len(vulns), "critical": critical, "high": high}
              total_critical += critical
              total_high += high
          except Exception as e:
              reports[report_type] = {"error": str(e)}
      
      print("\n" + "="*60)
      print("SECURITY SCAN SUMMARY")
      print("="*60)
      for scan_type, stats in reports.items():
          if "error" in stats:
              print(f"  {scan_type}: ERROR - {stats['error']}")
          else:
              print(f"  {scan_type}: {stats['total']} findings ({stats['critical']} CRITICAL, {stats['high']} HIGH)")
      print("-"*60)
      print(f"  TOTAL: {total_critical} CRITICAL, {total_high} HIGH")
      print("="*60)
      
      if total_critical > 0:
          print("\n❌ CRITICAL vulnerabilities found - pipeline should be blocked!")
          sys.exit(1)
      elif total_high > 5:
          print(f"\n⚠️  More than 5 HIGH vulnerabilities found ({total_high})")
          sys.exit(1)
      else:
          print("\n✅ Security scan passed")
      EOF
  artifacts:
    paths: ["gl-*-report.json"]
    when: always
  when: always
```

---

## 8. Intégration Nexus IQ / Lifecycle

### 8.1 Nexus IQ Server dans le pipeline

```yaml
# Nécessite Nexus IQ Server (Sonatype Lifecycle)
sca:nexus-iq:
  stage: security
  image: sonatype/nexus-iq-cli:latest
  variables:
    NEXUS_IQ_URL: "https://nexus-iq.internal"
    NEXUS_IQ_APP_ID: "myapp"
    NEXUS_IQ_STAGE: "build"    # develop, build, stage-release, release, operate
  script:
    - |
      java -jar /sonatype/nexus-iq-cli.jar \
        -s "$NEXUS_IQ_URL" \
        -a "${NEXUS_IQ_USER}:${NEXUS_IQ_PASSWORD}" \
        -i "$NEXUS_IQ_APP_ID" \
        -t "$NEXUS_IQ_STAGE" \
        -r iq-report.json \
        .
  artifacts:
    paths: [iq-report.json]
    when: always
    expire_in: 30 days
  allow_failure: false   # Bloquer si policy violation
```

### 8.2 Politique de sécurité Nexus (configuration)

```
Dans Nexus IQ / Sonatype Lifecycle :

Politiques recommandées :

1. "Critical CVE - Build Break"
   Condition : CVSSv3 Score >= 9.0
   Action : FAIL (bloque le build)

2. "Known Malware"
   Condition : Composant marqué malware
   Action : FAIL

3. "Copyleft License - Warn"
   Condition : Licence GPL-3.0 ou AGPL-3.0
   Action : WARN (avertissement, pas de blocage)

4. "Proprietary License - Fail"
   Condition : Licence commerciale sans approbation
   Action : FAIL

5. "Deprecated Component - Warn"
   Condition : Composant déprécié depuis > 1 an
   Action : WARN
```

---

## 📋 Récapitulatif Module 06

| Scan | Outil | Quand bloquer |
|------|-------|---------------|
| SAST | Semgrep + GitLab | HIGH/CRITICAL en MR |
| SCA | Trivy fs + GitLab DS | HIGH/CRITICAL non corrigé |
| Container | Trivy image | CRITICAL avant déploiement |
| DAST | OWASP ZAP | CRITICAL après déploiement staging |
| Secrets | Gitleaks | Toujours, dès `.pre` |
| IaC | Checkov + KICS | CRITICAL/HIGH sur Terraform/K8s |
| Licences | pip-licenses | Licences copyleft bloquantes |
| Nexus IQ | Sonatype CLI | Policy violation selon config |

🔒 **Règle d'or** : Fail fast sur les secrets (`.pre`), bloquer sur les CRITICAL avant merge, autoriser les HIGH avec justification documentée.

---

[← Module 05](./05_NEXUS_INTEGRATION.md) | [→ Module 07 - Pipeline Patterns](./07_PIPELINE_PATTERNS.md)
